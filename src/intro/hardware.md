# Conoce tu hardware

Vamos a familiarizarnos con el hardware con el que vamos a trabajar.

## STM32F3DISCOVERY (El "F3")

<p align="center">
<img title="F3" src="../assets/f3.jpg">
</p>

¿Qué contiene esta placa?

- Un microcontrolador [STM32F303VCT6](https://www.st.com/en/microcontrollers/stm32f303vc.html). Este microcontrolador tiene
  - Un procesador ARM Cortex-M4F de un solo núcleo con soporte de hardware para operaciones de punto flotante de precisión única y una frecuencia de reloj máxima de 72 MHz.

  - 256 KiB de memoria "Flash". (1 KiB = 10**24** bytes)

  - 48 KiB de memoria RAM.

  - Una variedad de periféricos integrados como temporizadores, I2C, SPI y USART.

  - Entradas y salidas de propósito general (GPIO) y otros tipos de pines accesibles a través de las dos filas de cabeceras a lo largo de la placa.
    
  - Una interfaz USB accesible a través del puerto USB etiquetado como "USB USER".

- Un [acelerómetro](https://es.wikipedia.org/wiki/Aceler%C3%B3metro) como parte del chip [LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html).

- Un [magnetómetro](https://es.wikipedia.org/wiki/Magnet%C3%B3metro) como parte del chip [LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html). 

- Un [giroscopio](https://es.wikipedia.org/wiki/Gir%C3%B3scopo) como parte del chip [L3GD20](https://www.pololu.com/file/0J563/L3GD20.pdf).

- 8 LEDs de usuario dispuestos en forma de brújula.

- Un segundo microcontrolador: un [STM32F103](https://www.st.com/en/microcontrollers/stm32f103cb.html). Este microcontrolador es en realidad parte de un programador / depurador a bordo y está conectado al puerto USB llamado "USB ST-LINK".


Para obtener una lista más detallada de las características y otras especificaciones de la placa, consulta el sitio web de [STMicroelectronics](https://www.st.com/en/evaluation-tools/stm32f3discovery.html).

Una advertencia: ten cuidado si quieres aplicar señales externas a la placa. Los pines del microcontrolador STM32F303VCT6 reciben una tensión nominal de 3,3 voltios. Para más información, consulta la sección [6.2 Valores máximos absolutos del manual](https://www.st.com/resource/en/datasheet/stm32f303vc.pdf)
