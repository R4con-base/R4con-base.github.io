# Máquina "Shared" de HackTheBox

## Características:

- Linux
- Media
- Web Enumeration
- SQL Injection (SQLI) in Cookie
- Cracking Hashes
- Abusing Cron Jobs
- iPython Arbitrary Code Execution - CVE-2022-21699 (User poisoning)
- Information Leakage
- Abusing Redis - Sandbox Escape (CVE-2022-0543) [Privilege Escalation]

## Útil en:

- eWPT
- OSCP

---

**IP:** 10.10.11.172

## Reconocimiento

### Escaneo de puertos inicial

```bash
nmap -p- --min-rate 10000 10.10.11.172
```

**Resultado:**
```
PORT    STATE SERVICE  VERSION   
22/tcp  open  ssh      OpenSSH
80/tcp  open  http     nginx    
443/tcp open  ssl/http nginx 
```

### Escaneo detallado de servicios

```bash
nmap -p 22,80,443 -sCV 10.10.11.172
```

```
PORT    STATE SERVICE  VERSION                                                                                                                
22/tcp  open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)                                                                          
80/tcp  open  http     nginx 1.18.0                                                                                                           
| http-methods:                                                                                                                               
|_  Supported Methods: GET HEAD POST OPTIONS                                                                                                  
|_http-server-header: nginx/1.18.0                                                                                                            
|_http-title: Did not follow redirect to http://shared.htb                                                                                    
443/tcp open  ssl/http nginx 1.18.0                                    
| http-methods:                    
|_  Supported Methods: GET HEAD POST OPTIONS                           
|_http-server-header: nginx/1.18.0                                     
|_http-title: Did not follow redirect to https://shared.htb                                                                                   
| ssl-cert: Subject: commonName=*.shared.htb/organizationName=HTB/stateOrProvinceName=None/countryName=US                                     
| Issuer: commonName=*.shared.htb/organizationName=HTB/stateOrProvinceName=None/countryName=US                                                
| Public Key type: rsa             
| Public Key bits: 2048            
| Signature Algorithm: sha256WithRSAEncryption                         
| Not valid before: 2022-03-20T13:37:14                                
| Not valid after:  2042-03-15T13:37:14                                
| MD5:   fb0b 4ab4 9ee7 d95d ae43 239a fca4 c59e                       
|_SHA-1: 6ccd a103 5d29 a441 0aa2 0e32 79c4 83e1 750a d0a0                                                                                    
| tls-alpn:
```

Agregamos `shared.htb` al archivo `/etc/hosts`.

## Análisis Web

![Sitio web principal](/secciones/posts/imagenes/shared/web1.png)

El sitio muestra una tienda de compras. Se pueden agregar artículos al carrito y se pueden ver parámetros GET que podrían ser vulnerables a SQL injection. Al intentar explotarlos, no parecen ser vulnerables. Al hacer clic en "proceder al pago", redirecciona a `checkout.shared.htb`, que agregaremos también al `/etc/hosts`.

### Fuzzing de directorios

#### Fuzzing en checkout.shared.htb

```bash
dirb https://checkout.shared.htb/
```

```
GENERATED WORDS: 4612                                                          

---- Scanning URL: https://checkout.shared.htb/ ----
+ https://checkout.shared.htb/akeeba.backend.log (CODE:403|SIZE:555)                
==> DIRECTORY: https://checkout.shared.htb/assets/                                  
==> DIRECTORY: https://checkout.shared.htb/config/                                  
==> DIRECTORY: https://checkout.shared.htb/css/                                     
+ https://checkout.shared.htb/development.log (CODE:403|SIZE:555)                   
+ https://checkout.shared.htb/production.log (CODE:403|SIZE:555)                    
+ https://checkout.shared.htb/spamlog.log (CODE:403|SIZE:555)                       
                                                                                    
---- Entering directory: https://checkout.shared.htb/assets/ ----
+ https://checkout.shared.htb/assets/akeeba.backend.log (CODE:403|SIZE:555)         
+ https://checkout.shared.htb/assets/development.log (CODE:403|SIZE:555)            
+ https://checkout.shared.htb/assets/favicon.ico (CODE:200|SIZE:23462)              
+ https://checkout.shared.htb/assets/production.log (CODE:403|SIZE:555)             
+ https://checkout.shared.htb/assets/spamlog.log (CODE:403|SIZE:555)                
                                                                                    
---- Entering directory: https://checkout.shared.htb/config/ ----
(!) WARNING: All responses for this directory seem to be CODE = 403.                
    (Use mode '-w' if you want to scan it anyway)
                                                                                    
---- Entering directory: https://checkout.shared.htb/css/ ----
+ https://checkout.shared.htb/css/akeeba.backend.log (CODE:403|SIZE:555)            
+ https://checkout.shared.htb/css/development.log (CODE:403|SIZE:555)               
+ https://checkout.shared.htb/css/production.log (CODE:403|SIZE:555)                
+ https://checkout.shared.htb/css/spamlog.log (CODE:403|SIZE:555)                   
                                                                                    
-----------------
END_TIME: Sun Sep 18 21:35:13 2022
DOWNLOADED: 13936 - FOUND: 13
```

