# Lista de chequeo de Patrones de Diseño HAL

- **Nomenclatura** _(la **crate** se alinea con las convenciones de nomenclatura de Rust)_
  - [ ] La _crate_ tiene el nombre apropiado ([C-CRATE-NAME])
- **Interoperabilidad** _( la **crate** interactúa bien con otras funcionalidades de la librería)_
  - [ ] Los tipos Wrapper proporcionan un método destructor ([C-FREE])
  - [ ] Las HAL reexportan su _crate_ de acceso a registros ([C-REEXPORT-PAC])
  - [ ] Los tipos implementan los _traits_ `embedded-hal` ([C-HAL-TRAITS])
- **Predictibilidad** _(**crate** permite código legible que actúa como parece)_
  - [ ] Se utilizan constructores en lugar de _traits_ de extensión ([C-CTOR])
- **Interfaces GPIO** _(Las Interfaces GPIO siguen un patrón común)_
  - [ ] Los tipos de pin son de tamaño cero por defecto ([C-ZST-PIN])
  - [ ] Los tipos de pin proporcionan métodos para borrar el pin y el puerto ([C-ERASED-PIN])
  - [ ] El estado de las pines debe codificarse como parámetros de tipo ([C-PIN-STATE])

[c-crate-name]: naming.html#c-crate-name
[c-free]: interoperability.html#c-free
[c-reexport-pac]: interoperabilidad.html#c-reexport-pac
[c-hal-traits]: interoperabilidad.html#c-hal-traits
[c-ctor]: predictability.html#c-ctor
[c-zst-pin]: gpio.md#c-zst-pin
[c-erased-pin]: gpio.md#c-erased-pin
[c-pin-state]: gpio.md#c-pin-state