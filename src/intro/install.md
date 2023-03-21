# Instalación de las herramientas

Esta página contiene instrucciones de instalación independientes del sistema operativo para algunas de las herramientas:

### Rust Toolchain

Instala rustup siguiendo las instrucciones en [https://rustup.rs](https://rustup.rs).

**NOTA** Asegúrate de que tienes una versión del compilador igual o más reciente que `1.31`. `rustc -V` debería devolver una fecha más reciente que la que se muestra a continuación.

```text
$ rustc -V
rustc 1.31.1 (b6c32da9b 2018-12-18)
```

Por cuestiones de ancho de banda y uso de disco, la instalación por defecto sólo soporta compilación nativa. Para añadir soporte de compilación cruzada para las arquitecturas ARM Cortex-M elija uno de los siguientes objetivos de compilación. Para la tarjeta STM32F3DISCOVERY utilizada para los ejemplos de este libro, utilice el objetivo `thumbv7em-none-eabihf`.

Cortex-M0, M0+ y M1 (arquitectura ARMv6-M):

```console
rustup target add thumbv6m-none-eabi
```

Cortex-M3 (arquitectura ARMv7-M):

```console
rustup target add thumbv7m-none-eabi
```

Cortex-M4 y M7 sin punto flotante por hardware (arquitectura ARMv7E-M):

```console
rustup target add thumbv7em-none-eabi
```

Cortex-M4F y M7F con punto flotante por hardware (arquitectura ARMv7E-M):

```console
rustup target add thumbv7em-none-eabihf
```

Cortex-M23 (arquitectura ARMv8-M):

```console
rustup target add thumbv8m.base-none-eabi
```

Cortex-M33 y M35P (arquitectura ARMv8-M):

```console
rustup target add thumbv8m.main-none-eabi
```

Cortex-M33F y M35PF con punto flotante por hardware (arquitectura ARMv8-M):

```console
rustup target add thumbv8m.main-none-eabihf
```

### `cargo-binutils`

```text
cargo install cargo-binutils

rustup component add llvm-tools-preview
```

WINDOWS: prerrequisito tener instalado C++ Build Tools para Visual Studio 2019. https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16

### `cargo-generate`

Lo usaremos más adelante para generar un proyecto a partir de una plantilla.

```console
cargo install cargo-generate
```

Nota: en algunas distribuciones Linux (p.e. Ubuntu) puede que necesites instalar los paquetes `libssl-dev` y `pkg-config` antes de instalar cargo-generate.

### Instrucciones específicas por sistema operativo

Ahora sigue las instrucciones específicas para el SO que estés usando:

- [Linux](install/linux.md)
- [Windows](install/windows.md)
- [macOS](install/macos.md)