#### Fuzzing en shared.htb

```bash
dirb https://shared.htb/
```

```
GENERATED WORDS: 4612                                                          

---- Scanning URL: https://shared.htb/ ----
+ https://shared.htb/akeeba.backend.log (CODE:403|SIZE:555)                         
==> DIRECTORY: https://shared.htb/app/                                              
==> DIRECTORY: https://shared.htb/bin/                                              
==> DIRECTORY: https://shared.htb/cache/                                            
==> DIRECTORY: https://shared.htb/classes/                                          
==> DIRECTORY: https://shared.htb/config/                                           
==> DIRECTORY: https://shared.htb/controllers/                                      
+ https://shared.htb/development.log (CODE:403|SIZE:555)                            
==> DIRECTORY: https://shared.htb/docs/                                             
==> DIRECTORY: https://shared.htb/download/                                         
==> DIRECTORY: https://shared.htb/img/                                              
+ https://shared.htb/index.php (CODE:200|SIZE:56215)                                
==> DIRECTORY: https://shared.htb/js/                                               
==> DIRECTORY: https://shared.htb/mails/                                            
+ https://shared.htb/Makefile (CODE:200|SIZE:88)                                    
==> DIRECTORY: https://shared.htb/modules/                                          
==> DIRECTORY: https://shared.htb/pdf/                                              
+ https://shared.htb/production.log (CODE:403|SIZE:555)                             
+ https://shared.htb/robots.txt (CODE:200|SIZE:2748)                                
+ https://shared.htb/spamlog.log (CODE:403|SIZE:555)                                
==> DIRECTORY: https://shared.htb/src/                                              
==> DIRECTORY: https://shared.htb/themes/                                           
==> DIRECTORY: https://shared.htb/tools/                                            
==> DIRECTORY: https://shared.htb/translations/                                     
==> DIRECTORY: https://shared.htb/upload/                                           
==> DIRECTORY: https://shared.htb/var/                                              
==> DIRECTORY: https://shared.htb/vendor/                                           
==> DIRECTORY: https://shared.htb/webservice/                                       

[... resto de directorios encontrados ...]
```

Los resultados del fuzzing no parecen conducir a nada útil. Revisamos con Wappalyzer:

![Análisis con Wappalyzer](/secciones/posts/imagenes/shared/wpa.png)

## Búsqueda de vulnerabilidades en PrestaShop

Buscando exploits en PrestaShop, se encuentran muchos artículos y vulnerabilidades:

- https://www.cvedetails.com/vulnerability-list/vendor_id-8950/Prestashop.html
- https://www.exploit-db.com/exploits/45964
- https://github.com/farisv/PrestaShop-CVE-2018-19126

Algunos de ellos requieren credenciales del portal. Entre ellos, llama la atención una vulnerabilidad en uno de los módulos cargados en el comercio electrónico, basada en SQL injection:

https://www.exploitalert.com/view-details.html?id=36763

Parecería ser una inyección ciega, que explota cualquier comportamiento (como el manejo de una excepción o el retraso en la respuesta de la consulta) para comprender la estructura de la base de datos o los datos que contiene.

### Intento de explotación

```bash
https://shared.htb/index.php?fc=module&module=productcomments&controller=CommentGrade&id_products[]=(select*from(select(sleep(2)))a)
```

**Respuesta:**
```json
{"products":[{"id_product":0,"comments_nb":null,"average_grade":null}]}
```

Todo esto sin resultados favorables.

## Análisis de comunicación entre portales

Al colocar algunos artículos en el carrito del primer portal y proceder al pago, se observa que los artículos están reportados en la factura emitida por el segundo portal. Por tanto, debe existir algún tipo de comunicación entre los portales. Al analizar el segundo portal, se descubre cómo se pasa la información del carrito.