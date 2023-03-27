# macOS

Todas las herramientas pueden ser instaladas usando [Homebrew] o [MacPorts]:

[homebrew]: http://brew.sh/
[macports]: https://www.macports.org/

## Instala las herramientas con [Homebrew]

```text
$ # GDB
$ brew install armmbed/formulae/arm-none-eabi-gcc

$ # OpenOCD
$ brew install openocd

$ # QEMU
$ brew install qemu
```

> **NOTA** Si OpenOCD se cae puede que necesites instalar la última versión usando:

```text
$ brew install --HEAD openocd
```

## Instala las herramientas con [MacPorts]

```text
$ # GDB
$ sudo port install arm-none-eabi-gcc

$ # OpenOCD
$ sudo port install openocd

$ # QEMU
$ sudo port install qemu
```

¡Eso es todo! anda a la [siguiente sección].

[siguiente sección]: verify.md
