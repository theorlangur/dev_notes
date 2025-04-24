# Various info regarding nrf52480 (specifically pro micro nrf52480)

## Memory map of the dev board
[here](https://learn.adafruit.com/introducing-the-adafruit-nrf52840-feather/hathach-memory-map)

## Entering UF2 bootloader mode (USB flash disk)
Need to pull `RST` to `GND` 2 times during 0.5 seconds.
If the board is already in some failed state (red LED is quickly pulsing) 
a single bridging of `RST` to `GND` is enough to enter `UF2 bootloader` mode.

## Erasing flash
Could not find a proper way of erasing the settings partitions. 
Tried making a `uf2` file that writes to the `settings_storage` partition addresses with zeroes.
Didn't work. Apparently here's what `ZMK` [does](https://zmk.dev/docs/troubleshooting/connection-issues#building-a-reset-firmware) in this case.
And building that `settings_reset` is just linking additionally a c-file `app/src/settings/reset_settings_on_start.c` with `SYS_INIT` macro call that schedules 
`zmk_settings_erase` call.
And according to [this](https://zmk.dev/docs/troubleshooting/connection-issues#reset-split-keyboard-procedure) it should be applied (flashed) once
and then non-reset firmware should be flashed.

## Powering up the board
Apparently aside from `B+` and `B-` you can use `VDD` and `GND` on the back of the board near the `DIO` and `CLK` pins (needed for `SWD` debugging).
I guess this would be the way to power the board with a cell (or AA, AAA) battery (with some step up boost converter naturally)

## Reading from serial

With a simple `cat` app we can read printf's from the board as: 
```
cat /dev/<tty device>
```
example:
```
cat /dev/ttyACM0
```

## How to get command line with a proper environment after installing NRF Connect DK via VSCode

1. on the 'nRF Connect' extension tab click on `Manage toolchains`
2. select `Open Terminal Profile` in the opened dropdown
3. in the terminal execute (more info [here](https://docs.nordicsemi.com/bundle/nrfutil/page/nrfutil-toolchain-manager/nrfutil-toolchain-manager_0.14.1.html)): 
```
nrfutil toolchain-manager env --as-script > ~/path/to/env_file.sh
```
4. in the exported file add also export of the following env variable:
```
export ZEPHYR_BASE=/path/to/ncs/toolchain/zephyr (e.g. /home/user/ncs/v2.9.0/zephyr)
```


## Using LLVM tools (clang, lld) to build

### Configuring environment + project's config
It makes sense to make a copy of the environment script and: 
1. set the value of the `ZEPHYR_TOOLCHAIN_VARIANT` to `llvm`
2. in the application that is built, in the project's conf file (default: `prj.conf`):
   set `CONFIG_LLVM_USE_LLD=y`

### Fixing include/linking issues
I made llvm toolchain work with a picolibc so far (also C++). As a result the following cmake toolchain file
needs to be provided to the build process:

```
set(triple arm-zephyr-eabi)
set(arch thumb)
set(instr v7e-m)
set(XCMAKE_SYSROOT /home/user/ncs/toolchains/b77d8c1312/opt/zephyr-sdk/${triple}/${triple})
set(XMAKE_SYSROOT /home/user/ncs/toolchains/b77d8c1312/opt/zephyr-sdk/${triple})

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -I${XCMAKE_SYSROOT}/include/c++/12.2.0")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -I${XCMAKE_SYSROOT}/include/c++/12.2.0/${triple}/${arch}/${instr}/nofp")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${XMAKE_SYSROOT}/picolibc/${triple}/lib/${arch}/${instr}/nofp")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${XMAKE_SYSROOT}/picolibc/lib/gcc/${triple}/12.2.0/${arch}/${instr}/nofp")
```

Such a toolchain file can be passed by using a `--toolchain <path/to/cmake_toolchain_file>` option.

Sometimes it might be required to explicitly state in the `prj.conf`: `CONFIG_PICOLIBC=y`

Commandline west invocation example to configure and build the project:
```
west build --build-dir build . --sysbuild --pristine --board promicro_nrf52840/nrf52840/uf2 -- -DNCS_TOOLCHAIN_VERSION=NONE -DCONF_FILE=prj-pico.conf --toolchain /home/user/ncs/toolchains/llvm_toolchain.cmake
```

`--sysbuild` strictly speaking is not needed as it's implicit, but still.
`-DCONF_FILE=...` allows to use an arbitrary file as a configuration, not just a default `prj.conf`

Also it's important in the `CMakeLists.txt` to explicitly link against `c` library like:
```
target_link_libraries(app PRIVATE c)
```

otherwise there will be linker errors about undefined symbols.

#### LLD Linking changes
in the `cmake/linker/lld/linker_libraries.cmake` file we need to comment out the line:
`set_linker_property(TARGET linker PROPERTY c++_library "-lc++;-lc++abi")`


### Enabling C++ support

Add to `prj.conf`:
```
CONFIG_CPP=y
```

to enable `C++23` support:
```
CONFIG_STD_CPP2B=y
```

In `CMakeLists.txt`:
```
target_link_libraries(app PRIVATE stdc++ supc++)
```

## Building sysbuild VS nosysbuild

Building as `nosysbuild` allows flashing onto the pro micro nrf52840 in uf2 as-is. Binary just works.
When the same project is built as `sysbuild` it doesn't work as-is.

To make it work it's important to have `partition manager` subsys from `nrf` enabled 
(in case of a trivial hello world it was just working. I guess the mere fact of `sysbuild` enables it) 
and make sure that `app` partition starts not at `0x0`, but at `0x26000`. 
This can be achieved by providing a `pm_static.yml` with the following
content:
```
reserved_off:
  address: 0x00000
  size: 0x26000
```

Whatever partitions are specified: all declared flash memory must be covered.
`app` is allocated dynamically based on the other partitions.

Static Partition configuration for Zigbee (not yet proven) that at least allows the build:
```
reserved_off:
  address: 0x00000
  size: 0x26000
zboss_nvram:
  address: 0xed000
  size: 0x4000
zboss_product_config:
  address: 0xf1000
  size: 0x0800
settings_storage:
  address: 0xf1800
  size: 0x2800
bootloader:
  address: 0xf4000
  size: 0xc000
```

`app` will be located between 0x26000 and 0xed000

***Important***
It seems that for ZigBee applications we have to build with `sysbuild` and it awaits `pm_config.h` file to exist.
This file is generated by the partition manager and only in `sysbuild` mode. Direct configuration via cmake doesn't
cut it. Need to go via `west`.

### Sysbuild and llvm cmake toolchain file
By default whatever toolchain file you pass to the original cmake call it's not forwarded to the internal
cmake call for sysbuild.
To make that happen in the file `/path/to/zephyr/share/sysbuild/cmake/modules/sysbuild_extensions.cmake`
we need to find a variable `shared_cmake_variables_list` and add `CMAKE_TOOLCHAIN_FILE` to the list

## Zigbee

ncs-zigbee addon (a non-deprecated way of doing zigbee) doesn't seem to support nrf52840 at the moment.
So the only working way left is via `nrf` and `zigbee` subsystem.

### Channel selection

To make it check and attempt to steer on multiple channels:
```
CONFIG_ZIGBEE_CHANNEL_SELECTION_MODE_MULTI=y
CONFIG_ZIGBEE_CHANNEL_MASK=0x7FFF800
```

### Including zboss and other zigbee headers in C++ project

it's important for the zigbee API to have C names, no C++ name mangling.
So zigbee headers should be included like:
```cpp
extern "C"
{
#include <zboss_api.h>
#include <zboss_api_addons.h>
#include <zb_mem_config_med.h>
#include <zigbee/zigbee_app_utils.h>
#include <zigbee/zigbee_error_handler.h>
#include <zb_nrf_platform.h>
}
```

## Bootloader
[This](https://nicekeyboards.com/assets/nice_nano_bootloader-0.6.0_s140_6.1.1.hex) one worked (I was able to flash it and restore one of my pro micro boards)
Flashed with a command executed via OpenOCD:
```
flash write_image erase nice_nano_bootloader-0.6.0_s140_6.1.1.hex
```
**Note**: [here](https://nicekeyboards.com/docs/nice-nano/#bootloader) it's claimed that Adafruit's bootloader for nrf52 could work
**Update**: it does work. e.g.\
`adm_b_nrf52840_1_bootloader-0.9.2_s140_6.1.1.hex`

### Bootloader and `nrf5 mass_erase`
It appears that after `mass_erase` flashing the chip with OpenOCD it's important to flash again the bootloader as described earlier. 
However it doesn't for some reason allow flashing the app with `program ...`. It gets flashed but it doesn't get executed.
What helps is connecting via USB and flashing with UF2 image. After that OpenOCD works again.

## Debugging with OpenOCD

### Start openocd
To start `openocd` with this board using STLink-V2:
```
openocd -f interface/stlink.cfg -f target/nrf52.cfg
```
***Update***
debugging with STLink-V2 sucks, stepping with a debugger is a nightmare since it always halts in panic for
whatever reason.
A cheap `CMSIS DAP/DAPLink Simulator STM32 Debugger` delivers a much better experience.
To start `openocd` with `CMSIS-DAP`:
```
openocd -f interface/cmsis-dap.cfg -f target/nrf52.cfg
```
Pins are to be connected as:

| SWD Pin| DAPLink Pin |
|--------|-------------|
| SWDCLK | TCK/CK      |
| SWDDIO | TMS/IO      |
| GND    | GND         |
| 3.3V   | 3.3V        |

### Flash image
Connect with `telnet` to `localhost:4444` like:
```
telnet localhost 4444
```
To flash image (important: zephyr.***bin***):
```
program path/to/zephyr.bin 0x26000 reset
```

### Example of VSCode debug configuration
***Important*** GDB needs an `ELF` file, not `BIN`
don't forget to set `useExtendedRemote` to `true`
```
{
  "name": "Launch",
    "type": "cppdbg",
    "request": "launch",
    "program": "${workspaceRoot}/path/to/zephyr.elf",
    "cwd": "${workspaceFolder}",
    "miDebuggerServerAddress": "localhost:3333",
    "useExtendedRemote": true,
    "MIMode": "gdb"
}
```

### Logging with `printk` and `LOG_*` over OpenOCD
The way that worked is: `SEGGER RTT` (Real-Time Transfer)
#### Zephyr application part
Additional configuration needed in `prj.conf`:
```
CONFIG_LOG=y
CONFIG_LOG_MAX_LEVEL=4
CONFIG_LOG_BACKEND_RTT=y
CONFIG_USE_SEGGER_RTT=y
CONFIG_PRINTK=y
```
`CONFIG_LOG` was the part that forced linking.
It requires the existence of 2 functions (otherwise it fails with linking errors):
```cpp
void zephyr_rtt_mutex_lock();
void zephyr_rtt_mutex_unlock();
```
It seems that there's no ready-to-use implementations for these and we need to provide
those.
This has worked:
```cpp
K_MUTEX_DEFINE(rtt_term_mutex);
extern "C" void zephyr_rtt_mutex_lock()
{
	k_mutex_lock(&rtt_term_mutex, K_FOREVER);
}

extern "C" void zephyr_rtt_mutex_unlock()
{
	k_mutex_unlock(&rtt_term_mutex);
}
```

#### OpenOCD part
After [starting `OpenOCD`](#start-openocd) execute the following commands:
```
# Halt the CPU to safely scan RAM for the RTT block
halt

# Setup RTT: Scan RAM (0x20000000 is start of RAM for nRF52840)
# Scan size (e.g., 0x10000 = 64kB, the full RAM)
# ID ("SEGGER RTT" is the default used by Zephyr/SEGGER)
rtt setup 0x20000000 0x10000 "SEGGER RTT"

# Start RTT processing in OpenOCD
rtt start

# Start a TCP server on port 9090 for RTT channel 0 (terminal output)
# You can choose a different port if 9090 is busy
rtt server start 9090 0

# Resume CPU execution
resume
```

After this connecting with a telnet to `localhost:9090` should show
the logging messaging.

## Power consumption notes
### Zigbee
ZBOSS stack issues a dedicated signal `ZB_COMMON_SIGNAL_CAN_SLEEP`.
The normal default thing to do is to call `zb_sleep_now`.
The `zigbee_default_signal_handler` does exactly that if you let it handle the signal.

**Important**

`zb_sleep_now` won't put the chip to sleep if it's configured to be a `router` or a `coordinator`.
It **needs** to be an `end device` in order to be able to actually sleep.

Before enabling Zigbee the following functions should be called to configure/enable 
the power saving techniques:

```cpp
zb_set_rx_on_when_idle(false);
zigbee_configure_sleepy_behavior(true);
```

### Zephyr
#### CONFIG_PM

For nrf52840 it makes no sense to enable that. You won't even be able to, because Nordic
has intentionally removed that since according to their claims the system [*automagically*](https://github.com/nrfconnect/sdk-zephyr/commit/96b38273138f05dd06cf7a58fa361f401e773e5e)
goes into a low-power state without hints from the developer. Seems to be true.
#### CONFIG_PM_DEVICE

Enabling this option in theory should've allowed various drivers to properly react to the events
of suspending/resuming the system. In practice however it has resulted in the inability of
the board to lower the power consumption below 1.5mA (without it values **<10uA** were achieved).
#### CONFIG_RAM_POWER_DOWN_LIBRARY

This option will allow you to have the functionality to power down the unused sections of RAM.
A call to `power_down_unused_ram();` will do the trick. Naturally no heap usage should occur
and all the required memory should be accounted for statically at compile time 
(constexpr, constinit et al are your friends).
#### Disabling periphery in Device Tree

Unused periphery should also be disabled in the device tree by providing an overlay file
that turns them off.
The content of such file may look like:
```
&i2c1 {
    status = "disabled";
};
&adc {
    status = "disabled";
};
&bt_hci_sdc {
    status = "disabled";
};
```
To use the overlay file the following should be put into CMakeLists.txt **before**
calling `find_package(Zephyr...`:
```
set(EXTRA_DTC_OVERLAY_FILE "dts.overlay")
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
```
