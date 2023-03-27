# Un poco de C con tu Rust

Usar C o C++ dentro de un proyecto Rust consta de dos partes principales:

- Envolver la API C expuesta para usarla con Rust
- Construir tu código C o C++ para integrarlo con el código Rust

Como C++ no tiene una ABI estable para el compilador de Rust, se recomienda usar la ABI `C` cuando se combina Rust con C o C++.

## Definir la interfaz

Antes de consumir código C o C++ desde Rust, es necesario definir (en Rust) qué tipos de datos y firmas de función existen en el código enlazado. En C o C++, se incluiría un fichero de cabecera (`.h` o `.hpp`) que defina estos datos. En Rust, es necesario traducir manualmente estas definiciones a Rust, o utilizar una herramienta para generar estas definiciones.

Primero, cubriremos la traducción manual de estas definiciones de C/C++ a Rust.

### Envolviendo funciones C y Datatypes

Típicamente, las bibliotecas escritas en C o C++ proporcionan un fichero de cabecera que define todos los tipos y funciones utilizados en las interfaces públicas. Un archivo de ejemplo puede tener este aspecto:

```C
/* File: cool.h */
typedef struct CoolStruct {
    int x;
    int y;
} CoolStruct;

void cool_function(int i, char c, CoolStruct* cs);
```

Traducida a Rust, esta interfaz quedaría así:

```rust,ignore
/* File: cool_bindings.rs */
#[repr(C)]
pub struct CoolStruct {
    pub x: cty::c_int,
    pub y: cty::c_int,
}

extern "C" {
    pub fn cool_function(
        i: cty::c_int,
        c: cty::c_char,
        cs: *mut CoolStruct
    );
}
```

Echemos un vistazo a esta definición pieza por pieza, para explicar cada una de las partes.

```rust,ignore
#[repr(C)]
pub struct CoolStruct { ... }
```

Por defecto, Rust no garantiza el orden, el relleno o el tamaño de los datos incluidos en una `struct`. Para garantizar la compatibilidad con el código C, incluimos el atributo `#[repr(C)]`, que indica al compilador de Rust que utilice siempre las mismas reglas que C para organizar los datos dentro de una estructura.

```rust,ignore
pub x: cty::c_int,
pub y: cty::c_int,
```

Debido a la flexibilidad de cómo C o C++ definen un `int` o un `char`, se recomienda usar tipos de datos primitivos definidos en `cty`, que mapeará tipos de C a tipos en Rust.

```rust,ignore
extern "C" { pub fn cool_function( ... ); }
```

Esta sentencia define la firma de una función que usa la ABI de C, llamada `cool_function`. Al definir la firma sin definir el cuerpo de la función, la definición de esta función necesitará ser proporcionada en otro lugar, o enlazada en la librería final o binaria desde una librería estática.

```rust,ignore
    i: cty::c_int,
    c: cty::c_char,
    cs: *mut CoolStruct
```

Al igual que en el caso anterior, definimos los tipos de datos de los argumentos de las funciones utilizando definiciones compatibles con C. También mantenemos los mismos nombres de argumentos para mayor claridad.

Aquí tenemos un nuevo tipo, `*mut CoolStruct`. Como C no tiene el concepto de referencias de Rust, que se vería así: `&mut CoolStruct`, en su lugar tenemos un puntero crudo. Como hacer referencia a este puntero es `unsafe`, y el puntero puede ser de hecho un puntero `null`, hay que tener cuidado para asegurar las garantías típicas de Rust cuando se interactúa con código C o C++.

### Generación automática de la interfaz

En lugar de generar manualmente estas interfaces, lo que puede ser tedioso y propenso a errores, existe una herramienta llamada [bindgen] que realizará estas conversiones automáticamente. Para instrucciones de uso de [bindgen], consulte el [manual de usuario de bindgen], sin embargo el proceso típico consiste en lo siguiente:

