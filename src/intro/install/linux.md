# Linux

Aquí están los comandos de instalación para algunas distribuciones de Linux.

## Paquetes

- Ubuntu 18.04 o más reciente / Debian stretch o más reciente

> **NOTA** `gdb-multiarch` es el comando GDB que usarás para depurar tus programas ARM Cortex-M

<!-- Debian stretch -->
<!-- GDB 7.12 -->
<!-- OpenOCD 0.9.0 -->
<!-- QEMU 2.8.1 -->

<!-- Ubuntu 18.04 -->
<!-- GDB 8.1 -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.11.1 -->

``` console
sudo apt install gdb-multiarch openocd qemu-system-arm
```

- Ubuntu 14.04 y 16.04

> **NOTA** `arm-none-eabi-gdb` es el comando GDB que usarás para depurar tus programas ARM Cortex-M

<!-- Ubuntu 14.04 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.7.0 (?) -->
<!-- QEMU 2.0.0 (?) -->

``` console
sudo apt install gdb-arm-none-eabi openocd qemu-system-arm
```

- Fedora 27 o más reciente

<!-- Fedora 27 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.10.2 -->

``` console
sudo dnf install gdb openocd qemu-system-arm
```

- Arch Linux

> **NOTA** `arm-none-eabi-gdb` es el comando GDB que usarás para depurar programas ARM Cortex-M

``` console
sudo pacman -S arm-none-eabi-gdb qemu-arch-extra openocd
```

## reglas udev

Esta regla le permite utilizar OpenOCD con la tarjeta Discovery sin privilegios de root.

Cree el archivo `/etc/udev/rules.d/70-st-link.rules` con el contenido que se muestra a continuación.

``` text
# STM32F3DISCOVERY rev A/B - ST-LINK/V2
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", TAG+="uaccess"

# STM32F3DISCOVERY rev C+ - ST-LINK/V2-1
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", TAG+="uaccess"
```

Y luego, recargar todas las reglas udev con:

``` console
sudo udevadm control --reload-rules
```

Si tenías la tarjeta conectada al portátil, desconéctala y vuelve a conectarla.

Puedes comprobar los permisos ejecutando este comando:

``` console
lsusb
```

Que debería mostrar algo como

```text
(..)
Bus 001 Device 018: ID 0483:374b STMicroelectronics ST-LINK/V2.1
(..)
```

Tome nota de los números de bus y dispositivo. Use esos números para crear una ruta como `/dev/bus/usb/<bus>/<device>`. Luego usa esta ruta así:

``` console
ls -l /dev/bus/usb/001/018
```

```text
crw-------+ 1 root root 189, 17 Sep 13 12:34 /dev/bus/usb/001/018
```

```console
getfacl /dev/bus/usb/001/018 | grep user
```

```text
user::rw-
user:you:rw-
```

El signo `+` añadido a los permisos indica la existencia de un permiso ampliado. El comando `getfacl` le dice al usuario `you` que puede hacer uso de este dispositivo.

Ahora, ve a la [siguiente sección].

[siguiente sección]: verify.md