# Ataques de Desautenticación y Disociación WiFi

## Tipos de Ataques

Existen dos tipos distintos de ataques:

### Desautenticación (Deauthentication)
Es un tipo de ataque de denegación de servicio que targets la comunicación entre un usuario y un punto de acceso Wi-Fi.

### Disociación (Disassociation)
Involucra enviar marcos de disociación falsificados a un punto de acceso inalámbrico o dispositivo cliente, causando que el dispositivo se desconecte de la red.

## Funcionamiento Técnico

Cuando un cliente necesita desconectarse del punto de acceso inalámbrico, envía marcos especiales conocidos como marcos de desautenticación o disociación. Debido a que no están encriptados, estos marcos no requieren un usuario autenticado. Por esto, un atacante puede crear estos marcos y enviarlos al punto de acceso.

El ataque de desautenticación es uno de los más comunes y fácilmente ejecutados. Este ataque está diseñado para desconectar un cliente de un Punto de Acceso Inalámbrico (WAP) y, si se continúa repetidamente, puede hacer una red inutilizable para uno o todos los dispositivos cliente.

## Herramientas Comunes

Las herramientas más utilizadas incluyen:

### Aircrack-ng Suite
Para desautenticaciones dirigidas, aireplay-ng envía un total de 128 paquetes por cada desautenticación que especifiques. 64 paquetes se envían al AP mismo y 64 paquetes se envían al cliente.

**Comando básico:**
```bash
aireplay-ng --deauth 0 -a [ROUTER_BSSID] -c [TARGET_MAC_ADDRESS] wlan1
```

Donde `--deauth 0` significa que enviarás paquetes de desautenticación infinitos. Para de enviar paquetes cuando detienes la ejecución del programa.

### Otras Herramientas
- **MDK3**: Framework para ataques wireless
- **Void11**: Herramienta específica para ataques de desautenticación
- **Scapy**: Librería Python para manipulación de paquetes
- **Zulu**: Suite de herramientas wireless
- **WiFi Pineapple**: Dispositivo hardware para ataques wireless

## Impacto del Ataque

Este ataque envía paquetes de disociación a uno o más clientes que están actualmente asociados con un punto de acceso particular. Los ataques de desautenticación permiten desconectar cualquier dispositivo de cualquier red, incluso si no estás conectado a la red.

