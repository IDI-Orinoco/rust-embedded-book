# Registros Mapeados en Memoria

Los sistemas embebidos sólo pueden llegar hasta cierto punto ejecutando código Rust normal y moviendo datos en la RAM. Si queremos obtener cualquier información dentro o fuera de nuestro sistema (ya sea parpadeando un LED, detectando la pulsación de un botón o comunicándonos con un periférico fuera del chip en algún tipo de bus) vamos a tener que sumergirnos en el mundo de los periféricos y sus 'registros mapeados en memoria'.

Es muy posible que el código que necesitas para acceder a los periféricos de tu microcontrolador ya esté escrito, en alguno de los siguientes niveles:

<p align="center">
<img title="Common crates" src="../assets/crates.png">
</p>

- Micro-architecture Crate - Este tipo de _crate_ maneja cualquier rutina útil común al núcleo del procesador que tu microcontrolador está utilizando, así como cualquier periférico que sea común a todos los microcontroladores que utilizan ese tipo particular de núcleo de procesador. Por ejemplo, la _crate_ [cortex-m] le proporciona funciones para activar y desactivar interrupciones, que son las mismas para todos los microcontroladores basados en Cortex-M. También le da acceso al periférico 'SysTick' incluido con todos los microcontroladores basados en Cortex-M.
- Peripheral Access Crate (PAC) - Este tipo de _crate_ es una fina envoltura sobre los diversos registros de memoria definidos para el número de parte del micro-controlador que está utilizando. Por ejemplo, [tm4c123x] para la serie Texas Instruments Tiva-C TM4C123, o [stm32f30x] para la serie ST-Micro STM32F30x. Aquí, interactuarás con los registros directamente, siguiendo las instrucciones de funcionamiento de cada periférico dadas en el Manual de Referencia Técnica de tu micro-controlador.
- HAL Crate - Estas _crates_ ofrecen una API más amigable para tu procesador en particular, a menudo implementando algunos _traits_ comunes definidos en [embedded-hal]. Por ejemplo, esta _crate_ podría ofrecer una estructura `Serial`, con un constructor que toma un conjunto apropiado de pines GPIO y una tasa de baudios, y ofrece algún tipo de función `write_byte` para enviar datos. Ver el capítulo sobre [Portabilidad] para más información sobre [embedded-hal].
- Board Crate - Estas _crates_ van un paso más allá que una HAL _Crate_ preconfigurando varios periféricos y pines GPIO para adaptarse al kit de desarrollo específico o tarjeta que esté utilizando, como [stm32f3-discovery] para la tarjeta STM32F3DISCOVERY.

[cortex-m]: https://crates.io/crates/cortex-m
[tm4c123x]: https://crates.io/crates/tm4c123x
[stm32f30x]: https://crates.io/crates/stm32f30x
[embedded-hal]: https://crates.io/crates/embedded-hal
[portability]: ../portability/index.md
[stm32f3-discovery]: https://crates.io/crates/stm32f3-discovery
[discovery]: https://rust-embedded.github.io/discovery/

## Board Crate

Una _board crate_ o crate de tarjeta es el punto de partida perfecto, si eres nuevo en Rust embebido. Abstraen muy bien los detalles de HW que pueden ser abrumadores cuando se empieza a estudiar este tema, y hace que las tareas estándar sean fáciles, como encender o apagar un LED. La funcionalidad que expone varía mucho entre tarjetas. Dado que este libro tiene como objetivo permanecer agnóstico al hardware, las _board crates_ no serán cubiertos por este libro.

Si quieres experimentar con la tarjeta STM32F3DISCOVERY, es muy recomendable echar un vistazo a la caja de la tarjeta [stm32f3-discovery], que proporciona funcionalidad para hacer parpadear los LEDs de la tarjeta, acceder a su brújula, bluetooth y mucho más. El libro [Discovery] ofrece una gran introducción al uso de una board crate.

Pero si estás trabajando en un sistema que todavía no tiene una board crate dedicado, o necesitas una funcionalidad que no proporcionan las _crates_ existentes, sigue leyendo mientras empezamos desde abajo, con las _crates_ de micro-arquitectura.

## Micro-architecture crate

Veamos el periférico SysTick que es común a todos los microcontroladores basados en Cortex-M. Podemos encontrar una API de muy bajo nivel en la [cortex-m] crate, y podemos usarlo así:

