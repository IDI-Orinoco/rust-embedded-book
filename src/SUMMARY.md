# Contenido

<!--

Definition of the organization of this book is still a work in process.

Refer to https://github.com/rust-embedded/book/issues for
more information and coordination

-->

- [Introducción](./intro/index.md)
    - [Hardware](./intro/hardware.md)
    - [`no_std`](./intro/no-std.md)
    - [Herramientas](./intro/tooling.md)
    - [Instalación](./intro/install.md)
        - [Linux](./intro/install/linux.md)
        - [MacOS](./intro/install/macos.md)
        - [Windows](./intro/install/windows.md)
        - [Verify Installation](./intro/install/verify.md)
- [Cómo empezar](./start/index.md)
  - [QEMU](./start/qemu.md)
  - [Hardware](./start/hardware.md)
  - [Registros mapeados en memoria](./start/registers.md)
  - [Semihosting](./start/semihosting.md)
  - [Pánico](./start/panicking.md)
  - [Excepciones](./start/exceptions.md)
  - [Interrupciones](./start/interrupts.md)
  - [IO](./start/io.md)
- [Periféricos](./peripherals/index.md)
    - [Un primer intento en Rust](./peripherals/a-first-attempt.md)
    - [El comprobador de préstamos](./peripherals/borrowck.md)
    - [Singletons](./peripherals/singletons.md)
- [Garantías estáticas](./static-guarantees/index.md)
    - [Programación por Typestate](./static-guarantees/typestate-programming.md)
    - [Periféricos como máquinas de estado](./static-guarantees/state-machines.md)
    - [Contratos de diseño](./static-guarantees/design-contracts.md)
    - [Abstracciones de costo cero](./static-guarantees/zero-cost-abstractions.md)
- [Portabilidad](./portability/index.md)
- [Concurrencia](./concurrency/index.md)
- [Colecciones](./collections/index.md)
- [Design Patterns](./design-patterns/index.md)
    - [HALs](./design-patterns/hal/index.md)
        - [Lista de control](./design-patterns/hal/checklist.md)
        - [Nomenclatura](./design-patterns/hal/naming.md)
        - [Interoperabilidad](./design-patterns/hal/interoperability.md)
        - [Previsibilidad](./design-patterns/hal/predictability.md)
        - [GPIO](./design-patterns/hal/gpio.md)
- [Consejos para desarrolladores de C embebido](./c-tips/index.md)
    <!-- TODO: Define Sections -->
- [Interoperabilidad](./interoperability/index.md)
    - [Un poco de C con tu Rust](./interoperability/c-with-rust.md)
    - [Un poco de Rust con tu C](./interoperability/rust-with-c.md)
- [Temas sin clasificar](./unsorted/index.md)
  - [Optimizaciones: El equilibrio entre velocidad y tamaño](./unsorted/speed-vs-size.md)
  - [Realización de funciones matemáticas](./unsorted/math.md)

---

[Apéndice A: Glosario](./appendix/glossary.md)
