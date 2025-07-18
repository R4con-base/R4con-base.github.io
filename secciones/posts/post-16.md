# Máquina "Backend" de HackTheBox

Caracteristicas:

- Linux  
- Media  
- API Enumeration Abusing API 
- Registering a new user 
- Abusing API 
- Logging in as the created user 
- Enumerating FastAPI EndPoints through Docs 
- Abusing FastAPI 
- We managed to change the admin password 
- Abusing FastAPI 
- We get the ability to read files from the machine (Source Analysis) 
- Creating our own privileged JWT Abusing FastAPI 
- We achieved remote command execution through the exec endpoint Information Leakage (Privilege Escalation)

Util en:

- eWPT
- OSWE
- OSCP

        IP 10.10.11.161 

- nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.161 -oG all ports

    PORT   STATE SERVICE

    22/tcp open  ssh

    80/tcp open  http

- sudo nmap -sCV  -p22,80 10.10.11.161 -oN targeted

    PORT   STATE SERVICE VERSION

    22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)

    80/tcp open  http    uvicorn
    
    | fingerprint-strings: 
    |   DNSStatusRequestTCP, DNSVersionBindReqTCP, GenericLines, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
    |     HTTP/1.1 400 Bad Request
    |   GetRequest: 
    |     HTTP/1.1 200 OK
    |     date: Mon, 11 Apr 2022 19:35:50 GMT
    |     server: uvicorn
    |     content-length: 29
    |     content-type: application/json
    |     Connection: close
    |     {"msg":"UHC API Version 1.0"}

Según la versión de OpenSSH ,se ejecuta un Ubuntu focal 20.04
visitamos la pagina web y vemos que devuelve datos JSON que muestra Firefox:

![Texto alternativo](/secciones/posts/imagenes/backend/page1.webp)

podemos definir que es la seccion de una api. continuaremos fuzeando esta vez sin hilos para no romperlo 

- wfuzz -c --hc=404 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.11.161/FUZZ

obtenemos /api y /docs aun que docs nos da 401 respuesta no autorizada. Necesito más acceso para acceder a los documentos. 
Visitando /api devuelve una lista de puntos finales, en este caso, que contiene uno, v1 y /api/v1 muestra dos puntos finales más.
/api/v1/admin redirecciona a /api/v1/admin/, que luego devuelve 401 No autorizado y extrañamente, /admin/v1/user devuelve 404 no encontrado

- sudo wfuzz -c --hc=404,422 --hh=4 -z list,GET-PUT-POST -X FUZZ  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt 10.10.11.161/api/v1/user/FUZZ

y como salida obtenemos /test luego haremos

- sudo wfuzz -c --hc=404,422 --hh=4 -z list,GET-PUT-POST -X FUZZ  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt 10.10.11.161/api/v1/user/FUZ2Z

Y en este codigo despues de la bandera -z se asignara una lista con los metodos mencionados para que haga las consultas con los 3 metodos
sacamos el codigo de estado 422 para poder ver mas salidas utiles

hacemos

- curl -s -X GET http://10.10.11.161/api/v1/user/login | jq

la bandera jq de al final es para que interprete codigo javascript 

- curl -s -X POST  http://10.10.11.161/api/v1/user/signup -d 'username=admin@htb.local&password=test' | jq

nos devuelve:

    {
    "detail": [
        {
        "loc": [
            "body"
        ],
        "msg": "value is not a valid dict",
        "type": "type_error.dict"
        }
    ]
    }

como alternativa para que nos devuelva otro estado
podemos intentar mandarlo en formato json

- curl -s -X POST  http://10.10.11.161/api/v1/user/signup -d '{"username": "admin@htb.local", "password": "test"}' | jq

asi que agregamos content type, ya que normalmente hay que indicarlo para que lo fuerze, le agregamos la linea antes de lanzar 

- curl -v -s -X POST -d '{"email": "test@htb.htb", "password": "test123"}' http://10.10.11.161/api/v1/user/signup -H "Content-Type: application/json" | jq .

