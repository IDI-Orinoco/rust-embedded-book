# Introducción

Bienvenido al Libro de Rust Embebido: Un libro introductorio sobre el uso del lenguaje de programación Rust en sistemas embebidos "Bare Metal", como los microcontroladores.

## A quién va dirigido Rust Embebido

Rust Embebido es para todos aquellos que quieran hacer programación embebida aprovechando los conceptos de alto nivel y las garantías de seguridad que ofrece el lenguaje Rust.
(Ver también [Para quién es Rust](https://doc.rust-lang.org/book/ch00-00-introduction.html))

## Alcance

Los objetivos de este libro son:

- Poner a los desarrolladores al día con el desarrollo de Rust embebido, es decir, cómo configurar un entorno de desarrollo.

- Compartir las mejores prácticas _actuales_ sobre el uso de Rust para el desarrollo embebido, es decir, cómo utilizar mejor las características del lenguaje Rust para escribir un software embebido más correcto.

- Servir como un recetario en algunos casos. Por ejemplo, ¿Cómo puedo mezclar C y Rust en un solo proyecto?

Este libro intenta ser lo más general posible, pero para facilitar las cosas tanto a los lectores como a los escritores, utiliza la arquitectura ARM Cortex-M en todos sus ejemplos. Sin embargo, el libro no asume que el lector esté familiarizado con esta arquitectura en particular y explica detalles particulares de esta arquitectura cuando es necesario.

## A quién va dirigido este libro

Este libro está dirigido a personas con algún tipo de experiencia en sistemas embebidos o con algún tipo de experiencia en Rust, sin embargo creemos que todos los curiosos de la programación en Rust embebido pueden obtener algo de este libro. Para aquellos que no tengan ningún conocimiento previo, les sugerimos que lean la sección "Supuestos y Prerrequisitos" y se pongan al día con los conocimientos que les faltan para sacar más provecho del libro y mejorar su experiencia de lectura. Puedes consultar la sección "Otros recursos" para encontrar recursos sobre temas en los que quieras ponerte al día.

### Supuestos y prerrequisitos

- Te sientes cómodo usando el lenguaje de programación Rust, y has escrito, ejecutado y depurado aplicaciones Rust en un entorno de escritorio. También debes estar familiarizado con los modismos de la [edición 2018], ya que este libro está dirigido a Rust 2018.

[edición 2018]: https://doc.rust-lang.org/edition-guide/

- Te sientes cómodo desarrollando y depurando sistemas embebidos en otro lenguaje como C, C++ o Ada, y estás familiarizado con conceptos como:
  - Compilación cruzada
  - Periféricos mapeados en memoria
  - Interrupciones
  - Interfaces comunes como I2C, SPI, Serial, etc.

### Otros recursos

Si no estás familiarizado con nada de lo mencionado anteriormente o si quieres más información sobre un tema específico mencionado en este libro, puede que algunos de estos recursos te resulten útiles.

| Tema                               | Recurso                                                | Descripción                                                                                   |
| ---------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Rust                               | [Rust Book]                                            | Si aún no te sientes cómodo con Rust, te sugerimos que leas este libro.                       |
| Rust, Embebido                     | [Discovery Book]                                       | Si nunca has hecho nada de programación embebida, este libro podría ser un mejor comienzo.    |
| Rust, Embebido                     | [Embedded Rust Bookshelf]                              | Aquí puedes encontrar otros recursos proporcionados por el Grupo de Trabajo de Rust Embebido. |
| Rust, Embebido                     | [Embedonomicon]                                        | Los detalles más importantes de la programación embebida en Rust.                             |
| Rust, Embebido                     | [embedded FAQ]                                         | Preguntas frecuentes sobre Rust en un contexto embebido.                                      |
| Interrupciones                     | [Interrupt]                                            | -                                                                                             |
| IO/Periféricos mapeados en memoria | [Memory-mapped I/O]                                    | -                                                                                             |
| SPI, UART, RS232, USB, I2C, TTL    | [Stack Exchange about SPI, UART, and other interfaces] | -                                                                                             |

[rust book]: (https://doc.rust-lang.org/book/)
[discovery book]: (https://docs.rust-embedded.org/discovery/)
[embedded rust bookshelf]: (https://docs.rust-embedded.org)
[embedonomicon]: (https://docs.rust-embedded.org/embedonomicon/)
[embedded faq]: (https://docs.rust-embedded.org/faq.html)
[interrupt]: (https://en.wikipedia.org/wiki/Interrupt)
[memory-mapped i/o]: (https://en.wikipedia.org/wiki/Memory-mapped_I/O)
[stack exchange about spi, uart, and other interfaces]: (https://electronics.stackexchange.com/questions/37814/usart-uart-rs232-usb-spi-i2c-ttl-etc-what-are-all-of-these-and-how-do-th)

### Traducciones

Este libro ha sido traducido por generosos voluntarios. Si quieres que tu traducción aparezca aquí, por favor abre una PR para añadirla.

- [Japonés](https://tomoyuki-nakabayashi.github.io/book/)
  ([repositorio](https://github.com/tomoyuki-nakabayashi/book))

- [Chino](https://xxchang.github.io/book/)
  ([repositorio](https://github.com/XxChang/book))

## Cómo utilizar este libro

En general, este libro supone que se lee de principio a fin. Los capítulos posteriores se basan en los conceptos de los capítulos anteriores, y es posible que los capítulos anteriores no profundicen en los detalles de un tema, sino que vuelvan a tratar el tema en un capítulo posterior.

Este libro utilizará la tarjeta de desarrollo [STM32F3DISCOVERY] de STMicroelectronics para la mayoría de los ejemplos que contiene. Esta tarjeta está basada en la arquitectura ARM Cortex-M, y aunque la funcionalidad básica es la misma en la mayoría de las CPUs basadas en esta arquitectura, los periféricos y otros detalles de implementación de los microcontroladores son diferentes entre los distintos proveedores, y a menudo incluso diferentes entre las familias de microcontroladores del mismo proveedor.

Por esta razón, sugerimos adquirir la tarjeta de desarrollo [STM32F3DISCOVERY] para seguir los ejemplos de este libro.

[stm32f3discovery]: http://www.st.com/en/evaluation-tools/stm32f3discovery.html

## Contribución a este libro

El trabajo de este libro está coordinado en [este repositorio] y es desarrollado principalmente por el [equipo de recursos].

[este repositorio]: https://github.com/rust-embedded/book
[equipo de recursos]: https://github.com/rust-embedded/wg#the-resources-team

Si tienes problemas para seguir las instrucciones de este libro o encuentras que alguna sección del libro no es lo suficientemente clara o difícil de seguir, entonces eso es un error y debe ser reportado en el [rastreador de problemas] de este libro.

[rastreador de problemas]: https://github.com/rust-embedded/book/issues/

Las PR para corregir errores tipográficos y añadir nuevos contenidos son muy bienvenidas.

## Reutilización de este material

Este libro se distribuye bajo las siguientes licencias:

- Los ejemplos de código y los proyectos independientes de Cargo contenidos en este libro están licenciados bajo los términos de la [Licencia MIT] y la [Licencia Apache v2.0].
- La prosa escrita, las imágenes y los diagramas contenidos en este libro están bajo los términos de la licencia Creative Commons [CC-BY-SA v4.0].

[licencia mit]: https://opensource.org/licenses/MIT
[licencia apache v2.0]: http://www.apache.org/licenses/LICENSE-2.0
[cc-by-sa v4.0]: https://creativecommons.org/licenses/by-sa/4.0/legalcode

TL;DR: Si quieres utilizar nuestro texto o imágenes en tu trabajo, tienes que:

- Dar el crédito apropiado (es decir, mencionar este libro en su diapositiva, y proporcionar un enlace a la página correspondiente)
- Proporcionar un enlace a la licencia [CC-BY-SA v4.0]
- Indicar si ha modificado el material de alguna manera, y hacer que cualquier cambio en nuestro material esté disponible bajo la misma licencia

Además, por favor, haznos saber si este libro te resulta útil.
