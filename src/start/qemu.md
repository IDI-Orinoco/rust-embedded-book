# QEMU

Empezaremos escribiendo un programa para el [LM3S6965], un microcontrolador Cortex-M3. Hemos elegido este como nuestro objetivo inicial porque [puede ser emulado](https://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm) usando QEMU por lo que no es necesario juguetear con el hardware en esta sección y podemos centrarnos en las herramientas y el proceso de desarrollo.

[lm3s6965]: http://www.ti.com/product/LM3S6965

**IMPORTANTE**
Usaremos el nombre "app" para el nombre del proyecto en este tutorial. Siempre que veas la palabra "app" debes sustituirla por el nombre que hayas seleccionado para tu proyecto. O, también podrías nombrar tu proyecto "app" y evitar las sustituciones.

## Creando un programa Rust no estándar

Usaremos la plantilla de proyecto [`cortex-m-quickstart`] para generar un nuevo proyecto a partir de ella. El proyecto creado contendrá una aplicación barebone: un buen punto de partida para una nueva aplicación Rust embebida. Además, el proyecto contendrá un directorio `examples`, con varias aplicaciones separadas, destacando algunas de las funcionalidades clave de rust embebido.

[`cortex-m-quickstart`]: https://github.com/rust-embedded/cortex-m-quickstart

### Usando `cargo-generate`

Primero instala cargo-generate

```console
cargo install cargo-generate
```

Luego genera un nuevo proyecto

```console
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```

```text
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app
```

```console
cd app
```

### Usando `git`

Clonar el repositorio

```console
git clone https://github.com/rust-embedded/cortex-m-quickstart app
cd app
```

Y luego rellena los marcadores de posición en el archivo `Cargo.toml`.

```toml
[package]
authors = ["{{authors}}"] # "{{authors}}" -> "John Smith"
edition = "2018"
name = "{{project-name}}" # "{{project-name}}" -> "app"
version = "0.1.0"

# ..

[[bin]]
name = "{{project-name}}" # "{{project-name}}" -> "app"
test = false
bench = false
```

### Sin usar herramientas

Toma la última instantánea de la plantilla `cortex-m-quickstart` y extráela.

```console
curl -LO https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip
unzip master.zip
mv cortex-m-quickstart-master app
cd app
```

O puedes navegar hasta [`cortex-m-quickstart`], hacer clic en el botón verde "Clonar o descargar" y luego en "Descargar ZIP".

A continuación, rellena los marcadores de posición en el archivo `Cargo.toml` como se hace en la segunda parte de la versión "Usando `git`".

## Resumen del Programa

Por conveniencia aquí están las partes más importantes del código fuente en `src/main.rs`:

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        // your code goes here
    }
}
```

Este programa es un poco diferente de un programa estándar de Rust, así que echemos un vistazo más de cerca.

`#![no_std]` indica que este programa _no_ se enlazará a la _crate_ estándar, `std`. En su lugar, se enlazará a su subconjunto: la `core` _crate_.

`#![no_main]` indica que este programa no usará la interfaz estándar `main` que usan la mayoría de los programas Rust. La principal razón para usar `no_main` es que usar la interfaz `main` en el contexto `no_std` requiere nightly.

`use panic_halt as _;`. Esta _crate_ proporciona un `panic_handler` que define el comportamiento de pánico del programa. Cubriremos esto con más detalle en el capítulo [Panicking](panicking.md) del libro.

[`#[entry]`][entry] es un atributo proporcionado por la _crate_ [`cortex-m-rt`] que se utiliza para marcar el punto de entrada del programa. Como no estamos usando la interfaz estándar `main` necesitamos otra forma de indicar el punto de entrada del programa y esa sería `#[entry]`.

