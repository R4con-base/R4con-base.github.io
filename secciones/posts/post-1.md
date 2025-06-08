El objetivo de la definición del estándar `ISO/OSI` fue crear un modelo de referencia que permitiera la comunicación de diferentes sistemas técnicos a través de diversos dispositivos y tecnologías, y que además garantizara la compatibilidad. Para lograr este objetivo, el modelo `OSI` utiliza `siete` capas diferentes, que se basan jerárquicamente entre sí. Estas capas representan las fases del establecimiento de cada conexión por las que pasan los paquetes enviados. De esta manera, el estándar se creó para visualizar cómo se estructura y establece una conexión.

|**Capa**|**Función**|
|---|---|
|`7. Application`|Entre otras cosas, esta capa controla la entrada y salida de datos y proporciona las funciones de la aplicación.|
|`6. Presentation`|La tarea de la capa de presentación es transferir la presentación de datos dependiente del sistema a una forma independiente de la aplicación.|
|`5. Session`|La capa de sesión controla la conexión lógica entre dos sistemas y evita, por ejemplo, interrupciones de la conexión u otros problemas.|
|`4. Transport`|La Capa 4 se utiliza para el control integral de los datos transferidos. La Capa de Transporte puede detectar y evitar situaciones de congestión y segmentar los flujos de datos.|
|`3. Network`|En la capa de red, las conexiones se establecen en redes de conmutación de circuitos y los paquetes de datos se reenvían en redes de conmutación de paquetes. Los datos se transmiten por toda la red, desde el emisor hasta el receptor.|
|`2. Data Link`|La función principal de la capa 2 es permitir transmisiones fiables y sin errores en el medio correspondiente. Para ello, los flujos de bits de la capa 1 se dividen en bloques o tramas.|
|`1. Physical`|Las técnicas de transmisión utilizadas son, por ejemplo, señales eléctricas, señales ópticas u ondas electromagnéticas. A través de la capa 1, la transmisión se realiza mediante líneas de transmisión cableadas o inalámbricas.|

Las capas `2-4` son capas `orientadas al transporte`, y las capas `5-7` son capas `orientadas a la aplicación`. En cada capa se realizan tareas definidas con precisión, y las interfaces con las capas vecinas se describen con precisión.

Cada capa ofrece servicios para la capa inmediatamente superior. Para que estos servicios estén disponibles, la capa utiliza los servicios de la capa inferior y realiza las tareas de su capa.

Si dos sistemas se comunican usando `OSI`, se ejecutan al menos las siete capas del modelo `dos veces`, ya que tanto el emisor como el receptor deben tener en cuenta el modelo de capas. Por lo tanto, se deben realizar numerosas tareas en cada capa para garantizar la seguridad, la fiabilidad y el rendimiento de la comunicación.

Cuando una aplicación envía un paquete a otro sistema, el sistema trabaja las capas mostradas de arriba hacia abajo, desde la capa `7` hasta la capa `1`, y el sistema receptor descomprime el paquete recibido de abajo hacia arriba, desde la capa `1` hasta la capa `7`.