```rust,ignore
#![no_std]
#![no_main]
use cortex_m::peripheral::{syst, Peripherals};
use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let peripherals = Peripherals::take().unwrap();
    let mut systick = peripherals.SYST;
    systick.set_clock_source(syst::SystClkSource::Core);
    systick.set_reload(1_000);
    systick.clear_current();
    systick.enable_counter();
    while !systick.has_wrapped() {
        // Loop
    }

    loop {}
}
```

Las funciones de la estructura `SYST` se asemejan bastante a la funcionalidad definida en el Manual de Referencia Técnica de ARM para este periférico. No hay nada en esta API sobre 'retrasar X milisegundos' - tenemos que implementarlo nosotros mismos usando un bucle `while`. Ten en cuenta que no podemos acceder a nuestra estructura `SYST` hasta que hayamos llamado a `Peripherals::take()` - esta es una rutina especial que garantiza que sólo hay una estructura `SYST` en todo nuestro programa. Para más información, consulta la sección [Periféricos].

[periféricos]: ../peripherals/index.md

## Usando una _Peripheral Access Crate_ (PAC) o una _Crate_ de Acceso a Periféricos 

No llegaremos muy lejos con nuestro desarrollo de software embebido si nos limitamos sólo a los periféricos básicos incluidos con cada Cortex-M. En algún momento, vamos a necesitar escribir algún código que sea específico para el micro-controlador en particular que estamos utilizando. En este ejemplo, vamos a suponer que tenemos un Texas Instruments TM4C123 - un Cortex-M4 medio de 80MHz con 256 KiB de Flash. Vamos a utilizar la _crate_ [tm4c123x] para hacer uso de este chip.

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _; // panic handler

use cortex_m_rt::entry;
use tm4c123x;

#[entry]
pub fn init() -> (Delay, Leds) {
    let cp = cortex_m::Peripherals::take().unwrap();
    let p = tm4c123x::Peripherals::take().unwrap();

    let pwm = p.PWM0;
    pwm.ctl.write(|w| w.globalsync0().clear_bit());
    // Mode = 1 => Count up/down mode
    pwm._2_ctl.write(|w| w.enable().set_bit().mode().set_bit());
    pwm._2_gena.write(|w| w.actcmpau().zero().actcmpad().one());
    // 528 cycles (264 up and down) = 4 loops per video line (2112 cycles)
    pwm._2_load.write(|w| unsafe { w.load().bits(263) });
    pwm._2_cmpa.write(|w| unsafe { w.compa().bits(64) });
    pwm.enable.write(|w| w.pwm4en().set_bit());
}

