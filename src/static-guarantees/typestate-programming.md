# Programación de estado de tipos (typestates)

El concepto de [typestates] describe la codificación de información sobre el estado actual de un objeto en el tipo de ese objeto. Aunque esto puede sonar un poco arcano, si ha usado [Patrón Constructor] en Rust, ¡ya ha comenzado a usar Typestate Programming!

[typestates]: https://en.wikipedia.org/wiki/Typestate_analysis
[Patrón Constructor]: https://doc.rust-lang.org/1.0.0/style/ownership/builders.html

```rust
pub mod foo_module {
    #[derive(Debug)]
    pub struct Foo {
        inner: u32,
    }

    pub struct FooBuilder {
        a: u32,
        b: u32,
    }

    impl FooBuilder {
        pub fn new(starter: u32) -> Self {
            Self {
                a: starter,
                b: starter,
            }
        }

        pub fn double_a(self) -> Self {
            Self {
                a: self.a * 2,
                b: self.b,
            }
        }

        pub fn into_foo(self) -> Foo {
            Foo {
                inner: self.a + self.b,
            }
        }
    }
}

fn main() {
    let x = foo_module::FooBuilder::new(10)
        .double_a()
        .into_foo();

    println!("{:#?}", x);
}
```

En este ejemplo, no hay una forma directa de crear un objeto `Foo`. Debemos crear un `FooBuilder` e inicializarlo correctamente antes de que podamos obtener el objeto `Foo` que queremos.

Este ejemplo mínimo codifica dos estados:

- `FooBuilder`, que representa un estado "desconfigurado" o "configuración en proceso"
- `Foo`, que representa un estado "configurado" o "listo para usar".

## Tipos Fuertes

Debido a que Rust tiene un [Sistema de Tipos Fuerte], no hay una manera fácil de crear mágicamente una instancia de `Foo`, o de convertir un `FooBuilder` en un `Foo` sin llamar al método `into_foo()`. Además, llamar al método `into_foo()` consume la estructura original de `FooBuilder`, lo que significa que no se puede reutilizar sin la creación de una nueva instancia.

[sistema de tipos fuerte]: https://en.wikipedia.org/wiki/Strong_and_weak_typing

Esto nos permite representar los estados de nuestro sistema como tipos e incluir las acciones necesarias para las transiciones de estado en los métodos que intercambian un tipo por otro. Al crear un `FooBuilder` e intercambiarlo por un objeto `Foo`, hemos recorrido los pasos de una máquina de estado básica.
