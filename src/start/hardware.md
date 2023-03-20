# Hardware

A estas alturas ya deberías estar familiarizado con las herramientas y el proceso de desarrollo. En esta sección cambiaremos a hardware real; el proceso seguirá siendo en gran medida el mismo. Empecemos.

## Conoce tu hardware

Antes de empezar es necesario identificar algunas características del dispositivo de destino, ya que se utilizarán para configurar el proyecto:

- El núcleo ARM. Por ejemplo, Cortex-M3.

- ¿Incluye el núcleo ARM una FPU? Los núcleos Cortex-M4**F** y Cortex-M7**F** sí.

- ¿Cuánta memoria Flash y RAM tiene el dispositivo de destino? Por ejemplo, 256 KiB de Flash y 32 KiB de RAM.

- ¿Dónde están mapeadas la memoria Flash y la RAM en el espacio de direcciones? Por ejemplo, la RAM se encuentra normalmente en la dirección `0x2000_0000`.

Puedes encontrar esta información en la hoja de datos o en el manual de referencia de tu dispositivo.

En esta sección utilizaremos nuestro hardware de referencia, la STM32F3DISCOVERY. Esta tarjeta contiene un microcontrolador STM32F303VCT6. Este microcontrolador tiene:

- Un núcleo Cortex-M4F que incluye una FPU de precisión simple

- 256 KiB de Flash localizados en la dirección 0x0800_0000.

- 40 KiB de RAM localizados en la dirección 0x2000_0000. (Hay otra región de RAM pero por simplicidad la ignoraremos).

## Configuración

Empezaremos desde cero con una instancia de plantilla nueva. Consulta la [sección anterior sobre QEMU] para refrescar cómo hacer esto sin `cargo-generate`.

[sección anterior sobre qemu]: qemu.md

```text
$ cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app

$ cd app
```

El paso número uno es establecer un objetivo de compilación por defecto en `.cargo/config.toml`.

```console
tail -n5 .cargo/config.toml
```

```toml
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

Usaremos `thumbv7em-none-eabihf` ya que cubre el núcleo Cortex-M4F.

El segundo paso es introducir la información de la región de memoria en el archivo `memory.x`.

```text
$ cat memory.x
/* Linker script for the STM32F303VCT6 */
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```

> **NOTA**: Si por alguna razón ha cambiado el archivo `memory.x` después de haber hecho
> la primera compilación de un objetivo de compilación específico, entonces haz `cargo clean` antes de
> de `cargo build`, porque `cargo build` puede no seguir las actualizaciones de `memory.x`.

Empezaremos con el ejemplo hello de nuevo, pero primero tenemos que hacer un pequeño cambio.

En `examples/hello.rs`, asegúrate de que la llamada `debug::exit()` está comentada o eliminada. Se utiliza sólo para ejecutar en QEMU.

```rust,ignore
#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // exit QEMU
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

Ahora puedes compilar programas usando `cargo build` e inspeccionar los binarios usando `cargo-binutils` como hacías antes. La crate `cortex-m-rt` se encarga de toda la magia necesaria para que tu chip funcione, ya que prácticamente todas las CPUs Cortex-M arrancan de la misma manera.

```console
cargo build --example hello
```

## Depuración

