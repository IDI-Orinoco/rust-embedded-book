# Verificar la instalación

En esta sección comprobamos que algunas de las herramientas / controladores necesarios se han instalado y configurado correctamente.

Conecta tu computador portátil / PC a la tarjeta discovery mediante un cable micro USB. La placa tiene dos conectores USB; utilice el etiquetado "USB ST-LINK" que se encuentra en el centro del borde de la placa.

Comprueba también que el cabezal ST-LINK está lleno. Ve la imagen siguiente; el cabezal ST-LINK está resaltado.

<p align="center">
<img title="Connected discovery board" src="../../assets/verify.jpeg">
</p>

Ahora ejecuta el siguiente comando:

```console
openocd -f interface/stlink.cfg -f target/stm32f3x.cfg
```

> **NOTA**: Las versiones antiguas de openocd, incluida la versión 0.10.0 de 2017, no contienen el nuevo (y preferible) archivo `interface/stlink.cfg`; en su lugar, es posible que tenga que utilizar `interface/stlink-v2.cfg` o `interface/stlink-v2-1.cfg`..

Debería obtener la siguiente salida y el programa debería bloquear la consola:

```text
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.919881
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

Puede que el contenido no coincida exactamente, pero deberías obtener la última línea acerca de los _breakpoints_ y _watchpoints_. Si lo obtienes, finaliza el proceso OpenOCD y pasa a la [siguiente sección].

[siguiente sección]: ../../start/index.md

Si no obtienes la línea "breakpoints" entonces intenta uno de los siguientes comandos.

```console
openocd -f interface/stlink-v2.cfg -f target/stm32f3x.cfg
```

```console
openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg
```

Si uno de esos comandos funciona, significa que tienes una revisión de hardware antigua de la tarjeta discovery. Esto no será un problema, pero recuerda este hecho ya que necesitarás configurar las cosas de manera diferente más adelante. Puedes pasar a la [siguiente sección].

Si ninguno de los comandos funciona como usuario normal, intenta ejecutarlos con permisos de root (por ejemplo, `sudo openocd ..`). Si los comandos funcionan con permisos de root, comprueba que las [reglas udev] se han configurado correctamente.

[reglas udev]: linux.md#udev-rules

Si has llegado a este punto y OpenOCD no funciona, abre [una incidencia] y te ayudaremos.

[una incidencia]: https://github.com/rust-embedded/book/issues
