# Interoperabilidad

La interoperabilidad entre el código Rust y C depende siempre de la transformación de datos entre los dos lenguajes. Para ello hay dos módulos dedicados en `stdlib` llamados [`std::ffi`](https://doc.rust-lang.org/std/ffi/index.html) y [`std::os::raw`](https://doc.rust-lang.org/std/os/raw/index.html).

`std::os::raw` trata con tipos primitivos de bajo nivel que pueden ser convertidos implícitamente por el compilador porque la disposición de memoria entre Rust y C es lo suficientemente similar o la misma.

`std::ffi` proporciona alguna utilidad para convertir tipos más complejos como Strings, mapeando tanto `&str` como `String` a tipos C que son más fáciles y seguros de manejar.

Ninguno de estos módulos está disponible en `core`, pero puedes encontrar una versión compatible con `#![no_std]` de `std::ffi::{CStr,CString}` en la _crate_ [`cstr_core`], y la mayoría de los tipos `std::os::raw` en la _crate_ [`cty`].

[`cstr_core`]: https://crates.io/crates/cstr_core
[`cty`]: https://crates.io/crates/cty

| Tipo Rust | Intermedio | Tipo C       |
| --------- | ---------- | ------------ |
| String    | CString    | \*char       |
| &str      | CStr       | \*const char |
| ()        | c_void     | void         |
| u32 o u64 | c_uint     | unsigned int |
| etc.      | ...        | ...          |

Como se mencionó anteriormente, los tipos primitivos pueden ser convertidos por el compilador implícitamente.

```rust,ignore
unsafe fn foo(num: u32) {
    let c_num: c_uint = num;
    let r_num: u32 = c_num;
}
```

## Interoperabilidad con otros sistemas de compilación

Un requisito común para incluir Rust en tu proyecto embebido es combinar Cargo con tu sistema de compilación existente, como make o cmake.

Estamos recopilando ejemplos y casos de uso para esto en nuestro issue tracker en [issue #61].

[issue #61]: https://github.com/rust-embedded/book/issues/61

## Interoperabilidad con Sistemas Operativos en Tiempo Real (RTOSs)

Integrar Rust con un RTOS como FreeRTOS o ChibiOS es todavía un trabajo en progreso; en particular llamar a funciones RTOS desde Rust puede ser complicado.

Estamos recopilando ejemplos y casos de uso para esto en nuestro issue tracker en [issue #62].

[issue #62]: https://github.com/rust-embedded/book/issues/62
