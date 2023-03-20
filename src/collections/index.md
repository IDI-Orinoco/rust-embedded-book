# Colecciones

Eventualmente querrás usar estructuras de datos dinámicas (también conocidas como colecciones) en tu programa. `std` proporciona un conjunto de colecciones comunes: [`Vec`], [`String`], [`HashMap`], etc. Todas las colecciones implementadas en `std` utilizan un asignador global de memoria dinámica (también conocido como heap).

[`Vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[`String`]: https://doc.rust-lang.org/std/string/struct.String.html
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html

Como `core` está, por definición, libre de asignaciones de memoria, estas implementaciones no están disponibles allí, pero se pueden encontrar en la _crate_ `alloc` que viene con el compilador.

Si necesitas colecciones, una implementación asignada a heap no es tu única opción. También puedes usar colecciones de _capacidad fija_; una de estas implementaciones se encuentra en la _crate_ [`heapless`].

[`heapless`]: https://crates.io/crates/heapless

En esta sección, exploraremos y compararemos estas dos implementaciones.

## Usando `alloc`

La _crate_ `alloc` viene con la distribución estándar de Rust. Para importar la _crate_ puedes `usarla` directamente _sin_ declararla como dependencia en tu archivo `Cargo.toml`.

``` rust,ignore
#![feature(alloc)]

extern crate alloc;

use alloc::vec::Vec;
```

Para poder utilizar cualquier colección primero tendrás que utilizar el atributo `global_allocator` para declarar el asignador global que utilizará tu programa. Es necesario que el asignador que selecciones implemente el _trait_ [`GlobalAlloc`].

[`GlobalAlloc`]: https://doc.rust-lang.org/core/alloc/trait.GlobalAlloc.html

Para completar y mantener esta sección lo más autocontenida posible, implementaremos un simple asignador de punteros y lo usaremos como asignador global. Sin embargo, te sugerimos encarecidamente que utilices un asignador probado en batalla que esté en crates.io en tu programa en lugar de este asignador.

``` rust,ignore
// Bump pointer allocator implementation

use core::alloc::{GlobalAlloc, Layout};
use core::cell::UnsafeCell;
use core::ptr;

use cortex_m::interrupt;

// Bump pointer allocator for *single* core systems
struct BumpPointerAlloc {
    head: UnsafeCell<usize>,
    end: usize,
}

unsafe impl Sync for BumpPointerAlloc {}

unsafe impl GlobalAlloc for BumpPointerAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // `interrupt::free` is a critical section that makes our allocator safe
        // to use from within interrupts
        interrupt::free(|_| {
            let head = self.head.get();
            let size = layout.size();
            let align = layout.align();
            let align_mask = !(align - 1);

            // move start up to the next alignment boundary
            let start = (*head + align - 1) & align_mask;

            if start + size > self.end {
                // a null pointer signal an Out Of Memory condition
                ptr::null_mut()
            } else {
                *head = start + size;
                start as *mut u8
            }
        })
    }

    unsafe fn dealloc(&self, _: *mut u8, _: Layout) {
        // this allocator never deallocates memory
    }
}

// Declaration of the global memory allocator
// NOTE the user must ensure that the memory region `[0x2000_0100, 0x2000_0200]`
// is not used by other parts of the program
#[global_allocator]
static HEAP: BumpPointerAlloc = BumpPointerAlloc {
    head: UnsafeCell::new(0x2000_0100),
    end: 0x2000_0200,
};
```

Además de seleccionar un asignador global, el usuario también tendrá que definir cómo se gestionan los errores de falta de memoria (OOM) usando el atributo _unstable_ `alloc_error_handler`.

``` rust,ignore
#![feature(alloc_error_handler)]

use cortex_m::asm;

#[alloc_error_handler]
fn on_oom(_layout: Layout) -> ! {
    asm::bkpt();

    loop {}
}
```

Una vez que todo esto está en su lugar, el usuario puede finalmente utilizar las colecciones en `alloc`.

```rust,ignore
#[entry]
fn main() -> ! {
    let mut xs = Vec::new();

    xs.push(42);
    assert!(xs.pop(), Some(42));

    loop {
        // ..
    }
}
```

Si has usado las colecciones en la _crate_ `std` entonces estas te serán familiares ya que son exactamente la misma implementación.

## Usando `heapless`

`heapless` no requiere configuración ya que sus colecciones no dependen de un asignador de memoria global. Simplemente `use` sus colecciones y procede a instanciarlas:

```rust,ignore
// heapless version: v0.4.x
use heapless::Vec;
use heapless::consts::*;

