# Excepciones

Las excepciones, y las interrupciones, son un mecanismo de hardware mediante el cual el procesador gestiona eventos asíncronos y errores fatales (p.e. la ejecución de una instrucción no válida). Las excepciones implican prioridad e involucran manejadores de excepciones, subrutinas ejecutadas en respuesta a la señal que desencadenó el evento.

La _crate_ `cortex-m-rt` proporciona un atributo [`exception`] para declarar los manejadores de excepciones.

[`exception`]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.exception.html

```rust,ignore
// Exception handler for the SysTick (System Timer) exception
#[exception]
fn SysTick() {
    // ..
}
```

Aparte del atributo `exception` los manejadores de excepciones parecen funciones simples pero hay una diferencia más: Los manejadores de excepciones _no_ pueden ser llamados por software. Siguiendo el ejemplo anterior, la sentencia `SysTick();` produciría un error de compilación.

Este comportamiento es intencionado y es necesario para proporcionar una característica: las variables `static mut` declaradas _dentro_ de los manejadores de `exception` son _seguras_ de usar.

```rust,ignore
#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;

    // `COUNT` has transformed to type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

Como sabrás, usar variables `static mut` en una función la convierte en [_non-reentrant_](<https://en.wikipedia.org/wiki/Reentrancy_(computing)>). Es un comportamiento indefinido llamar a una función no-reentrante, directa o indirectamente, desde más de un manejador de excepciones / interrupciones o desde `main` y uno o más manejadores de excepciones / interrupciones.

Rust seguro nunca debe dar lugar a un comportamiento indefinido por lo que las funciones no reentrantes deben ser marcadas como `unsafe`. Sin embargo, me acaban de decir que los manejadores de `excepciones` pueden usar variables `static mut` de forma segura. ¿Cómo es esto posible? Esto es posible porque los manejadores de `excepción` no pueden ser llamados por software, por lo que la reentrada no es posible.

> Tenga en cuenta que el atributo `exception` transforma las definiciones de variables
> estáticas dentro de la función envolviéndolas en bloques "inseguros" y
> proporcionándonos con nuevas variables apropiadas de tipo `&mut` del mismo nombre.
> De este modo podemos derefenciar la referencia mediante `*` para acceder a los valores
> de las variables sin necesidad de envolverlas en bloques de tipo `unsafe`.

## Un ejemplo completo

Este es un ejemplo que utiliza el temporizador del sistema para lanzar una excepción `SysTick` aproximadamente cada segundo. El manejador de la excepción `SysTick` mantiene un registro de cuantas veces ha sido llamado en la variable `COUNT` y luego imprime el valor de `COUNT` en la consola del host usando semihosting.

> **NOTA**: Puedes ejecutar este ejemplo en cualquier dispositivo Cortex-M;
> también puedes ejecutarlo en QEMU

```rust,ignore
#![deny(unsafe_code)]
#![no_main]
#![no_std]

use panic_halt as _;

use core::fmt::Write;

use cortex_m::peripheral::syst::SystClkSource;
use cortex_m_rt::{entry, exception};
use cortex_m_semihosting::{
    debug,
    hio::{self, HStdout},
};

#[entry]
fn main() -> ! {
    let p = cortex_m::Peripherals::take().unwrap();
    let mut syst = p.SYST;

    // configures the system timer to trigger a SysTick exception every second
    syst.set_clock_source(SystClkSource::Core);
    // this is configured for the LM3S6965 which has a default CPU clock of 12 MHz
    syst.set_reload(12_000_000);
    syst.clear_current();
    syst.enable_counter();
    syst.enable_interrupt();

    loop {}
}

