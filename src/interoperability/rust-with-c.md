# Un poco de Rust con tu C

Usar código Rust dentro de un proyecto C o C++ consiste principalmente en dos partes.

- Crear una API amigable con C en Rust
- Incrustar tu proyecto Rust en un sistema de compilación externo

Aparte de `cargo` y `meson`, la mayoría de los sistemas de compilación no tienen soporte nativo para Rust. Así que lo mejor es que utilices `cargo` para compilar tu _crate_ y cualquier dependencia.

## Creando un proyecto

Crea un nuevo proyecto `cargo` como de costumbre.

Hay banderas para decirle a `cargo` que emita una biblioteca de sistemas, en lugar de su objetivo rust normal. Esto también te permite establecer un nombre de salida diferente para tu biblioteca, si quieres que difiera del resto de tu _crate_.

```toml
[lib]
name = "your_crate"
crate-type = ["cdylib"]      # Creates dynamic lib
# crate-type = ["staticlib"] # Creates static lib
```

## Creación de una API `C`

Debido a que C++ no tiene una ABI estable para el compilador de Rust, usamos `C` para cualquier interoperabilidad entre diferentes lenguajes. Esto no es una excepción cuando usamos Rust dentro de código C y C++.

### `#[no_mangle]`

El compilador de Rust manipula los nombres de los símbolos de forma diferente a lo que esperan los enlazadores de código nativo. Como tal, cualquier función que Rust exporte para ser usada fuera de Rust necesita que se le diga que no sea manipulada por el compilador.

### `extern "C"`

Por defecto, cualquier función que escribas en Rust usará la ABI de Rust (que tampoco está estabilizada). En cambio, cuando se construyen APIs FFI orientadas al exterior, necesitamos decirle al compilador que use la ABI del sistema.

Dependiendo de tu plataforma, puede que quieras apuntar a una versión ABI específica, que están documentadas [aquí](https://doc.rust-lang.org/reference/items/external-blocks.html).

---

Juntando estas partes, se obtiene una función que se parece más o menos a esto.

```rust,ignore
#[no_mangle]
pub extern "C" fn rust_function() {

}
```

Igual que cuando usas código `C` en tu proyecto Rust ahora necesitas transformar los datos desde y hacia una forma que el resto de la aplicación entienda.

## Enlazado y mayor contexto del proyecto.

Así que, esa es una mitad del problema resuelto. ¿Cómo usas esto ahora?

**Esto depende mucho de tu proyecto y/o sistema de compilación**

`cargo` creará un archivo `my_lib.so`/`my_lib.dll` o `my_lib.a`, dependiendo de tu plataforma y configuración. Esta biblioteca puede ser simplemente enlazada por tu sistema de compilación.

Sin embargo, llamar a una función Rust desde C requiere un archivo de cabecera para declarar las firmas de la función.

Cada función en tu API Rust-ffi necesita tener una función de cabecera correspondiente.

```rust,ignore
#[no_mangle]
pub extern "C" fn rust_function() {}
```

se convertiría en

```C
void rust_function();
```

etc.

Existe una herramienta para automatizar este proceso, llamada [cbindgen], que analiza tu código Rust y genera a partir de él cabeceras para tus proyectos C y C++.

[cbindgen]: https://github.com/eqrion/cbindgen

Llegados a este punto, ¡utilizar las funciones Rust desde C es tan sencillo como incluir la cabecera y llamarlas!

```C
#include "my-rust-project.h"
rust_function();
```
