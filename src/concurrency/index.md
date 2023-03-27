# Concurrencia

La concurrencia se produce cuando diferentes partes del programa se ejecutan en momentos diferentes o fuera de orden. En un contexto embebido, esto incluye:

* Manejadores de interrupción, que se ejecutan cada vez que se produce la interrupción asociada,
* varias formas de multihilo, en las que el microprocesador intercambia regularmente partes del programa,
* y, en algunos sistemas, microprocesadores multinúcleo, en los que cada núcleo puede ejecutar de forma independiente una parte diferente del programa al mismo tiempo.

Dado que muchos programas embebidos necesitan lidiar con interrupciones, la concurrencia aparecerá tarde o temprano, y es también donde muchos errores sutiles y difíciles pueden ocurrir. Afortunadamente, Rust proporciona una serie de abstracciones y garantías de seguridad para ayudarnos a escribir código correcto.

## Sin concurrencia

La concurrencia más simple para un programa embebido es la no concurrencia: tu software consiste en un único bucle principal que sigue corriendo, y no hay interrupciones en absoluto. A veces esto es perfectamente adecuado para el problema en cuestión. Normalmente, el bucle leerá algunas entradas, realizará algún procesamiento y escribirá algunas salidas.

```rust,ignore
#[entry]
fn main() {
    let peripherals = setup_peripherals();
    loop {
        let inputs = read_inputs(&peripherals);
        let outputs = process(inputs);
        write_outputs(&peripherals, outputs);
    }
}
```

Como no hay concurrencia, no hay necesidad de preocuparse por compartir datos entre partes de tu programa o sincronizar el acceso a los periféricos. Si puedes salirte con la tuya con un enfoque tan simple esta puede ser una gran solución.

## Datos Mutables Globales

A diferencia del Rust no embebido, normalmente no tendremos el lujo de crear asignaciones de heap y pasar referencias a esos datos a un hilo recién creado. En su lugar, nuestros manejadores de interrupciones pueden ser llamados en cualquier momento y deben saber cómo acceder a cualquier memoria compartida que estemos utilizando. En el nivel más bajo, esto significa que debemos tener memoria mutable _asignada estáticamente_, a la que tanto el manejador de interrupciones como el código principal puedan referirse.

En Rust, tales variables [`static mut`] son siempre inseguras para leer o escribir, porque sin tener especial cuidado, podrías provocar una condición de carrera, donde tu acceso a la variable es interrumpido a mitad de camino por una interrupción que también accede a esa variable.

[`static mut`]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

Para un ejemplo de cómo este comportamiento puede causar errores sutiles en tu código, considera un programa embebido que cuenta los flancos ascendentes de alguna señal de entrada en cada periodo de un segundo (un contador de frecuencia):

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // DANGER - Not actually safe! Could cause data races.
            unsafe { COUNTER += 1 };
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

Cada segundo, la interrupción del temporizador vuelve a poner el contador a 0. Mientras tanto, el bucle principal mide continuamente la señal, e incrementa el contador cuando ve un cambio de bajo a alto. Hemos tenido que usar `unsafe` para acceder a `COUNTER`, ya que es `static mut`, y eso significa que estamos prometiendo al compilador que no causaremos ningún comportamiento indefinido. ¿Puedes detectar la condición de carrera? El incremento en `COUNTER` _no_ está garantizado que sea atómico - de hecho, en la mayoría de las plataformas embebidas, se dividirá en una carga, luego el incremento, y luego un almacenamiento. Si la interrupción se disparara después de la carga pero antes del almacenamiento, el reset a 0 sería ignorado después de que la interrupción volviera - y contaríamos el doble de transiciones para ese periodo.

## Secciones críticas

