# Máquina "Ready" de HackTheBox

## Características
- Linux
- Dificultad media
- Enumeración de Docker 
- Eludir la protección SSRF
- Inyección CRLF
- Escape del contenedor privilegiado de Docker 
- GitLab vulnerabilidad v11.4.7 Community Edition

**IP:** 10.10.10.214

## Reconocimiento

### Escaneo nmap

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.149.69
```

Encontramos dos puertos que ejecutaban servicios como SSH (22) y NGINX (5080). La sesión de escaneo sugiere también una lista corta de carpetas para investigar con el navegador y una redirección a una página de inicio de sesión. La página de inicio de sesión y el título de la página es GITLAB.

Verificamos el archivo robots.txt:

```
http://ready.htb:5080/robots.txt 
```

Al visitar la página de inicio, nos lleva a la página de inicio de sesión. Nos registramos e iniciamos sesión.

![Ready New Project](/secciones/posts/imagenes/Ready/ready1.png)

## Enumeración de la aplicación

Una vez que iniciamos sesión, consultamos la sección de ayuda para obtener información sobre la versión de esta aplicación GitLab. La **versión 11.4.7** se lanzó en noviembre de 2018, es una versión de más de 2 años. Al buscar un poco en Google, podemos encontrar vulnerabilidades relacionadas con esa versión y también algunos POCs.

## Detalles de vulnerabilidad

Básicamente existen dos vulnerabilidades diferentes en la versión 11.4.7 de GitLab:
- **SSRF CVE-2018-19571** 
- **Inyección CRLF CVE-2018-19585**

Si verificamos las confirmaciones de GitLab de 11.4.8, veremos un par de correcciones de seguridad para la versión anterior.

## Explotación

Podemos explotar estas vulnerabilidades encadenando SSRF y CRLF. Para más detalles:
https://liveoverflow.com/gitlab-11-4-7-remote-code-execution-real-world-ctf-2018/

Cambiando una pequeña parte de la carga útil podemos obtener la shell inversa.

### Pasos para la explotación

1. Iniciamos sesión en GitLab
2. Hacemos clic en **Nuevo proyecto**

![Ready New Project](/secciones/posts/imagenes/Ready/readynewproject.png)

3. Hacemos clic en **Crear proyecto** mientras capturamos la solicitud en Burpsuite. Debería verse así:

![Burp Request](/secciones/posts/imagenes/Ready/burp1.png)

4. Seleccionamos el `authenticity_token` y decodificamos la URL presionando `CTRL+SHIFT+U`
5. Copiamos el token y la cookie. El exploit code snippet debería verse así:

![Code Snippet](/secciones/posts/imagenes/Ready/codesnipet.png)

6. En una ventana lanzamos netcat en modo escucha con el puerto 1234 y ejecutamos el exploit:

![Exploit Execution](/secciones/posts/imagenes/Ready/nvyexploit.png)

### Acceso inicial

Accedimos como usuario `dude`.

Configuramos la shell:

```bash
script /dev/null -c bash
# Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
uname -a
```

Capturamos la bandera del usuario:

```bash
cat /home/dude/user.txt
```

## Escalada de privilegios 

Encontramos un archivo llamado `gitlab.rb`, localizado en la ruta `/opt/backup/`. Al inspeccionar el contenido de dicho archivo, encontramos algo útil: la contraseña de usuario SMTP. Esta contraseña nos ayudará más tarde.

```bash
cat /opt/backup/gitlab.rb | grep smtp
```

![Save Password](/secciones/posts/imagenes/Ready/savepasswd.png)

### Acceso a root

Cambiamos al usuario root:

```bash
su root
# Password: wW59U!ZKMbG9+*#h
```

```bash
whoami && id
```

Buscamos la bandera sin éxito. Después de más enumeración, debemos explotar un contenedor Docker privilegiado.

## Escape del contenedor Docker

Aquí puedes ver más sobre escape de contenedores Docker privilegiados:
https://medium.com/better-programming/escaping-docker-privileged-containers-a7ae7d17f5a1

Usamos este script para escapar del contenedor:

![Root Exploit](/secciones/posts/imagenes/Ready/exploitroot.png)

### En la máquina atacante:

```bash
cat root_flag.sh
sudo python3 -m http.server 80
```

### En la víctima:

```bash
cd /dev/shm
# Descargamos y ejecutamos el script de escape
```

Y tenemos la flag de root.

---

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También si encuentra faltas de ortografía o cualquier error, puedes contactarme a mi correo.