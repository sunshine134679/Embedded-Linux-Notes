# 第8讲 U-Boot 图形化界面配置

> 来源：正点原子官方笔记整理

## 一、U-Boot 图形化配置方法

1. 通过终端配置。
2. 进入 U-Boot 源码根目录。
3. 先执行默认配置：

   ```bash
   make mx6ull_alientek_emmc_defconfig
   ```

4. 输入 `make menuconfig`，打开图形化配置界面。
5. 新电脑需要注意安装 `ncurses` 库。

图形化配置界面对功能编译有 3 种模式：

| 模式 | 含义 |
| ---- | ---- |
| `Y` | 对应功能编译进 U-Boot |
| `N` | 对应功能不编译进 U-Boot |
| `M` | 对应功能编译为模块，Linux 内核中常用 |

配置完成后，需要保存自己的配置文件。

## 二、`menuconfig` 图形化配置原理

U-Boot 源码根目录下的 `Kconfig` 是图形化界面配置文件。

### 2.1 `mainmenu`

`mainmenu` 表示主菜单。

### 2.2 调用子目录下的 `Kconfig`

```kconfig
source "xxx/Kconfig"
```

### 2.3 `menu` / `endmenu`

`menu` 和 `endmenu` 之间表示子菜单。

### 2.4 `choice` / `endchoice`

`choice` 和 `endchoice` 表示多选一。

### 2.5 `config` 条目

`config` 条目以 `config` 开头，后面紧跟配置项，例如 `LOCALVERSION`。

使能条目后，会在 `.config` 中生成：

```make
CONFIG_LOCALVERSION=y
```

### 2.6 `depends on` 和 `select`

- `select`：选中某项后，`select` 对应的项目也会被选中。
- `depends on`：指定依赖项，依赖项需要先被选中，否则当前项目不能选择。

### 2.7 `menuconfig`

`menuconfig` 用于定义带菜单入口的配置项。

## 三、添加自定义菜单

后续可以通过修改 `Kconfig` 添加自己的自定义菜单。