Entonces, ¿qué podemos hacer con las carreras de datos? Un enfoque sencillo es utilizar _secciones críticas_, un contexto en el que las interrupciones están deshabilitadas. Envolviendo el acceso a `COUNTER` en `main` en una sección crítica, podemos estar seguros de que la interrupción del temporizador no se disparará hasta que hayamos terminado de incrementar `COUNTER`:

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // New critical section ensures synchronised access to COUNTER
            cortex_m::interrupt::free(|_| {
                unsafe { COUNTER += 1 };
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

En este ejemplo, usamos `cortex_m::interrupt::free`, pero otras plataformas tendrán mecanismos similares para ejecutar código en una sección crítica. Esto también es lo mismo que desactivar las interrupciones, ejecutar algo de código, y luego volver a activar las interrupciones.

Observa que no necesitamos poner una sección crítica dentro de la interrupción del temporizador, por dos razones:

  * Escribir 0 en `COUNTER` no puede verse afectado por una carrera ya que no lo leemos.
  * De todas formas nunca será interrumpido por el hilo `main

Si `COUNTER` estuviera siendo compartido por múltiples manejadores de interrupción que pudieran _prevenir_ a cada uno, entonces cada uno podría requerir una sección crítica también.

Esto resuelve nuestro problema inmediato, pero seguimos escribiendo mucho código inseguro sobre el que tenemos que razonar cuidadosamente, y podríamos estar usando secciones críticas innecesariamente. Dado que cada sección crítica pausa temporalmente el procesamiento de interrupciones, hay un costo asociado de algún tamaño de código extra y mayor latencia de interrupción y jitter (las interrupciones pueden tardar más en ser procesadas, y el tiempo hasta que son procesadas será más variable). Que esto sea un problema depende de tu sistema, pero en general, nos gustaría evitarlo.

Vale la pena señalar que, si bien una sección crítica garantiza que no se produzcan interrupciones, ¡no proporciona una garantía de exclusividad en sistemas multinúcleo!  El otro núcleo podría estar accediendo felizmente a la misma memoria que tu núcleo, incluso sin interrupciones. Necesitarás primitivas de sincronización más fuertes si estás usando múltiples núcleos.

## Acceso atómico

En algunas plataformas existen instrucciones atómicas especiales que garantizan las operaciones de lectura-modificación-escritura. Específicamente para Cortex-M: `thumbv6` (Cortex-M0, Cortex-M0+) sólo proporciona instrucciones atómicas de carga y almacenamiento, mientras que `thumbv7` (Cortex-M3 y superiores) proporciona instrucciones completas de Comparación e Intercambio (CAS). Estas instrucciones CAS ofrecen una alternativa a la pesada deshabilitación de todas las interrupciones: podemos intentar el incremento, que tendrá éxito la mayoría de las veces, pero si se interrumpe se reintentará automáticamente toda la operación de incremento. Estas operaciones atómicas son seguras incluso en múltiples núcleos.

```rust,ignore
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // Use `fetch_add` to atomically add 1 to COUNTER
            COUNTER.fetch_add(1, Ordering::Relaxed);
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // Use `store` to write 0 directly to COUNTER
    COUNTER.store(0, Ordering::Relaxed)
}
```

Esta vez `COUNTER` es una variable `static` segura. Gracias al tipo `AtomicUsize` `COUNTER` puede modificarse de forma segura tanto desde el manejador de interrupciones como desde el hilo principal sin desactivar las interrupciones. Cuando sea posible, esta es la mejor solución - pero puede que no esté soportada en tu plataforma.

Una nota sobre [`Ordering`]: esto afecta a cómo el compilador y el hardware pueden reordenar las instrucciones, y también tiene consecuencias sobre la visibilidad de la caché. Asumiendo que el objetivo es una plataforma mononúcleo, `Relaxed` es suficiente y la opción más eficiente en este caso particular. Un ordenamiento más estricto hará que el compilador emita barreras de memoria alrededor de las operaciones atómicas; ¡dependiendo de para qué uses las atómicas puedes necesitarlo o no! Los detalles precisos del modelo atómico son complicados y es mejor describirlos en otro lugar.

Para más detalles sobre atómicos y ordenación, vea el [nomicon].

[`Ordering`]: https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html
[nomicon]: https://doc.rust-lang.org/nomicon/atomics.html


## Abstracciones, envío y sincronización

Ninguna de las soluciones anteriores es especialmente satisfactoria. Requieren bloques "inseguros" que deben ser comprobados cuidadosamente y no son ergonómicos. ¡Seguramente podemos hacerlo mejor en Rust!

Podemos abstraer nuestro contador en una interfaz segura que pueda ser utilizada con seguridad en cualquier otra parte de nuestro código. Para este ejemplo, usaremos el contador de sección crítica, pero podrías hacer algo muy similar con atomics.

```rust,ignore
use core::cell::UnsafeCell;
use cortex_m::interrupt;

// Our counter is just a wrapper around UnsafeCell<u32>, which is the heart
// of interior mutability in Rust. By using interior mutability, we can have
// COUNTER be `static` instead of `static mut`, but still able to mutate
// its counter value.
struct CSCounter(UnsafeCell<u32>);

const CS_COUNTER_INIT: CSCounter = CSCounter(UnsafeCell::new(0));

impl CSCounter {
    pub fn reset(&self, _cs: &interrupt::CriticalSection) {
        // By requiring a CriticalSection be passed in, we know we must
        // be operating inside a CriticalSection, and so can confidently
        // use this unsafe block (required to call UnsafeCell::get).
        unsafe { *self.0.get() = 0 };
    }

    pub fn increment(&self, _cs: &interrupt::CriticalSection) {
        unsafe { *self.0.get() += 1 };
    }
}

// Required to allow static CSCounter. See explanation below.
unsafe impl Sync for CSCounter {}

// COUNTER is no longer `mut` as it uses interior mutability;
// therefore it also no longer requires unsafe blocks to access.
static COUNTER: CSCounter = CS_COUNTER_INIT;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // No unsafe here!
            interrupt::free(|cs| COUNTER.increment(cs));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We do need to enter a critical section here just to obtain a valid
    // cs token, even though we know no other interrupt could pre-empt
    // this one.
    interrupt::free(|cs| COUNTER.reset(cs));

    // We could use unsafe code to generate a fake CriticalSection if we
    // really wanted to, avoiding the overhead:
    // let cs = unsafe { interrupt::CriticalSection::new() };
}
```

Hemos movido nuestro código inseguro (`unsafe`) al interior de nuestra abstracción cuidadosamente planificada, y ahora el código de nuestra aplicación no contiene ningún bloque inseguro (`unsafe`).

Este diseño requiere que la aplicación pase un token `CriticalSection`: estos tokens sólo son generados de forma segura por `interrupt::free`, así que al requerir que se pase uno, nos aseguramos de que estamos operando dentro de una sección crítica, sin tener que hacer el bloqueo nosotros mismos. Esta garantía la proporciona estáticamente el compilador: no habrá ninguna sobrecarga en tiempo de ejecución asociada a `cs`. Si tuviéramos múltiples contadores, a todos se les podría dar el mismo `cs`, sin necesidad de múltiples secciones críticas anidadas.

Esto también trae a colación un tema importante para la concurrencia en Rust: los _traits_ [`Send` y `Sync`]. Para resumir el libro de Rust, un tipo es Send cuando puede ser movido con seguridad a otro hilo, mientras que es Sync cuando puede ser compartido con seguridad entre múltiples hilos. En un contexto embebido, consideramos que las interrupciones se ejecutan en un hilo separado del código de la aplicación, por lo que las variables accedidas tanto por una interrupción como por el código principal deben ser Sync.

[`Send` y `Sync`]: https://doc.rust-lang.org/nomicon/send-and-sync.html

Para la mayoría de los tipos en Rust, ambos _traits_ se derivan automáticamente por el compilador. Sin embargo, debido a que `CSCounter` contiene una [`UnsafeCell`], no es Sync, y por lo tanto no podríamos hacer un `static CSCounter`: las variables `static` _deben_ ser Sync, ya que pueden ser accedidas por múltiples hilos.

[`UnsafeCell`]: https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html

Para indicar al compilador que nos hemos asegurado de que el `CSCounter` es seguro para compartir entre hilos, implementamos el rasgo Sync explícitamente. Al igual que con el uso anterior de secciones críticas, esto sólo es seguro en plataformas mononúcleo: con múltiples núcleos, tendrías que ir más lejos para garantizar la seguridad.

## Mutexes

Hemos creado una abstracción útil específica para nuestro problema de contadores, pero hay muchas abstracciones comunes utilizadas para la concurrencia.

Una de estas _primitivas de sincronización_ es un mutex, abreviatura de exclusión mutua. Estas construcciones aseguran el acceso exclusivo a una variable, como nuestro contador. Un hilo puede intentar _bloquear_ (o _adquirir_) el mutex, y o bien lo consigue inmediatamente, o se bloquea esperando a que se adquiera el bloqueo, o devuelve un error de que el mutex no se ha podido bloquear. Mientras ese hilo mantiene el bloqueo, se le concede acceso a los datos protegidos. Cuando el hilo termina, _desbloquea_ (o _libera_) el mutex, permitiendo que otro hilo lo bloquee. En Rust, normalmente implementaríamos el desbloqueo usando el rasgo [`Drop`] para asegurarnos de que siempre se libera cuando el mutex sale del ámbito.

[`Drop`]: https://doc.rust-lang.org/core/ops/trait.Drop.html

Usar un mutex con manejadores de interrupciones puede ser complicado: normalmente no es aceptable que el manejador de interrupciones se bloquee, y sería especialmente desastroso que se bloqueara esperando a que el hilo principal liberara un bloqueo, ya que entonces nos _deadlock_ (el hilo principal nunca liberará el bloqueo porque la ejecución permanece en el manejador de interrupciones). El deadlock no se considera inseguro: es posible incluso en Rust seguro.

Para evitar completamente este comportamiento, podríamos implementar un mutex que requiera una sección crítica para bloquearse, como en nuestro ejemplo del contador. Mientras la sección crítica dure lo mismo que el bloqueo, podemos estar seguros de que tenemos acceso exclusivo a la variable envuelta sin necesidad de seguir el estado de bloqueo/desbloqueo del mutex.

De hecho, ¡esto lo hace por nosotros la _crate_ `cortex_m`! Podríamos haber escrito nuestro contador usándolo:

```rust,ignore
use core::cell::Cell;
use cortex_m::interrupt::Mutex;

static COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            interrupt::free(|cs|
                COUNTER.borrow(cs).set(COUNTER.borrow(cs).get() + 1));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We still need to enter a critical section here to satisfy the Mutex.
    interrupt::free(|cs| COUNTER.borrow(cs).set(0));
}
```

Ahora estamos utilizando [`Cell`], que junto con su hermano `RefCell` se utiliza para proporcionar mutabilidad interior segura. Ya hemos visto `UnsafeCell` que es la capa inferior de mutabilidad interior en Rust: te permite obtener múltiples referencias mutables a su valor, pero sólo con código inseguro. Una `Cell` es como una `UnsafeCell` pero proporciona una interfaz segura: sólo permite tomar una copia del valor actual o reemplazarlo, no tomar una referencia, y como no es Sync, no puede ser compartida entre hilos. Estas restricciones hacen que su uso sea seguro, pero no podríamos usarlo directamente en una variable `static` ya que una `static` debe ser Sync.

[`Cell`]: https://doc.rust-lang.org/core/cell/struct.Cell.html

¿Por qué funciona el ejemplo anterior? El `Mutex<T>` implementa Sync para cualquier `T` que sea Send - como una `Cell`. Puede hacer esto de forma segura porque sólo da acceso a su contenido durante una sección crítica. Por lo tanto, podemos obtener un contador seguro sin código inseguro.

Esto es genial para tipos simples como el `u32` de nuestro contador, pero ¿qué pasa con tipos más complejos que no son Copy? Un ejemplo extremadamente común en un contexto embebido es una estructura periférica, que generalmente no es Copy. Para eso, podemos recurrir a `RefCell`.

## Compartiendo Periféricos

Las _crates_ de dispositivos generadas usando `svd2rust` y abstracciones similares proporcionan un acceso seguro a los periféricos al imponer que sólo una instancia de la estructura periférica puede existir a la vez. Esto garantiza la seguridad, pero dificulta el acceso a un periférico tanto desde el hilo principal como desde un manejador de interrupciones.

Para compartir de forma segura el acceso a los periféricos, podemos utilizar el `Mutex` que vimos antes. También necesitaremos usar [`RefCell`], que utiliza una comprobación en tiempo de ejecución para asegurar que sólo se da una referencia a un periférico cada vez. Esto tiene más sobrecarga que el simple `Cell`, pero ya que estamos dando referencias en lugar de copias, debemos estar seguros de que sólo existe una a la vez.

[`RefCell`]: https://doc.rust-lang.org/core/cell/struct.RefCell.html

Por último, también tendremos que tener en cuenta la forma de mover el periférico a la variable compartida después de que se haya inicializado en el código principal. Para ello podemos utilizar el tipo `Option`, inicializado a `None` y posteriormente establecido a la instancia del periférico.

```rust,ignore
use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};
use stm32f4::stm32f405;

