---
title: SSTAR平台引脚使用注意事项
date: 2026-10-06 21:15:10
tags:
    - Linux
    - C/C++
---

## SSTAR 平台引脚使用注意事项

### 引脚的默认状态

在 sstar 的 HW checklist 表格的 GPIO List 页中，被标注为 default GPIO mode 的引脚的默认复用功能就是 GPIO。如果想确认某个引脚是否为 GPIO，需要检查它是否在设备树及 padmux.dtsi 中被显式占用了。若要将某个默认不是 GPIO 引脚作为 GPIO，该引脚从上电到软件 padmux 配置生效之间会有一段默认状态时间，使用这种引脚时需要注意。

### GPIO 开漏输出模式

sstar 平台的引脚在配置为 GPIO 模式时不能设置为开漏输出，只能是推挽输出。如果要在用户态模拟开漏输出，如模拟 i2c 的线与操作，只能在想让 SDA 或 SCL 线进入高电平状态时，把 GPIO 设置为浮空输入模式，使引脚进入高阻态，才能让外部上拉把电平拉高(如果没有外部上拉，就把 GPIO 设置为上拉输入模式)。

补充：sstar 官方关于如何配置内核模拟 i2c 驱动的[文档](https://dev.comake.online/home/article/2735?searchId=241645)，这个方式应该是各个嵌入式 linux 平台都较为通用的，总结如下：

1. 确认模拟 i2c 相关的引脚的复用功能都是 GPIO（这个步骤在不同平台的操作是不同的）。
2. 进入 kernel 的 menuconfig，打开如下设置：
    ```
    Device Drivers  --->
        [*] GPIO Support  --->
        I2C support  --->
            [*] I2C support
            I2C Hardware Bus support  --->
                <*> GPIO-based bitbanging I2C
    ```
3. 修改设备树（这个步骤在不同平台的操作是相似的）：
    ``` c
    diff
    ...
        gpio:gpio{
            compatible = "sstar,gpio";
            // 表示其他设备引用这个 GPIO 控制器时，需要在 GPIO phandle 后面提供两个 cell 的参数（通常就是指两个参数）
            // 例如下面的 <&gpio 84 0>，84 指 GPIO 编号，0 指 GPIO 标志位（flag）
            // 具体 flags 的含义由 GPIO binding 定义。常见情况包括：
            // 0：默认有效电平，GPIO_ACTIVE_HIGH：高电平有效，GPIO_ACTIVE_LOW：低电平有效。
    +       #gpio-cells = <2>;
    +       status = "ok";
        };
    ...
        aliases {
            console = &uart0;
            serial0 = &uart0;
            serial1 = &uart1;
            serial2 = &fuart;
            serial3 = &uart2;
            // 给新增的模拟 i2c 节点的标签定义别名，注意不能与已有的 i2c 别名冲突
            // 这里的 i2c_gpio 是一个标签，给节点定义标签是方便让其他节点引用该节点
            // 别名会保留在编译后的设备树中。所以别名主要用于让驱动程序读取设备树
            // 当前示例硬件已经支持 i2c0 和 i2c1，这里使用 i2c2 防止冲突
            // 在 Linux 设备树体系中，i2c 的别名还通常用于确定 i2c 总线的编号
            // i2c2 通常会让内核生成对应该 i2c 控制器的 /dev/i2c-2 节点，但不保证一定如此，要看平台的具体实现
    +       i2c2    = &i2c_gpio;
        };
    ...
    +   i2c_gpio:i2c_gpio@0 {
            // 表示这个节点由 Linux 内核的 GPIO 模拟 i2c 驱动处理
    +       compatible = "i2c-gpio";
            // 表示该节点下面的子节点的 reg 地址的数据长度为一个 cell
            // 一个 cell 通常是 32 位，reg 在不同场景下的含义不同
            // 子节点就是挂载到这个 i2c 总线上的设备，i2c 设备地址通常是 7 位
    +       #address-cells = <1>;
            // 表示该节点下面的子节点的 reg 大小的数据长度为 0，reg 属性将不包含 size 字段
            // i2c 设备通常只有一个设备地址，没有内存设备那样需要描述的一段空间
            // 如 goodix_gt911 节点中 reg = <0x5d>; 只有一个 address 字段
    +       #size-cells = <0>;
            // 定义 SDA 和 SCL 线的引脚
            // 这两个 GPIO 的定义顺序非常重要，通常是先 SDA，后 SCL
    +       gpios = <&gpio 84 0>, // sda pin
    +               <&gpio 86 0>; // scl pin
            // 设置模拟 i2c 在操作 GPIO 时的时序延迟，单位是微秒
    +       i2c-gpio,delay-us = <5>;
    +       status = "ok";
    +    };
    ...

    ```
