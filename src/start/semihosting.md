# Semihosting

Semihosting es un mecanismo que permite a los dispositivos embebidos realizar E/S en el anfitrión y se utiliza principalmente para registrar mensajes en la consola del anfitrión. Semihosting requiere una sesión de depuración y prácticamente nada más (¡sin cables adicionales!) por lo que es muy conveniente de usar. La desventaja es que es super lento: cada operación de escritura puede tardar varios milisegundos dependiendo del depurador de hardware (por ejemplo, ST-Link) que utilices.

La crate [`cortex-m-semihosting`] proporciona una API para realizar operaciones semihosting en dispositivos Cortex-M. El programa de abajo es la versión semihosting de "Hello, world!":

[`cortex-m-semihosting`]: https://crates.io/crates/cortex-m-semihosting

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    loop {}
}
```

Si ejecutas este programa en hardware verás el mensaje "Hello, world!" dentro de los logs de OpenOCD.

```text
$ openocd
(..)
Hello, world!
(..)
```

Es necesario activar primero el semihosting en OpenOCD desde GDB:

```console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

QEMU entiende las operaciones de semihosting por lo que el programa anterior también funcionará con `qemu-system-arm` sin tener que iniciar una sesión de depuración. Ten en cuenta que tendrás que pasar la bandera `-semihosting-config` a QEMU para habilitar el soporte de semihosting; estas banderas ya están incluidas en el archivo `.cargo/config.toml` de la plantilla.

```text
$ # this program will block the terminal
$ cargo run
     Running `qemu-system-arm (..)
Hello, world!
```

También hay una operación de semi-hosting `exit` que se puede utilizar para terminar el proceso QEMU. Importante: **no** uses `debug::exit` en hardware; esta función puede corromper tu sesión de OpenOCD y no podrás depurar más programas hasta que la reinicies.

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    if roses == "red" {
        debug::exit(debug::EXIT_SUCCESS);
    } else {
        debug::exit(debug::EXIT_FAILURE);
    }

    loop {}
}
```

```text
$ cargo run
     Running `qemu-system-arm (..)

$ echo $?
1
```

Un último consejo: puedes configurar el comportamiento de pánico a `exit(EXIT_FAILURE)`. Esto te permitirá escribir pruebas ejecutar-pasar `no_std` que puedes ejecutar en QEMU.

Para mayor comodidad, la crate `panic-semihosting` tiene una función "exit" que cuando está activada invoca `exit(EXIT_FAILURE)` después de registrar el mensaje de pánico en la stderr del host.

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _; // features = ["exit"]

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    assert_eq!(roses, "red");

    loop {}
}
```

```text
$ cargo run
     Running `qemu-system-arm (..)
panicked at 'assertion failed: `(left == right)`
  left: `"blue"`,
 right: `"red"`', examples/hello.rs:15:5

$ echo $?
1
```

**NOTA**: Para habilitar esta prestación en `panic-semihosting`, edite su sección de dependencias `Cargo.toml` donde se especifica `panic-semihosting` con:

```toml
panic-semihosting = { version = "VERSION", features = ["exit"] }
```

donde `VERSION` es la versión deseada. Para más información sobre las prestaciones de las dependencias, consulta la sección [`specifying dependencies`] del libro de Cargo.

[`specifying dependencies`]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
