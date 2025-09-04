# Cómo Hacer Baterías Portátiles con Celdas de Litio

## Tipos de Baterías

Se pueden reutilizar baterías desechadas de **polímero de litio** o de **iones de litio**.
*   **Polímero de Litio:** Son aquellas baterías cuadradas y flexibles que se encuentran en notebooks, celulares, etc.
*   **Iones de Litio:** Suelen ser más potentes, con entregas de voltaje más eficientes y mayor durabilidad. Las de notebooks suelen fallar por el chip de carga o una celda dañada.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/baterias-litio.jpg)

## Entendiendo la Capacidad (mAh) y el Voltaje (V)

Todas las baterías marcan su **amperaje** (capacidad) y **voltaje**.
*   El amperaje, medido en **miliamperios-hora (mAh)**, indica la cantidad de energía que puede almacenar y, por lo tanto, cuánto rendirá.
    *   `1 A = 1000 mAh`
*   El voltaje, medido en **voltios (V)**, es la fuerza con la que se entrega esa energía.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/bateria-polimero-mAh.png)

**Ejemplo:** Si una batería marca **3000 mAh** (3 Ah) y el dispositivo que alimenta consume **0.5 A**, la batería rendirá aproximadamente **6 horas** de uso continuo (`3 Ah / 0.5 A = 6 h`). Si el consumo fuera de **1 A**, el tiempo se reduciría a **3 horas**.

Para aumentar la autonomía, se pueden conectar varias baterías.

## Conexión de Baterías: Serie y Paralelo

### 1. Conexión en Paralelo (Aumentar la Capacidad / mAh)

Se conectan todos los polos positivos entre sí y todos los negativos entre sí.
*   **Resultado:** El **voltaje final se mantiene** (ej: 3.7V), pero la **capacidad (mAh) se suma**.
*   **Ventaja:** Mayor tiempo de uso.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/suma-amperaje.png)

### 2. Conexión en Serie (Aumentar el Voltaje / V)

Se conecta el polo positivo de una batería al negativo de la siguiente.
*   **Resultado:** El **voltaje se suma** (ej: 3.7V + 3.7V = 7.4V), pero la **capacidad (mAh) se mantiene**.
*   **Ventaja:** Se obtiene un voltaje mayor necesario para algunos dispositivos.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/serie.png)

### 3. Combinación Serie-Parallel (Aumentar Voltaje y Capacidad)

Se pueden crear "bancos" de baterías. Por ejemplo, con dos pares de baterías en paralelo (cada par de 3.7V y 3Ah) conectados en serie, se obtendría un total de **7.4V y 3Ah**.

A estos grupos se les puede llamar **celdas**. La cantidad de baterías en paralelo define la capacidad de la celda, y la cantidad de celdas en serie define el voltaje total.

## Carga y Protección de las Baterías

### Módulos de Carga

Para cargar celdas de **3.7V** (1S), se puede usar un módulo **TP4056**. Es compatible con cualquier batería de litio recargable de 3.7V nominales.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/tp4056.png)

*   **Precaución:** No hay problema en cargar baterías en **paralelo** con un TP4056, ya que se comportan como una sola celda grande. Sin embargo, **NO** se deben cargar baterías en **serie** con un TP4056, ya que requiere un cargador específico.

Para cargar baterías en serie (ej: 7.4V para 2 celdas, 11.1V para 3 celdas), existen módulos de carga especializados para múltiples celdas (2S, 3S, 4S, etc.).

![Delivery Web](/secciones/posts/imagenes/Baterias-1/modulosdecargaceldas.png)

![Delivery Web](/secciones/posts/imagenes/Baterias-1/carga-2-seldas.png)

### Módulos de Balanceo y Protección (BMS)

Todas las baterías de litio tienen una vida útil limitada y pueden ser peligrosas si se dañan, se golpean o se sobrecargan (las de polímero suelen inflarse e incluso reventar).

Es **crucial** usar un **módulo de protección y balanceo** (BMS o Protection Circuit Module - PCM). Los cargadores como el TP4056 incluyen esta protección para una celda, cortando la carga cuando está completa.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/balanceador-de-carga.png)

Muchas baterías de polímero de litio ya lo traen integrado.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/balanceador-polimero.png)

