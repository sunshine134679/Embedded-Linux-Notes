# 第21讲 RGBLCD实验

> 来源：正点原子官方笔记整理

## 一、RGB LCD 显示原理简介

### 1.1 像素点

液晶屏、手机、平板、RGB LCD 屏幕都由一个个彩色像素点构成。每个彩色像素点通常包含红色、绿色、蓝色三个子像素，也就是 RGB。

RGB 是光的三原色，通过调整三种颜色的比例，可以显示不同颜色。

### 1.2 分辨率

要显示文字、图片、视频等内容，需要很多像素点。分辨率表示像素点个数。

示例：

```text
1080P = 1920 * 1080
```

表示一行有 1920 个像素点，一列有 1080 个像素点。

常见分辨率：

- 720P
- 1080P
- 2K
- 4K
- 8K

屏幕尺寸不变时，分辨率越高，显示效果越细腻。

正点原子 RGB 屏幕示例：

| 尺寸 | 分辨率 |
| ---- | ---- |
| 4.3 寸 | `480 * 272` |
| 7 寸 | `800 * 480` |
| 7 寸 | `1024 * 600` |
| 10.1 寸 | `1280 * 800` |

### 1.3 像素格式

像素格式用于描述 RGB 三种颜色如何量化。

常见格式：

| 格式 | 说明 |
| ---- | ---- |
| RGB888 | R/G/B 各 8 bit，共 24 bit，可描述约 1677 万种颜色 |
| ARGB8888 | 在 RGB888 基础上增加 8 bit Alpha 通道，共 32 bit |
| RGB101010 | R/G/B 各 10 bit，常用于 HDR10 等场景 |

### 1.4 LCD 屏幕接口

RGB 格式屏幕通常称为 RGB 接口屏。

常见屏幕接口：

- MIPI
- LVDS
- MCU
- RGB

正点原子屏幕 ID 通过在 RGB LCD 屏幕的 `R7`、`G7`、`B7` 上焊接上拉或下拉电阻实现，用于识别不同屏幕。

ALPHA 底板 RGB 屏幕接口使用了 3 个 3157 模拟开关，目的是防止 LCD 屏幕上的 ID 电阻影响 6ULL 启动。

### 1.5 LCD 时间参数和时序

水平同步信号：

- `HSYNC`：行同步信号，出现 `HSYNC` 表示新的一行开始显示。

一行显示过程：

1. 产生 `HSYNC` 信号，表示新的一行开始，持续时间为 `HSPW`。
2. `HSYNC` 信号结束后，需要一段延时，称为 `HBP`。
3. 显示一行像素数据，例如 1024 个像素点需要 1024 个 `CLK`。
4. 一行显示完成后，到下一行 `HSYNC` 产生前的延时称为 `HFP`。

一行总时钟数：

```text
HSPW + HBP + WIDTH + HFP
= 20 + 140 + 1024 + 160
= 1344 CLK
```

垂直同步信号：

- `VSYNC`：帧同步信号，出现 `VSYNC` 表示新的一帧开始显示。

一帧显示过程：

1. `VSYNC` 信号持续一段时间，称为 `VSPW`。
2. `VSYNC` 信号结束后，需要一段延时，称为 `VBP`。
3. 显示有效行数，例如 600 行。
4. 所有行显示完成后，进入 `VFP` 延时。

一帧所需时钟：

```text
(VSPW + VBP + HEIGHT + VFP) * (HSPW + HBP + WIDTH + HFP)
```

### 1.6 显存

显存就是显示存储空间。

如果使用 `ARGB8888`：

```text
ARGB8888 = 32 bit = 4 B
```

对于 `1024 * 600` 屏幕：

```text
1024 * 600 * 4 = 2.5 MB
```

因此需要预留约 2.5 MB 内存给 LCD 使用。

示例：

```c
u32 lcdframe[1024 * 600];
```

## 二、6ULL LCDIF 控制器接口原理

本实验使用 DOTCLK 接口：

- `VSYNC`
- `HSYNC`
- `ENABLE` / `DE`
- `DOTCLK` / `PCLK`

关键寄存器：

| 寄存器 | 作用 |
| ---- | ---- |
| `LCDIF_CTRL` | LCDIF 主控制寄存器 |
| `LCDIF_CTRL1` | 数据格式相关控制 |
| `LCDIF_TRANSFER_COUNT` | 设置水平和垂直像素数量 |
| `LCDIF_VDCTRL0` ~ `LCDIF_VDCTRL4` | 设置显示时序 |
| `LCDIF_CUR_BUF` | 当前显存首地址 |
| `LCDIF_NEXT_BUF` | 下一帧显存首地址 |