La depuración será un poco diferente. De hecho, los primeros pasos pueden parecer diferentes dependiendo del dispositivo de destino. En esta sección mostraremos los pasos necesarios para depurar un programa que se ejecuta en el STM32F3DISCOVERY. Esto debe servir como referencia; para información específica del dispositivo sobre depuración consulte el [Debugonomicon](https://github.com/rust-embedded/debugonomicon).

Como antes haremos depuración remota y el cliente será un proceso GDB. Esta vez, sin embargo, el servidor será OpenOCD.

Como se hizo durante la sección [verificar] conecte la tarjeta discovery a su portátil / PC y compruebe que la cabecera ST-LINK está poblada.

[verificar]: ../intro/install/verify.md

En un terminal, ejecute `openocd` para conectarse al ST-LINK de la tarjeta discovery. Ejecute este comando desde la raíz de la plantilla; `openocd` leerá el archivo `openocd.cfg` que indica qué archivo de interfaz y archivo de destino utilizar.

```console
cat openocd.cfg
```

```text
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board

# Depending on the hardware revision you got you'll have to pick ONE of these
# interfaces. At any time only one interface should be commented out.

# Revision C (newer revision)
source [find interface/stlink.cfg]

# Revision A and B (older revisions)
# source [find interface/stlink-v2.cfg]

source [find target/stm32f3x.cfg]
```

> **NOTA** Si descubriste que tienes una revisión anterior de la tarjeta discovery
> durante la sección [verificar] entonces debe modificar el archivo `openocd.cfg`
> en este punto para usar `interface/stlink-v2.cfg`.

```text
$ openocd
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.913879
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

En otro terminal ejecute GDB, también desde la raíz de la plantilla.

```text
gdb-multiarch -q target/thumbv7em-none-eabihf/debug/examples/hello
```

**NOTA**: como antes, puede que necesites otra versión de gdb en lugar de `gdb-multiarch` dependiendo de la que haya instalado en el capítulo de instalación. También puede ser `arm-none-eabi-gdb` o simplemente `gdb`.

A continuación conecte GDB a OpenOCD, que está esperando una conexión TCP en el puerto 3333 utilizando el comando `target`.

```console
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

Ahora proceda a _flashear_ (cargar) el programa en el microcontrolador utilizando el comando `load`.

```console
(gdb) load
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1518 lma 0x8000400
Loading section .rodata, size 0x414 lma 0x8001918
Start address 0x08000400, load size 7468
Transfer rate: 13 KB/sec, 2489 bytes/write.
```

El programa ya está cargado. Este programa usa semihosting así que antes de hacer cualquier llamada a semihosting tenemos que decirle a OpenOCD que habilite semihosting. Puedes enviar comandos a OpenOCD usando el comando `monitor`.

```console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

> Puedes ver todos los comandos de OpenOCD invocando el comando `monitor help`.

Como antes podemos saltar hasta `main` usando un _breakpoint_ y el comando `continue`.

```console
(gdb) break main
Breakpoint 1 at 0x8000490: file examples/hello.rs, line 11.
Note: automatically using hardware breakpoints for read-only addresses.

(gdb) continue
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main_trampoline () at examples/hello.rs:11
11      #[entry]
```

> **NOTA** Si GDB bloquea el terminal en lugar de parar en el _breakpoint_ después de
> ejecutar el comando `continue` anterior, es posible que quieras volver a comprobar que
> la información de la región de memoria en el archivo `memory.x` está configurada correctamente.
> para tu dispositivo (tanto los inicios como las longitudes).

Entra en la función principal con `step`.

```console
(gdb) step
halted: PC: 0x08000496
hello::__cortex_m_rt_main () at examples/hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

Después de hacer avanzar el programa con `next` deberías ver "Hello, world!" impreso en la consola de OpenOCD, entre otras cosas.

```console
$ openocd
(..)
Info : halted: PC: 0x08000502
Hello, world!
Info : halted: PC: 0x080004ac
Info : halted: PC: 0x080004ae
Info : halted: PC: 0x080004b0
Info : halted: PC: 0x080004b4
Info : halted: PC: 0x080004b8
Info : halted: PC: 0x080004bc
```

El mensaje sólo se muestra una vez cuando el programa está a punto de entrar en el bucle infinito definido en la línea 19: `loop {}`.

Ahora puedes salir de GDB usando el comando `quit`.

```console
(gdb) quit
A debugging session is active.

        Inferior 1 [Remote target] will be detached.

Quit anyway? (y or n)
```

La depuración ahora requiere algunos pasos más, así que hemos empaquetado todos esos pasos en un único script GDB llamado `openocd.gdb`. El archivo fue creado durante el paso `cargo generate`, y debería funcionar sin modificaciones. Echemos un vistazo:

```console
cat openocd.gdb
```

```text
target extended-remote :3333

# print demangled symbols
set print asm-demangle on

# detect unhandled exceptions, hard faults and panics
break DefaultHandler
break HardFault
break rust_begin_unwind

monitor arm semihosting enable

load

# start the process but immediately halt the processor
stepi
```

Ahora ejecutando `<gdb> -x openocd.gdb target/thumbv7em-none-eabihf/debug/examples/hello` conectará inmediatamente GDB a OpenOCD, habilitará el semihosting, cargará el programa e iniciará el proceso.

Alternativamente, puedes convertir `<gdb> -x openocd.gdb` en un runner personalizado para hacer que `cargo run` construya un programa _y_ comience una sesión GDB. Este runner está incluido en `.cargo/config.toml` pero está comentado.

```console
head -n10 .cargo/config.toml
```

```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
# runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
# uncomment ONE of these three option to make `cargo run` start a GDB session
# which option to pick depends on your system
runner = "arm-none-eabi-gdb -x openocd.gdb"
# runner = "gdb-multiarch -x openocd.gdb"
# runner = "gdb -x openocd.gdb"
```

```text
$ cargo run --example hello
(..)
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
(gdb)
```
