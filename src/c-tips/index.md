# Consejos para desarrolladores de C embebido

Este capítulo recoge una serie de consejos que pueden ser útiles para desarrolladores experimentados en C embebido que quieran empezar a escribir Rust. Destacará especialmente cómo las cosas a las que ya podrías estar acostumbrado en C son diferentes en Rust.

## Preprocesador

En C embebido es muy común usar el preprocesador para una variedad de propósitos, tales como:

- Selección en tiempo de compilación de bloques de código con `#ifdef`.
- Cálculos y tamaños de matrices en tiempo de compilación
- Macros para simplificar patrones comunes (para evitar el costo de las llamadas a funciones)

En Rust no hay preprocesador, por lo que muchos de estos casos de uso se abordan de manera diferente. En el resto de esta sección cubrimos varias alternativas al uso del preprocesador.

### Selección de Código en Tiempo de Compilación

Lo más parecido a `#ifdef ... #endif` en Rust son las [prestaciones de Cargo]. Éstas son un poco más formales que el preprocesador de C: todas las prestaciones posibles se listan explícitamente por _crate_, y sólo pueden estar activadas o desactivadas. Las prestaciones se activan cuando listas una _crate_ como dependencia, y son aditivas: si cualquier _crate_ en tu árbol de dependencias activa una prestación para otra _crate_, esa prestación se activará para todos los usuarios de esa _crate_.

[prestaciones de Cargo]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section

Por ejemplo, puedes tener una _crate_ que proporcione una biblioteca de primitivas de procesamiento de señales. Cada una de ellas puede requerir un tiempo extra de compilación o declarar una gran tabla de constantes que te gustaría evitar. Podrías declarar una función de Cargo para cada componente en tu `Cargo.toml`:

```toml
[features]
FIR = []
IIR = []
```

A continuación, en tu código, utiliza `#[cfg(feature="FIR")]` para controlar lo que se incluye.

```rust
/// In your top-level lib.rs

#[cfg(feature="FIR")]
pub mod fir;

#[cfg(feature="IIR")]
pub mod iir;
```

De forma similar, puede incluir bloques de código sólo si una prestación _no_ está habilitada, o si cualquier combinación de prestaciones está o no está habilitada.

Además, Rust proporciona una serie de condiciones de configuración automática que puedes utilizar, como `target_arch` para seleccionar código diferente basado en la arquitectura. Para más detalles sobre el soporte de compilación condicional, consulta el capítulo [compilación condicional] de la referencia de Rust.

[compilación condicional]: https://doc.rust-lang.org/reference/conditional-compilation.html

La compilación condicional sólo se aplicará a la siguiente sentencia o bloque. Si un bloque no puede ser utilizado en el ámbito actual, entonces el atributo `cfg` tendrá que ser utilizado varias veces. Vale la pena señalar que la mayoría de las veces es mejor simplemente incluir todo el código y permitir que el compilador elimine el código muerto al optimizar: es más simple para ti y tus usuarios, y en general el compilador hará un buen trabajo eliminando el código no utilizado.

### Computación y Tamaños en Tiempo de Compilación

Rust soporta `const fn`, funciones que se garantiza que son evaluables en tiempo de compilación y por lo tanto se pueden utilizar cuando se requieren constantes, como en el tamaño de las matrices. Esto puede utilizarse junto con las funciones mencionadas anteriormente, por ejemplo:

```rust
const fn array_size() -> usize {
    #[cfg(feature="use_more_ram")]
    { 1024 }
    #[cfg(not(feature="use_more_ram"))]
    { 128 }
}

static BUF: [u32; array_size()] = [0u32; array_size()];
```

Estos son nuevos para Rust estable a partir de 1.31, por lo que la documentación es todavía escasa. La funcionalidad disponible para `const fn` también es muy limitada en el momento de escribir esto; en futuras versiones de Rust se espera ampliar lo que se permite en un `const fn`.

### Macros

Rust proporciona un [sistema de macros] extremadamente potente. Mientras que el preprocesador de C opera casi directamente sobre el texto de tu código fuente, el sistema de macros de Rust opera a un nivel superior. Hay dos variedades de macros en Rust: _macros a través de ejemplo_ y _macros procedimentales_. Las primeras son más simples y comunes; se parecen a las llamadas a funciones y pueden expandirse a una expresión, declaración, elemento o patrón completo. Las macros procedimentales son más complejas pero permiten adiciones extremadamente potentes al lenguaje Rust: pueden transformar sintaxis arbitraria de Rust en nueva sintaxis de Rust.

