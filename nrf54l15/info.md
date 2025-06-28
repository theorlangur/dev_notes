# Various info regarding nrf54l15 (specifically regarding Ezurio BL54L15)

## Device tree for Zephyr
Basically a copy of pieces from nrf54l15DK was taken and it looks something like:

```dts
/dts-v1/;

#include <nordic/nrf54l15_cpuapp.dtsi>
#include "pinctrl.dtsi"

/ {
	compatible = "nordic,orlangur_ezurio_nrf54l15-cpuapp";
	model = "Orlangur Ezurio nRF54L15 Breakout";

	chosen {
		zephyr,code-partition = &code_partition;
		zephyr,sram = &cpuapp_sram;
		zephyr,flash-controller = &rram_controller;
		zephyr,flash = &cpuapp_rram;
		zephyr,ieee802154 = &ieee802154;
		zephyr,console = &uart30;
	};

	leds {
		compatible = "gpio-leds";
		led0: led_0 {
			gpios = <&gpio2 9 GPIO_ACTIVE_HIGH>;
			label = "Red LED 0";
		};
	};

	aliases {
		led0 = &led0;
	};
};

&cpuapp_sram {
	status = "okay";
};

&uart20 {
	current-speed = <115200>;
	pinctrl-0 = <&uart20_default>;
	pinctrl-1 = <&uart20_sleep>;
	pinctrl-names = "default", "sleep";
};

&uart30 {
	status = "okay";
	current-speed = <115200>;
	pinctrl-0 = <&uart30_default>;
	pinctrl-1 = <&uart30_sleep>;
	pinctrl-names = "default", "sleep";
};

&lfxo {
	load-capacitors = "internal";
	load-capacitance-femtofarad = <12500>;
};

&hfxo {
	load-capacitors = "internal";
	load-capacitance-femtofarad = <15000>;
};

&regulators {
	status = "okay";
};

&vregmain {
	status = "okay";
	regulator-initial-mode = <NRF5X_REG_MODE_DCDC>;
};

&grtc {
	owned-channels = <0 1 2 3 4 5 6 7 8 9 10 11>;
	/* Channels 7-11 reserved for Zero Latency IRQs, 3-4 for FLPR */
	child-owned-channels = <3 4 7 8 9 10 11>;
	status = "okay";
};

&gpio0 {
	status = "okay";
};

&gpio1 {
	status = "okay";
};

&gpio2 {
	status = "okay";
};

&gpiote20 {
	status = "okay";
};

&gpiote30 {
	status = "okay";
};

&radio {
	status = "okay";
};

&ieee802154 {
	status = "okay";
};

&temp {
	status = "okay";
};

&clock {
	status = "okay";
};

&cpuapp_rram {
	status = "okay";
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;
		code_partition: partition@0 {
			label = "code-0";
			reg = <0x0 DT_SIZE_K(388)>;
		};
	};
};

&adc {
	status = "okay";
};
```

This describes a system without a dedicated bootloader.

## Flashing with OpenOCD over CMSIS-DAP adapter
OpenOCD must be new enough (built it from sources, from main) to include `target/nordic/nrf54l.cfg` file.
The flashing is not supported via `program` or `flash` commands yet, but according to [this](https://review.openocd.org/c/openocd/+/8609/1?tab=comments)
the following set of commands does the job:
```
mww 0x5004b500 0x101
nrf54l.dap memaccess 12
load_image <path/to/build/merged.hex>
```

It seems that flashing `merged.hex` (and not `zephyr.hex`) works at least for hello world (blinky).

## Problem with k_msleep
When system never wakes up after calling `k_msleep` the following configuration could help and
should be present:

```
# Start SYSCOUNTER on driver init
CONFIG_NRF_GRTC_START_SYSCOUNTER=y
```

## Configuring LFXO
`CONFIG_CLOCK_CONTROL_NRF_K32SRC_XTAL` seems to be the default, but still:

```
CONFIG_CLOCK_CONTROL_NRF_K32SRC_XTAL=y
CONFIG_CLOCK_CONTROL_NRF_K32SRC_20PPM=y
```

## Communicating over UART
This config allows seeing `printk` messages over `uart` (e.g. `CH340` `USB-to-UART` dongle):

```
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_PRINTK=y
CONFIG_UART_INTERRUPT_DRIVEN=y
```
