# Portabilidad

En entornos embebidos, la portabilidad es un tema muy importante: Cada proveedor e incluso cada familia de un mismo fabricante ofrece diferentes periféricos y capacidades y, de forma similar, las formas de interactuar con los periféricos variarán.

Una forma común de igualar estas diferencias es a través de una capa llamada capa de Abstracción de Hardware o **HAL**.

> Las abstracciones de hardware son conjuntos de rutinas en software que emulan algunos detalles específicos de la plataforma, dando a los programas acceso directo a los recursos de hardware.
>
> Suelen permitir a los programadores escribir aplicaciones independientes del dispositivo y de alto rendimiento al proporcionar llamadas estándar del sistema operativo (SO) al hardware.
>
> _Wikipedia: [capa de abstracción de hardware]_

[capa de abstracción de hardware]: https://en.wikipedia.org/wiki/Hardware_abstraction

Los sistemas embebidos son un poco especiales en este sentido, ya que normalmente no tenemos sistemas operativos y software instalable por el usuario, sino imágenes de firmware que se compilan como un todo, así como una serie de otras limitaciones. Así que, aunque el enfoque tradicional tal y como lo define Wikipedia podría funcionar, probablemente no sea el enfoque más productivo para garantizar la portabilidad.

¿Cómo lo hacemos en Rust? Entra **embedded-hal**...

## ¿Qué es embedded-hal?

En pocas palabras es un conjunto de _traits_ que definen contratos de implementación entre **implementaciones de HAL**, **controladores** y **aplicaciones (o firmwares)**. Estos contratos incluyen tanto capacidades (es decir, si un _trait_ se implementa para un determinado tipo, la **implementación de HAL** proporciona una determinada capacidad) como métodos (es decir, si puedes construir un tipo implementando un _trait_, está garantizado que dispones de los métodos especificados en el _trait_).

Una estratificación típica podría ser la siguiente

![](../assets/rust_layers.svg)

Algunos de los traits definidos en **embedded-hal** son:

- GPIO (pines de entrada y salida)
- Comunicación serie
- I2C
- SPI
- Temporizadores/Cuentas atrás
- Conversión analógico-digital

La razón principal para tener los _traits_ **embedded-hal** y las crates que los implementan y utilizan es mantener la complejidad bajo control. Si se tiene en cuenta que una aplicación puede tener que implementar el uso del periférico en el hardware, así como la aplicación y, potencialmente, los controladores para los componentes de hardware adicionales, entonces debería ser fácil ver que la reutilización es muy limitada. Expresado matemáticamente, si **M** es el número de implementaciones de periféricos HAL y **N** el número de drivers, entonces si tuviéramos que reinventar la rueda para cada aplicación acabaríamos con **M\*N** implementaciones, mientras que usando la _API_ proporcionada por los _traits_ **embedded-hal** la complejidad de la implementación se acercará a **M+N**. Por supuesto hay beneficios adicionales que se pueden obtener, tales como menos ensayo-y-error debido a una API bien definida y lista para usar.

## Usuarios de embedded-hal

Como se ha dicho anteriormente hay tres usuarios principales de la HAL:

### Implementación HAL

Una implementación HAL proporciona la interfaz entre el hardware y los usuarios de los _traits_ HAL. Las implementaciones típicas constan de tres partes:

- Uno o más tipos específicos de hardware
- Funciones para crear e inicializar dicho tipo, a menudo proporcionando varias opciones de configuración (velocidad, modo de funcionamiento, pines de uso, etc.)
- Uno o más `trait` `impl` de _traits_ **embedded-hal** para ese tipo.

La implementación de **HAL** puede ser de varios tipos:

- Mediante acceso a hardware de bajo nivel, p.e. a través de registros.
- A través del sistema operativo, p.e. mediante el uso de `sysfs` en Linux
- A través de un adaptador, p.e. un simulador de tipos para pruebas unitarias.
- A través de controladores para adaptadores de hardware, p.e. multiplexores I2C o expansores GPIO.

### Controlador

Un driver implementa un conjunto de funcionalidades personalizadas para un componente interno o externo, conectado a un periférico que implementa los _traits_ embedded-hal. Ejemplos típicos para tales controladores incluyen varios sensores (temperatura, magnetómetro, acelerómetro, luz), dispositivos de visualización (matrices de LED, pantallas LCD) y actuadores (motores, transmisores).

Un controlador tiene que ser inicializado con una instancia de tipo que implemente un determinado `trait` del embedded-hal que se asegura a través de _trait bound_ y proporciona su propia instancia de tipo con un conjunto personalizado de métodos que permiten interactuar con el dispositivo controlado.

### Aplicación

La aplicación une las distintas partes y asegura que se consigue la funcionalidad deseada. Al portar entre diferentes sistemas, esta es la parte que requiere más esfuerzos de adaptación, ya que la aplicación necesita inicializar correctamente el hardware real a través de la implementación HAL y la inicialización de diferentes hardware difiere, a veces drásticamente. También la elección del usuario juega a menudo un papel importante, ya que los componentes pueden conectarse físicamente a diferentes terminales, los buses de hardware a veces necesitan hardware externo para ajustarse a la configuración o hay que hacer diferentes compensaciones en el uso de los periféricos internos (p.e. se dispone de múltiples temporizadores con diferentes capacidades o los periféricos entran en conflicto con otros).