La respuesta es 201 Creado y devuelve un dictado vacío. 
Si eso funcionó y tengo una cuenta, probaré una metodología similar en el /api/v1/user/login punto final. Éste va directo a que le falta nombre de usuario 
y contraseña, sin quejarse primero de la falta de un cuerpo o de que el cuerpo no es un dict vamos a probar exactamente lo que hice arriba

    curl -s -d '{"username": "test1@htb.htb", "password": "test1"}' -X POST http://10.10.11.161/api/v1/user/login -H "Content-Type: application/json" | jq .
    {
    "detail": [
        {
        "loc": [
            "body",
            "username"
        ],
        "msg": "field required",
        "type": "value_error.missing"
        },
        {
        "loc": [
            "body",
            "password"
        ],
        "msg": "field required",
        "type": "value_error.missing"
        }
    ]
    }

Dados los diferentes mensajes de error, quizás esté configurado para utilizar un cuerpo HTTP estándar. Eso funciono.

curl -s -d 'username=0xdf@htb.htb&password=0xdf0xdf' http://10.10.11.161/api/v1/user/login | jq .
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjUwNDEzMTMwLCJpYXQiOjE2NDk3MjE5MzAsInN1YiI6IjMiLCJpc19zdXBlcnVzZXIiOmZhbHNlLCJndWlkIjoiNTg4MjI2N2YtYTA2My00MzZiLWE5NmUtNTIyZjk2ZjNkOTY5In0.Y6OfDmvwWEHsZhtwxZqN2X6-q29B5C2dYDRIhOik_jo",
  "token_type": "bearer"
}

y nos devolvio un json web token que llevaremos a la pagina jwt.io

![Texto alternativo](/secciones/posts/imagenes/backend/jwt1.png)

vemos (is_superuser: falso) la forma estándar de utilizar un token al portador es incluir un encabezado Authorization, con la cadena bearer [token]. 
Eso funciona aquí ya que ahora puedo acceder. /docs: 

- curl http://10.10.11.161/docs -H "Authorization: bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjUwNDEzMTMwLCJpYXQiOjE2NDk3MjE5MzAsInN1YiI6IjMiLCJpc19zdXBlcnVzZXIiOmZhbHNlLCJndWlkIjoiNTg4MjI2N2YtYTA2My00MzZiLWE5NmUtNTIyZjk2ZjNkOTY5In0.Y6OfDmvwWEHsZhtwxZqN2X6-q29B5C2dYDRIhOik_jo"

        <!DOCTYPE html>
        <html>
        <head>
        <link type="text/css" rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swagger-ui-dist@3/swagger-ui.css">
        <link rel="shortcut icon" href="https://fastapi.tiangolo.com/img/favicon.png">
        <title>docs</title>
        </head>
        <body>
        <div id="swagger-ui">
        </div>
        < script src="https://cdn.jsdelivr.net/npm/swagger-ui-dist@3/swagger-ui-bundle.js">< / script>
        <!--  SwaggerUIBundle is now available on the page -->
        < script >
        const ui = SwaggerUIBundle({
            url: '/openapi.json',
        "dom_id": "#swagger-ui",
    "layout": "BaseLayout",
    "deepLinking": true,
    "showExtensions": true,
    "showCommonExtensions": true,

        presets: [
            SwaggerUIBundle.presets.apis,
            SwaggerUIBundle.SwaggerUIStandalonePreset
            ],
        })
        < /script >
        </body>
        </html>

Agrego encabezados de modificación simples a Firefox y agrego el token de portador de esta manera.

![Texto alternativo](/secciones/posts/imagenes/backend/modifi.webp)

y ahora vemos.

![Texto alternativo](/secciones/posts/imagenes/backend/webnew.webp)

y SecretFlagEndpoint nos da la primera bandera. Al continuar mirando veo la seccion updatepass. 

![Texto alternativo](/secciones/posts/imagenes/backend/update1.webp)

podemos establecer cualquier contraseña, asi que tenemos una guía de administración. 

![Texto alternativo](/secciones/posts/imagenes/backend/guide1.webp)

