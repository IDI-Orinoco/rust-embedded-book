# Un entorno Rust `no_std`

El término Programación Embebida se utiliza para una amplia gama de diferentes clases de programación. Va desde la programación de MCUs de 8 bits (como el [ST72325xx](https://www.st.com/resource/en/datasheet/st72325j6.pdf)) con sólo unos pocos KB de RAM y ROM, hasta sistemas como la Raspberry Pi ([Modelo B 3+](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications)) que tiene un Cortex-A53 de 4 núcleos de 32/64 bits a 1,4 GHz y 1GB de RAM. Se aplicarán diferentes restricciones/limitaciones cuando estes escribiendo el código dependiendo del tipo de objetivo y del caso de uso que tengas.

Hay dos clasificaciones generales de programación embebida:

## Entornos alojados
Este tipo de entornos se asemeja a un entorno de PC normal. Esto significa que se le proporciona una interfaz de sistema, por ejemplo [POSIX](https://en.wikipedia.org/wiki/POSIX), que te proporcionan primitivas para interactuar con varios sistemas, como sistemas de archivos, redes, gestión de memoria, hilos, etc. Las bibliotecas estándar, a su vez, suelen depender de estas primitivas para implementar su funcionalidad. También puede tener algún tipo de sysroot y restricciones en el uso de RAM/ROM, y quizás algunos HW o I/Os especiales. En general, se siente como codificar en un entorno de PC de propósito especial.

## Entornos Bare Metal
En un entorno bare metal no se ha cargado ningún código antes del programa. Sin el software proporcionado por un SO no podemos cargar la biblioteca estándar. En su lugar, el programa, junto con las crates que utiliza, sólo puede utilizar el hardware (bare metal) para ejecutarse. Para evitar que rust cargue la biblioteca estándar utilice `no_std`. Las partes agnósticas a la plataforma de la biblioteca estándar están disponibles a través de [libcore](https://doc.rust-lang.org/core/). libcore también excluye cosas que no siempre son deseables en un entorno embebido. Una de estas cosas es un asignador de memoria para la asignación de memoria dinámica. Si necesitas esta u otras funcionalidades, a menudo hay crates que las proporcionan.

### El ejecutable de libstd
Como se mencionó antes, el uso de [libstd](https://doc.rust-lang.org/std/) requiere algún tipo de integración del sistema, pero esto no es sólo porque [libstd](https://doc.rust-lang.org/std/) está proporcionando una forma común de acceder a las abstracciones del sistema operativo, sino que también proporciona un ejecutable. Este ejecutable, entre otras cosas, se encarga de configurar la protección contra el desbordamiento de pila, procesar los argumentos de la línea de comandos y generar el hilo principal antes de que se invoque la función principal de un programa. Este tiempo de ejecución tampoco estará disponible en un entorno `no_std`.

## Resumen
`#![no_std]` es un atributo a nivel de crate que indica que el crate se enlazará con el core-crate en lugar de con el std-crate. El crate [libcore](https://doc.rust-lang.org/core/) es a su vez un subconjunto agnóstico de plataforma del crate std que no hace suposiciones sobre el sistema en el que se ejecutará el programa. Como tal, proporciona APIs para primitivas del lenguaje como floats, strings y slices, así como APIs que exponen características del procesador como operaciones atómicas e instrucciones SIMD. Sin embargo, carece de APIs para cualquier cosa que implique la integración de la plataforma. Debido a estas propiedades, el código de no\_std y [libcore](https://doc.rust-lang.org/core/) se puede utilizar para cualquier tipo de código de arranque (fase 0) como bootloaders, firmware o kernels.

### Resumen

| característica                                            | no\_std | std |
|-----------------------------------------------------------|--------|-----|
| pila (memoria dinámica)                                   |   *    |  ✓  |
| colecciones (Vec, HashMap, etc)                           |  **    |  ✓  |
| protección contra desbordamiento de pila                  |   ✘    |  ✓  |
| ejecuta el código init antes del main                     |   ✘    |  ✓  |
| libstd disponible                                         |   ✘    |  ✓  |
| libcore disponible                                        |   ✓    |  ✓  |
| escribir el código de firmware, kernel o bootloader       |   ✓    |  ✘  |

\* Sólo si usas el crate `alloc` y utilizas un asignador adecuado como [alloc-cortex-m].

\** Sólo si usas el crate `collections` y configuras un asignador global por defecto.

[alloc-cortex-m]: https://github.com/rust-embedded/alloc-cortex-m

## Ver también
* [RFC-1184](https://github.com/rust-lang/rfcs/blob/master/text/1184-stabilize-no_std.md)
