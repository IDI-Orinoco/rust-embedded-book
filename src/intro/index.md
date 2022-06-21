# Introducción

Bienvenido al Libro de Rust Embebido: Un libro introductorio sobre el uso del lenguaje de programación Rust en sistemas embebidos "Bare Metal", como los microcontroladores.

## A quién va dirigido Rust Embebido

Rust Embebido es para todos aquellos que quieran hacer programación embebida aprovechando los conceptos de alto nivel y las garantías de seguridad que ofrece el lenguaje Rust. 
(Ver también [Para quién es Rust](https://doc.rust-lang.org/book/ch00-00-introduction.html))

## Alcance

Los objetivos de este libro son:

* Poner a los desarrolladores al día con el desarrollo de Rust embebido, es decir, cómo configurar un entorno de desarrollo.

* Compartir las mejores prácticas *actuales* sobre el uso de Rust para el desarrollo embebido, es decir, cómo utilizar mejor las características del lenguaje Rust para escribir un software embebido más correcto.

* Servir como un recetario en algunos casos. Por ejemplo, ¿Cómo puedo mezclar C y Rust en un solo proyecto?

Este libro intenta ser lo más general posible, pero para facilitar las cosas tanto a los lectores como a los escritores, utiliza la arquitectura ARM Cortex-M en todos sus ejemplos. Sin embargo, el libro no asume que el lector esté familiarizado con esta arquitectura en particular y explica detalles particulares de esta arquitectura cuando es necesario.

## A quién va dirigido este libro

Este libro está dirigido a personas con algún tipo de experiencia en sistemas embebidos o con algún tipo de experiencia en Rust, sin embargo creemos que todos los curiosos de la programación en Rust embebido pueden obtener algo de este libro. Para aquellos que no tengan ningún conocimiento previo, les sugerimos que lean la sección "Supuestos y Prerrequisitos" y se pongan al día con los conocimientos que les faltan para sacar más provecho del libro y mejorar su experiencia de lectura. Puedes consultar la sección "Otros recursos" para encontrar recursos sobre temas en los que quieras ponerte al día.

### Supuestos y prerrequisitos

* Te sientes cómodo usando el lenguaje de programación Rust, y has escrito, ejecutado y depurado aplicaciones Rust en un entorno de escritorio. También debes estar familiarizado con los modismos de la [edición 2018], ya que este libro está dirigido a Rust 2018.

[edición 2018]: https://doc.rust-lang.org/edition-guide/

* Te sientes cómodo desarrollando y depurando sistemas embebidos en otro lenguaje como C, C++ o Ada, y estás familiarizado con conceptos como:
    * Compilación cruzada
    * Periféricos mapeados en memoria
    * Interrupciones
    * Interfaces comunes como I2C, SPI, Serial, etc.

### Other Resources
If you are unfamiliar with anything mentioned above or if you want more information about a specific topic mentioned in this book you might find some of these resources helpful.

| Topic        | Resource | Description |
|--------------|----------|-------------|
| Rust         | [Rust Book](https://doc.rust-lang.org/book/) | If you are not yet comfortable with Rust, we highly suggest reading this book. |
| Rust, Embedded | [Discovery Book](https://docs.rust-embedded.org/discovery/) | If you have never done any embedded programming, this book might be a better start |
| Rust, Embedded | [Embedded Rust Bookshelf](https://docs.rust-embedded.org) | Here you can find several other resources provided by Rust's Embedded Working Group. |
| Rust, Embedded | [Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) | The nitty gritty details when doing embedded programming in Rust. |
| Rust, Embedded | [embedded FAQ](https://docs.rust-embedded.org/faq.html) | Frequently asked questions about Rust in an embedded context. |
| Interrupts | [Interrupt](https://en.wikipedia.org/wiki/Interrupt) | - |
| Memory-mapped IO/Peripherals | [Memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) | - |
| SPI, UART, RS232, USB, I2C, TTL | [Stack Exchange about SPI, UART, and other interfaces](https://electronics.stackexchange.com/questions/37814/usart-uart-rs232-usb-spi-i2c-ttl-etc-what-are-all-of-these-and-how-do-th) | - |

### Translations

This book has been translated by generous volunteers. If you would like your
translation listed here, please open a PR to add it.

* [Japanese](https://tomoyuki-nakabayashi.github.io/book/)
  ([repository](https://github.com/tomoyuki-nakabayashi/book))

* [Chinese](https://xxchang.github.io/book/)
  ([repository](https://github.com/XxChang/book))

## How to Use This Book

This book generally assumes that you’re reading it front-to-back. Later
chapters build on concepts in earlier chapters, and earlier chapters may
not dig into details on a topic, revisiting the topic in a later chapter.

This book will be using the [STM32F3DISCOVERY] development board from
STMicroelectronics for the majority of the examples contained within. This board
is based on the ARM Cortex-M architecture, and while basic functionality is
the same across most CPUs based on this architecture, peripherals and other
implementation details of Microcontrollers are different between different
vendors, and often even different between Microcontroller families from the same
vendor.

For this reason, we suggest purchasing the [STM32F3DISCOVERY] development board
for the purpose of following the examples in this book.

[STM32F3DISCOVERY]: http://www.st.com/en/evaluation-tools/stm32f3discovery.html

## Contributing to This Book

The work on this book is coordinated in [this repository] and is mainly
developed by the [resources team].

[this repository]: https://github.com/rust-embedded/book
[resources team]: https://github.com/rust-embedded/wg#the-resources-team

If you have trouble following the instructions in this book or find that some
section of the book is not clear enough or hard to follow then that's a bug and
it should be reported in [the issue tracker] of this book.

[the issue tracker]: https://github.com/rust-embedded/book/issues/

Pull requests fixing typos and adding new content are very welcome!

## Re-using this material

This book is distributed under the following licenses:

* The code samples and free-standing Cargo projects contained within this book are licensed under the terms of both the [MIT License] and the [Apache License v2.0].
* The written prose, pictures and diagrams contained within this book are licensed under the terms of the Creative Commons [CC-BY-SA v4.0] license.

[MIT License]: https://opensource.org/licenses/MIT
[Apache License v2.0]: http://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA v4.0]: https://creativecommons.org/licenses/by-sa/4.0/legalcode

TL;DR: If you want to use our text or images in your work, you need to:

* Give the appropriate credit (i.e. mention this book on your slide, and provide a link to the relevant page)
* Provide a link to the [CC-BY-SA v4.0] licence
* Indicate if you have changed the material in any way, and make any changes to our material available under the same licence

Also, please do let us know if you find this book useful!