static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // Obtain the peripheral singletons and configure it.
    // This example is from an svd2rust-generated crate, but
    // most embedded device crates will be similar.
    let dp = stm32f405::Peripherals::take().unwrap();
    let gpioa = &dp.GPIOA;

    // Some sort of configuration function.
    // Assume it sets PA0 to an input and PA1 to an output.
    configure_gpio(gpioa);

    // Store the GPIOA in the mutex, moving it.
    interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
    // We can no longer use `gpioa` or `dp.GPIOA`, and instead have to
    // access it via the mutex.

    // Be careful to enable the interrupt only after setting MY_GPIO:
    // otherwise the interrupt might fire while it still contains None,
    // and as-written (with `unwrap()`), it would panic.
    set_timer_1hz();
    let mut last_state = false;
    loop {
        // We'll now read state as a digital input, via the mutex
        let state = interrupt::free(|cs| {
            let gpioa = MY_GPIO.borrow(cs).borrow();
            gpioa.as_ref().unwrap().idr.read().idr0().bit_is_set()
        });

        if state && !last_state {
            // Set PA1 high if we've seen a rising edge on PA0.
            interrupt::free(|cs| {
                let gpioa = MY_GPIO.borrow(cs).borrow();
                gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // This time in the interrupt we'll just clear PA0.
    interrupt::free(|cs| {
        // We can use `unwrap()` because we know the interrupt wasn't enabled
        // until after MY_GPIO was set; otherwise we should handle the potential
        // for a None value.
        let gpioa = MY_GPIO.borrow(cs).borrow();
        gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().clear_bit());
    });
}
```

Es mucho para asimilar, así que vamos a desglosar las líneas importantes.

```rust,ignore
static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));
```

Nuestra variable compartida es ahora un `Mutex` alrededor de una `RefCell` que contiene una `Option`. El `Mutex` asegura que sólo tenemos acceso durante una sección crítica, y por lo tanto hace que la variable sea Sync, a pesar de que una `RefCell` normal no sería Sync. La `RefCell` nos da mutabilidad interior con referencias, que necesitaremos para usar nuestro `GPIOA`. La `Option` nos permite inicializar esta variable a algo vacío, y sólo después realmente mover la variable. No podemos acceder al singleton periférico estáticamente, sólo en tiempo de ejecución, así que esto es necesario.

```rust,ignore
interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
```

Dentro de una sección crítica podemos llamar a `borrow()` en el mutex, que nos da una referencia a la `RefCell`. Luego llamamos a `replace()` para mover nuestro nuevo valor a la `RefCell`.

```rust,ignore
interrupt::free(|cs| {
    let gpioa = MY_GPIO.borrow(cs).borrow();
    gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
});
```

Por último, utilizamos `MY_GPIO` de forma segura y concurrente. La sección crítica evita que la interrupción se dispare como de costumbre, y nos permite tomar prestado el mutex. La `RefCell` nos da una `&Option<GPIOA>`, y controla cuánto tiempo permanece prestada - una vez que la referencia sale del ámbito, la `RefCell` se actualiza para indicar que ya no está prestada.

Como no podemos mover el `GPIOA` fuera de `&Option`, tenemos que convertirlo en un `&Option<&GPIOA>` con `as_ref()`, que finalmente podemos `unwrap()` para obtener el `&GPIOA` que nos permite modificar el periférico.

Si necesitamos una referencia mutable a un recurso compartido, entonces debemos usar `borrow_mut` y `deref_mut` en su lugar. El siguiente código muestra un ejemplo utilizando el temporizador TIM2.

```rust,ignore
use core::cell::RefCell;
use core::ops::DerefMut;
use cortex_m::interrupt::{self, Mutex};
use cortex_m::asm::wfi;
use stm32f4::stm32f405;