1. Reúne todas las cabeceras C o C++ que definan interfaces o tipos de datos que quieras usar con Rust.
2. Escribe un archivo `bindings.h`, que `#include "..."` cada uno de los archivos que reuniste en el paso uno.
3. Introduce este fichero `bindings.h`, junto con cualquier bandera de compilación utilizada para compilar tu código en `bindgen`. Consejo: utilice `Builder.ctypes_prefix("cty")` / `--ctypes-prefix=cty` y `Builder.use_core()` / `--use-core` para hacer compatible el código generado `#![no_std]`.
4. `bindgen` producirá el código Rust generado a la salida de la ventana de terminal. Este archivo puede ser canalizado a un archivo de tu proyecto, como `bindings.rs`. Puedes usar este archivo en tu proyecto Rust para interactuar con código C/C++ compilado y enlazado como una librería externa. Consejo: no te olvides de usar la _crate_ [`cty`](https://crates.io/crates/cty) si tus tipos en los bindings generados están prefijados con `cty`.

[bindgen]: https://github.com/rust-lang/rust-bindgen
[manual de usuario de bindgen]: https://rust-lang.github.io/rust-bindgen/

## Construyendo tu código C/C++

Como el compilador Rust no sabe directamente cómo compilar código C o C++ (o código de cualquier otro lenguaje, que presente una interfaz C), es necesario compilar tu código no Rust con antelación.

Para proyectos embebidos, esto significa normalmente compilar el código C/C++ en un archivo estático (como `cool-library.a`), que puede combinarse con tu código Rust en el paso final de enlazado.

Si la biblioteca que deseas utilizar ya se distribuye como un archivo estático, no es necesario reconstruir su código. Sólo tienes que convertir el fichero de cabecera de interfaz proporcionado como se ha descrito anteriormente, e incluir el archivo estático en el momento de compilar/enlazar.

Si tu código existe como un proyecto fuente, será necesario compilar tu código C/C++ a una biblioteca estática, ya sea activando tu sistema de compilación existente (como `make`, `CMake`, etc.), o portando los pasos de compilación necesarios para utilizar una herramienta llamada `cc` _crate_. Para ambos pasos, es necesario utilizar un script `build.rs`.

### Scripts de compilación `build.rs` de Rust

Un script `build.rs` es un archivo escrito en sintaxis Rust, que se ejecuta en tu máquina de compilación, DESPUÉS de que las dependencias de tu proyecto hayan sido construidas, pero ANTES de que tu proyecto sea construido.

La referencia completa se puede encontrar [aquí](https://doc.rust-lang.org/cargo/reference/build-scripts.html). Los scripts `build.rs` son útiles para generar código (como a través de [bindgen]), llamando a sistemas de compilación externos como `Make`, o compilando directamente C/C++ a través del uso de la _crate_ `cc`.

### Activación de sistemas de compilación externos

Para proyectos con complejos proyectos externos o sistemas de compilación, puede ser más fácil usar [`std::process::Command`] para "enviar" a tus otros sistemas de compilación atravesando rutas relativas, llamando a un comando fijo (como `make library`), y luego copiando la biblioteca estática resultante a la ubicación adecuada en el directorio de compilación `target`.

Mientras que tu _crate_ puede estar dirigida a una plataforma embebida `no_std`, tu `build.rs` se ejecuta sólo en las máquinas que compilan tu _crate_. Esto significa que puedes usar cualquier _crate_ de Rust que se ejecute en tu anfitrión de compilación.

[`std::process::command`]: https://doc.rust-lang.org/std/process/struct.Command.html

### Construyendo código C/C++ con la _crate_ `cc`

Para proyectos con dependencias o complejidad limitadas, o para proyectos donde es difícil modificar el sistema de compilación para producir una librería estática (en lugar de un binario final o ejecutable), puede ser más fácil utilizar la [_crate_ `cc`], que proporciona una interfaz idiomática de Rust al compilador proporcionado por el anfitrión.

[_crate_ `cc`]: https://github.com/alexcrichton/cc-rs

En el caso más sencillo de compilar un único archivo C como dependencia de una biblioteca estática, un script `build.rs` de ejemplo utilizando la [_crate_ `cc`] tendría el siguiente aspecto:

```rust,ignore
fn main() {
    cc::Build::new()
        .file("foo.c")
        .compile("libfoo.a");
}
```
