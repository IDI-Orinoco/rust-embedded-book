# Optimizaciones: el compromiso entre velocidad y tamaño

Todo el mundo quiere que su programa sea superrápido y superpequeño, pero normalmente no es posible tener ambas características. Esta sección discute los diferentes niveles de optimización que `rustc` proporciona y cómo afectan al tiempo de ejecución y al tamaño binario de un programa.

## Sin optimizaciones

Esta es la opción por defecto. Cuando se llama a `cargo build` se utiliza el perfil de desarrollo (AKA `dev`). Este perfil está optimizado para la depuración, por lo que habilita la información de depuración y _no_ habilita ninguna optimización, es decir, utiliza `-C opt-level = 0`.

Al menos para el desarrollo bare metal, debuginfo es costo cero en el sentido de que no ocupará espacio en Flash / ROM por lo que en realidad recomendamos que active debuginfo en el perfil de liberación -- está desactivado por defecto. Esto le permitirá utilizar puntos de interrupción al depurar las versiones.

```toml
[profile.release]
# symbols are nice and they don't increase the size on Flash
debug = true
```

Sin optimizaciones es genial para la depuración, porque el paso a través del código se siente como si estuviera ejecutando el programa declaración por declaración, además de que puede `print` variables de la pila y los argumentos de función en GDB. Cuando el código está optimizado, al intentar imprimir variables se imprime `$0 = <valor optimizado>`.

El mayor inconveniente del perfil `dev` es que el binario resultante será enorme y lento. El tamaño es normalmente el mayor problema porque los binarios no optimizados pueden ocupar docenas de KiB de Flash, que tu dispositivo de destino puede no tener -- el resultado: ¡tu binario no optimizado no cabe en tu dispositivo!

¿Podemos tener binarios más pequeños y fáciles de depurar? Sí, hay un truco.

### Optimizar las dependencias

Existe una función de Cargo llamada [`profile-overrides`] que te permite anular el nivel de optimización de las dependencias. Puedes usar esta característica para optimizar todas las dependencias por tamaño mientras mantienes la _crate_ superior sin optimizar y amigable para el depurador.

[`profile-overrides`]: https://doc.rust-lang.org/cargo/reference/profiles.html#overrides

Aquí tienes un ejemplo:

```toml
# Cargo.toml
[package]
name = "app"
# ..

[profile.dev.package."*"] # +
opt-level = "z" # +
```

Sin la anulación:

```text
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 9060   0x8000400
.rodata               1708   0x8002780
.data                    0  0x20000000
.bss                     4  0x20000000
```

Con la anulación:

```text
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 3490   0x8000400
.rodata               1100   0x80011c0
.data                    0  0x20000000
.bss                     4  0x20000000
```

Eso es una reducción de 6 KiB en el uso de Flash sin ninguna pérdida en la depurabilidad de la _crate_ superior. Si entras en una dependencia entonces empezarás a ver esos mensajes de `<value optimized out>` de nuevo, pero normalmente lo que quieres es depurar la _crate_ superior y no las dependencias. Y si _necesitas_ depurar una dependencia entonces puedes usar la característica `profile-overrides` para excluir una dependencia en particular de ser optimizada. Véase el ejemplo siguiente:

```toml
# ..

# don't optimize the `cortex-m-rt` crate
[profile.dev.package.cortex-m-rt] # +
opt-level = 0 # +

# but do optimize all the other dependencies
[profile.dev.package."*"]
codegen-units = 1 # better optimizations
opt-level = "z"
```

¡Ahora la _crate_ superior y `cortex-m-rt` son amigables con el depurador!

## Optimizar para velocidad

A partir de 2018-09-18 `rustc` soporta tres niveles de "optimización para velocidad": `opt-level = 1`, `2` y `3`. Cuando ejecutas `cargo build --release` estás usando el perfil release que por defecto es `opt-level = 3`.

Tanto `opt-level = 2` como `3` optimizan la velocidad a expensas del tamaño del binario, pero el nivel `3` hace más vectorización e inlining que el nivel `2`. En particular, verás que en `opt-level` igual o mayor que `2` LLVM desenrollará bucles. El desenrollado de bucles tiene un costo bastante alto en términos de Flash / ROM (por ejemplo, de 26 bytes a 194 para un bucle de matriz cero), pero también puede reducir a la mitad el tiempo de ejecución dadas las condiciones adecuadas (por ejemplo, el número de iteraciones es lo suficientemente grande).

Actualmente no hay forma de deshabilitar el desenrollado de bucles en `opt-level = 2` y `3` así que si no puedes permitirte su costo, deberías optimizar tu programa por tamaño.

## Optimizar por tamaño

A partir de 2018-09-18 `rustc` soporta dos niveles de "optimización por tamaño": `opt-level = "s"` y `"z"`. Estos nombres fueron heredados de clang / LLVM y no son demasiado descriptivos, pero `"z"` pretende dar la idea de que produce binarios más pequeños que `"s"`.

Si quieres que tus binarios de lanzamiento estén optimizados para el tamaño, entonces cambia el ajuste `profile.release.opt-level` en `Cargo.toml` como se muestra a continuación.

```toml
[profile.release]
# or "z"
opt-level = "s"
```

Estos dos niveles de optimización reducen en gran medida el umbral inline de LLVM, una métrica utilizada para decidir si hacer una función `inline` o no. Uno de los principios de Rust son las abstracciones de costo cero; estas abstracciones tienden a usar muchos newtypes y pequeñas funciones para mantener invariantes (por ejemplo, funciones que toman prestado un valor interno como `deref`, `as_ref`) por lo que un umbral inline bajo puede hacer que LLVM pierda oportunidades de optimización (por ejemplo, eliminar ramas muertas, llamadas inline a cierres).

Cuando se optimiza por tamaño se puede intentar aumentar el umbral inline para ver si tiene algún efecto en el tamaño del binario. La forma recomendada de cambiar el umbral en línea es añadir la bandera `-C inline-threshold` a las otras rustflags en `.cargo/config.toml`.

```toml
# .cargo/config.toml
# this assumes that you are using the cortex-m-quickstart template
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
rustflags = [
  # ..
  "-C", "inline-threshold=123", # +
]
```

¿Qué valor usar? [A partir de 1.29.0 estos son los umbrales en línea que utilizan los distintos niveles de optimización][inline-threshold]:

[inline-threshold]: https://github.com/rust-lang/rust/blob/1.29.0/src/librustc_codegen_llvm/back/write.rs#L2105-L2122

- `opt-level = 3` usa 275
- `opt-level = 2` usa 225
- `opt-level = "s"` usa 75
- `opt-level = "z"` usa 25

Deberías probar con `225` y `275` para optimizar el tamaño.