#### Función del Balanceador:
*   Se conecta directamente a los terminales de la batería o banco de baterías.
*   Detecta el voltaje de cada celda en serie.
*   **Balancea la carga:** Durante la carga, asegura que todas las celdas lleguen al mismo voltaje máximo. Sin él, las celdas más nuevas o en mejor estado se sobrecargan mientras las más viejas o débiles no se cargan por completo, estropeándose rápidamente.
*   **Protección:** Corta la salida de voltaje si el voltaje de la batería cede por debajo del mínimo seguro (aprox. **3.3V**) o supera el máximo (**4.2V**).

### Voltajes de Operación de una Celda de Li-ion
*   **Máximo de carga:** 4.2V
*   **Mínimo de descarga segura:** 3.3V
*   **Voltaje nominal:** 3.7V

## Cómo Identificar Baterías en Buen Estado

Usa un multímetro en la escala de **corriente continua (DC)** en **20V**.
*   Mide el voltaje en los polos de la batería.
*   **Buena condición:** Debe mostrar un voltaje residual, idealmente por encima de ~3.0V. Si marca **menos de 0.1V** o el multímetro muestra una **resistencia infinita** (como si no hubiera circuito), la batería está en mal estado y no es utilizable.

## Instalación Práctica de un Módulo de Protección (BMS)

El balanceador tiene terminales marcados con **+** y **-**. El **+** va al polo positivo de la batería y el **-** al negativo.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/balanceador-colocado.png)

**¡Importante!** La forma ideal de conectarlo a las celdas es soldando las láminas de níquel del BMS a los terminales de la batería usando un **soldador de puntos**. Soldar directamente con cautín puede dañar las celdas.

**Ejemplo de conexión incorrecta:** Si una batería está por debajo del voltaje mínimo (ej: 0.5V), el BMS cortará la salida. Al medir en sus terminales de salida no habrá voltaje, pero al medir directamente en los polos de la batería sí se verán los 0.5V. Esto confirma que el BMS está funcionando.

## Ejemplo de Montaje: TP4056 en una Celda (3.7V)

1.  Se sueldan los terminales **B-** y **B+** del TP4056 directamente a los polos negativo y positivo de la batería.
    ![Delivery Web](/secciones/posts/imagenes/Baterias-1/flux-v-.png)
    ![Delivery Web](/secciones/posts/imagenes/Baterias-1/flux-v+.png)

2.  Para las terminales de salida (**OUT+** y **OUT-**), se usan cables: **rojo para positivo** y **negro para negativo**.
    ![Delivery Web](/secciones/posts/imagenes/Baterias-1/negativo-negro-positivo-rojo.png)

3.  Estos cables de salida se conectan al balanceador/protector si se está usando uno externo.
    ![Delivery Web](/secciones/posts/imagenes/Baterias-1/balanceador-terminales.png)

4.  Con un tester, se puede verificar que el sistema funciona correctamente y que la batería acepta la carga.
    ![Delivery Web](/secciones/posts/imagenes/Baterias-1/tester-a-bateria.png)

## Esquemas de Carga para Múltiples Baterías

### Carga en Paralelo

Para cargar varias baterías en paralelo, se conectan todas como una sola celda grande y se usa un único módulo de carga (ej: TP4056) y un único BMS. Los balanceadores se encargarán de gestionar la carga de manera uniforme.

![Delivery Web](/secciones/posts/imagenes/Baterias-1/carga-en-paralelo.png)

### Carga en Serie (2 Celdas - 7.4V)

Para cargar baterías en serie se necesita un módulo de carga específico para el número de celdas (ej: 2S). La conexión es la siguiente:

*   El **V-** del balanceador de la primera celda y el **V-** del módulo de carga se conectan al polo negativo del conjunto (GND).
*   El **V+** del balanceador de la última celda y el **V+** del módulo de carga se conectan al polo positivo del conjunto.
*   El punto central (donde se unen las celdas) se conecta a la terminal correspondiente del módulo de carga (ej: "BAT" o "2S").

![Delivery Web](/secciones/posts/imagenes/Baterias-1/carga-en-serie.png)
![Delivery Web](/secciones/posts/imagenes/Baterias-1/modulo-2s.png)
![Delivery Web](/secciones/posts/imagenes/Baterias-1/modulo-a-bateria.png)