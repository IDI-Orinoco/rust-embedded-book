# Abstracciones de Costo Cero

Los estados de tipo también son un excelente ejemplo de abstracciones de costo cero: la capacidad de mover ciertos comportamientos para compilar el tiempo de ejecución o análisis. Estos estados de tipo no contienen datos reales y, en su lugar, se utilizan como marcadores. Dado que no contienen datos, no tienen una representación real en la memoria en tiempo de ejecución:

```rust,ignore
use core::mem::size_of;

let _ = size_of::<Enabled>();    // == 0
let _ = size_of::<Input>();      // == 0
let _ = size_of::<PulledHigh>(); // == 0
let _ = size_of::<GpioConfig<Enabled, Input, PulledHigh>>(); // == 0
```

## Tipos de tamaño cero

```rust,ignore
struct Enabled;
```

Las estructuras definidas de esta manera se denominan tipos de tamaño cero, ya que no contienen datos reales. Aunque estos tipos actúan como "reales" en tiempo de compilación, puede copiarlos, moverlos, tomar referencias a ellos, etc., sin embargo, el optimizador los eliminará por completo.

En este fragmento de código:

```rust,ignore
pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
    self.periph.modify(|_r, w| w.input_mode().high_z());
    GpioConfig {
        periph: self.periph,
        enabled: Enabled,
        direction: Input,
        mode: HighZ,
    }
}
```

El GpioConfig que devolvemos nunca existe en tiempo de ejecución. Llamar a esta función generalmente se reducirá a una sola instrucción de ensamblaje: almacenar un valor de registro constante en una ubicación de registro. Esto significa que la interfaz de estado de tipo que hemos desarrollado es una abstracción de costo cero: no usa más CPU, RAM o espacio de código para rastrear el estado de `GpioConfig`, y representa el mismo código de máquina como un acceso de registro directo.

## Anidado

En general, estas abstracciones se pueden anidar tan profundamente como desee. Siempre que todos los componentes utilizados sean tipos de tamaño cero, la estructura completa no existirá en tiempo de ejecución.

Para estructuras complejas o profundamente anidadas, puede ser tedioso definir todas las combinaciones posibles de estado. En estos casos, se pueden utilizar macros para generar todas las implementaciones.
