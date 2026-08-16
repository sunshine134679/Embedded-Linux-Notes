# 第2讲 正点原子官方 U-Boot 编译与体验

> 来源：正点原子官方笔记整理

## 一、何为 U-Boot？

- U-Boot 是一个较复杂的裸机程序，本质上是 Bootloader。
- U-Boot 的主要作用是启动 Linux 或其他系统。
- U-Boot 最核心的工作之一是初始化 DDR，因为 Linux 通常运行在 DDR 中。

一般情况下，Linux 镜像 `zImage` / `uImage` 和设备树 `.dtb` 存放在 SD、EMMC、NAND、SPI Flash 等外部存储介质中。U-Boot 需要把这些镜像从外部 Flash 拷贝到 DDR 中，再启动内核。

U-Boot 的主要目的就是为系统启动做准备。它不仅能启动 Linux，也可以启动其他系统，例如 VxWorks；Linux 也不一定只能通过 U-Boot 启动。

U-Boot 是通用 Bootloader，支持多种架构。

## 二、U-Boot 获取来源

1. **U-Boot 官网版本**

   优点是通用，缺点是对具体芯片、外设驱动和具体开发板的支持可能不完整。

2. **SoC 厂商定制版本**

   SoC 厂商通常会基于 U-Boot 官网某个版本加入对应 SoC 及驱动支持，例如 NXP 官方的 i.MX6ULL EVK 板。

3. **开发板厂商定制版本**

   开发板会参考 SoC 官方板，但硬件设计通常不同，因此开发板厂商还会继续修改 SoC 厂商版本，使其适配自己的板子。

## 三、正点原子官方 U-Boot 编译

1. 编译 U-Boot 前需要先配置。
2. 编译完成后会生成 `u-boot.bin`。
3. `u-boot.bin` 还需要添加头部信息。U-Boot 编译最后会通过 `tools/mkimage` 添加头部信息，生成 `u-boot.imx`。
4. 如果已经配置过 U-Boot，要注意脚本可能会清理整个工程，配置文件和配置项也会被删除。
5. 为了方便开发，建议直接在 U-Boot 顶层 `Makefile` 中设置 `ARCH` 和 `CROSS_COMPILE` 这两个变量的值。

## 四、正点原子官方 U-Boot 烧写
