# 第10讲 Linux内核源码目录分析

> 来源：正点原子官方笔记整理

## 一、Linux 源码目录分析

1. 分析 Linux 内核前，最好先编译一遍 Linux 内核，然后压缩源码并拷贝到 Windows 下阅读。
2. `arch/arm/boot/` 是 ARM 架构启动镜像目录。Linux 内核编译完成后，会在此目录下生成 `Image`、`zImage`。
3. `arch/arm/boot/dts` 存放 Linux 内核设备树文件，也包括 `.dts` 编译后对应的 `.dtb` 文件。

## 二、Linux 内核 VSCode 工程创建

后续可以基于编译过的 Linux 内核源码创建 VSCode 工程，方便跳转和分析。