#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;
    static mut STDOUT: Option<HStdout> = None;

    *COUNT += 1;

    // Lazy initialization
    if STDOUT.is_none() {
        *STDOUT = hio::hstdout().ok();
    }

    if let Some(hstdout) = STDOUT.as_mut() {
        write!(hstdout, "{}", *COUNT).ok();
    }

    // IMPORTANT omit this `if` block if running on real hardware or your
    // debugger will end in an inconsistent state
    if *COUNT == 9 {
        // This will terminate the QEMU process
        debug::exit(debug::EXIT_SUCCESS);
    }
}
```

```console
tail -n5 Cargo.toml
```

```toml
[dependencies]
cortex-m = "0.5.7"
cortex-m-rt = "0.6.3"
panic-halt = "0.2.0"
cortex-m-semihosting = "0.3.1"
```

```text
$ cargo run --release
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
123456789
```

Si ejecutas esto en la tarjeta Discovery verás la salida en la consola OpenOCD. Además, el programa _no_ se detendrá cuando la cuenta llegue a 9.

## El manejador de excepciones por defecto

Lo que el atributo `exception` realmente hace es _anular_ el manejador de excepciones por defecto para una excepción específica. Si no anulas el manejador para una excepción en particular, será manejada por la función `DefaultHandler`, que por defecto es:

```rust,ignore
fn DefaultHandler() {
    loop {}
}
```

Esta función es proporcionada por la _crate_ `cortex-m-rt` y marcada como `#[no_mangle]` para que puedas poner un breakpoint en "DefaultHandler" y atrapar excepciones _sin manejador_.

Es posible sobreescribir este `DefaultHandler` usando el atributo `exception`:

```rust,ignore
#[exception]
fn DefaultHandler(irqn: i16) {
    // custom default handler
}
```

El argumento `irqn` indica qué excepción se está procesando. Un valor negativo indica que se está sirviendo una excepción Cortex-M; y cero o un valor positivo indican que se está sirviendo una excepción específica del dispositivo, también conocida como interrupción.

## El manejador de fallas graves

La excepción `HardFault` es un poco especial. Esta excepción se dispara cuando el programa entra en un estado inválido por lo que su manejador no puede _no_ retornar ya que podría resultar en un comportamiento indefinido. Además, la _crate runtime_ hace algún trabajo antes de que el manejador `HardFault` definido por el usuario sea invocado para mejorar la depuración.

El resultado es que el manejador `HardFault` debe tener la siguiente firma: `fn(&ExceptionFrame) -> !`. El argumento del manejador es un puntero a los registros que fueron empujados a la pila por la excepción. Estos registros son una instantánea del estado del procesador en el momento en que se produjo la excepción y son útiles para diagnosticar una falla grave.

He aquí un ejemplo que realiza una operación ilegal: una lectura a una posición de memoria inexistente.

> **NOTA**: Este programa no funcionará, es decir, no se bloqueará, en QEMU porque
> `qemu-system-arm -machine lm3s6965evb` no comprueba las cargas de memoria y
> en lecturas a memoria inválida.

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use core::fmt::Write;
use core::ptr;

use cortex_m_rt::{entry, exception, ExceptionFrame};
use cortex_m_semihosting::hio;

#[entry]
fn main() -> ! {
    // read a nonexistent memory location
    unsafe {
        ptr::read_volatile(0x3FFF_FFFE as *const u32);
    }

    loop {}
}

#[exception]
fn HardFault(ef: &ExceptionFrame) -> ! {
    if let Ok(mut hstdout) = hio::hstdout() {
        writeln!(hstdout, "{:#?}", ef).ok();
    }

    loop {}
}
```

El manejador `HardFault` imprime el valor `ExceptionFrame`. Si ejecutas esto verás algo como esto en la consola de OpenOCD.

```text
$ openocd
(..)
ExceptionFrame {
    r0: 0x3ffffffe,
    r1: 0x00f00000,
    r2: 0x20000000,
    r3: 0x00000000,
    r12: 0x00000000,
    lr: 0x080008f7,
    pc: 0x0800094a,
    xpsr: 0x61000000
}
```

El valor `pc` es el valor del Contador de Programa en el momento de la excepción y apunta a la instrucción que disparó la excepción.

Si miras el desensamblado del programa:

```text
$ cargo objdump --bin app --release -- -d --no-show-raw-insn --print-imm-hex
(..)
ResetTrampoline:
 8000942:       movw    r0, #0xfffe
 8000946:       movt    r0, #0x3fff
 800094a:       ldr     r0, [r0]
 800094c:       b       #-0x4 <ResetTrampoline+0xa>
```

Puedes consultar el valor del contador de programa `0x0800094a` en el desensamblado. Verás que una operación de carga (`ldr r0, [r0]` ) causó la excepción. El campo `r0` de `ExceptionFrame` te dirá que el valor del registro `r0` era `0x3fff_fffe` en ese momento.
