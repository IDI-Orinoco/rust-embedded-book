# Periféricos

## ¿Qué son los periféricos?

La mayoría de los microcontroladores tienen algo más que una CPU, RAM o memoria Flash: contienen secciones de silicio que se utilizan para interactuar con sistemas externos al microcontrolador, así como para interactuar directa e indirectamente con su entorno en el mundo a través de sensores, controladores de motor o interfaces humanas como una pantalla o un teclado. Estos componentes se conocen colectivamente como periféricos.

Estos periféricos son útiles porque le permiten al desarrollador relegar procesamiento en ellos, evitando tener que gestionar todo en el software. Al igual que un desarrollador en una pc relegaría el procesamiento gráfico a una tarjeta de vídeo, los desarrolladores de sistemas embebidos pueden relegar algunas tareas a los periféricos, lo que permite a la CPU dedicar su tiempo a hacer otra cosa importante, o no hacer nada para ahorrar energía.

Si nos fijamos en la tarjeta principal de un computador doméstico antiguo de los años 70 u 80 (y en realidad, los PC de sobremesa de ayer no están tan alejados de los sistemas embebidos de hoy en día) esperaríamos ver:

- Un procesador
- Un chip de RAM
- Un chip de ROM
- Un controlador de E/S

El chip de RAM, el chip de ROM y el controlador de E/S (el periférico en este sistema) estarían unidos al procesador a través de una serie de pistas paralelas conocidas como 'bus'. Este bus transporta la información de la dirección, que selecciona con qué dispositivo del bus desea comunicarse el procesador, y un bus de datos que transporta los datos propiamente dichos. En nuestros microcontroladores embebidos se aplican los mismos principios, sólo que todo está empaquetado en una única pieza de silicio.

Sin embargo, a diferencia de las tarjetas gráficas, que normalmente tienen una API de software como Vulkan, Metal u OpenGL, los periféricos están expuestos a nuestro microcontrolador con una interfaz de hardware, que se asigna a un área de la memoria.

## Espacio de Memoria Lineal y Real

En un microcontrolador, escribir algunos datos en alguna otra dirección arbitraria, como `0x4000_0000` o `0x0000_0000`, también puede ser una acción completamente válida.

En un sistema de sobremesa, el acceso a la memoria está estrechamente controlado por la MMU, o Unidad de Gestión de Memoria. Este componente tiene dos responsabilidades principales: hacer cumplir los permisos de acceso a secciones de memoria (impidiendo que un proceso lea o modifique la memoria de otro proceso); y reasignar segmentos de la memoria física a rangos de memoria virtual utilizados en el software. Los microcontroladores no suelen tener una MMU, y en su lugar sólo utilizan direcciones físicas reales en el software.

Aunque los microcontroladores de 32 bits tienen un espacio de direcciones real y lineal desde `0x0000_0000`, y `0xFFFF_FFFF`, generalmente sólo utilizan unos pocos cientos de kilobytes de ese rango para la memoria real. Esto deja una cantidad significativa de espacio de direcciones restante. En capítulos anteriores, hablábamos de que la RAM estaba localizada en la dirección `0x2000_0000`. Si nuestra RAM tuviera 64 KiB de longitud (es decir, con una dirección máxima de 0xFFFF) entonces las direcciones `0x2000_0000` a `0x2000_FFFF` corresponderían a nuestra RAM. Cuando escribimos en una variable que esta en la dirección `0x2000_1234`, lo que ocurre internamente es que alguna lógica detecta la parte superior de la dirección (0x2000 en este ejemplo) y luego activa la RAM para que pueda actuar sobre la parte inferior de la dirección (0x1234 en este caso). En un Cortex-M también tenemos nuestra Flash ROM mapeada en la dirección `0x0000_0000` hasta, digamos, la dirección `0x0007_FFFF` (si tenemos una Flash ROM de 512 KiB). En lugar de ignorar todo el espacio restante entre estas dos regiones, los diseñadores de microcontroladores mapearon la interfaz para los periféricos en ciertas posiciones de memoria. Esto termina pareciendo algo como esto

![](../assets/nrf52-memory-map.png)

[Hoja de datos Nordic nRF52832 (pdf)]

## Periféricos Mapeados en Memoria

La interacción con estos periféricos es simple a primera vista - escribir los datos correctos en la dirección correcta. Por ejemplo, enviar una palabra de 32 bits a través de un puerto serial podría ser tan directo como escribir esa palabra de 32 bits en una determinada dirección de memoria. El periférico del puerto serial se encargaría entonces de enviar los datos automáticamente.

La configuración de estos periféricos funciona de forma similar. En lugar de llamar a una función para configurar un periférico, se expone un trozo de memoria que sirve como API de hardware. Escribe `0x8000_0000` en un Registro de Configuración de Frecuencia SPI, y el puerto SPI enviará datos a 8 Megabits por segundo. Escribe `0x0200_0000` en la misma dirección, y el puerto SPI enviará datos a 125 Kilobits por segundo. Estos registros de configuración se parecen un poco a esto:

![](../assets/nrf52-spi-frequency-register.png)

[Hoja de datos Nordic nRF52832 (pdf)]

Esta interfaz es la forma en que se realizan las interacciones con el hardware, independientemente del lenguaje que se utilice, ya sea ensamblador, C o Rust.

[hoja de datos nordic nrf52832 (pdf)]: http://infocenter.nordicsemi.com/pdf/nRF52832_PS_v1.1.pdf
