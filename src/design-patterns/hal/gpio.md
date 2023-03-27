# Recomendaciones para interfaces GPIO

<a id="c-zst-pin"></a>
## Los tipos de pin son de tamaño cero por defecto (C-ZST-PIN)

Las Interfaces GPIO expuestas por la HAL deberían proporcionar tipos dedicados de tamaño cero para cada pin en cada interfaz o puerto, resultando en una abstracción GPIO de costo cero cuando todas las asignaciones de pines son conocidas estáticamente.

Cada interfaz o puerto GPIO debe implementar un método `split` que devuelva una estructura con cada pin.

Ejemplo:

```rust
pub struct PA0;
pub struct PA1;
// ...

pub struct PortA;

impl PortA {
    pub fn split(self) -> PortAPins {
        PortAPins {
            pa0: PA0,
            pa1: PA1,
            // ...
        }
    }
}

pub struct PortAPins {
    pub pa0: PA0,
    pub pa1: PA1,
    // ...
}
```

<a id="c-erased-pin"></a>
## Los tipos de pines proporcionan métodos para borrar pines y puertos (C-ERASED-PIN)

Los pines deben proporcionar métodos de borrado de tipos que trasladen sus propiedades de tiempo de compilación a tiempo de ejecución, y permitan una mayor flexibilidad en las aplicaciones.

Ejemplo:

```rust
/// Port A, pin 0.
pub struct PA0;

impl PA0 {
    pub fn erase_pin(self) -> PA {
        PA { pin: 0 }
    }
}

/// A pin on port A.
pub struct PA {
    /// The pin number.
    pin: u8,
}

impl PA {
    pub fn erase_port(self) -> Pin {
        Pin {
            port: Port::A,
            pin: self.pin,
        }
    }
}

pub struct Pin {
    port: Port,
    pin: u8,
    // (these fields can be packed to reduce the memory footprint)
}

enum Port {
    A,
    B,
    C,
    D,
}
```

<a id="c-pin-state"></a>
## El estado de los pines debería codificarse como parámetros de tipo (C-PIN-STATE)

Los pines pueden configurarse como entrada o salida con diferentes características dependiendo del chip o familia. Este estado debe codificarse en el sistema de tipos para evitar el uso de pines en estados incorrectos.

Los estados adicionales específicos de cada chip (por ejemplo, la intensidad de accionamiento) también pueden codificarse de este modo, utilizando parámetros de tipo adicionales.

Los métodos para cambiar el estado del pin deben proporcionarse como métodos `into_input` y `into_output`.

Además, se deben proporcionar métodos `with_{input,output}_state` que reconfiguren temporalmente un pin en un estado diferente sin moverlo.

Se deben proporcionar los siguientes métodos para cada tipo de pin (es decir, tanto los tipos de pin borrados como los no borrados deben proporcionar la misma API):

* `pub fn into_input<N: InputState>(self, input: N) -> Pin<N>`
* `pub fn into_output<N: OutputState>(self, output: N) -> Pin<N>`
* ```ignore
  pub fn with_input_state<N: InputState, R>(
      &mut self,
      input: N,
      f: impl FnOnce(&mut PA1<N>) -> R,
  ) -> R
  ```
* ```ignore
  pub fn with_output_state<N: OutputState, R>(
      &mut self,
      output: N,
      f: impl FnOnce(&mut PA1<N>) -> R,
  ) -> R
  ```


El estado de los pines debe estar limitado por _traits_ sellados. Los usuarios de la HAL no deberían tener necesidad de añadir su propio estado. Los _traits_ pueden proporcionar métodos específicos de la HAL necesarios para implementar la API de estado de los pines.

Ejemplo:

```rust
# use std::marker::PhantomData;
mod sealed {
    pub trait Sealed {}
}

pub trait PinState: sealed::Sealed {}
pub trait OutputState: sealed::Sealed {}
pub trait InputState: sealed::Sealed {
    // ...
}

pub struct Output<S: OutputState> {
    _p: PhantomData<S>,
}

impl<S: OutputState> PinState for Output<S> {}
impl<S: OutputState> sealed::Sealed for Output<S> {}

pub struct PushPull;
pub struct OpenDrain;

impl OutputState for PushPull {}
impl OutputState for OpenDrain {}
impl sealed::Sealed for PushPull {}
impl sealed::Sealed for OpenDrain {}

pub struct Input<S: InputState> {
    _p: PhantomData<S>,
}

impl<S: InputState> PinState for Input<S> {}
impl<S: InputState> sealed::Sealed for Input<S> {}

pub struct Floating;
pub struct PullUp;
pub struct PullDown;

impl InputState for Floating {}
impl InputState for PullUp {}
impl InputState for PullDown {}
impl sealed::Sealed for Floating {}
impl sealed::Sealed for PullUp {}
impl sealed::Sealed for PullDown {}

pub struct PA1<S: PinState> {
    _p: PhantomData<S>,
}

impl<S: PinState> PA1<S> {
    pub fn into_input<N: InputState>(self, input: N) -> PA1<Input<N>> {
        todo!()
    }

    pub fn into_output<N: OutputState>(self, output: N) -> PA1<Output<N>> {
        todo!()
    }

    pub fn with_input_state<N: InputState, R>(
        &mut self,
        input: N,
        f: impl FnOnce(&mut PA1<N>) -> R,
    ) -> R {
        todo!()
    }

    pub fn with_output_state<N: OutputState, R>(
        &mut self,
        output: N,
        f: impl FnOnce(&mut PA1<N>) -> R,
    ) -> R {
        todo!()
    }
}

// Same for `PA` and `Pin`, and other pin types.
```
