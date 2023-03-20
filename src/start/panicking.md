# Pánico

El pánico es una parte esencial del lenguaje Rust. Las operaciones integradas, como la indexación, se comprueban en tiempo de ejecución para garantizar la seguridad de la memoria. Cuando se intenta indexar fuera de los límites, se produce un pánico.

En la biblioteca estándar el pánico tiene un comportamiento definido: despliega la pila del hilo en pánico, a menos que el usuario haya optado por abortar el programa en caso de pánico.

En programas sin biblioteca estándar, sin embargo, el comportamiento de pánico se deja indefinido. Se puede elegir un comportamiento declarando una función `#[panic_handler]`. Esta función debe existir exactamente _una vez_ en el grafo de dependencias de un programa, y debe tener la siguiente firma: `fn(&PanicInfo) -> !`, donde [`PanicInfo`] es una estructura que contiene información sobre la localización del pánico.

[`panicinfo`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

Dado que los sistemas embebidos van desde los orientados al usuario hasta los de seguridad crítica (que no pueden colapsar), no hay un comportamiento de pánico talla única, pero hay un montón de comportamientos comúnmente utilizados. Estos comportamientos comunes han sido empaquetados en _crates_ que definen la función `#[panic_handler]`. Algunos ejemplos incluyen:

- [`panic-abort`]. Un pánico provoca la ejecución de la instrucción abort.
- [`panic-halt`]. Un pánico hace que el programa, o el hilo actual, se detenga entrando en un bucle infinito.
- [`panic-itm`]. El mensaje de pánico se registra utilizando el ITM, un periférico específico de ARM Cortex-M.
- [`panic-semihosting`]. El mensaje de pánico se registra en el host utilizando la técnica semihosting.

[`panic-abort`]: https://crates.io/crates/panic-abort
[`panic-halt`]: https://crates.io/crates/panic-halt
[`panic-itm`]: https://crates.io/crates/panic-itm
[`panic-semihosting`]: https://crates.io/crates/panic-semihosting

Puedes encontrar más _crates_ buscando la palabra clave [`panic-handler`] en crates.io.

[`panic-handler`]: https://crates.io/keywords/panic-handler

Un programa puede elegir uno de estos comportamientos simplemente enlazando con la _crate_ correspondiente. El hecho de que el comportamiento de pánico se exprese en el código fuente de una aplicación como una única línea de código no sólo es útil como documentación, sino que también puede utilizarse para cambiar el comportamiento de pánico en función del perfil de compilación. Por ejemplo:

```rust,ignore
#![no_main]
#![no_std]

// dev profile: easier to debug panics; can put a breakpoint on `rust_begin_unwind`
#[cfg(debug_assertions)]
use panic_halt as _;

// release profile: minimize the binary size of the application
#[cfg(not(debug_assertions))]
use panic_abort as _;

// ..
```

En este ejemplo la _crate_ enlaza con la _crate_ `panic-halt` cuando se construye con el perfil dev (`cargo build`), pero enlaza con la _crate_ `panic-abort` cuando se construye con el perfil release (`cargo build --release`).

> La forma `use panic_abort as _;` de la sentencia `use` se usa para asegurar que el manejador de pánico `panic_abort`
> se incluye en nuestro ejecutable final, dejando claro al compilador que no usaremos explícitamente nada
> de la crate. Sin el cambio de nombre `as _`, el compilador nos avisaría de que tenemos una importación sin usar.
> A veces se puede ver `extern crate panic_abort` en su lugar, que es un estilo más antiguo utilizado antes de la
> edición 2018 de Rust, y ahora sólo debería usarse para crates "sysroot" (los que se distribuyen con el propio Rust) tales
> como `proc_macro`, `alloc`, `std`, y `test`.

## Un ejemplo

He aquí un ejemplo que intenta indexar un array más allá de su longitud. La operación resulta en un pánico.

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len() + 1;
    let _y = xs[i]; // out of bounds access

    loop {}
}
```

Este ejemplo elige el comportamiento `panic-semihosting` que imprime el mensaje de pánico en la consola del host usando semihosting.

``` text
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

Puedes probar a cambiar el comportamiento a `panic-halt` y confirmar que no se imprime ningún mensaje en ese caso.
