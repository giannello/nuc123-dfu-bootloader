# NUC123 DFU Bootloader

## Why?

As a requirement for porting QMK to my Metadot Das Keyboard 4 Professional, it's quite handy to have a bootloader
capable of programming the NUC123 APROM, without having to resort to SWD/JTAG.

## How?

Nuvoton provides a working example as part of their NUC123BSP, but getting it to compile was a hassle.

This project can be imported in NuEclipse, built, and will build a `.hex` file that can be flashed via SWD.

This project differs from the upstream sample as following:
* uses `picolibc` instead of `newlib-nano`, for smaller binaries
* uses the `picolibc.ld` linker script, with the correct build parameters for `NUC123SD4AE`
* you can enter DFU mode by keeping the "Suspend" key pressed while plugging the keyboard
* once in DFU mode, you can resume the boot process by pressing the "Mute" key

## Requirements

* a working installation of `arm-none-eabi-gcc` and `picolibc` (`apt install gcc-arm-none-eabi picolibc-arm-none-eabi` on debian testing works fine)

## Flashing

Warning: OpenOCD doesn't currently support the `NUC123SD4AE` part. Apply the [following patch](https://review.openocd.org/c/openocd/+/7417) and recompile OpenOCD.
Bigger warning: the following operation will FULLY ERASE your device. Only attempt if you know what you are doing. After flashing this bootloader, you need to
use `dfu-tool` to flash a suitable firmware. There is currently no public, working QMK port.


Use OpenOCD with the following configuration: https://gist.github.com/elfmimi/3b5f8e42c594c670adc1422d7a38ccf7
Save a copy of the config bits configuration (`Config0` and `Config1`):
```
$ openocd -f NUCxxx.cfg -c ReadConfigRegs -c exit
Open On-Chip Debugger 0.11.0-rc2
[...]
Reading Configuration registers.
Device ID (PDID)    :0x10012315 (NUC123SD4AE0)
target halted due to debug-request, current mode: Thread 
xPSR: 0x81000000 pc: 0x0000426a psp: 0x20000b4c
Config0 (0x00300000):0xF7FFFF7E
Config1 (0x00300004):0x0001C000
```

Flash with
```
$ openocd -f NUCxxx.cfg -c "WriteConfigRegs 0xFFFFFF7E 0xFFFFFFFF; ReadConfigRegs; program nuc123-dfu-bootloader.hex 0x100000; SysReset ldrom run; exit"
Open On-Chip Debugger 0.11.0-rc2
[...]
target halted due to debug-request, current mode: Thread 
xPSR: 0x41000000 pc: 0x0000366a psp: 0x20000e48
Device ID (PDID)    :0x10012315 (NUC123SD4AE0)
Configuration registers have been Written
Reading Configuration registers.
Device ID (PDID)    :0x10012315 (NUC123SD4AE0)
Config0 (0x00300000):0xFFFFFF7E
Config1 (0x00300004):0xFFFFFFFF
target halted due to debug-request, current mode: Thread 
xPSR: 0xc1000000 pc: 0x00000190 msp: 0x20000400
** Programming Started **
Info : Device ID: 0x10012315
Info : Device Name: NUC123SD4AE
Info : bank base = 0x00000000, size = 0x00011000
Info : Device ID: 0x10012315
Info : Device Name: NUC123SD4AE
Info : bank base = 0x0001f000, size = 0x00001000
Info : Device ID: 0x10012315
Info : Device Name: NUC123SD4AE
Info : bank base = 0x00100000, size = 0x00001000
Warn : Adding extra erase range, 0x00100fd0 .. 0x00100fff
Info : Nuvoton NuMicro: Sector Erase ... (0 to 7)
Info : Nuvoton NuMicro: Flash Write ...
** Programming Finished **
```

`CONFIG0` and `CONFIG1` are still a bit messy, haven't found the time to play around with them properly.
* Setting `CONFIG1` to `0x0001C000`, and then flashing via DFU, ends with an unbootable device
* Setting `CONFIG0` to `0xx7xxxxxx` activates a watchdog, and the device will restart periodically unless properly handled (not supported in my QMK port yet)
* Setting `CONFIG0` to `0xxxxxxxFx` enabled booting from APROM, and the bootloader will be entirely bypassed
* Setting `CONFIG0` to `0xxxxxxx7x` enabled booting from LDROM, and the bootloader will be executed

## References

OpenOCD configuration https://gist.github.com/elfmimi/3b5f8e42c594c670adc1422d7a38ccf7

NUC123 makefile https://gist.github.com/elfmimi/e28761a42e14940b0c5f29656316e734

sample bootloader https://github.com/JCSTwo/Nuvoton-Application

NUC123 supporting libraries and sample code https://github.com/OpenNuvoton/NUC123BSP

Many thanks to @elfmimi for the precious tips