[sistema de macros]: https://doc.rust-lang.org/book/ch19-06-macros.html

En general, donde podría haber utilizado una macro del preprocesador de C, probablemente quiera ver si una macro por ejemplo puede hacer el trabajo en su lugar. Pueden ser definidas en su _crate_ y fácilmente usadas por su propio _crate_ o exportadas para otros usuarios. Tenga en cuenta que, dado que deben expandirse a expresiones, sentencias, elementos o patrones completos, algunos casos de uso de macros del preprocesador de C no funcionarán, por ejemplo, una macro que se expanda a parte del nombre de una variable o a un conjunto incompleto de elementos de una lista.

Al igual que ocurre con las funciones de Cargo, conviene plantearse si se necesita la macro. En muchos casos, una función regular es más fácil de entender y será _inline_ en el mismo código que una macro. Los [atributos] `#[inline]` e `#[inline(always)]` ofrecen un mayor control sobre este proceso, aunque también en este caso hay que tener cuidado: el compilador alineará automáticamente funciones de la misma _crate_ cuando sea necesario, por lo que forzarlo a hacerlo de forma inadecuada podría reducir el rendimiento.

[atributos]: https://doc.rust-lang.org/reference/attributes.html#inline-attribute

Explicar todo el sistema de macros de Rust está fuera del alcance de esta página de consejos, así que te animamos a consultar la documentación de Rust para más detalles.

## Sistema de construcción

La mayoría de los _crates_ de Rust se construyen usando Cargo (aunque no es obligatorio). Esto resuelve muchos problemas difíciles con los sistemas de construcción tradicionales. Sin embargo, es posible que desee personalizar el proceso de construcción. Cargo proporciona los [scripts `build.rs`] para este propósito. Se trata de scripts de Rust que pueden interactuar con el sistema de compilación de Cargo según sea necesario.

[scripts `build.rs`]: https://doc.rust-lang.org/cargo/reference/build-scripts.html

Los casos de uso comunes para los scripts de compilación incluyen:

- proporcionar información en tiempo de compilación, por ejemplo incrustando estáticamente la fecha de compilación o el hash de commit de Git en el ejecutable
- generar secuencias de comandos del enlazador en tiempo de compilación en función de las prestaciones seleccionadas u otra lógica
- modificar la configuración de compilación de Cargo
- añadir bibliotecas estáticas adicionales con las que enlazar

Actualmente no hay soporte para scripts posteriores a la compilación, que tradicionalmente se han utilizado para tareas como la generación automática de binarios a partir de los objetos de compilación o la impresión de información de compilación.

### Compilación cruzada

El uso de Cargo como sistema de compilación también simplifica la compilación cruzada. En la mayoría de los casos basta con decirle a Cargo `--target thumbv6m-none-eabi` y encontrar un ejecutable adecuado en `target/thumbv6m-none-eabi/debug/myapp`.

Para plataformas no soportadas nativamente por Rust, necesitarás compilar `libcore` para ese objetivo por ti mismo. En tales plataformas, [Xargo] puede ser utilizado como un sustituto de Cargo que automáticamente construye `libcore` para ti.

[Xargo]: https://github.com/japaric/xargo

## Iteradores vs Acceso a Matrices

En C probablemente estés acostumbrado a acceder a arrays directamente por su índice:

```c
int16_t arr[16];
int i;
for(i=0; i<sizeof(arr)/sizeof(arr[0]); i++) {
    process(arr[i]);
}
```

En Rust esto es un anti-patrón: el acceso indexado puede ser más lento (ya que necesita ser comprobado) y puede impedir varias optimizaciones del compilador. Esta es una distinción importante y vale la pena repetirla: Rust comprobará los accesos fuera de los límites en la indexación manual de arrays para garantizar la seguridad de la memoria, mientras que C indexará alegremente fuera del array.

En su lugar, usa iteradores:

```rust,ignore
let arr = [0u16; 16];
for element in arr.iter() {
    process(*element);
}
```

