# 第3讲 U-Boot 命令使用

> 来源：正点原子官方笔记整理

## 一、U-Boot 启动 Log 简析

带有调试性质的开发中，U-Boot 通常烧写到 SD 卡中，便于频繁更新和验证。

## 二、U-Boot 命令使用

### 2.1 `help` 命令

查看某个命令的帮助信息：

```text
? 命令名
```

### 2.2 信息查询

- `bdinfo`
- `printenv`：查看当前板子的环境变量，重要。

### 2.3 `setenv` 命令

`setenv` 用于设置环境变量，也可以自定义或删除环境变量，是重点命令。

### 2.4 `saveenv` 命令

`saveenv` 用于保存环境变量，是重点命令。

### 2.5 新建环境变量

使用 `setenv` 新建环境变量。

### 2.6 删除环境变量

使用 `setenv` 删除环境变量。

### 2.7 内存操作命令

- `md`
- `nm`
- `mm`
- `mw`
- `cp`
- `cmp`

### 2.8 网络操作命令

网线插入 ENET2，保证开发板和电脑处于同一个网段内。

- `ping`：重点。
- `dhcp`
- `nfs`：重点，目的主要是调试程序。
- `tftp`：重点。

### 2.9 EMMC / SD 卡操作命令

- `mmc`
- `mmc info`
- `mmc rescan`
- `mmc list`
- `mmc dev`
- `mmc part`
- `mmc read`
- `mmc write`
- `mmc erase`：最好不要使用。

### 2.10 FAT 格式文件系统操作命令

对于 i.MX6U 来说，SD / EMMC 通常分为三个分区：

| 分区 | 用途 |
| ---- | ---- |
| 第一个分区 | 存放 U-Boot |
| 第二个分区 | 存放 Linux `zImage` 和 `.dtb`，FAT 格式 |
| 第三个分区 | 根文件系统，EXT4 格式 |

- `fatinfo`
- `fatls`
- `fstype`
- `fatload`
- `fatwrite`

### 2.11 EXT 格式文件系统操作命令

- `ext4ls`

### 2.12 NAND 操作命令

- `nand info`
- `nand write`
- `nand erase`
- `nand read`

### 2.13 BOOT 操作命令

- `bootz`

  启动 Linux 前，必须先将 `zImage` 和 `.dtb` 放到 DRAM 中。

- `bootm`
- `boot`

### 2.14 其他命令

- `reset`
- `go`
- `run`
- `mtest`
