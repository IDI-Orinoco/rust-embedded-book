# Interrupciones

Las interrupciones difieren de las excepciones en varios aspectos, pero su funcionamiento y uso es muy similar y también son manejadas por el mismo controlador de interrupciones. Mientras que las excepciones están definidas por la arquitectura Cortex-M, las interrupciones son siempre implementaciones específicas del proveedor (y a menudo incluso del chip), tanto en nombre como en funcionalidad.

Las interrupciones permiten una gran flexibilidad que debe tenerse en cuenta cuando se intenta utilizarlas de forma avanzada. No cubriremos esos usos en este libro, sin embargo es una buena idea tener en mente lo siguiente:

- Las interrupciones tienen prioridades programables que determinan el orden de ejecución de sus manejadores.
- Las interrupciones pueden anidarse y priorizarse, es decir, la ejecución de un manejador de interrupción puede ser interrumpida por otra interrupción de mayor prioridad.
- En general, el motivo por el que se produce la interrupción debe borrarse para evitar que re-entre en el manejador de interrupciones indefinidamente.

Los pasos generales de inicialización en tiempo de ejecución son siempre los mismos:

- Configurar el(los) periférico(s) para generar peticiones de interrupción en las ocasiones deseadas
- Establecer la prioridad deseada del manejador de interrupciones en el controlador de interrupciones
- Habilitar el manejador de interrupciones en el controlador de interrupciones

De forma similar a las excepciones, la _crate_ `cortex-m-rt` proporciona un atributo [`interrupt`] para declarar los manejadores de interrupciones. Las interrupciones disponibles (y su posición en la tabla de manejadores de interrupción) se generan automáticamente a través de `svd2rust` a partir de una descripción SVD.

[`interrupt`]: https://docs.rs/cortex-m-rt-macros/0.1.5/cortex_m_rt_macros/attr.interrupt.html

``` rust,ignore
// Interrupt handler for the Timer2 interrupt
#[interrupt]
fn TIM2() {
    // ..
    // Clear reason for the generated interrupt request
}
```

Los manejadores de interrupciones parecen funciones simples (excepto por la falta de argumentos) similares a los manejadores de excepciones. Sin embargo, no pueden ser llamados directamente por otras partes del firmware debido a las convenciones especiales de llamada. Sin embargo, es posible generar peticiones de interrupción en el software para desencadenar un desvío al manejador de interrupciones.

De forma similar a los manejadores de excepciones, también es posible declarar variables `static mut` dentro de los manejadores de interrupciones para mantener el estado _safe_.

``` rust,ignore
#[interrupt]
fn TIM2() {
    static mut COUNT: u32 = 0;

    // `COUNT` has type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

Para una descripción más detallada sobre los mecanismos demostrados aquí, por favor ve a la [sección de excepciones].

[sección de excepciones]: ./exceptions.md