`LCDIF_CTRL` 常用位：

| 位 | 设置 | 作用 |
| ---- | ---- | ---- |
| `bit0` | `1` | 使能 LCDIF |
| `bit1` | `0` | 24 位数据全部有效 |
| `bit5` | `1` | 主机模式 |
| `bit9:8` | `3` | 输入像素格式为 24 bit |
| `bit11:10` | `3` | 数据传输宽度为 24 bit |
| `bit13:12` | `0` | 不进行数据交换 |
| `bit15:14` | `0` | 输入数据不交换 |
| `bit17` | `1` | DOTCLK 模式 |
| `bit19` | `1` | DOTCLK 模式相关要求 |
| `bit31` | `0` | 复位位必须为 0 |

其他寄存器说明：

- `LCDIF_CTRL1 bit19:16` 设置为 `0x7`，表示 24 位格式。
- `LCDIF_TRANSFER_COUNT bit15:0` 是一行像素数，例如 1024。
- `LCDIF_TRANSFER_COUNT bit31:16` 是行数，例如 600。
- `LCDIF_VDCTRL0 bit17:0` 为 `VSPW` 参数。
- `LCDIF_VDCTRL1` 表示两个 `VSYNC` 信号之间的长度，即 `VSPW + VBP + HEIGHT + VFP`。
- `LCDIF_VDCTRL2 bit17:0` 表示两个 `HSYNC` 信号之间的长度，即 `HSPW + HBP + WIDTH + HFP`。
- `LCDIF_VDCTRL2 bit31:18` 为 `HSPW`。
- `LCDIF_VDCTRL3 bit15:0` 为 `VBP + VSPW`。
- `LCDIF_VDCTRL3 bit27:16` 为 `HBP + HSPW`。
- `LCDIF_VDCTRL4 bit17:0` 是一行像素点数量，例如 1024。

还需要完成 LCD IO 初始化。

## 三、LCD 像素时钟设置

LCD 需要 `CLK` 信号，该信号由 6ULL 的 CLK 引脚发送给 RGB LCD。

例如 7 寸 `1024 * 600` 屏幕需要约 51.2 MHz 的像素时钟。

`LCDIF_CLK_ROOT` 是 6ULL 的像素时钟。本实验设置 `PLL5`，也就是 Video PLL，为 LCD 时钟源。

PLL5 设置：

```text
PLL5_CLK = Fref * DIV_SELECT
```

`DIV_SELECT` 是 `CCM_ANALOG_PLL_VIDEO bit6:0`，可选范围为 27 ~ 54。

其他配置：

1. 设置 `CCM_ANALOG_PLL_VIDEO bit20:19 = 2`，表示 1 分频。
2. 设置 `CCM_ANALOG_MISC2 bit31:30 = 0`，也就是 `VIDEO_DIV = 0`，1 分频。
3. 不使用小数分频器，因此：

   ```text
   CCM_ANALOG_PLL_VIDEO_NUM = 0
   CCM_ANALOG_PLL_VIDEO_DENOM = 0
   ```

LCDIF 时钟源和分频：

| 寄存器位 | 作用 | 设置 |
| ---- | ---- | ---- |
| `CCM_CSCDR2 bit17:15` | `LCDIF_PRE_CLK_SEL`，选择 `LCDIF_CLK_ROOT` 时钟源 | `0x2`，选择 PLL5 |
| `CCM_CSCDR2 bit14:12` | `LCDIF_PRED`，前级分频 | `0` ~ `7` 对应 1 ~ 8 分频 |
| `CCM_CBCMR bit25:23` | `LCDIF_PODF`，第二级分频 | `0` ~ `7` 对应 1 ~ 8 分频 |
| `CCM_CSCDR2 bit11:9` | `LCDIF_CLK_SEL`，选择 LCD CLK 最终时钟源 | `0`，来源于 pre-muxed LCDIF clock |

## 四、LCD 驱动程序编写

如果使用正点原子开发板和 RGB 屏幕，驱动 LCD 前需要先读取屏幕 ID。

## 五、LCD 操作 API 函数编写

基于显存地址和 LCDIF 配置，可以封装清屏、画点、显示字符、显示图片等 API。
