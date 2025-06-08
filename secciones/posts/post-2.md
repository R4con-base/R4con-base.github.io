Modelo de referencia en capas, a menudo denominado `Internet Protocol Suite`. El término `TCP/IP` representa los dos protocolos `Transmission Control Protocol` (`TCP`) e `Internet Protocol` (`IP`). `IP` se encuentra dentro de la `capa de red` (`Capa 3`) y `TCP` en la `capa de transporte` (`Capa 4`) del modelo de capas `OSI`.

|**Capa**|**Función**|
|---|---|
|`4. Application`|La capa de aplicación permite que las aplicaciones accedan a los servicios de las otras capas y define los protocolos que las aplicaciones utilizan para intercambiar datos.|
|`3. Transport`|La capa de transporte es responsable de proporcionar servicios de sesión (TCP) y datagramas (UDP) para la capa de aplicación.|
|`2. Internet`|La capa de Internet es responsable de las funciones de direccionamiento, empaquetado y enrutamiento del host.|
|`1. Link`|La capa de enlace se encarga de colocar los paquetes TCP/IP en el medio de red y recibir los paquetes correspondientes. TCP/IP está diseñado para funcionar independientemente del método de acceso a la red, el formato de trama y el medio.|

`IP` garantiza que el paquete de datos llegue a su destino, mientras que `TCP` controla la transferencia de datos y garantiza la conexión entre el flujo de datos y la aplicación.

![Diagrama TCP/IP](Pasted image 20250324215613.png)

Las tareas más importantes de `TCP/IP` son:

|**Tarea**|**Protocolo**|**Descripción**|
|---|---|---|
|`Direccionamiento Lógico`|`IP`|Debido a la gran cantidad de hosts en diferentes redes, es necesario estructurar la topología de la red y el direccionamiento lógico. En TCP/IP, IP asume el direccionamiento lógico de las redes y los nodos. Los paquetes de datos solo llegan a la red donde deben estar. Los métodos para lograrlo son `clases de red`, `subnetting` y `CIDR`.|
|`Enrutamiento`|`IP`|Para cada paquete de datos, se determina el siguiente nodo en cada nodo del trayecto del emisor al receptor. De esta manera, un paquete de datos se enruta a su receptor, incluso si el emisor desconoce su ubicación.|
|`Control de Errores y Flujo`|`TCP`|El emisor y el receptor se comunican frecuentemente mediante una conexión virtual. Por lo tanto, se envían mensajes de control continuamente para comprobar si la conexión sigue establecida.|
|`Soporte de Aplicaciones`|`TCP`|Los puertos TCP y UDP forman una abstracción de software para distinguir aplicaciones específicas y sus enlaces de comunicación.|
|`Resolución de Nombres`|`DNS`|El DNS proporciona resolución de nombres a través de nombres de dominio completamente calificados (FQDN) en direcciones IP, lo que nos permite llegar al host deseado con el nombre especificado en Internet.|