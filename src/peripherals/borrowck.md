## Estado Global Mutable

Desafortunadamente, el hardware no es más que un estado global mutable, lo que puede resultar muy aterrador para un desarrollador de Rust. El hardware existe independientemente de las estructuras del código que escribimos y puede ser modificado en cualquier momento por el mundo real.

## ¿Cuáles deberían ser nuestras reglas?

¿Cómo podemos interactuar de forma fiable con estos periféricos?

1. Utilice siempre métodos `volatile` para leer o escribir en la memoria periférica, ya que puede cambiar en cualquier momento
2. En el software, deberíamos poder compartir cualquier número de accesos de solo lectura a estos periféricos
3. Si algún software debe tener acceso de lectura y escritura a un periférico, debe contener la única referencia a ese periférico

## El Verificador de Préstamos (The Borrow Checker)

¡Las últimas dos de estas reglas suenan sospechosamente similares a lo que ya hace el Borrow Checker!

¿Imagínese si pudiéramos pasar la propiedad de estos periféricos u ofrecerles referencias inmutables o mutables?

Bueno, podemos, pero para el Borrow Checker, necesitamos tener exactamente una instancia de cada periférico, para que Rust pueda manejar esto correctamente. Bueno, afortunadamente en el hardware, solo hay una instancia de cualquier periférico dado, pero ¿cómo podemos exponer eso en la estructura de nuestro código?
