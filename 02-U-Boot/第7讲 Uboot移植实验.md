# 第7讲 U-Boot 移植实验

> 来源：正点原子官方笔记整理

## 一、NXP 官方 U-Boot 编译与测试

将 NXP 提供的 U-Boot 拷贝到 Ubuntu 中。

一块开发板要运行 U-Boot，至少需要 DDR / DRAM、串口、SD / EMMC / NAND 等基础资源可以正常工作。

测试结果：

1. U-Boot 能正常启动。
2. LCD 驱动需要根据所使用的屏幕修改。
3. NET 初始化失败。

## 二、移植 NXP 官方 U-Boot 到 ALPHA 开发板

### 2.1 添加板子默认配置文件

借鉴 NXP 官方 6ULL EVK 开发板，默认配置文件也沿用它作为基础。

### 2.2 添加板子对应的头文件

不同板子需要配置不同信息，一般会放在板级配置头文件中。

### 2.3 添加板子对应的板级文件夹

每个板子都有特有文件，也叫板级文件。这里直接拷贝 6ULL EVK 的板级文件作为基础。

### 2.4 修改 U-Boot 的配置界面

修改配置界面，让新板子可以在 U-Boot 配置系统中选择。

### 2.5 使用新添加的板子配置并编译 U-Boot

使用新增板子的默认配置文件进行配置，然后编译 U-Boot。

### 2.6 LCD 驱动修改

需要关注：

1. 确认 LCD IO 初始化正确，对应 `mx6ull_alientek_emmc.c` 中的 `lcd_pads`。
2. 确认 LCD 参数正确，对应 `mx6ull_alientek_emmc.c` 中的 `displays`。
3. `fb_videomode` 表示 RGB LCD 参数。

LCDIF 基地址计算：

```text
MX6ULL_LCDIF1_BASE_ADDR = AIPS2_OFF_BASE_ADDR + 0x48000
AIPS2_OFF_BASE_ADDR     = ATZ2_BASE_ADDR + 0x80000
ATZ2_BASE_ADDR          = AIPS2_ARB_BASE_ADDR
AIPS2_ARB_BASE_ADDR     = 0x02100000
```

所以：

```text
MX6ULL_LCDIF1_BASE_ADDR = 0x02100000 + 0x80000 + 0x48000
                         = 0x21c8000
```

`panel` 环境变量表示 LCD ID。

### 2.7 网络驱动修改

6ULL 网络方案采用内部 MAC + 外部 PHY。6ULL 官方开发板使用的 PHY 芯片是 `KSZ8081`，正点原子 ALPHA 开发板使用的是 `LAN8720A`，因此需要修改驱动。

`LAN8720A` 有一个管理接口叫 MDIO，两根线分别是 `MDIO` 和 `MDC`。一个 MDIO 接口可以管理 32 个 PHY 芯片，通过 PHY ADDR 区分访问哪个 PHY 芯片。

ALPHA 开发板 PHY 地址：

| 网口 | PHY ADDR | 复位引脚 |
| ---- | -------- | -------- |
| ENET1 | `0x0` | `SNVS_TAMPER7` |
| ENET2 | `0x1` | `SNVS_TAMPER8` |

因为所有 PHY 的前 32 个寄存器一模一样，U-Boot 中已经有通用 PHY 驱动，所以理论上不需要重新写完整 PHY 驱动。

驱动修改点：

1. 修改 PHY ADDR。
2. 删除原有 74LV595 相关代码。
3. 添加 ALPHA 开发板的网络复位 IO。

### 2.8 其他需要修改的地方

根据实际板级差异继续补充。

## 三、`bootcmd` 和 `bootargs` 环境变量

### 3.1 `bootcmd` 环境变量

宏 `CONFIG_BOOTCOMMAND` 也可以设置 `bootcmd` 的值。

示例：

```c
"run findfdt;" \
"mmc dev ${mmcdev};" \
"mmc dev ${mmcdev}; if mmc rescan; then " \
"if run loadbootscript; then " \
"run bootscript; " \
"else " \
"if run loadimage; then " \
"run mmcboot; " \
"else run netboot; " \
"fi; " \
"fi; " \
"else run netboot; fi"
```

