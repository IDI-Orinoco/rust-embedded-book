# Windows

## `arm-none-eabi-gdb`

ARM proporciona instaladores `.exe` para Windows. Toma uno de [aquí][gcc], y sigue las instrucciones. Justo antes de que termine el proceso de instalación, marca/selecciona la opción "Add path to environment variable". Luego verifica que las herramientas están en tu `%PATH%`:

``` text
$ arm-none-eabi-gdb -v
GNU gdb (GNU Tools for Arm Embedded Processors 7-2018-q2-update) 8.1.0.20180315-git
(..)
```

[gcc]: https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads

## OpenOCD

No hay una versión binaria oficial de OpenOCD para Windows, pero si no estás de humor para compilarla tú mismo, el proyecto xPack proporciona una distribución binaria, [aquí][openocd]. Sigue las instrucciones de instalación proporcionadas. A continuación, actualiza tu variable de entorno `%PATH%` para incluir la ruta donde se instalaron los binarios. (`C:\Users\USERNAME\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.10.0-13.1\.content\bin\`, si usaste la instalación fácil)

[openocd]: https://xpack.github.io/openocd/

Comprueba que OpenOCD está en tu `%PATH%` con:

``` text
$ openocd -v
Open On-Chip Debugger 0.10.0
(..)
```

## QEMU

Baja QEMU desde [el sitio web oficial][qemu].

[qemu]: https://www.qemu.org/download/#windows

## Controlador USB ST-LINK

También necesitarás instalar [este controlador USB] o OpenOCD no funcionará. Sigue las instrucciones del instalador y asegúrate de instalar la versión correcta (32-bit o 64-bit) del controlador.

[este controlador usb]: http://www.st.com/en/embedded-software/stsw-link009.html

¡Eso es todo! anda a la [siguiente sección].

[siguiente sección]: verify.md