Los iteradores proporcionan una potente gama de funcionalidades que tendrías que implementar manualmente en C, como encadenar, comprimir, enumerar, encontrar el mínimo o el máximo, sumar, y más. Los métodos de los iteradores también pueden encadenarse, lo que proporciona un código de procesamiento de datos muy legible.

Consulte [Iteradores en el libro] y [Documentación de los Iteradores] para obtener más información.

[Iteradores en el libro]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[Documentación de los Iteradores]: https://doc.rust-lang.org/core/iter/trait.Iterator.html

## Referencias vs Punteros

En Rust, los punteros (llamados punteros crudos ([_raw pointers_])) existen pero sólo se usan en circunstancias específicas, ya que desreferenciarlos siempre se considera inseguro (`unsafe`) -- Rust no puede proporcionar sus garantías habituales sobre lo que puede haber detrás del puntero.

[_raw pointers_]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer

En la mayoría de los casos, utilizamos _references_, indicadas por el símbolo `&`, o _mutable references_, indicadas por `&mut`. Las referencias se comportan de manera similar a los punteros, en el sentido de que pueden ser desreferenciadas para acceder a los valores subyacentes, pero son una parte clave del sistema de propiedad de Rust: Rust hará cumplir estrictamente que sólo se puede tener una referencia mutable _o_ múltiples referencias no mutables al mismo valor en un momento dado.

En la práctica, esto significa que tienes que ser más cuidadoso sobre si necesitas acceso mutable a los datos: mientras que en C el valor por defecto es mutable y debes ser explícito sobre `const`, en Rust ocurre lo contrario.

Una situación en la que todavía se pueden utilizar _punteros crudos_ es interactuando directamente con el hardware (por ejemplo, escribiendo un puntero a un buffer en un registro periférico DMA), y también se utilizan _tras bastidores_ en todas las _crates_ de acceso periférico para permitir la lectura y escritura de registros mapeados en memoria.

## Acceso Volátil

En C, las variables individuales pueden ser marcadas como `volatile`, indicando al compilador que el valor de la variable puede cambiar entre accesos. Las variables volátiles se usan comúnmente en un contexto embebido para registros mapeados en memoria.

En Rust, en lugar de marcar una variable como `volatile`, usamos métodos específicos para realizar accesos volátiles: [`core::ptr::read_volatile`] y [`core::ptr::write_volatile`]. Estos métodos toman un `*const T` o un `*mut T` (_raw pointers_, como se ha comentado anteriormente) y realizan una lectura o escritura volátil.

[`core::ptr::read_volatile`]: https://doc.rust-lang.org/core/ptr/fn.read_volatile.html
[`core::ptr::write_volatile`]: https://doc.rust-lang.org/core/ptr/fn.write_volatile.html

Por ejemplo, en C podrías escribir

```c
volatile bool signalled = false;

void ISR() {
    // Señal de que se ha producido la interrupción
    signalled = true;
}

void driver() {
    while(true) {
        // Dormir hasta la señal
        while(!signalled) { WFI(); }
        // Resetear la bandera de la señal
        signalled = false;
        // Realizar alguna tarea que estaba a la espera de la interrupción
        run_task();
    }
}
```

El equivalente en Rust usaría métodos volátiles en cada acceso:

```rust,ignore
static mut SIGNALLED: bool = false;

#[interrupt]
fn ISR() {
    // Señal de que se ha producido la interrupción
    // (En condiciones reales, debes considerar una primitiva de alto nivel,
    //  tal como un tipo atómico (atomic type)).
    unsafe { core::ptr::write_volatile(&mut SIGNALLED, true) };
}

fn driver() {
    loop {
        // Dormir hasta la señal
        while unsafe { !core::ptr::read_volatile(&SIGNALLED) } {}
        // Resetear la bandera de la señal
        unsafe { core::ptr::write_volatile(&mut SIGNALLED, false) };
        // Realizar alguna tarea que estaba a la espera de la interrupción
        run_task();
    }
}
```

Vale la pena notar algunas cosas en el ejemplo de código:

- Podemos pasar `&mut SIGNALLED` a la función que requiere `*mut T`, ya que `&mut T` se convierte automáticamente en `*mut T` (y lo mismo para `*const T`)
- Necesitamos bloques `unsafe` para los métodos `read_volatile`/`write_volatile`, ya que son funciones `unsafe`. Es responsabilidad del programador garantizar un uso seguro: consulte la documentación de los métodos para más detalles.

