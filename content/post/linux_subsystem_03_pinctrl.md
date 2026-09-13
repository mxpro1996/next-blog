---
title: "[转]（三）Linux 内核驱动 - Pinctrl子系统"
date: 2025-10-11T09:02:36+08:00
draft: false
---
#### **1. 概述**

在许多SoC（System-on-Chip）中，内部包含了pin控制器，这些控制器通过寄存器来配置引脚的功能和特性。为了统一管理不同SoC厂商的引脚配置，Linux内核提供了 **pinctrl** 子系统，用于管理引脚的复用、特性配置、GPIO等。

#### **2. Pinctrl子系统的主要功能**

* **管理系统中的所有可控制的pin**：在系统初始化时，pinctrl子系统会扫描所有引脚，并标识哪些引脚是可控的。
* **引脚复用（Multiplexing）**：引脚可根据需要配置为不同的功能，例如SPI、I2C、GPIO等。pinctrl子系统通过复用引脚的功能管理这些引脚的不同状态。
* **电气特性配置**：通过配置引脚的电气特性，如开启或关闭引脚的上下拉电阻、调整引脚的驱动能力等，确保硬件功能的稳定性。

#### **3. 核心概念**

* **Function 和 Pin Group**：

  * **Function** 表示一个硬件功能（例如，SPI、I2C）。在硬件设计中，可能有多个引脚可以实现同一个功能，通过复用控制将其引导到对应的引脚组（Pin Group）。
  * **Pin Group** 是一组物理引脚，这些引脚被映射到一个具体的硬件功能。在配置引脚时，必须指定相应的function和pin group，以保证功能的正常切换。

* **Pin Control State**：每个设备可能处于多个状态（例如：default、sleep、idle等），每个状态对应不同的引脚配置（包括复用功能、电气特性等）。pinctrl子系统通过 **pin control state** 管理设备的不同工作状态。

#### **4. 与GPIO子系统的关系**

GPIO（General Purpose Input/Output）引脚在Linux中也由pinctrl子系统进行管理。每个引脚可能会被配置为多个功能之一，如GPIO、I2C、SPI等。当一个引脚已经被配置为某个功能（如GPIO），它就不能再被重新配置为其他功能。因此，pinctrl子系统确保每个引脚的配置是唯一且不冲突的。

#### **5. 与统一设备驱动模型的关系**

在Linux中，设备驱动的加载和设备节点的绑定通常会通过统一的设备驱动模型（UDM）来完成。在驱动加载的过程中，pinctrl子系统会在设备的 **probe** 函数之前对引脚进行初始化配置，确保引脚已经处于合适的工作状态。驱动可以通过 **devm_pinctrl_get** 获取pinctrl的句柄，并调用 **pinctrl_select_state** 来切换引脚的状态。

#### **6. 与设备树（Device Tree）的关系**

设备树中的配置决定了设备所需的引脚状态。在设备树文件中，通过 `pinctrl-names` 和 `pinctrl-X` 等属性定义了设备所需要的引脚配置。每个设备的驱动程序会根据设备树中的配置动态加载相应的引脚设置。

例如，在设备树中，以下配置定义了不同的pin控制状态：

```dts
pinctrl-names = "sleep","default","idle";
pinctrl-0 = <&state_sleep>;
pinctrl-1 = <&state_default>;
pinctrl-2 = <&state_idle>;
```

这些状态在驱动程序中被引用，允许驱动在不同的工作状态下配置引脚。

#### **7. Pinctrl与主控驱动的关系**

主控驱动通过注册一个 `struct pinctrl_desc` 结构体，将主控的pinctrl硬件操作转换为符合Linux pinctrl子系统规范的结构。这个结构体包含了引脚控制器的详细信息，如引脚数量、控制操作函数、复用操作函数等。

```c
struct pinctrl_desc {
    const char *name;
    struct pinctrl_pin_desc const *pins;
    unsigned int npins;
    const struct pinctrl_ops *pctlops;
    const struct pinmux_ops *pmxops;
    const struct pinconf_ops *confops;
    struct module *owner;
};
```

#### **8. DTS中的Pinctrl配置**

在设备树文件（DTS）中，`pinctrl`配置可以通过以下方式进行定义：

```dts
pinctrl@e01b0000 {
    compatible = "actions,s700-pinctrl";
    reg = <0 0xe01b0000 0 0x1000>;
    pinctrl-names = "default";
    pinctrl-0 = <&state_default>;
};
```

设备树中的每个节点定义了引脚的复用功能、状态、驱动配置等。通过 `pinctrl-names` 和 `pinctrl-X` 属性，设备树描述了设备的各种引脚状态及其配置。

#### **9. 实战示例：动态切换I2C与GPIO**

通过一个实战例子，展示了如何动态切换I2C与GPIO功能的配置：

* 在设备树中，我们首先定义了两个pin状态：一个是I2C模式，另一个是GPIO模式。

```dts
gpio_i2c_exchage {
   compatible = "gpio-i2c-exchage";
   pinctrl-names = "default", "gpio";
   pinctrl-0 = <&i2c0_function>;
   pinctrl-1 = <&i2c0_gpio>;
};

i2c0_option {
    i2c0_function: i2c0-function {
        rockchip,pins = <2 GPIO_D0 RK_FUNC_1 &pcfg_pull_none_smt>,
                        <2 GPIO_D1 RK_FUNC_1 &pcfg_pull_none_smt>;
    };
    i2c0_gpio: i2c0-gpio {
        rockchip,pins = <2 GPIO_D0 RK_FUNC_GPIO &pcfg_pull_none_smt>,
                        <2 GPIO_D1 RK_FUNC_GPIO &pcfg_pull_none_smt>;
    };
};
```

* 在驱动代码中，通过 `pinctrl_select_state` 切换引脚状态。

```c
int biada_pinctrl_request_gpios(int state) {
    int result = 0;
    if(state == 0) {
        result = pinctrl_select_state(i2c0_pinctrl, i2c0_default);
    } else {
        result = pinctrl_select_state(i2c0_pinctrl, i2c0_gpio);
    }
    return result;
}
```

* 通过这种方式，可以在运行时切换引脚的功能，例如从I2C模式切换到GPIO模式，或者反之。

#### **10. 总结**

Linux的 **pinctrl** 子系统提供了一种统一、灵活的方式来管理和配置SoC中的引脚功能与电气特性。通过结合设备树、内核驱动和GPIO子系统，pinctrl子系统不仅能够支持引脚的复用和功能切换，还能够根据硬件需求进行动态配置。在开发过程中，合理使用设备树配置和pinctrl接口，能够显著提高硬件资源的利用率和系统的可配置性。

