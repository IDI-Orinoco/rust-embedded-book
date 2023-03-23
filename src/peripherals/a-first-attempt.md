# Un Primer Intento

## Los Registros

Echemos un vistazo al periférico 'SysTick' - un simple temporizador que viene con cada núcleo de procesador Cortex-M. Típicamente lo buscarás en la hoja de datos del fabricante del chip o en el _Manual de Referencia Técnica_, pero este ejemplo es común a todos los núcleos Cortex-M de ARM, miremos en el [manual de referencia de ARM]. Vemos que hay cuatro registros:

[manual de referencia de arm]: http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/Babieigh.html

| Offset | Nombre     | Descripción                      | Ancho   |
| ------ | ---------- | -------------------------------- | ------- |
| 0x00   | SYST_CSR   | Registro de control y estado     | 32 bits |
| 0x04   | SYST_RVR   | Registro de valor de recarga     | 32 bits |
| 0x08   | SYST_CVR   | Registro de valor actual         | 32 bits |
| 0x0C   | SYST_CALIB | Registro de Valor de Calibración | 32 bits |

## El enfoque C

En Rust, podemos representar una colección de registros exactamente de la misma manera que lo hacemos en C - con una `struct`.

```rust,ignore
#[repr(C)]
struct SysTick {
    pub csr: u32,
    pub rvr: u32,
    pub cvr: u32,
    pub calib: u32,
}
```

El calificador `#[repr(C)]` le dice al compilador de Rust que diseñe esta estructura como lo haría un compilador de C. Eso es muy importante, ya que Rust permite reordenar los campos de estructura, mientras que C no. ¡Puedes imaginar la depuración que tendríamos que hacer si el compilador reorganizara estos campos en silencio! Con este calificador en su lugar, tenemos nuestros cuatro campos de 32 bits que corresponden a la tabla anterior. Pero, por supuesto, esta `estructura` no sirve por sí sola, necesitamos una variable.

```rust,ignore
let systick = 0xE000_E010 as *mut SysTick;
let time = unsafe { (*systick).cvr };
```

## Accesos volátiles

Ahora, hay un par de problemas con el enfoque anterior.

1. Tenemos que usar `unsafe` cada vez que queramos acceder a nuestro Periférico.
2. No tenemos forma de especificar qué registros son de solo lectura o de lectura y escritura.
3. Cualquier pieza de código en cualquier parte de tu programa podría acceder al hardware a través de esta estructura.
4. Aún más importante, esto en realidad no funciona...

Ahora, el problema es que los compiladores son inteligentes. Si realiza dos escrituras en la misma pieza de RAM, una tras otra, el compilador puede notarlo y omitir la primera escritura por completo. En C, podemos marcar las variables como "volátiles" para garantizar que cada lectura o escritura ocurra según lo previsto. En Rust, en cambio, marcamos los _accesos_ como volátiles, no como variables.

```rust,ignore
let systick = unsafe { &mut *(0xE000_E010 as *mut SysTick) };
let time = unsafe { core::ptr::read_volatile(&mut systick.cvr) };
```

Entonces, hemos solucionado uno de nuestros cuatro problemas, ¡pero ahora tenemos aún más código `no seguro`! Afortunadamente, hay una _crate_ de terceros que puede ayudar: [`registro_volatil`].

[`registro_volatil`]: https://crates.io/crates/volatile_register

```rust,ignore
use volatile_register::{RW, RO};

#[repr(C)]
struct SysTick {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

fn get_systick() -> &'static mut SysTick {
    unsafe { &mut *(0xE000_E010 as *mut SysTick) }
}

fn get_time() -> u32 {
    let systick = get_systick();
    systick.cvr.read()
}
```

Ahora, los accesos volátiles se realizan automáticamente a través de los métodos `read` y `write`. Todavía es `inseguro` realizar escrituras, pero para ser justos, el hardware es un montón de estados mutables y no hay forma de que el compilador sepa si estas escrituras son realmente seguras, por lo que esta es una buena posición inicial.

## Un envoltorio Rust para SysTick

Necesitamos envolver esta `estructura` en una API de capa superior que sea segura para que llamen nuestros usuarios. Como autor del controlador, verificamos manualmente que el código inseguro sea correcto y luego presentamos una API segura para nuestros usuarios para que no tengan que preocuparse por eso (¡siempre que confíen en que lo haremos bien!).

Un ejemplo podría ser:

```rust,ignore
use volatile_register::{RW, RO};

pub struct SystemTimer {
    p: &'static mut RegisterBlock
}

#[repr(C)]
struct RegisterBlock {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

impl SystemTimer {
    pub fn new() -> SystemTimer {
        SystemTimer {
            p: unsafe { &mut *(0xE000_E010 as *mut RegisterBlock) }
        }
    }

    pub fn get_time(&self) -> u32 {
        self.p.cvr.read()
    }

    pub fn set_reload(&mut self, reload_value: u32) {
        unsafe { self.p.rvr.write(reload_value) }
    }
}

pub fn example_usage() -> String {
    let mut st = SystemTimer::new();
    st.set_reload(0x00FF_FFFF);
    format!("Time is now 0x{:08x}", st.get_time())
}
```

Ahora, el problema con este enfoque es que el siguiente código es perfectamente aceptable para el compilador:

```rust,ignore
fn thread1() {
    let mut st = SystemTimer::new();
    st.set_reload(2000);
}

fn thread2() {
    let mut st = SystemTimer::new();
    st.set_reload(1000);
}
```

Nuestro argumento `&mut self` para la función `set_reload` verifica que no haya otras referencias a _esa_ estructura `SystemTimer` en particular, ¡pero no impiden que el usuario cree un segundo `SystemTimer` que apunte exactamente al mismo periférico! El código escrito de esta manera funcionará si el autor es lo suficientemente diligente para detectar todas estas instancias de controlador 'duplicadas', pero una vez que el código se distribuye en varios módulos, controladores, desarrolladores y días, se vuelve cada vez más fácil hacer estos tipos de errores.