Es raro que necesites estas funciones directamente en tu código, ya que normalmente se encargarán de ellas las bibliotecas de alto nivel. Para los periféricos mapeados en memoria, las _crates_ de acceso a periféricos implementarán el acceso volátil automáticamente, mientras que para las primitivas de concurrencia hay mejores abstracciones disponibles (ver el [capítulo de concurrencia]).

[capítulo de concurrencia]: ../concurrency/index.md

## Tipos empaquetados y alineados

En C embebido es común decirle al compilador que una variable debe tener una cierta alineación o que una estructura debe estar empaquetada en lugar de alineada, normalmente para cumplir requisitos específicos de hardware o protocolo.

En Rust esto se controla mediante el atributo `repr` de una estructura o unión. La representación por defecto no ofrece garantías de disposición, por lo que no debe utilizarse para código que interopera con hardware o C. El compilador puede reordenar los miembros de la estructura o insertar relleno y el comportamiento puede cambiar con futuras versiones de Rust.

```rust
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7ffecb3511d0 0x7ffecb3511d4 0x7ffecb3511d2
// Nota que el orden se cambió a x, z, y para mejorar el empaquetado.
```

Para asegurar diseños interoperables con C, use `repr(C)`:

```rust
#[repr(C)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7fffd0d84c60 0x7fffd0d84c62 0x7fffd0d84c64
// El orden se preserva y la disposición no cambiará en el tiempo.
// `z` es alineado a dos bytes por lo que un byte de relleno existe entre `y` y `z`.
```

Para asegurar una representación empaquetada, use `repr(packed)`:

```rust
#[repr(packed)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    // La referencias siempre deben estar alineadas, por lo tanto para revisar
    // las direcciones de los campos de las estructuras, usamos `std::ptr::addr_of!()`
    // para obtener un puntero crudo en lugar de sólo imprimir `&v.x`.
    let px = std::ptr::addr_of!(v.x);
    let py = std::ptr::addr_of!(v.y);
    let pz = std::ptr::addr_of!(v.z);
    println!("{:p} {:p} {:p}", px, py, pz);
}

// 0x7ffd33598490 0x7ffd33598492 0x7ffd33598493
// No se ha insertado relleno entre `y` y `z`, por lo tanto, ahora `z` está desalineado.
```

Tenga en cuenta que el uso de `repr(packed)` también establece la alineación del tipo a `1`.

Finalmente, para especificar una alineación concreta, usa `repr(align(n))`, donde `n` es el número de bytes a alinear (y debe ser una potencia de dos):

```rust
#[repr(C)]
#[repr(align(4096))]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    let u = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
    println!("{:p} {:p} {:p}", &u.x, &u.y, &u.z);
}

// 0x7ffec909a000 0x7ffec909a002 0x7ffec909a004
// 0x7ffec909b000 0x7ffec909b002 0x7ffec909b004
// Las dos instancias `u` y `v` han sido colocadas en alineamientos de 4096 bytes,
// evidenciado por el `000` al final de sus direcciones.
```

Observe que podemos combinar `repr(C)` con `repr(align(n))` para obtener una disposición alineada y compatible con C. No es permisible combinar `repr(align(n))` con `repr(packed)`, ya que `repr(packed)` establece la alineación a `1`. Tampoco está permitido que un tipo `repr(packed)` contenga un tipo `repr(align(n))`.

Para más detalles sobre la disposición de tipos, consulta el capítulo [disposición de tipos] de la Referencia de Rust.

[disposición de tipos]: https://doc.rust-lang.org/reference/type-layout.html

## Otros Recursos

- En este libro:
  - [Un poco de C con tu Rust](../interoperability/c-with-rust.md)
  - [Un poco de Rust con tu C](../interoperability/rust-with-c.md)
- [Preguntas frecuentes sobre Rust embebido](https://docs.rust-embedded.org/faq.html)
- [Punteros Rust para programadores C](http://blahg.josefsipek.net/?p=580)
- [Usé punteros, ¿y ahora qué?](https://github.com/diwic/reffers-rs/blob/master/docs/Pointers.md)