# 第7讲 IMX启动方式

> 来源：正点原子官方笔记整理

## 一、硬件启动方式选择

### 1.1 启动方式选择

LED 实验从 SD 卡读取 `.bin` 文件并启动，说明 6ULL 支持从 SD 卡启动。6ULL 也支持多种其他启动方式。

`BOOT_MODE0` 和 `BOOT_MODE1` 用于选择启动模式：

- USB 启动。
- 内部 Boot 启动。

如果要烧写系统到开发板，可以选择从 USB 下载，再写入 SD 卡、EMMC、NAND 等外置存储。烧写完成后，设置为内部 Boot 启动，再从对应外置存储启动。

### 1.2 启动设备选择

前提是 `BOOT_MODE1 = 1`、`BOOT_MODE0 = 0`，也就是从内部 Boot 启动。

支持的启动设备包括：

- NOR Flash
- OneNAND
- NAND Flash
- QSPI Flash
- SD / EMMC
- EEPROM

常用的是 NAND、SD、EMMC，部分场景也会使用 QSPI Flash。

启动设备通过 `BOOT_CFG` 选择。`BOOT_CFG1`、`BOOT_CFG2`、`BOOT_CFG4` 各 8 位，由 `LCD_DATA0` ~ `LCD_DATA23` 设置。

ALPHA 开发板上大部分启动配置默认通过 47K 下拉电阻接地：

- `BOOT_CFG4` 8 根线全部接地。
- `BOOT_CFG2` 基本接地，除了 `BOOT_CFG2[3]`，该位用于选择 SD 卡启动接口。
- `BOOT_CFG1[0:2]` 固定，`BOOT_CFG1[3:7]` 可设置。

### 1.3 正点原子 ALPHA 开发板启动原理

核心板 `LCD_DATA0` ~ `LCD_DATA23` 基本使用 47K 下拉，配合拨码或硬件电路完成启动配置。

## 二、启动头文件

### 2.1 Boot ROM 做的事情

Boot ROM 主要完成：

1. 设置内核时钟为 396 MHz。
2. 使能 MMU 和 Cache，包括 L1 Cache、L2 Cache、MMU，用于加速启动。
3. 从 `BOOT_CFG` 指定的外置存储中读取 image，并做相应处理。

### 2.2 IVT 和 Boot Data

`.bin` 文件前需要添加头部。

烧写到 SD 卡中的 `load.imx` 起始地址是：

```text
0x400
```

也就是 1024 字节偏移。

头部大小为 3 KB，加上偏移的 1 KB，总共 4 KB。因此 SD 卡中真正 `.bin` 文件起始地址为：

```text
4096
```

`IVT` 大小：

```text
32 B / 4 = 8 条
```

`IVT + Boot Data` 中很多数据来自 NXP 官方 `u-boot.imx` 文件。

### 2.3 DCD 数据

`DCD` 是 Device Configuration Data，用于配置 6ULL 内部寄存器。

典型内容：

1. 将 `CCGR0` ~ `CCGR6` 全部写为 `0xffffffff`，打开所有外设时钟。
2. 配置 DDR 初始化参数。
3. 设置 DDR 控制器，完成 DDR 初始化。

### 2.4 其他数据

检查数据命令、NOP 命令、解锁命令等也属于 DCD 相关内容。
