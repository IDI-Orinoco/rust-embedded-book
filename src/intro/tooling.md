# Herramientas

Tratar con microcontroladores implica el uso de varias herramientas diferentes ya que estaremos con una arquitectura diferente a la de tu portátil y tendremos que ejecutar y depurar programas en un dispositivo *remoto*.

Utilizaremos todas las herramientas listadas a continuación. Cualquier versión reciente debería funcionar no se especifica una versión mínima, pero hemos enumerado las versiones que hemos probado.

- Rust 1.31, 1.31-beta, o un toolchain más reciente MÁS soporte de compilación ARM Cortex-M. soporte.
- [`cargo-binutils`](https://github.com/rust-embedded/cargo-binutils) ~0.1.4
- [`qemu-system-arm`](https://www.qemu.org/). Versiones probadas: 3.0.0
- OpenOCD >=0.8. Versiones probadas: v0.9.0 y v0.10.0
- GDB con soporte ARM. Versión 7.12 o más reciente muy recomendable. Versiones probadas de probadas: 7.10, 7.11, 7.12 y 8.1
- [`cargo-generate`](https://github.com/ashleygwilliams/cargo-generate) o `git`. Estas herramientas son opcionales pero facilitarán el seguimiento del libro.

El siguiente texto explica por qué utilizamos estas herramientas. Las instrucciones de instalación se encuentran en la página siguiente.

## `cargo-generate` O `git`

Los programas "bare metal" son programas Rust no estándar (`no_std`) que requieren algunos ajustes en el proceso de enlazado para poder obtener la disposición de memoria del programa. Esto requiere algunos archivos (como scripts del enlazador) y ajustes (como las banderas del enlazador) adicionales. Los hemos empaquetado en una plantilla para que sólo tengas que rellenar la información que falta (como el nombre del proyecto y las características de tu hardware de destino).

Nuestra plantilla es compatible con `cargo-generate`: un subcomando de Cargo para crear nuevos proyectos Cargo a partir de plantillas. También puedes descargarla usando `git`, `curl`, `wget`, o tu navegador web.

## `cargo-binutils`

`cargo-binutils` es una colección de subcomandos de Cargo que facilitan el uso de las herramientas LLVM que vienen con la cadena de herramientas de Rust. Estas herramientas incluyen las versiones LLVM de `objdump`, `nm` y `size` y se utilizan para inspeccionar binarios.

La ventaja de usar estas herramientas sobre las GNU binutils es que (a) instalar las herramientas de LLVM es la misma instalación con un solo comando (`rustup component add llvm-tools-preview`) independientemente de tu sistema operativo y (b) herramientas como `objdump` soportan todas las arquitecturas que `rustc` soporta -- desde ARM hasta x86_64 -- porque ambas comparten el mismo LLVM backend.

## `qemu-system-arm`

QEMU es un emulador. En este caso usamos la variante que puede emular completamente sistemas ARM. Usamos QEMU para ejecutar programas embebidos en el host. Gracias a esto puedes ¡puedes seguir algunas partes de este libro incluso si no tienes ningún hardware contigo!

## GDB

Un depurador es un componente muy importante del desarrollo embebido, ya que no siempre puedes permitirte el lujo de registrar cosas en la consola del host. En algunos casos, ¡puede que ni siquiera tengas LEDs parpadeando en tu hardware!

En general, LLDB funciona tan bien como GDB cuando se trata de depuración, pero no hemos encontrado una contraparte de LLDB al comando `load` de GDB, que carga el programa en el hardware de destino, por lo que actualmente recomendamos que utilices GDB.

## OpenOCD

GDB no es capaz de comunicarse directamente con el hardware de depuración ST-Link en tu tarjeta de desarrollo STM32F3DISCOVERY. Necesita un traductor y el Open On-Chip Debugger, OpenOCD, es ese traductor. OpenOCD es un programa que se ejecuta en tu laptop/PC y traduce entre el protocolo de depuración remota basado en TCP/IP de GDB y el protocolo USB de ST-Link. de GDB y el protocolo basado en USB de ST-Link.

OpenOCD también realiza otros trabajos importantes como parte de la traducción para la depuración del microcontrolador ARM Cortex-M basado en la tarjeta de desarrollo STM32F3DISCOVERY:
* Sabe cómo interactuar con los registros mapeados en memoria utilizados por el periférico de depuración ARM CoreSight. Son estos registros CoreSight los que permiten:
  * Manipulación de Breakpoint/Watchpoint
  * Lectura y escritura de los registros de la CPU
  * Detectar cuando la CPU ha sido detenida por un evento de depuración
  * Continuar la ejecución de la CPU después de un evento de depuración
  * etc.
* También sabe cómo borrar y escribir en la FLASH del microcontrolador.