para iniciar sesión 

![Texto alternativo](/secciones/posts/imagenes/backend/guide2.webp)

ahora actualiza el token en el complemento de encabezado de Firefox.
Como todo es un archivo en Linux, podemos ir a /proc/self/environ para obtener información sobre el proceso que estamos ejecutando. 

![Texto alternativo](/secciones/posts/imagenes/backend/environ1.webp)

podemos ver que tenemos un APP_MODULE en app.main. Así que revisemos /app/main.py ya que se trata de una aplicación python.
Esto me da un error del servidor. Intentemos recrear toda la ruta con la información de contraseña que obtuvimos. 

![Texto alternativo](/secciones/posts/imagenes/backend/environ2.webp)

en la salida podemos ver un archivo llamado config en /app/core lo buscamos a traves de la aplicacion y vemos

![Texto alternativo](/secciones/posts/imagenes/backend/jwtsecret.webp)

y encontramos la clave de depuracion me dirijo a jwt.io generamos el token como admin ponemos la contraseña y obtuvimos ejecucion de codigo 

![Texto alternativo](/secciones/posts/imagenes/backend/debugtrue1.webp)

nc no funciona asi que enviaremos una revshell con python

![Texto alternativo](/secciones/posts/imagenes/backend/rev1.webp)

Y ahora a escalar privilegios.

Enumeración

El uhc El directorio de inicio está configurado principalmente para la aplicación, pero también hay un archivo auth.log le hacemos un cat y Mirándolo, 
hay un montón de inicios de sesión de administrador, junto con un único error.

    htb@Backend:~/uhc$ cat auth.log 
    04/11/2022, 21:11:25 - Login Success for admin@htb.local
    04/11/2022, 21:14:45 - Login Success for admin@htb.local
    04/11/2022, 21:28:05 - Login Success for admin@htb.local
    04/11/2022, 21:31:25 - Login Success for admin@htb.local
    04/11/2022, 21:36:25 - Login Success for admin@htb.local
    04/11/2022, 21:39:45 - Login Success for admin@htb.local
    04/11/2022, 21:53:05 - Login Success for admin@htb.local
    04/11/2022, 22:01:25 - Login Success for admin@htb.local
    04/11/2022, 22:03:05 - Login Success for admin@htb.local
    04/11/2022, 22:09:45 - Login Success for admin@htb.local
    04/11/2022, 22:18:05 - Login Failure for Tr0ub4dor&3
    04/11/2022, 22:19:40 - Login Success for admin@htb.local
    04/11/2022, 22:19:45 - Login Success for admin@htb.local
    04/11/2022, 22:20:05 - Login Success for admin@htb.local
    04/11/2022, 22:21:25 - Login Success for admin@htb.local
    04/11/2022, 22:26:25 - Login Success for admin@htb.local
    04/11/2022, 22:33:05 - Login Success for admin@htb.local
    04/11/2022, 22:55:27 - Login Success for root@ippsec.rocks
    04/11/2022, 22:58:12 - Login Success for root@ippsec.rocks
    04/11/2022, 22:59:30 - Login Success for admin@htb.local
    04/12/2022, 00:04:02 - Login Failure for test@htb.htb
    04/12/2022, 00:04:32 - Login Failure for test
    04/12/2022, 00:04:45 - Login Failure for test@htb.htb
    04/12/2022, 00:05:30 - Login Success for test@htb.htb
    04/12/2022, 01:15:16 - Login Success for test@htb.htb
    04/12/2022, 01:51:06 - Login Success for admin@htb.local
    04/12/2022, 01:52:23 - Login Success for admin@htb.local

vemos "Tr0ub4dor&3" que parece ser una contraseña, asi que lanzamos

- su -

Somos root buscamos la flag y terminada 

------------------------------------------

**Algunos de los writeups en esta página, pueden tener contenido de otras páginas o tener muy pocas imágenes, esto 
debido a que en algunas de las máquinas que realice, no tome los apuntes o no tome capturas de pantalla, así que he decidido buscar varios writeups**


