# Apéndice A: Glosario

El ecosistema embebido está lleno de diferentes protocolos, componentes de hardware y específicos de cada proveedor que utilizan sus propios términos y abreviaturas. Este glosario intenta enumerarlos con indicaciones para entenderlos mejor.

### BSP

Una _Crate_ de Soporte de Tarjeta (Board Support _Crate_) proporciona una interfaz de alto nivel configurada para una tarjeta específica. Normalmente depende de una _crate_ [HAL](#hal).
Hay una descripción más detallada en la [página de registros de memoria mapeada](../start/registers.md) o para una visión más amplia ver [este video](https://youtu.be/vLYit_HHPaY).

### FPU

Unidad de punto flotante. Un 'procesador matemático' que ejecuta sólo operaciones con números de punto flotante.

### HAL

Una Capa de Abstracción de Hardware proporciona una interfaz amigable para el desarrollador a las características y periféricos de un microcontrolador. Normalmente se implementa sobre una [_Crate_ de Acceso Periférico (PAC)](#pac).
También se puede implementar _traits_ de la _crate_ [`embedded-hal`](https://crates.io/crates/embedded-hal).
Hay una descripción más detallada en la [página de registros de memoria mapeada](../start/registers.md) o para una visión más amplia ver [este video](https://youtu.be/vLYit_HHPaY).

### I2C

También llamado `I²C` o Inter-IC. Es un protocolo destinado a la comunicación de hardware dentro de un único circuito integrado. Ver [aquí][i2c] para más detalles

[i2c]: https://en.wikipedia.org/wiki/I2c

### PAC

Una _Crate_ de Acceso Periférico proporciona acceso a los periféricos de un microcontrolador. Es uno de nivel inferior y normalmente se genera directamente a partir del [SVD](#svd) proporcionado, a menudo utilizando [svd2rust] (#svd). utilizando [svd2rust](https://github.com/rust-embedded/svd2rust/). La [Capa de Abstracción de Hardware](#hal) dependerá normalmente de este _crate_.
Hay una descripción más detallada en la [página de registros de memoria mapeada](../start/registers.md) o para una visión más amplia ver [este video](https://youtu.be/vLYit_HHPaY).

### SPI

Interfaz Periférica Serial. Ver [aquí][spi] para más información.

[spi]: https://en.wikipedia.org/wiki/Serial_peripheral_interface

### SVD

Descripción de la Vista del Sistema es un formato de archivo XML utilizado para describir la vista de los programadores de un dispositivo microcontrolador. Puede leer más sobre él en [el sitio de documentación ARM CMSIS](https://www.keil.com/pack/doc/CMSIS/SVD/html/index.html).

### UART

Receptor-Transmisor Asíncrono Universal. Ver [aquí][uart] para más información.

[uart]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter

### USART

Receptor-Transmisor Síncrono y Asíncrono Universal. Ver [aquí][usart] para más información.

[usart]: https://en.wikipedia.org/wiki/Universal_synchronous_and_asynchronous_receiver-transmitter
