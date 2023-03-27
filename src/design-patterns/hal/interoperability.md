# Interoperabilidad


<a id="c-free"></a>
## Los tipos Wrapper proporcionan un método destructor (C-FREE)

Cualquier tipo de wrapper que no sea `Copy` proporcionado por la HAL debería proporcionar un método `free` que consuma el wrapper y devuelva el periférico en bruto (y posiblemente otros objetos) a partir del cual fue creado.

El método debe apagar y reiniciar el periférico si es necesario. Llamar a `new` con el periférico devuelto por `free` no debería fallar debido a un estado inesperado del periférico.

Si el tipo de HAL requiere que se construyan otros objetos que no sean `Copy` (por ejemplo pines de E/S), cualquier objeto de este tipo debería ser liberado y devuelto por `free` también. En ese caso, `free` debería devolver una tupla.

Por ejemplo:

```rust
# pub struct TIMER0;
pub struct Timer(TIMER0);

impl Timer {
    pub fn new(periph: TIMER0) -> Self {
        Self(periph)
    }

    pub fn free(self) -> TIMER0 {
        self.0
    }
}
```

<a id="c-reexport-pac"></a>
## HALs reexportan su _crate_ de acceso a registros (C-REEXPORT-PAC)

Las HALs pueden ser escritas sobre PACs generados por [svd2rust], o sobre otras _crates_ que proveen acceso a registros sin procesar. Las HALs siempre deben reexportar la _crate_ de acceso a registros en el que se basan en su _crate_ raíz.

Un PAC debe ser reexportado bajo el nombre `pac`, independientemente del nombre real de la _crate_, como el nombre de la HAL ya debe dejar claro qué PAC se está accediendo.

[svd2rust]: https://github.com/rust-embedded/svd2rust

<a id="c-hal-traits"></a>
## Los tipos implementan los rasgos `embedded-hal` (C-HAL-TRAITS)

Los tipos proporcionados por la HAL deben implementar todos los rasgos aplicables proporcionados por la _crate_ [`embedded-hal`].

Pueden implementarse múltiples rasgos para el mismo tipo.

[`embedded-hal`]: https://github.com/rust-embedded/embedded-hal