```

Hemos accedido al periférico `PWM0` exactamente de la misma forma que antes accedimos al periférico `SYST`, excepto que hemos llamado a `tm4c123x::Peripherals::take()`. Como esta crate fue auto-generada usando [svd2rust], las funciones de acceso para nuestros campos de registro toman una _closure_, en lugar de un argumento numérico. Aunque esto parece un montón de código, el compilador de Rust puede utilizarlo para realizar un montón de comprobaciones por nosotros, ¡pero luego genera código máquina que es bastante parecido al ensamblador escrito a mano! Cuando el código autogenerado no es capaz de determinar que todos los posibles argumentos de una función accesoria en particular son válidos (por ejemplo, si el SVD define el registro como de 32 bits, pero no dice si algunos de esos valores de 32 bits tienen un significado especial), entonces la función se marca como `unsafe`. Podemos ver esto en el ejemplo anterior cuando se establecen los subcampos `load` y `compa` usando la función `bits()`.

### Lectura

La función `read()` devuelve un objeto que da acceso de sólo lectura a los distintos subcampos dentro de este registro, tal y como se define en el archivo SVD del fabricante para este chip. Puede encontrar todas las funciones disponibles en el tipo de retorno especial `R` para este registro en particular, en este periférico en particular, en este chip en particular, en la [documentación tm4c123x][tm4c123x documentation R].

```rust,ignore
if pwm.ctl.read().globalsync0().is_set() {
    // Do a thing
}
```

### Escritura

La función `write()` toma una _closure_ con un único argumento. Típicamente lo llamamos `w`. Este argumento da acceso de lectura-escritura a varios subcampos dentro de este registro, como se define en el archivo SVD del fabricante para este chip. De nuevo, puedes encontrar todas las funciones disponibles en `w` para este registro en particular, en este periférico en particular, en este chip en particular, en la [documentación tm4c123x][tm4c123x documentation W]. Tenga en cuenta que todos los subcampos que no establezcamos se establecerán a un valor por defecto para nosotros - cualquier contenido existente en el registro se perderá.

```rust,ignore
pwm.ctl.write(|w| w.globalsync0().clear_bit());
```

### Modificación

Si deseamos cambiar sólo un subcampo concreto de este registro y dejar los demás subcampos sin cambios, podemos utilizar la función `modify`. Esta función toma un cierre con dos argumentos - uno para lectura y otro para escritura. Normalmente los llamamos `r` y `w` respectivamente. El argumento `r` se puede utilizar para inspeccionar el contenido actual del registro, y el argumento `w` se puede utilizar para modificar el contenido del registro.

```rust,ignore
pwm.ctl.modify(|r, w| w.globalsync0().clear_bit());
```

La función `modify` muestra realmente el poder de las _closures_. En C, tendríamos que leer un valor temporal, modificar los bits correctos y volver a escribir el valor. Esto significa que hay un margen de error considerable:

```C
uint32_t temp = pwm0.ctl.read();
temp |= PWM0_CTL_GLOBALSYNC0;
pwm0.ctl.write(temp);
uint32_t temp2 = pwm0.enable.read();
temp2 |= PWM0_ENABLE_PWM4EN;
pwm0.enable.write(temp); // Oh oh! Variable equivocada!
```

[svd2rust]: https://crates.io/crates/svd2rust
[tm4c123x documentation r]: https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.R.html
[tm4c123x documentation w]: https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.W.html

## Usando una _crate_ HAL

La _crate_ HAL para un chip funciona típicamente implementando un _Trait_ personalizado para las estructuras crudas expuestas por la PAC. A menudo este _trait_ definirá una función llamada `constrain()` para periféricos simples o `split()` para cosas como puertos GPIO con múltiples pines. Esta función consumirá la estructura subyacente del periférico y devolverá un nuevo objeto con una API de alto nivel. Esta API también puede hacer cosas como que la función `new` del puerto serie requiera un préstamo en alguna estructura `Clock`, que sólo puede ser generada llamando a la función que configura los PLLs y establece todas las frecuencias de reloj. De esta forma, es estáticamente imposible crear un objeto puerto serie sin haber configurado antes las frecuencias de reloj, o que el objeto puerto serie convierta erróneamente la velocidad de transmisión en ticks de reloj. Algunos crates incluso definen rasgos especiales para los estados en los que puede estar cada pin GPIO, requiriendo que el usuario ponga un pin en el estado correcto (digamos, seleccionando el Modo de Función Alternativo apropiado) antes de pasar el pin a Periférico. Todo ello sin costo alguno en tiempo de ejecución.

Veamos un ejemplo:

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _; // panic handler

use cortex_m_rt::entry;
use tm4c123x_hal as hal;
use tm4c123x_hal::prelude::*;
use tm4c123x_hal::serial::{NewlineMode, Serial};
use tm4c123x_hal::sysctl;

#[entry]
fn main() -> ! {
    let p = hal::Peripherals::take().unwrap();
    let cp = hal::CorePeripherals::take().unwrap();

    // Wrap up the SYSCTL struct into an object with a higher-layer API
    let mut sc = p.SYSCTL.constrain();
    // Pick our oscillation settings
    sc.clock_setup.oscillator = sysctl::Oscillator::Main(
        sysctl::CrystalFrequency::_16mhz,
        sysctl::SystemClock::UsePll(sysctl::PllOutputFrequency::_80_00mhz),
    );
    // Configure the PLL with those settings
    let clocks = sc.clock_setup.freeze();

    // Wrap up the GPIO_PORTA struct into an object with a higher-layer API.
    // Note it needs to borrow `sc.power_control` so it can power up the GPIO
    // peripheral automatically.
    let mut porta = p.GPIO_PORTA.split(&sc.power_control);

    // Activate the UART.
    let uart = Serial::uart0(
        p.UART0,
        // The transmit pin
        porta
            .pa1
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // The receive pin
        porta
            .pa0
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // No RTS or CTS required
        (),
        (),
        // The baud rate
        115200_u32.bps(),
        // Output handling
        NewlineMode::SwapLFtoCRLF,
        // We need the clock rates to calculate the baud rate divisors
        &clocks,
        // We need this to power up the UART peripheral
        &sc.power_control,
    );

    loop {
        writeln!(uart, "Hello, World!\r\n").unwrap();
    }
}
```
