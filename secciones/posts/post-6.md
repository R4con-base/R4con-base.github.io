# ARP Spoofing y ARP Cache Poisoning

## Definición

`ARP spoofing`, también conocido como `ARP cache poisoning` o `ARP poison routing`, es un ataque donde se envían mensajes ARP falsificados a través de una red local (LAN). El atacante inunda las tablas ARP de todos los participantes de la red usando respuestas ARP falsas. El objetivo es asociar nuestra dirección MAC con la dirección IP de un dispositivo legítimo en la red, permitiendo interceptar el tráfico dirigido a dicho dispositivo.

## Herramientas Principales

### Ettercap
[Ettercap](https://github.com/Ettercap/ettercap) es la herramienta más utilizada para este tipo de ataques. El flujo general de un ataque ARP spoofing con Ettercap es unirse a una red que quieres atacar, localizar hosts en la red, asignar objetivos a un archivo de "targets", y luego ejecutar el ataque en los objetivos.

### Cain & Abel
[Cain & Abel](https://github.com/xchwarze/Cain) es otra herramienta conocida para Windows que permite realizar ataques de ARP poisoning.

### Otras herramientas
El atacante usa herramientas como Arpspoof o Driftnet para enviar respuestas ARP falsificadas.

## Ejemplo de Ataque

```
Tiempo    Origen        -> Destino       Protocolo  Descripción
1  0.000000 10.129.12.100 -> 10.129.12.101 ARP 60    10.129.12.100 is at AA:AA:AA:AA:AA:AA
2  0.000015 10.129.12.100 -> 10.129.12.255 ARP 60    Who has 10.129.12.101? Tell 10.129.12.100
3  0.000030 10.129.12.101 -> 10.129.12.100 ARP 60    10.129.12.101 is at BB:BB:BB:BB:BB:BB
4  0.000045 10.129.12.100 -> 10.129.12.101 ARP 60    10.129.12.100 is at AA:AA:AA:AA:AA:AA
```

La primera y cuarta línea muestran al atacante (`10.129.12.100`) enviando mensajes ARP falsificados al objetivo, asociando su dirección MAC con la dirección IP del objetivo (`10.129.12.101`). La segunda y tercera línea muestran al objetivo enviando una solicitud ARP y respondiendo a nuestra dirección MAC. Esto indica que hemos envenenado la caché ARP del objetivo y todo el tráfico dirigido a él se enviará ahora a nuestra dirección MAC.

## Funcionamiento del Ataque

Si un participante de la red quiere establecer una conexión al servidor, por ejemplo, primero busca en su tabla ARP para ver si una dirección MAC está almacenada para la dirección IP del servidor. Dado que cada dirección IP en la tabla ARP del cliente apunta a la dirección MAC del atacante, la conexión no se establece al servidor sino al atacante.

Las máquinas víctima entonces reenviarán incorrectamente el tráfico de red al atacante. Herramientas como Ettercap permiten al atacante actuar como un proxy, viendo o modificando información antes de enviar el tráfico a su destino previsto.

## Propósitos del Ataque

Podemos usar el envenenamiento de ARP para realizar diversas actividades:
- Robar información confidencial
- Redirigir el tráfico 
- Lanzar ataques MITM (Man-in-the-Middle)
- Los desarrolladores también usan ARP spoofing para debuggear tráfico de red insertando intencionalmente un intermediario entre dos hosts

## Métodos de Protección

### Entradas ARP Estáticas
La forma más simple de certificación es el uso de entradas estáticas de solo lectura para servicios críticos en el caché ARP de un host. Estas ARP se añaden al caché y se retienen de forma permanente, sirviendo como mapeos permanentes entre direcciones MAC e IP.

Esta solución involucra mucha sobrecarga administrativa y solo se recomienda para redes más pequeñas. Implica añadir una entrada ARP para cada máquina en una red en cada computadora individual.

### Dynamic ARP Inspection (DAI)
La mayoría de switches Ethernet administrados tienen características diseñadas para mitigar ataques de ARP Poisoning. Típicamente conocidas como Dynamic ARP Inspection (DAI), estas características evalúan la validez de cada mensaje ARP y descartan paquetes que parecen sospechosos o maliciosos.

### Monitoreo de Tablas ARP
Mantener esta tabla bajo vigilancia verifica cualquier actividad sospechosa cuando los elementos en la tabla son alterados o cambiados. Al monitorear tablas de caché ARP, las entradas estáticas requieren la mayor atención, ya que estas son ingresadas manualmente y permanecen hasta que son removidas manualmente.

### Protocolos de Red Seguros
Para protegerse contra la suplantación de ARP, es importante utilizar:
- IPSec
- SSL/TLS
- VPNs para conexiones cifradas
- Firewalls
- Sistemas de detección de intrusos

## Detección del Ataque

Para descubrir ARP spoofing en una red grande se pueden usar herramientas especializadas y monitoreo continuo del protocolo ARP.

