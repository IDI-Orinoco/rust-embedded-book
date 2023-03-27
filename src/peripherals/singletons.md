# Singletons

> En ingeniería de software, el patrón singleton es un patrón de diseño de software que restringe la instanciación de una clase a un objeto.
>
> *Wikipedia: [Singleton Pattern], [Singleton](es)*

[Singleton Pattern]: https://en.wikipedia.org/wiki/Singleton_pattern
[Singleton]: https://es.wikipedia.org/wiki/Singleton

## Pero, ¿por qué no podemos simplemente usar variables globales?

Podríamos hacer que todo sea público estático, como esto:

```rust,ignore
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    };
}
```

Pero esto tiene algunos problemas. Es una variable global mutable, y en Rust, siempre es inseguro interactuar con estas. Estas variables también son visibles en todo su programa, lo que significa que el verificador de préstamos no puede ayudarte a rastrear las referencias y la propiedad de estas variables.

## ¿Cómo hacemos esto en Rust?

En lugar de convertir nuestro periférico en una variable global, podríamos decidir crear una estructura, en este caso denominada `PERIPHERALS`, que contiene una `Opción<T>` para cada uno de nuestros periféricos.

```rust,ignore
struct Peripherals {
    serial: Option<SerialPort>,
}
impl Peripherals {
    fn take_serial(&mut self) -> SerialPort {
        let p = replace(&mut self.serial, None);
        p.unwrap()
    }
}
static mut PERIPHERALS: Peripherals = Peripherals {
    serial: Some(SerialPort),
};
```

Esta estructura nos permite obtener una única instancia de nuestro periférico. Si intentamos llamar a `take_serial()` más de una vez, ¡nuestro código entrará en pánico!

```rust,ignore
fn main() {
    let serial_1 = unsafe { PERIPHERALS.take_serial() };
    // This panics!
    // let serial_2 = unsafe { PERIPHERALS.take_serial() };
}
```

Aunque interactuar con esta estructura es `unsafe`, una vez que tenemos el `SerialPort` que contiene, ya no necesitamos usar `unsafe`, o la estructura `PERIPHERALS` en absoluto.

Esto tiene una pequeña sobrecarga en la ejecución porque debemos envolver la estructura `SerialPort` en una opción, y necesitaremos llamar a `take_serial()` una vez, sin embargo, este pequeño costo inicial nos permite aprovechar el verificador de préstamos en el resto de nuestro programa.

## Soporte de biblioteca existente

Aunque creamos nuestra propia estructura `Peripherals` arriba, no es necesario hacer esto para su código. La _crate_ `cortex_m` contiene una macro llamada `singleton!()` que realizará esta acción por ti.

```rust,ignore
use cortex_m::singleton;

fn main() {
    // OK if `main` is executed only once
    let x: &'static mut bool =
        singleton!(: bool = false).unwrap();
}
```

[cortex_m docs](https://docs.rs/cortex-m/latest/cortex_m/macro.singleton.html)

Además, si usas [`cortex-m-rtic`](https://github.com/rtic-rs/cortex-m-rtic), todo el proceso de definición y obtención de estos periféricos es abstraído para ti, y tu, en cambio, recibes una estructura `Peripherals` que contiene una versión que es no-`Option<T>` de todos los elementos que definas.

```rust,ignore
// cortex-m-rtic v0.5.x
#[rtic::app(device = lm3s6965, peripherals = true)]
const APP: () = {
    #[init]
    fn init(cx: init::Context) {
        static mut X: u32 = 0;

        // Cortex-M peripherals
        let core: cortex_m::Peripherals = cx.core;

        // Device specific peripherals
        let device: lm3s6965::Peripherals = cx.device;
    }
}
```

## ¿Pero por qué?

Pero, ¿cómo estos Singleton marcan una diferencia notable en el funcionamiento de nuestro código Rust?

```rust,ignore
impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

    fn read_speed(
        &self // <------ This is really, really important
    ) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }
}
```

Hay dos factores importantes en juego aquí:

- Debido a que estamos usando un singleton, solo hay una forma o lugar para obtener una estructura `SerialPort`
- Para llamar al método `read_speed()`, debemos tener propiedad o una referencia a una estructura `SerialPort`

Estos dos factores juntos significan que solo es posible acceder al hardware si hemos satisfecho adecuadamente el verificador de préstamos, lo que significa que en ningún momento tenemos múltiples referencias mutables al mismo hardware.

```rust,ignore
fn main() {
    // missing reference to `self`! Won't work.
    // SerialPort::read_speed();

    let serial_1 = unsafe { PERIPHERALS.take_serial() };

    // you can only read what you have access to
    let _ = serial_1.read_speed();
}
```

## Trate su hardware como si fueran datos

Además, dado que algunas referencias son mutables y otras inmutables, es posible ver si una función o método podría modificar potencialmente el estado del hardware. Por ejemplo,

Esto está permitido para cambiar la configuración del hardware:

```rust,ignore
fn setup_spi_port(
    spi: &mut SpiPort,
    cs_pin: &mut GpioPin
) -> Result<()> {
    // ...
}
```

Esto no está permitido:

```rust,ignore
fn read_button(gpio: &GpioPin) -> bool {
    // ...
}
```

Esto nos permite imponer si el código debe o no realizar cambios en el hardware en **tiempo de compilación**, en lugar de en tiempo de ejecución. Como nota, esto generalmente solo funciona en una aplicación, pero para los sistemas completos, nuestro software se compilará en una sola aplicación, por lo que esto no suele ser una restricción.
