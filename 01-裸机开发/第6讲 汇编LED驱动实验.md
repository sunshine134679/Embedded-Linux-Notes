# 第6讲 汇编LED驱动实验

> 来源：正点原子官方笔记整理

## 一、汇编 LED 原理分析

### 1.1 为什么学习 Cortex-A 汇编

需要学习汇编的原因：

1. 部分 SoC 外设需要用汇编初始化。
2. 某些平台需要用汇编初始化 DDR，i.MX6U 不需要手写这部分。
3. 需要设置 SP 指针，一般指向 DDR，从而准备好 C 语言运行环境。

### 1.2 ALPHA 开发板 LED 硬件原理

STM32 IO 初始化流程：

1. 使能 GPIO 时钟。
2. 设置 IO 复用，将其复用为 GPIO。
3. 配置 GPIO 电气属性。
4. 使用 GPIO 输出高 / 低电平。

i.MX6ULL IO 初始化流程：

1. 使能外设时钟。

   `CCGR0` ~ `CCGR6` 共 7 个寄存器控制 6ULL 所有外设时钟。为了简单，可以设置为 `0xffffffff`，相当于使能所有外设时钟。

2. 配置 IO 复用。

   将 `IOMUXC_SW_MUX_CTL_PAD_GPIO1_IO03` 的 `bit3:0` 设置为 `0101`，即 `5`，让 `GPIO1_IO03` 复用为 GPIO。

3. 配置电气属性。

   `IOMUXC_SW_PAD_CTL_PAD_GPIO1_IO03` 用于设置 `GPIO1_IO03` 的电气属性，包括压摆率、速度、驱动能力、开漏、上下拉等。

4. 配置 GPIO 功能和输出电平。

   设置 GPIO 方向为输出，再通过数据寄存器输出高 / 低电平。

## 二、汇编简介

汇编由一条条指令构成。

C 语言示例：

```c
int a, b;
a = b;
```

假设 `a` 地址为 `0x20`，`b` 地址为 `0x30`，汇编可以表示为：

```asm
LDR R0, =0x30
LDR R1, [R0]

LDR R0, =0x20
STR R1, [R0]
```

编写裸机驱动时，最常用的是 `LDR` 和 `STR` 这两个指令。

## 三、编写驱动

根据 LED 对应 IO 的寄存器地址，完成时钟、复用、电气属性和 GPIO 输出配置。

## 四、编译程序

编译流程：

1. 使用 `arm-linux-gnueabihf-gcc` 将 `.c` / `.s` 文件编译为 `.o`。
2. 将所有 `.o` 文件链接为 ELF 格式可执行文件。
3. 将 ELF 文件转换为 `.bin` 文件。
4. 将 ELF 文件反汇编为汇编文件，便于分析。

链接就是将所有 `.o` 文件连接在一起，并放到指定运行地址。本实验需要指定链接起始地址。

6ULL 的 RAM 分为内部 RAM 和外部 DDR：

| 类型 | 地址范围 |
| ---- | ---- |
| 内部 RAM | `0x900000` ~ `0x91ffff` |
| 512 MB DDR | `0x80000000` ~ `0x9fffffff` |
| 256 MB DDR | `0x80000000` ~ `0x8fffffff` |

本系列裸机代码链接起始地址为：

```text
0x87800000
```

要使用 DDR，必须先初始化 DDR。对于 i.MX 系列 SoC，`.bin` 文件不能直接运行，需要添加一个头部。头部信息包含 DDR 初始化参数，SoC 内部 Boot ROM 会从 SD 卡、EMMC 等外置存储读取头部信息，初始化 DDR，并将 `.bin` 文件拷贝到指定位置。

`.bin` 的运行地址必须和链接起始地址一致，位置无关代码除外。

## 五、烧写 `.bin` 文件

STM32 通常烧写到内部 Flash。6ULL 支持从 SD 卡、EMMC、NAND、NOR、SPI Flash 等介质启动。裸机例程选择烧写到 SD 卡。

在 Ubuntu 下向 SD 卡烧写裸机 `.bin` 文件时，不是把 `.bin` 拷贝到 SD 卡文件系统中，而是烧写到 SD 卡绝对地址上。

对 i.MX 而言，不能直接烧写 `.bin` 文件，需要先在 `.bin` 前添加头部。这个工作使用正点原子提供的 `imxdownload` 软件完成。

给 `imxdownload` 添加执行权限：

```bash
chmod 777 imxdownload
```

烧写示例：

```bash
./imxdownload led.bin /dev/sdf
```

`imxdownload` 会向 `led.bin` 添加头部，生成新的 `load.imx` 文件。`load.imx` 才是最终烧写到 SD 卡中的文件。