关键环境变量和命令：

- `findfdt`：设置 `fdt_file` 环境变量，也就是 `.dtb` 文件名。
- `mmc dev 1`：切换到 EMMC。
- `fatload mmc 1:1 80800000 zImage`：加载内核镜像。

加载设备树并启动：

```bash
fatload mmc 1:1 83000000 imx6ull-14x14-evk.dtb
bootz 80800000 - 83000000
```

`loadbootscript` 展开：

```c
"loadbootscript=" \
"fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${script};\0" \
```

展开后：

```bash
fatload mmc 1:1 80800000 boot.scr
```

`loadimage` 展开：

```c
"loadimage=fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${image}\0" \
```

展开后：

```bash
fatload mmc 1:1 80800000 zImage
```

`loadfdt` 展开：

```c
"loadfdt=fatload mmc ${mmcdev}:${mmcpart} ${fdt_addr} ${fdt_file}\0" \
```

展开后：

```bash
fatload mmc 1:1 83000000 imx6ull-14x14-evk.dtb
```

启动内核：

```bash
bootz ${loadaddr} - ${fdt_addr}
bootz 80800000 - 83000000
```

### 3.2 `bootargs` 环境变量

宏 `CONFIG_BOOTARGS` 也可以设置 `bootargs` 的值。

示例：

```c
mmcargs=setenv bootargs console=${console},${baudrate} " \
CONFIG_BOOTARGS_CMA_SIZE \
"root=${mmcroot}\0" \
```

展开后：

```text
bootargs console=ttymxc0,115200 root=/dev/mmcblk1p2
```

`bootargs` 会传递给 Linux 内核，用于设置内核命令行参数。

## 四、U-Boot 启动 Linux 测试

### 4.1 从 EMMC 启动

首先查看 EMMC 中是否已有系统、Linux 镜像 `zImage` 和 `.dtb` 文件。

```bash
mmc dev 1
fatls mmc 1:1
fatload mmc 1:1 80800000 zImage
fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb
bootz 80800000 - 83000000
```

说明：

- `mmc dev 1`：切换到 EMMC。
- `fatls mmc 1:1`：查看 EMMC 分区 1 中的文件。
- `fatload mmc 1:1 80800000 zImage`：将 `zImage` 加载到 DDR 的 `0x80800000`。
- `fatload mmc 1:1 83000000 ...dtb`：将 `.dtb` 加载到 `0x83000000`。

如果内核启动成功，说明 U-Boot 支持 EMMC 启动，验证成功。

### 4.2 从网络启动

```bash
tftp 80800000 zImage
tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb
bootz 80800000 - 83000000
```

说明：

- 第一条命令从 TFTP 服务器下载 `zImage`。
- 第二条命令从 TFTP 服务器下载 `.dtb`。
- 第三条命令启动系统。

## 五、U-Boot DDR 初始化

### 5.1 裸机

使用 `imxdownload` 软件下载时，会在 `.bin` 文件头部添加 IVT 和 DCD 数据。

### 5.2 U-Boot

U-Boot 编译生成 `u-boot.imx`，其中已经包含 IVT 和 DCD 数据。

需要关注的问题：

1. `u-boot.imx` 的头部信息是如何添加的？
2. `u-boot.imx` 的 DCD 中 DDR 初始化代码应该如何修改？

U-Boot 编译时会输出：

```bash
./tools/mkimage -n board/freescale/mx6ull_alientek_emmc/imximage.cfg.cfgtmp -T imximage -e 0x87800000 -d u-boot.bin u-boot.imx
```

可以看出，U-Boot 使用 `tools/mkimage` 工具向 `u-boot.bin` 添加 `board/freescale/mx6ull_alientek_emmc/imximage.cfg.cfgtmp` 文件信息，从而得到 `u-boot.imx`。

默认只有 `imximage.cfg` 文件，它保存的就是 DCD 数据，DDR 初始化也在此文件中。

如果要修改 DDR 初始化代码，就需要修改 `imximage.cfg`。这个文件默认拷贝自 NXP 给 i.MX6ULL EVK 开发板写的配置，默认是给 512MB DDR3L 使用的。
