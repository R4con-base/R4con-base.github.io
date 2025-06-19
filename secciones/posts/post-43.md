# Máquina "StreamIO" de HackTheBox

## Información General

- **Sistema Operativo:** Windows  
- **Dificultad:** Media 
- **Categorías:** Active Directory  

## Técnicas y Herramientas Utilizadas

- SSL Certificate Enumeration 
- SMB Enumeration 
- Kerberos User Enumeration (Kerbrute) 
- ASREPRoast Attack (Failed) 
- SQL Injection (MSSQL) 
- WAF Bypass NTLM Hash Stealing through SQL Injection (xp_dirtree) 
- Cracking Hashes Local File Inclusion (LFI) 
- LFI + Wrappers (base64 encoding) 
- Remote File Inclusion (RFI) 
- RFI + RCE via malicious PHP script 
- Information Leakage 
- Database administrator user credentials 
- Enumerating the database with sqlcmd 
- Password spraying with CrackMapExec 
- Abusing WinRM 
- EvilWinRM Abusing Firefox Stored Profile Passwords 
- Firepwd Bloodhound Enumeration 
- Playing with SharpHound.ps1 
- Puckiestyle Abusing WriteOwner privilege over a group 
- PowerView.ps1 Playing with Add-DomainObjectAcl && Add-DomainGroupMember utilities 
- Getting LAPS Passwords 
- ldapsearch [Privilege Escalation]

## Utilidad para Certificaciones

- eWPT 
- eWPTXv2 
- OSWE 
- OSCP 
- OSEP 
- Active Directory

## Reconocimiento

**IP:** 10.10.10.151 

### Escaneo de Puertos

Primero realizamos un escaneo rápido de todos los puertos:

```bash
nmap -p- --min-rate 10000 10.10.11.158
```

**Resultado:**
```
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
443/tcp   open  https
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49701/tcp open  unknown
55088/tcp open  unknown
```

### Escaneo Detallado

Realizamos un escaneo más detallado de los puertos identificados:

```bash
nmap -p 53,80,88,135,139,389,443,445,464,593,636,3268,3269,5985,9389 -sCV 10.10.11.158
```

**Resultado:**
```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-09-13 00:29:25Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
443/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
| ssl-cert: Subject: commonName=streamIO/countryName=EU
| Subject Alternative Name: DNS:streamIO.htb, DNS:watch.streamIO.htb
| Not valid before: 2022-02-22T07:03:28
|_Not valid after:  2022-03-24T07:03:28
|_ssl-date: 2022-09-13T00:32:25+00:00; +7h03m09s from scanner time.
| tls-alpn: 
|_  http/1.1
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf        .NET Message Framing
```

## Análisis Inicial

Podemos confirmar que es una computadora **Domain Controller (DC)**. También podemos ver dos subdominios: `streamio.htb` y `watch.streamio.htb` que agregamos al `/etc/hosts`.

### Enumeración SMB

Vemos que el puerto 445 está abierto, así que con CrackMapExec y SMB trataremos de ver qué servicio es:

```bash
crackmapexec smb 10.10.11.158
```

**Resultado:**
```
SMB         10.10.11.158    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:False)
```

Intentaremos ver los recursos compartidos:

```bash
crackmapexec smb 10.10.11.158 --shares
```

Esto falla, así que intentaremos con smbclient:

```bash
smbclient -L 10.10.11.158 -N
```

También falla.

### Enumeración SSL

Como tiene HTTPS podemos inspeccionar el certificado SSL:

```bash
openssl s_client -connect 10.10.11.158:443
```

Si abrimos la IP en el navegador, podemos dar clic al candado > More Information > View Certificate > DNS Names, etc., para más enumeración.

### Enumeración Web

Veremos el IIS de la siguiente manera:

```bash
curl -s -X GET "http://10.10.11.158" -I | grep "Server"
```

**Resultado:**
```
Server: Microsoft-IIS/10.0
```

Luego un whatweb:

```bash
whatweb http://10.10.11.158
```

**Resultado:**
```
http://10.10.11.158 [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/10.0], IP[10.10.11.158], Microsoft-IIS[10.0], Title[IIS Windows Server], X-Powered-By[ASP.NET]
```

El sitio está powered by ASP.NET.

Podemos pensar que si logramos subir un archivo ASPX o ASP podemos lograr ejecución remota de comandos. Así que abriremos la IP desde el navegador usando HTTP.

Vemos `https://streamio.htb` y observamos una página de películas.

### Enumeración de Usuarios con Kerberos

Como teníamos el puerto 88 y Kerberos abierto, tenemos una forma potencial para enumerar usuarios, así que usaremos Kerbrute:

```bash
./kerbrute --dc 10.10.11.158 -d streamIO.htb users.txt
```

---

## Nota

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes. Esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentra faltas de ortografía o cualquier error, puedes contactarme a mi correo.