[entry]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html
[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt

`fn main() -> !`. Nuestro programa será el _único_ proceso ejecutándose en el hardware de destino, ¡así que no queremos que termine! Usamos una [función divergente](https://doc.rust-lang.org/rust-by-example/fn/diverging.html) (el bit `-> !` en la firma de la función) para asegurar en tiempo de compilación que ese será el caso.

## Compilación cruzada

El siguiente paso es _compilar_ el programa para la arquitectura Cortex-M3. Esto es tan sencillo como ejecutar `cargo build --target $TRIPLE` si tienes claro cuál debe ser el objetivo de compilación (`$TRIPLE`). Por suerte, el archivo `.cargo/config.toml` de la plantilla tiene la respuesta:

```console
tail -n6 .cargo/config.toml
```

```toml
[build]
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
# target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

Para realizar la compilación cruzada para la arquitectura Cortex-M3 tenemos que utilizar `thumbv7m-none-eabi`. Ese _target_ no se instala automáticamente al instalar el toolchain de Rust, ahora sería un buen momento para añadir ese _target_ al toolchain, si aún no lo has hecho:

```console
rustup target add thumbv7m-none-eabi
```

Dado que el objetivo de compilación `thumbv7m-none-eabi` ha sido establecido por defecto en tu archivo `.cargo/config.toml`, los dos comandos siguientes hacen lo mismo:

```console
cargo build --target thumbv7m-none-eabi
cargo build
```

## Inspeccionando

Ahora tenemos un binario ELF no nativo en `target/thumbv7m-none-eabi/debug/app`. Podemos inspeccionarlo usando `cargo-binutils`.

Con `cargo-readobj` podemos imprimir las cabeceras ELF para confirmar que se trata de un binario ARM.

```console
cargo readobj --bin app -- --file-headers
```

Ten en cuenta que:

- `--bin app` es azúcar para inspeccionar el binario en `target/$TRIPLE/debug/app`.
- `--bin app` también (re)compilará el binario, si es necesario

```text
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0x0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x405
  Start of program headers:          52 (bytes into file)
  Start of section headers:          153204 (bytes into file)
  Flags:                             0x5000200
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         19
  Section header string table index: 18
```

`cargo-size` puede imprimir el tamaño de las secciones de enlace del binario.

```console
cargo size --bin app --release -- -A
```

usamos `--release` para inspeccionar la versión optimizada

```text
app  :
section             size        addr
.vector_table       1024         0x0
.text                 92       0x400
.rodata                0       0x45c
.data                  0  0x20000000
.bss                   0  0x20000000
.debug_str          2958         0x0
.debug_loc            19         0x0
.debug_abbrev        567         0x0
.debug_info         4929         0x0
.debug_ranges         40         0x0
.debug_macinfo         1         0x0
.debug_pubnames     2035         0x0
.debug_pubtypes     1892         0x0
.ARM.attributes       46         0x0
.debug_frame         100         0x0
.debug_line          867         0x0
Total              14570
```

> Un repaso a las secciones del enlazador ELF
>
> - `.text` contiene las instrucciones del programa
> - `.rodata` contiene valores constantes como cadenas
> - `.data` contiene variables asignadas estáticamente cuyos valores iniciales son _no_ cero
> - `.bss` también contiene variables asignadas estáticamente cuyos valores iniciales _son_ cero
> - `.vector_table` es una sección _no_ estándar que utilizamos para almacenar la tabla de vectores (interrupciones)
> - Las secciones `.ARM.attributes` y `.debug_*` contienen metadatos y _no_ se cargarán en el objetivo al flashear el binario.

**IMPORTANTE**: Los archivos ELF contienen metadatos como información de depuración, por lo que su _tamaño en disco_ _no_ refleja con exactitud el espacio que ocupará el programa al ser flasheado en un dispositivo. _Usa_ siempre `cargo-size` para comprobar el tamaño real de un binario.

Se puede usar `cargo-objdump` para desensamblar el binario.

```console
cargo objdump --bin app --release -- --disassemble --no-show-raw-insn --print-imm-hex
```

> **NOTA** si el comando anterior se queja de `Unknown command line argument` mira el siguiente informe de error: https://github.com/rust-embedded/book/issues/269

> **NOTA** esta salida puede diferir en su sistema. Nuevas versiones de rustc, LLVM y bibliotecas pueden generar diferentes ensamblados. Hemos truncado algunas de las instrucciones para mantener el fragmento pequeño.

```text
app:  file format ELF32-arm-little

Disassembly of section .text:
main:
     400: bl  #0x256
     404: b #-0x4 <main+0x4>

Reset:
     406: bl  #0x24e
     40a: movw  r0, #0x0
     < .. truncated any more instructions .. >

DefaultHandler_:
     656: b #-0x4 <DefaultHandler_>

UsageFault:
     657: strb  r7, [r4, #0x3]

DefaultPreInit:
     658: bx  lr

__pre_init:
     659: strb  r7, [r0, #0x1]

__nop:
     65a: bx  lr

HardFaultTrampoline:
     65c: mrs r0, msp
     660: b #-0x2 <HardFault_>

HardFault_:
     662: b #-0x4 <HardFault_>

HardFault:
     663: <unknown>
```

## Ejecutando

A continuación, ¡vamos a ver cómo ejecutar un programa embebido en QEMU! Esta vez usaremos el ejemplo `hello` que realmente hace algo.

Por conveniencia aquí está el código fuente de `examples/hello.rs`:

```rust,ignore
//! Escribe "Hola, mundo!"en la consola del anfitrión usando semihosting

#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
    hprintln!("Hola, mundo!").unwrap();

    // salir de QEMU
    // NOTA no ejecute esto en el hardware; puede corromper el estado del OpenOCD
    debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

Este programa usa algo llamado semihosting para imprimir texto en la consola _host_. Cuando se usa hardware real esto requiere una sesión de depuración pero cuando se usa QEMU esto simplemente funciona.

Empecemos compilando el ejemplo:

```console
cargo build --example hello
```

El binario de salida estará localizado en `target/thumbv7m-none-eabi/debug/examples/hello`.

Para ejecutar este binario en QEMU ejecute el siguiente comando:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

```text
Hello, world!
```

El comando debería salir con éxito (código de salida = 0) después de imprimir el texto. En \*nix puedes comprobarlo con el siguiente comando:

```console
echo $?
```

```text
0
```

Vamos a desglosar ese comando QEMU:

- `qemu-system-arm`. Este es el emulador QEMU. Hay algunas variantes de estos binarios QEMU; éste hace una emulación completa del _sistema_ de máquinas _ARM_, de ahí el nombre.

- `-cpu cortex-m3`. Esto le dice a QEMU que emule una CPU Cortex-M3. Especificar el modelo de CPU nos permite detectar algunos errores de compilación: por ejemplo, ejecutar un programa compilado para Cortex-M4F, que tiene una FPU por hardware, hará que QEMU se equivoque durante su ejecución.

- `-machine lm3s6965evb`. Esto le dice a QEMU que emule el LM3S6965EVB, una tarjeta de evaluación que contiene un microcontrolador LM3S6965.

- `-nographic`. Esto le dice a QEMU que no lance su GUI.

- `-semihosting-config (..)`. Esto le dice a QEMU que habilite semihosting. Semihosting permite al dispositivo emulado, entre otras cosas, utilizar el stdout, stderr y stdin del host y crear archivos en el host.

- `-kernel $file`. Esto le dice a QEMU qué binario cargar y ejecutar en la máquina emulada.

¡Escribir ese largo comando QEMU es demasiado trabajo! Podemos configurar un _runner_ personalizado para simplificar el proceso. El archivo `.cargo/config.toml` tiene un _runner_ comentado que invoca a QEMU; vamos a descomentarlo:

```console
head -n3 .cargo/config.toml
```

```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```

Este _runner_ sólo se aplica al objetivo `thumbv7m-none-eabi`, que es nuestro objetivo de compilación por defecto. Ahora `cargo run` compilará el programa y lo ejecutará en QEMU:

```console
cargo run --example hello --release
```

```text
   Compiling app v0.1.0 (file:///tmp/app)
    Finished release [optimized + debuginfo] target(s) in 0.26s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/hello`
Hello, world!
```

## Depuración

La depuración es crítica para el desarrollo embebido. Veamos cómo se hace.

Depurar un dispositivo embebido implica depuración _remota_ ya que el programa que queremos depurar no se estará ejecutando en la máquina que está ejecutando el programa depurador (GDB o LLDB).

La depuración remota implica un cliente y un servidor. En una configuración QEMU, el cliente será un proceso GDB (o LLDB) y el servidor será el proceso QEMU que también está ejecutando el programa embebido.

En esta sección usaremos el ejemplo `hello` que ya hemos compilado.

El primer paso para depurar es lanzar QEMU en modo depuración:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -gdb tcp::3333 \
  -S \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

Este comando no imprimirá nada en la consola y bloqueará el terminal. Esta vez hemos pasado dos banderas extra

- `-gdb tcp::3333`. Esto le dice a QEMU que espere una conexión GDB en el puerto TCP 3333.

- `-S`. Esto le dice a QEMU que congele la máquina en el arranque. Sin esto el programa habría llegado al final de main ¡antes de que tuviéramos la oportunidad de lanzar el depurador!

A continuación lanzamos GDB en otro terminal y le decimos que cargue los símbolos de depuración del ejemplo:

```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

**NOTA**: puede que necesite otra versión de gdb en lugar de `gdb-multiarch` dependiendo de la que haya instalado en el capítulo de instalación. También podría ser `arm-none-eabi-gdb` o simplemente `gdb`.

A continuación, dentro de la shell GDB nos conectamos a QEMU, usando el comando `target`, que está esperando una conexión en el puerto TCP 3333.

```console
target remote :3333
```

```text
Remote debugging using :3333
Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473
473     pub unsafe extern "C" fn Reset() -> ! {
```

Verás que el proceso se detiene y que el contador del programa apunta a una función llamada `Reset`. Este es el manejador de reset: lo que los núcleos Cortex-M ejecutan al arrancar.

> Ten en cuenta que en algunas configuraciones, en lugar de mostrar la línea `Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473` como se muestra arriba, gdb puede imprimir algunas advertencias como:
>
> `core::num::bignum::Big32x40::mul_small () at src/libcore/num/bignum.rs:254` 
> `    src/libcore/num/bignum.rs: No such file or directory.`
>
> Es un fallo conocido. Puedes ignorar con seguridad esas advertencias, lo más probable es que estés en Reset().

Este manejador de reset eventualmente llamará a nuestra función principal. Saltemos todo el camino hasta allí usando un _breakpoint_ y el comando `continue`. Para establecer el punto de ruptura, primero echemos un vistazo a dónde nos gustaría hacer el _break_ en nuestro código, con el comando `list`.

```console
list main
```

Esto mostrará el código fuente, del archivo examples/hello.rs.

```text
6       use panic_halt as _;
7
8       use cortex_m_rt::entry;
9       use cortex_m_semihosting::{debug, hprintln};
10
11      #[entry]
12      fn main() -> ! {
13          hprintln!("Hello, world!").unwrap();
14
15          // exit QEMU
```

Nos gustaría añadir un _breakpoint_ justo antes del "¡Hola, mundo!", que está en la línea 13. Lo hacemos con el comando `break`:

```console
break 13
```

Ahora podemos ordenar a gdb que ejecute hasta nuestra función principal, con el comando `continue`:

```console
continue
```

```text
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main () at examples\hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

Ya estamos cerca del código que imprime "Hello, world!". Avancemos usando el comando `next`.

```console
next
```

```text
16          debug::exit(debug::EXIT_SUCCESS);
```

En este punto deberías ver "Hello, world!" impreso en el terminal que está ejecutando `qemu-system-arm`.

```text
$ qemu-system-arm (..)
Hello, world!
```

Llamando de nuevo a `next` terminará el proceso QEMU.

```console
next
```

```text
[Inferior 1 (Remote target) exited normally]
```

Ahora puedes salir de la sesión GDB.

```console
quit
```
