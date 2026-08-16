# 第10讲 NXP官方SDK使用实验

> 来源：正点原子官方笔记整理

## 一、官方 SDK 移植

### 1.1 新建 `cc.h`

SDK 包会用到很多数据类型，因此需要在 `cc.h` 中定义常用数据类型。

### 1.2 移植文件

需要移植的文件：

- `fsl_common.h`
- `fsl_iomuxc.h`
- `MCIMX6Y2.h`

设备为：

```text
MCIMX6Y2
```

## 二、编写驱动

基于 SDK 提供的寄存器定义和函数封装编写驱动。

## 三、IO 函数

`IOMUXC_SetPinMux` 参数示例：

```c
IOMUXC_SetPinMux(
	uint32_t muxRegister,     /* 0x020e0068U */
	uint32_t muxMode,         /* 0x5U */
	uint32_t inputRegister,   /* 0x00000000U */
	uint32_t inputDaisy,      /* 0x0U */
	uint32_t configRegister,  /* 0x020e02f4U */
	uint32_t inputOnfield     /* 0 */
);
```

`IOMUXC_SetPinConfig` 参数示例：

```c
IOMUXC_SetPinConfig(
	uint32_t muxRegister,
	uint32_t muxMode,
	uint32_t inputRegister,
	uint32_t inputDaisy,
	uint32_t configRegister,  /* 0x020e02f4U */
	uint32_t configValue      /* 0x10b0 */
);
```