static G_TIM: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> =
	Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let mut cp = cm::Peripherals::take().unwrap();
    let dp = stm32f405::Peripherals::take().unwrap();

    // Some sort of timer configuration function.
    // Assume it configures the TIM2 timer, its NVIC interrupt,
    // and finally starts the timer.
    let tim = configure_timer_interrupt(&mut cp, dp);

    interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(tim));
    });

    loop {
        wfi();
    }
}

#[interrupt]
fn timer() {
    interrupt::free(|cs| {
        if let Some(ref mut tim)) =  G_TIM.borrow(cs).borrow_mut().deref_mut() {
            tim.start(1.hz());
        }
    });
}

```

¡Uf! Esto es seguro, pero también es un poco difícil de manejar. ¿Hay algo más que podamos hacer?

## RTIC

Una alternativa es el [RTIC framework], abreviatura de Real Time Interrupt-driven Concurrency. Refuerza las prioridades estáticas y rastrea los accesos a variables `static mut` ("recursos") para asegurar estáticamente que siempre se accede a los recursos compartidos de forma segura, sin requerir la sobrecarga de entrar siempre en secciones críticas y usar el conteo de referencias (como en `RefCell`). Esto tiene una serie de ventajas, como garantizar que no se produzcan bloqueos y ofrecer una sobrecarga de tiempo y memoria extremadamente baja.

[RTIC framework]: https://github.com/rtic-rs/cortex-m-rtic

El marco también incluye otras características como el paso de mensajes, que reduce la necesidad de un estado compartido explícito, y la capacidad de programar tareas para que se ejecuten en un momento dado, lo que puede utilizarse para implementar tareas periódicas. Consulta [la documentación] para obtener más información.

[la documentación]: https://rtic.rs

## Sistemas operativos en tiempo real

Otro modelo común para la concurrencia embebida es el sistema operativo en tiempo real (RTOS). Aunque actualmente están menos explorados en Rust, son ampliamente utilizados en el desarrollo embebido tradicional. Ejemplos de código abierto incluyen [FreeRTOS] y [ChibiOS]. Estos RTOSs proporcionan soporte para ejecutar múltiples hilos de aplicación que la CPU intercambia, ya sea cuando los hilos ceden el control (llamado multitarea cooperativa) o basado en un temporizador regular o interrupciones (multitarea preventiva). Los RTOS suelen proporcionar mutexes y otras primitivas de sincronización, y a menudo interoperan con funciones de hardware como los motores DMA.

[FreeRTOS]: https://freertos.org/
[ChibiOS]: http://chibios.org/

En el momento de escribir esto, no hay muchos ejemplos de Rust RTOS a los que apuntar, pero es un área interesante así que ¡mira este espacio!

## Múltiples núcleos

Cada vez es más común tener dos o más núcleos en procesadores embebidos, lo que añade una capa extra de complejidad a la concurrencia. Todos los ejemplos que utilizan una sección crítica (incluyendo el `cortex_m::interrupt::Mutex`) asumen que el único otro hilo de ejecución es el hilo de interrupción, pero en un sistema multinúcleo eso ya no es cierto. En su lugar, necesitaremos primitivas de sincronización diseñadas para múltiples núcleos (también llamadas SMP, por multiprocesamiento simétrico).

Éstas suelen utilizar las instrucciones atómicas que hemos visto antes, ya que el sistema de procesamiento garantizará que la atomicidad se mantenga en todos los núcleos.

Cubrir estos temas en detalle está actualmente fuera del alcance de este libro, pero los patrones generales son los mismos que para el caso de un solo núcleo.