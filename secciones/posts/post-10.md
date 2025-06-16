[El Protocolo de Control de Transmisión](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) (TPC `TCP`) y [el Protocolo de Datagramas de Usuario](https://en.wikipedia.org/wiki/User_Datagram_Protocol) (UDP `UDP`) son protocolos utilizados en la transmisión de información y datos en Internet. Normalmente, las conexiones TCP transmiten datos importantes, como páginas web y correos electrónicos. Por el contrario, las conexiones UDP transmiten datos en tiempo real, como la transmisión de vídeo o los juegos en línea.

TCP  garantiza la recepción de todos los datos enviados de una computadora a otra. ambas partes permanecen conectadas hasta que finaliza la llamada. Si ocurre un error al enviar datos, el receptor responde para que el emisor pueda reenviar los datos faltantes. Esto lo hace `TCP`confiable y más lento que UDP, ya que requiere más tiempo para la transmisión y la recuperación de errores.

UDP es un protocolo sin conexión. Se utiliza cuando la velocidad es más importante que la fiabilidad, como en la transmisión de vídeo o los juegos en línea. Con [ `UDP`Nombre del protocolo], no se verifica que los datos recibidos estén completos y sin errores.
es posible que se pierdan algunos datos `UDP`, pero la transmisión general es más rápida.

## Paquete IP

Un paquete [de Protocolo de Internet](https://en.wikipedia.org/wiki/Internet_Protocol) ( `IP`) es el área de datos que utiliza la capa de red del modelo [de Interconexión de Sistemas Abiertos](https://en.wikipedia.org/wiki/OSI_model) ( `OSI`)  Consta de un encabezado y la carga útil (los datos de la carga útil propiamente dichos).

#### Encabezado IP

El encabezado de un paquete IP contiene varios campos que tienen información importante.

|**Campo**|**Descripción**|
|---|---|
|`Version`|Indica qué versión del protocolo IP se está utilizando|
|`Internet Header Length`|Indica el tamaño del encabezado en palabras de 32 bits.|
|`Class of Service`|Significa lo importante que es la transmisión de los datos|
|`Total length`|Especifica la longitud total del paquete en bytes|
|`Identification (ID)`|Se utiliza para identificar fragmentos del paquete cuando se fragmenta en partes más pequeñas.|
|`Flags`|Se utiliza para indicar fragmentación|
|`Fragment Offset`|Indica dónde se coloca el fragmento actual en el paquete.|
|`Time to Live`|Especifica cuánto tiempo puede permanecer el paquete en la red|
|`Protocol`|Especifica qué protocolo se utiliza para transmitir los datos, como TCP o UDP|
|`Checksum`|Se utiliza para detectar errores en el encabezado.|
|`Source/Destination`|Indique desde dónde se envió el paquete y a dónde se envía|
|`Options`|Contiene información opcional para el enrutamiento|
|`Padding`|Rellena el paquete hasta obtener una longitud de palabra completa|

`IP ID`campo. Se utiliza para identificar fragmentos de un paquete IP cuando se fragmenta en partes más pequeñas. Es un `16-bit`campo con un número único que va desde `0-65535`.

Si una computadora tiene varias direcciones IP, el `IP ID`campo será diferente para cada paquete enviado desde la computadora, pero muy similar. En TCPdump, el tráfico de red podría verse así:

#### Detección de redes

  Conexiones TCP/UDP

```shell-session
IP 10.129.1.100.5060 > 10.129.1.1.5060: SIP, length: 1329, id 1337
IP 10.129.1.100.5060 > 10.129.1.1.5060: SIP, length: 1329, id 1338
IP 10.129.1.100.5060 > 10.129.1.1.5060: SIP, length: 1329, id 1339
IP 10.129.2.200.5060 > 10.129.1.1.5060: SIP, length: 1329, id 1340
IP 10.129.2.200.5060 > 10.129.1.1.5060: SIP, length: 1329, id 1341
IP 10.129.2.200.5060 > 10.129.1.1.5060: SIP, length: 1329, id 1342
```

#### Campo de ruta de registro IP

El `Record-Route field`encabezado IP también registra la ruta a un dispositivo de destino. Cuando el dispositivo de destino devuelve el `ICMP Echo Reply`paquete, las direcciones IP de todos los dispositivos que lo atraviesan se listan en `Record-Route field`el encabezado IP. Esto ocurre, por ejemplo, al usar el siguiente comando:

  Conexiones TCP/UDP

```shell-session
chvrn@htb[/htb]$ ping -c 1 -R 10.129.143.158

PING 10.129.143.158 (10.129.143.158) 56(124) bytes of data.
64 bytes from 10.129.143.158: icmp_seq=1 ttl=63 time=11.7 ms
RR: 10.10.14.38
        10.129.0.1
        10.129.143.158
        10.129.143.158
        10.10.14.1
        10.10.14.38


--- 10.129.143.158 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 11.688/11.688/11.688/0.000 ms
```


La salida indica que `ping`se envió una solicitud y se recibió una respuesta del dispositivo de destino, y también se muestra `Record-Route field`en el encabezado IP del `ICMP Echo Request`paquete. El campo "Registrar ruta" contiene las direcciones IP de todos los dispositivos que pasaron por el `ICMP Echo Request`paquete en su camino hacia el dispositivo de destino. En este caso, `Record-Route field`contiene las direcciones IP:

|   |   |   |
|---|---|---|
|10.10.14.38|10.129.0.1|10.129.143.158|
|10.129.143.158|10.10.14.1|10.10.14.38|

La `traceroute`herramienta también se puede utilizar para rastrear la ruta a un destino con mayor precisión, ya que utiliza el método de tiempo de espera TCP para determinar cuándo se ha rastreado completamente la ruta.

1. Enviamos un paquete TCP SYN al dispositivo de destino con un TTL de 1 en el encabezado IP.
    
    Cuando un paquete TCP SYN con un TTL mayor que 1 llega a un enrutador, el valor del TTL se reduce en 1 y el paquete se reenvía al siguiente dispositivo. Si un paquete TCP SYN con un TTL de 1 llega a un enrutador, se descarta y el enrutador nos envía un paquete ICMP de tiempo excedido.
    
2. Recibimos el paquete ICMP Time-Exceeded y anotamos la dirección IP del enrutador que envió el paquete.
    
3. Después de eso, enviamos otro paquete TCP SYN al destino, aumentando el TTL en 1.


El proceso se repite hasta que el paquete TCP SYN llega al host de destino y recibe una `TCP SYN/ACK`respuesta `TCP RST`del dispositivo objetivo. Una vez recibida la respuesta del dispositivo de destino, sabemos que hemos rastreado la ruta al destino y finalizado el proceso de traceroute.

#### Carga útil de IP

La carga útil (también denominada `IP Data`) es la carga útil real del paquete. Contiene los datos de varios protocolos, como TCP o UDP, que se transmiten, al igual que el contenido de la carta en el sobre.

## TCP

Los paquetes TCP, también conocidos como paquetes TCP `segments`, se dividen en varias secciones llamadas encabezados y cargas útiles. Los segmentos TCP se encapsulan en el paquete IP enviado.

El encabezado contiene varios campos con información importante. El puerto de origen indica el ordenador desde el que se envió el paquete. El puerto de destino indica a qué ordenador se envía. El número de secuencia indica el orden en que se enviaron los datos. El número de confirmación se utiliza para confirmar que todos los datos se recibieron correctamente. Los indicadores de control indican si el paquete marca el final de un mensaje, si es una confirmación de recepción de datos o si contiene una solicitud de repetición. El tamaño de la ventana indica la cantidad de datos que puede recibir el receptor.

La suma de comprobación se utiliza para detectar errores en el encabezado y la carga útil. El puntero urgente alerta al receptor de que hay datos importantes en la carga útil.


La carga útil es la carga útil real del paquete y contiene los datos que se están transmitiendo, al igual que el contenido de una conversación entre dos personas.

## UDP

UDP transfiere `datagrams`(pequeños paquetes de datos) entre dos hosts. Es un protocolo sin conexión, lo que significa que no `not`necesita establecer una conexión entre el emisor y el receptor antes de enviar los datos. En su lugar, los datos se envían directamente al host de destino sin ninguna conexión previa.

Cuando `traceroute`se utiliza con UDP, se recibe un mensaje " `Destination Unreachable`y `Port Unreachable`" cuando el paquete de datagrama UDP llega al dispositivo de destino. Generalmente, los paquetes UDP se envían `traceroute`en hosts Unix.