# Predictibilidad

<a id="c-ctor"></a>
## Se utilizan constructores en lugar de _traits_ de extensión (C-CTOR)

Todos los periféricos a los que la HAL añade funcionalidad deben envolverse en un nuevo tipo, incluso si no se requieren campos adicionales para esa funcionalidad.

Deben evitarse los _traits_ de extensión implementados para el periférico directamente.

<a id="c-inline"></a>
## Los métodos están decorados con `#[inline]` cuando es apropiado (C-INLINE)

El compilador de Rust no realiza por defecto inlining completo a través de los límites de las _crates_. Como las aplicaciones embebidas son sensibles a aumentos inesperados del tamaño del código, `#[inline]` debería ser usado para guiar al compilador de la siguiente manera:

- Todas las funciones "pequeñas" deben marcarse como `#[inline]`. Lo que se califica como "pequeño" es subjetivo, pero generalmente todas las funciones que se espera que compilen en secuencias de instrucciones de un solo dígito se califican como pequeñas.
- Las funciones que es muy probable que tomen valores constantes como parámetros deben marcarse como `#[inline]`. Esto permite al compilador calcular incluso la complicada lógica de inicialización en tiempo de compilación, siempre que se conozcan las entradas de la función.
