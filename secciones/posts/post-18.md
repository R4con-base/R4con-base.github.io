# Máquina "Beep" de HackTheBox

**Características:**

- Linux  
- Fácil  
- Elastix 2.2.0 Exploitation 
- Local File Inclusion (LFI) Information Leakage 
- Vtiger CRM Exploitation 
- Abusing File Upload (1st way) [RCE] 
- Shellshock Attack (2nd way) [RCE]
- External
- Apache
- PHP
- Penetration Tester Level 2
- Local File Inclusion
- Network
- SMTP
- Python
- Remote Code Execution
- VoIP

**Útil en:**

- eWPT

**IP:** `10.10.10.7`

## Escaneo inicial

```bash
nmap -sT -p- --min-rate 5000 -oA nmap/alltcp 10.10.10.7
```

**Resultados:**
```
Starting Nmap 7.70 ( https://nmap.org ) at 2018-08-09 12:44 EDT
Nmap scan report for 10.10.10.7
Host is up (0.12s latency).
Not shown: 65519 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
111/tcp   open  rpcbind
143/tcp   open  imap
443/tcp   open  https
745/tcp   open  unknown
993/tcp   open  imaps
995/tcp   open  pop3s
3306/tcp  open  mysql
4190/tcp  open  sieve
4445/tcp  open  upnotifyp
4559/tcp  open  hylafax
5038/tcp  open  unknown
10000/tcp open  snet-sensor-mgmt
```

**Servicios identificados:** SSH, SMTP, HTTP, POP3, RPCbind, IMAP y HTTPS.

## Escaneo avanzado

```bash
nmap -sC -sV -p 22,25,80,110,111,143,443,745,993,995,3306,4190,4445,4559,5038,10000 -oA nmap/scriptstcp 10.10.10.7
```

**Hallazgos clave:**
- Versión de Apache indica CentOS (probablemente CentOS 5, muy antiguo)
- Archivo `robots.txt` con una entrada deshabilitada
- Página de inicio de sesión de Elastix en HTTPS

![Página de inicio de Elastix](/secciones/posts/imagenes/beep/web1.webp)

## Explotación de vulnerabilidad LFI

Al investigar posibles exploits para Elastix, encontramos una vulnerabilidad de Local File Inclusion (LFI) que nos permite acceder a información sensible.

![Explotación LFI](/secciones/posts/imagenes/beep/web2.webp)

**Credenciales obtenidas:**
- Usuario: `admin`
- Contraseña: `jEhdIekWmdjE`

## Acceso al panel de administración

Con las credenciales obtenidas, accedemos al panel de administración:

![Panel de administración](/secciones/posts/imagenes/beep/panel1.webp)

## Obtención de shell de usuario

Mediante un exploit RCE específico para esta versión de Elastix, logramos obtener un shell como usuario `fany`, donde encontramos la primera flag.

![Escalada de privilegios](/secciones/posts/imagenes/beep/escalada.webp)

## Fuerza bruta con Hydra

Realizamos fuerza bruta contra el servicio SSH con las siguientes credenciales potenciales:

**Usuarios posibles:**
- root
- asterisk
- admin
- asteriskuser
- cyrus
- fanis
- spamfilter
- mysql

**Contraseñas posibles:**
- jEhdIekWmdjE
- amp109

```bash
hydra -L beepuser -P beeppasswd ssh://10.10.10.7
```

## Acceso como root

Finalmente, logramos acceder como root usando:
- Usuario: `root`
- Contraseña: `jEhdIekWmdjE`

**Nota:** Algunos writeups pueden tener contenido de otras páginas o pocas imágenes debido a que en algunas máquinas no se tomaron apuntes completos o capturas de pantalla. Se han recopilado las mejores explicaciones de varios writeups para este documento. Si encuentras errores ortográficos o de cualquier tipo, por favor contáctame.