#[entry]
fn main() -> ! {
    let mut xs: Vec<_, U8> = Vec::new();

    xs.push(42).unwrap();
    assert_eq!(xs.pop(), Some(42));
    loop {}
}
```

Notarás dos diferencias entre estas colecciones y las de `alloc`.

Primero, tienes que declarar por adelantado la capacidad de la colección. Las colecciones `heapless` nunca se reasignan y tienen capacidades fijas; esta capacidad forma parte de la firma de tipo de la colección. En este caso hemos declarado que `xs` tiene una capacidad de 8 elementos, es decir, que el vector puede contener, como máximo, 8 elementos. Esto se indica mediante la `U8` (véase [`typenum`]) en la firma de tipo.

[`typenum`]: https://crates.io/crates/typenum

En segundo lugar, el método `push`, y muchos otros métodos, devuelven un `Result`. Dado que las colecciones `heapless` tienen una capacidad fija, todas las operaciones que insertan elementos en la colección pueden fallar potencialmente. La API refleja este problema devolviendo un `Result` que indica si la operación ha tenido éxito o no. Por el contrario, las colecciones `alloc` se reasignan a sí mismas en el heap para aumentar su capacidad.

A partir de la versión v0.4.x todas las colecciones `heapless` almacenan todos sus elementos en línea. Esto significa que una operación como `let x = heapless::Vec::new();` asignará la colección en la pila, pero también es posible asignar la colección en una variable `static`, o incluso en el montón (`Box<Vec<_, _>>`).

## Ventajas y desventajas

Ten en cuenta lo siguiente a la hora de elegir entre colecciones reubicables asignadas al heap y colecciones de capacidad fija.

### Out Of Memory y gestión de errores

Con las asignaciones de heap, el Out Of Memory es siempre una posibilidad y puede ocurrir en cualquier lugar donde una colección pueda necesitar crecer: por ejemplo, todas las invocaciones `alloc::Vec.push` pueden potencialmente generar una condición OOM. Por tanto, algunas operaciones pueden fallar _implícitamente_. Algunas colecciones `alloc` exponen métodos `try_reserve` que te permiten comprobar posibles condiciones OOM al hacer crecer la colección, pero debes ser proactivo a la hora de utilizarlos.

Si usas exclusivamente colecciones `heapless` y no usas un asignador de memoria para nada más, entonces una condición OOM es imposible. En su lugar, tendrás que lidiar con las colecciones que se queden sin capacidad caso por caso. Es decir, tendrás que lidiar con _todos_ los `Result` devueltos por métodos como `Vec.push`.

Los fallos OOM pueden ser más difíciles de depurar que, por ejemplo, `unwrap` todos los `Result` devueltos por `heapless::Vec.push` porque la localización observada del fallo puede _no_ coincidir con la localización de la causa del problema. Por ejemplo, incluso `vec.reserve(1)` puede desencadenar un OOM si el asignador está casi agotado porque alguna otra colección estaba perdiendo memoria (las fugas de memoria son posibles en Rust seguro).

### Uso de memoria

Razonar sobre el uso de memoria de las colecciones asignadas al heap es difícil porque la capacidad de las colecciones de larga duración puede cambiar en tiempo de ejecución. Algunas operaciones pueden implícitamente reasignar la colección incrementando su uso de memoria, y algunas colecciones exponen métodos como `shrink_to_fit` que pueden potencialmente reducir la memoria usada por la colección -- en última instancia, depende del asignador decidir si realmente reduce la asignación de memoria o no. Además, el asignador puede tener que lidiar con la fragmentación de la memoria, lo que puede aumentar el uso de memoria _aparente_.

Por otro lado, si utilizas exclusivamente colecciones de capacidad fija, almacenas la mayoría de ellas en variables `static` y estableces un tamaño máximo para la pila de llamadas, el enlazador detectará si intentas utilizar más memoria de la que está físicamente disponible.

Además, las colecciones de capacidad fija asignadas en la pila serán reportadas por la bandera [`-Z emit-stack-sizes`] lo que significa que las herramientas que analizan el uso de la pila (como [`stack-sizes`]) las incluirán en su análisis.

[`-Z emit-stack-sizes`]: https://doc.rust-lang.org/beta/unstable-book/compiler-flags/emit-stack-sizes.html
[`stack-sizes`]: https://crates.io/crates/stack-sizes

Sin embargo, las colecciones de capacidad fija _no_ pueden reducirse, lo que puede dar lugar a factores de carga (la relación entre el tamaño de la colección y su capacidad) inferiores a los que pueden conseguir las colecciones reubicables.

### Tiempo de ejecución en el peor de los casos (WCET)

Si estás construyendo aplicaciones sensibles al tiempo o aplicaciones de tiempo real duro, entonces te preocupas, quizás mucho, por el tiempo de ejecución en el peor de los casos de las diferentes partes de tu programa. Las colecciones `alloc` pueden reasignarse, por lo que el WCET de las operaciones que pueden hacer crecer la colección también incluirá el tiempo que se tarda en reasignar la colección, que a su vez depende de la capacidad _runtime_ de la colección. Esto dificulta la determinación del WCET de, por ejemplo, la operación `alloc::Vec.push`, ya que depende tanto del asignador utilizado como de su capacidad en tiempo de ejecución.

Por otro lado, las colecciones de capacidad fija nunca se reasignan, por lo que todas las operaciones tienen un tiempo de ejecución predecible. Por ejemplo, `heapless::Vec.push` se ejecuta en tiempo constante.

### Facilidad de uso

`alloc` requiere configurar un asignador global mientras que `heapless` no. Sin embargo, `heapless` requiere que elijas la capacidad de cada colección que instales.

La API `alloc` será familiar para prácticamente todos los desarrolladores de Rust. La API `heapless` intenta imitar de cerca la API `alloc` pero nunca será exactamente igual debido a su gestión explícita de errores -- algunos desarrolladores pueden sentir que la gestión explícita de errores es excesiva o demasiado engorrosa.