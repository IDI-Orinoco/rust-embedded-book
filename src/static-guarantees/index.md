# Garantías Estáticas

El sistema de tipos de Rust evita carreras de datos en tiempo de compilación (consulte los _traits_ [`Send`] y [`Sync`]). El sistema de tipos también se puede usar para verificar otras propiedades en tiempo de compilación; reduciendo la necesidad de controles de tiempo de ejecución en algunos casos.

[`Send`]: https://doc.rust-lang.org/core/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/core/marker/trait.Sync.html

Cuando se aplican a programas embebidos, estas _comprobaciones estáticas_ se pueden usar, por ejemplo, para hacer cumplir que la configuración de las interfaces de E/S se realice correctamente. Por ejemplo, se puede diseñar una API donde solo es posible inicializar una interfaz serial configurando primero los pines que utilizará la interfaz.

También se puede verificar estáticamente que las operaciones, como establecer un pin bajo, solo se pueden realizar en periféricos configurados correctamente. Por ejemplo, intentar cambiar el estado de salida de un pin configurado en modo de entrada flotante generaría un error de compilación.

Y, como se vio en el capítulo anterior, el concepto de propiedad se puede aplicar a los periféricos para garantizar que solo ciertas partes de un programa puedan modificar un periférico. Este _control de acceso_ hace que sea más fácil razonar sobre el software en comparación con la alternativa de tratar los periféricos como un estado mutable global.
