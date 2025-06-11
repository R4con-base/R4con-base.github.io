# Análisis de Tráfico de Red con Wireshark, TShark y Termshark

## ¿Qué es Wireshark?

Wireshark es un analizador de tráfico de red gratuito y de código abierto que permite una inspección profunda de paquetes, proporcionando una visión detallada del tráfico de red. Su potencia radica en sus capacidades de análisis avanzado y su compatibilidad con cientos de protocolos diferentes.

### Características Principales

- **Inspección profunda** de paquetes para cientos de protocolos
- **Interfaces gráficas y de terminal** (TTY)
- **Multiplataforma** - funciona en la mayoría de sistemas operativos
- **Soporte de protocolos**: Ethernet, IEEE 802.11, PPP/HDLC, ATM, Bluetooth, USB, Token Ring, Frame Relay, FDDI
- **Capacidades de descifrado**: IPsec, ISAKMP, Kerberos, SNMPv3, SSL/TLS, WEP, WPA/WPA2

## Las Tres Herramientas: GUI vs Terminal

### 1. Wireshark (Interfaz Gráfica)
Ideal para análisis detallado en entornos de escritorio con todas las funcionalidades visuales disponibles.

### 2. TShark (Terminal)
Versión de línea de comandos que comparte funciones y sintaxis con Wireshark. Perfecto para:
- Servidores sin entorno gráfico
- Automatización de scripts
- Transferencia de datos a otras herramientas

### 3. Termshark (TUI - Terminal User Interface)
Combina lo mejor de ambos mundos: interfaz similar a Wireshark directamente en terminal.

![Texto alternativo](/secciones/posts/images/termshark.webp)

## Comandos Básicos de TShark

| Comando | Función |
|---------|---------|
| `-D` | Mostrar interfaces disponibles |
| `-L` | Listar medios de capa de enlace |
| `-i` | Seleccionar interfaz (-i eth0) |
| `-f` | Aplicar filtro de captura |
| `-c` | Capturar cantidad específica de paquetes |
| `-a` | Condición de parada automática |
| `-r` | Leer desde archivo |
| `-W` | Escribir a archivo (formato pcapng) |
| `-P` | Imprimir resumen mientras escribe |
| `-x` | Mostrar salida hexadecimal y ASCII |

### Ejemplos Prácticos

```bash
# Mostrar interfaces disponibles
tshark -D

# Capturar en interfaz específica
tshark -i eth0 -w captura.pcap

# Aplicar filtro por host
sudo tshark -i eth0 -f "host 172.16.146.2"

# Capturar con resumen visible
tshark -i eth0 -P -w captura.pcap
```

## Interfaz de Wireshark: Los Tres Paneles

### 1. Lista de Paquetes (Naranja)
Muestra resumen de cada paquete con campos como:
- **Número**: Orden de llegada
- **Tiempo**: Formato Unix
- **Origen/Destino**: IPs involucradas
- **Protocolo**: Tipo de protocolo usado
- **Información**: Detalles específicos del protocolo

### 2. Detalles del Paquete (Azul)
Análisis detallado por capas siguiendo el modelo OSI (en orden inverso: capa inferior arriba, superior abajo).

### 3. Bytes del Paquete (Verde)
Contenido en formato ASCII y hexadecimal. Al seleccionar campos en paneles superiores, se resalta la ubicación correspondiente.

## Filtros: La Clave del Análisis Eficiente

### Filtros de Captura (Pre-captura)
Utilizan sintaxis BPF, se aplican antes de iniciar la captura:

| Filtro | Resultado |
|--------|-----------|
| `host x.x.x.x` | Solo tráfico de/hacia host específico |
| `net x.x.x.x/24` | Tráfico de red específica |
| `port 80` | Solo tráfico del puerto 80 |
| `not port 22` | Todo excepto puerto 22 |
| `tcp and port 443` | Tráfico TCP en puerto 443 |

### Filtros de Visualización (Post-captura)
Sintaxis propia de Wireshark, más flexible:

| Filtro | Resultado |
|--------|-----------|
| `ip.addr == x.x.x.x` | Tráfico de/hacia IP (OR) |
| `ip.src == x.x.x.x` | Solo tráfico desde IP |
| `tcp.port == 80` | Puerto TCP específico |
| `dns` | Solo tráfico DNS |
| `http and ip.addr == x.x.x.x` | HTTP de IP específica |

## Mejores Prácticas

### Para Capturas Grandes
- **Dividir archivos grandes** en fragmentos más pequeños
- **Usar filtros de captura** para reducir datos en disco
- **Aplicar filtros específicos** para evitar procesamientos largos

### Diferencia Importante: Protocolo vs Puerto
- **Filtrar por puerto 80**: Muestra todo el tráfico del puerto, independientemente del protocolo
- **Filtrar por HTTP**: Busca marcadores específicos del protocolo (GET, POST, etc.)

### Instalación de Termshark
```bash
# Descargar desde GitHub
wget https://github.com/gcla/termshark/releases/latest
# Extraer y usar directamente
```

## Conclusión

El análisis de tráfico de red es fundamental para la seguridad y el diagnóstico de redes. Wireshark y sus variantes ofrecen herramientas poderosas para diferentes escenarios:

- **Wireshark GUI**: Análisis detallado e interactivo
- **TShark**: Automatización y análisis programático  
- **Termshark**: Interfaz visual en terminal

La elección de la herramienta depende del entorno y los requisitos específicos de análisis. Dominar los filtros es crucial para un análisis eficiente, especialmente al trabajar con grandes volúmenes de tráfico de red.