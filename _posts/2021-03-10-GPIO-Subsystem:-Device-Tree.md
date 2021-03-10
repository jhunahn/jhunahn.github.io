---
title: "GPIO Subsystem: Device Tree"
layout: post
comments: true
date: 2021-03-10 13:30
image:
headerImage:
tag:
- Linux
- GPIO
star: false
category: Linux
author: jihun
---

<!-- more -->

# GPIO Mapping Using Device Tree

The property `gpio-controller` of device node wrote in the device tree (`.dts` or `.dtsi`) means the *GPIO controller node*.

```dts
/dts-v1/;

/ {
	node {
		gpio1: gpio1 {
		    gpio-controller;
		    #gpio-cells = <2>;
		};
		gpio2: gpio2 {
		    gpio-controller;
		    #gpio-cells = <1>;
		};

		[...]

		enable-gpios = <&gpio2 2>;
		data-gpios =   <&gpio1 12 0>,
		               <&gpio1 13 0>,
		               <&gpio1 14 0>,
		               <&gpio1 15 0>;
	};
};
```

## cell

The `#gpio-cells = <#>` property means the number of cell data arguments used.
In the example above, the *gpio1 controller* uses  two-cells and the device driver receives and uses 2 arguments. And the *gpio2 controller* uses just one-cell and the device driver receives the argument and uses it.

> If not specified, two-cell method is used by defaults.

## Interlocking (with pin control subsystem)

In the gpio controller node, use the `gpio-ranges` property.

The phandle pointed to by the `gpio-ranges`  must point to the associated *pin controller* node. 1~3 arguments can be used, and supportes the use of arrays.

When using 1~2 arguments, the `gpio-ranges-group-names`  must be used.
- First: starting gpio number (offset)
- Second: Use the pin start number and number corresponding to the group name specified in the `gpio-ranges-group-names` property.
> if this item is used, it must be 0.

```dts
	gpio-ranges = <&pinctrl 0 0>;
    gpio-ranges-group-names = "gpioa"
```

When using 3 arguments, the `gpio-ranges-group-names` property is not required, and **should be an empty string even if used**.
- First: starting gpio number (offset)
- Second: starting pin number (offset)
- 3rd: gpio count

```dts
	gpio-ranges = <&pinctrl 0 0 16>;
```

When used in array,
```dts
	gpio-ranges = <&pinctrl 0 16 3>, <&pinctrl 14 30 2>;
```

### example

```dts
iomux: iomux@FF10601c {
        compatible = "abilis,tb10x-iomux";
        reg = <0xFF10601c 0x4>;
        pctl_gpio_a: pctl-gpio-a {
                abilis,function = "gpioa";
        };
        pctl_uart0: pctl-uart0 {
                abilis,function = "uart0";
        };
};
uart@FF100000 {
        compatible = "snps,dw-apb-uart";
        reg = <0xFF100000 0x100>;
        clock-frequency = <166666666>;
        interrupts = <25 1>;
        reg-shift = <2>;
        reg-io-width = <4>;
        pinctrl-names = "default";
        pinctrl-0 = <&pctl_uart0>;
};
gpioa: gpio@FF140000 {
        compatible = "abilis,tb10x-gpio";
        reg = <0xFF140000 0x1000>;
        gpio-controller;
        #gpio-cells = <2>;
        ngpios = <3>;
        gpio-ranges = <&iomux 0 0>;
        gpio-ranges-group-names = "gpioa";
};
```

#### iomux pin controller node

The `pctl-gpio-a` node declares two pinmux nodes.

This vendor's device tree node map conversion not specified that neither the pins  nor the groups property. In this case, the `compact` node name used by the consumer device side is used as the group name. So, group names of two consumer devices `uart@FF100000` and `gpio@FF140000` are *uart* and *gpio* because `compact` name is used as the group name.

#### uart (pin control consumer) node
It register *pinmux* / *pinconf* mappings to the phandle node specified in the `pinctrl-0` property using the `default` state. Map the `uart` group and the `uart0` function defined in the pinmux node.
> Related functions: `drivers/pinctrl/pinctrl-tb10x.c` :: `tb10x_dt_node_to_map()`

#### gpioa (pin control consumer) node
It uses the `gpio-ranges` property to associate with the iomux pin controller node. From `gpio 0`, among the group names implemented in pin controller, the pins corresponding to the `gpioa` group name specified in the `gpio-ranges-group-names` property are registered as gpio pins.

## line

Before registering the gpio controller, you must first know the number of gpio pins used by the gpio controller by registering the number of gpio in `gpio_chip->ngpios` or specifying the **number of gpio pins** using the `ngpios` property. After that, use the `gpio-line-names` property to specify the names of gpio descriptors as many as the number of gpio pins.

> If the pin names of the gpio descriptors are already set as the internal code of the gpio device driver, you do not need to use the `gpio-line-names` property separately.

```dts
gpio-controller@00000000 {
	compatible = "foo";
	reg = <0x00000000 0x1000>;
	gpio-controller;
	#gpio-cells = <2>;
	ngpios = <18>;
	gpio-line-names = "MMC-CD", "MMC-WP", "VDD eth", "RST eth", "LED R",
		"LED G", "LED B", "Col A", "Col B", "Col C", "Col D",
		"Row A", "Row B", "Row C", "Row D", "NMI button",
		"poweroff", "reset";
}
```

## hog

When the `gpio-hog` property is found in the lower nodes of the gpio controller, the specified gpio pin state is immediately set.
> It added to `kernel v.4.1-rc1`

The handling of the argument corresponding to the `gpios` property is implemented in the gpio controller provided by the vendor. In the gpio simple conversion using two cells, the gpio start number (offset) is used without conversion, and the second argument corresponding to the gpio setting value is also used without conversion.
> Most vendors do not implement it and use the gpio simple transformation.

```dts
qe_pio_a: gpio-controller@1400 {
	compatible = "fsl,qe-pario-bank-a", "fsl,qe-pario-bank";
	reg = <0x1400 0x18>;
	gpio-controller;
	#gpio-cells = <2>;

	line_b {
		gpio-hog;
		gpios = <6 0>;
		output-low;
		line-name = "foo-bar-gpio";
	};
};
```

# Reference

- [GPIO Subsystem -3- (Device Tree)](http://jake.dothome.co.kr/gpio-3/)
