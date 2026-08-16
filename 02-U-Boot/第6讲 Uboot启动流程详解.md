# 第6讲 U-Boot 启动流程详解

> 来源：正点原子官方笔记整理

## 一、链接脚本 `u-boot.lds` 详解

从 `u-boot.lds` 可知，U-Boot 入口地址为 `_start`。

关键地址和段信息：

```text
__image_copy_start -> 0x87800000
.vectors           -> 0x87800000，存放中断向量表
arch/arm/cpu/armv7/start.o
start.c
__image_copy_end   -> 0x8785dc6c
```

重定位相关段：

```text
__rel_dyn_start -> 0x8785dc6c，rel 段
__rel_dyn_end   -> 0x878668a4
```

镜像结束位置：

```text
__end              -> 0x878668a4
_image_binary_end  -> 0x878668a4
```

BSS 段：

```text
__bss_start -> 0x8785dc6c
__bss_end   -> 0x878a8d74
```

## 二、U-Boot 启动流程

### 2.1 `reset` 函数

`bicne` 可以理解为 `bic + ne`。

`reset` 函数主要完成：

1. 将处理器设置为 SVC 模式，并关闭 FIQ 和 IRQ。
2. 设置中断向量。
3. 初始化 CP15。

### 2.2 `lowlevel_init` 函数

`CONFIG_SYS_INIT_SP_ADDR` 的计算过程：

```c
#define CONFIG_SYS_INIT_SP_ADDR \
	(CONFIG_SYS_INIT_RAM_ADDR + CONFIG_SYS_INIT_SP_OFFSET)
```

```c
#define CONFIG_SYS_INIT_SP_OFFSET \
	(CONFIG_SYS_INIT_RAM_SIZE - GENERATED_GBL_DATA_SIZE)
```

```c
#define CONFIG_SYS_INIT_RAM_SIZE IRAM_SIZE
#define IRAM_SIZE                0x00020000
```

```c
#define CONFIG_SYS_INIT_RAM_ADDR IRAM_BASE_ADDR
#define IRAM_BASE_ADDR           0x00900000  /* 6ULL 内部 OCRAM */
```

```c
#define GENERATED_GBL_DATA_SIZE 256
```

最终计算：

```text
0x00900000 + CONFIG_SYS_INIT_SP_OFFSET
= 0x00900000 + CONFIG_SYS_INIT_RAM_SIZE - GENERATED_GBL_DATA_SIZE
= 0x00900000 + 0x00020000 - 256
= 0x0091ff00
```

该阶段会设置 SP 指针和 R9 寄存器。

### 2.3 `s_init` 函数

`s_init` 是空函数。

### 2.4 `_main` 函数

`_main` 会继续引导到后续板级初始化流程。

### 2.5 `board_init_f` 函数

`initcall_run_list` 会调用一系列初始化函数，这些函数保存在 `init_sequence_f` 数组中。

版本字符串相关宏：

```c
version_string[] = U_BOOT_VERSION_STRING
#define U_BOOT_VERSION_STRING U_BOOT_VERSION " (" U_BOOT_DATE " - " \
	U_BOOT_TIME " " U_BOOT_TZ ")" CONFIG_IDENT_STRING
```

版本号示例：

```make
U_BOOT_VERSION = U-Boot 2016.03
```

内存相关配置：

```c
TOTAL_MALLOC_LEN      (CONFIG_SYS_MALLOC_LEN + CONFIG_ENV_SIZE)
CONFIG_SYS_MALLOC_LEN (16 * SZ_1M)
CONFIG_ENV_SIZE       SZ_8K
```

`mx6ullevk.c` 和 `mx6ullevk.h` 是后续移植中经常打交道的文件。

### 2.6 `relocate_code` 函数

`relocate_code` 有一个参数：

```text
r0 = gd->relocaddr = 0x9ff47000，U-Boot 重定位后的首地址
r1 = 0x87800000，源地址起始地址
r4 = 0x9ff47000 - 0x87800000 = 0x18747000，偏移
r2 = 0x8785dc6c
```

如果只是简单地把 U-Boot 从 `0x87800000` 拷贝到其他位置，函数调用和全局变量引用会出问题。U-Boot 对这个问题的处理方法是使用位置无关码，并借助 `.rel.dyn` 段完成重定位修正。

重定位前示例：

```asm
8785dcf8 <rel_a>:
8785dcf8: 00000000  andeq r0, r0, r0
```

```asm
878042b4 <rel_test>:
878042b4: e59f300c  ldr r3, [pc, #12]   ; 878042c8 <rel_test+0x14>
878042b8: e3a02064  mov r2, #100        ; 0x64
878042bc: e59f0008  ldr r0, [pc, #8]    ; 878042cc <rel_test+0x18>
878042c0: e5832000  str r2, [r3]
878042c4: ea00d64c  b   87839bfc <printf>
878042c8: 8785dcf8
878042cc: 87842aaf
```

`r3` 会被设置为 `878042b4 + 8 + 12 = 878042c8` 处保存的值，即 `r3 = 8785dcf8`。这里并没有直接读取 `rel_a` 的地址，而是借助 `878042c8` 这个 Label。

重定位后示例：

```asm
9ffa4cf8 <rel_a>:
9ffa4cf8: 00000000  andeq r0, r0, r0
```

```asm
9ff4b2b4 <rel_test>:
9ff4b2b4: e59f300c  ldr r3, [pc, #12]
9ff4b2b8: e3a02064  mov r2, #100        ; 0x64
9ff4b2bc: e59f0008  ldr r0, [pc, #8]
9ff4b2c0: e5832000  str r2, [r3]
9ff4b2c4: ea00d64c  b   87839bfc <printf>
9ff4b2c8: 8785dcf8
9ff4b2cc: 87842aaf
```

此时 Label 中的值还是旧地址，所以必须把 `8785dcf8` 修正为重定位后的 `rel_a` 地址：

```text
8785dcf8 + 0x18747000 = 0x9ffa4cf8
```

因此，重定位后需要把所有 Label 保存的数据加上偏移。

`.rel.dyn` 段中的记录示例：

```asm
8785dcec: 87800020
8785dcf0: 00000017
...
8785e2fc: 878042c8
8785e300: 00000017
```

含义可以理解为：

```text
878042c8 + offset = 读取新的 Label 处的数据 + offset
```

完成这个功能时，链接阶段需要加上 `-pie`。

### 2.7 `relocate_vectors` 函数

设置 VBAR 寄存器为重定位后的中断向量表起始地址。

### 2.8 `board_init_r` 函数

`board_init_r` 和 `board_init_f` 类似，也会执行 `init_sequence_r` 初始化序列。

### 2.9 `run_main_loop` 函数

主循环调用链：

```text
run_main_loop
-> main_loop
-> bootdelay_process
-> autoboot_command
-> abortboot
-> abortboot_normal
-> cli_loop
-> parse_file_outer
-> parse_stream_outer
-> parse_stream
-> run_list
-> run_list_real
-> run_pipe_real
-> cmd_process
```

关键点：

- `bootdelay_process` 获取 `bootdelay` 的值，并保存到 `stored_bootdelay` 全局变量中。
- `bootdelay_process` 还会获取 `bootcmd` 环境变量的值并返回。
- `autoboot_command` 的参数是 `bootcmd` 的值。
- `abortboot` / `abortboot_normal` 会处理倒计时。
- `cli_loop` 进入 U-Boot 命令模式。
- `cmd_process` 负责处理并执行命令。

### 2.10 `cli_loop` 函数

`cli_loop` 负责 U-Boot 命令行交互。

### 2.11 `cmd_process` 函数

U-Boot 使用 `U_BOOT_CMD` 定义命令，使用 `CONFIG_CMD_XXX` 使能某个命令。

`U_BOOT_CMD` 最终会定义一个 `cmd_tbl_t` 类型的变量，所有命令最终都存放在 `.u_boot_list` 段中。`cmd_tbl_t` 的 `cmd` 成员就是具体命令执行函数，命令执行函数通常命名为 `do_xxx`。

命令处理流程：

```text
cmd_process
-> find_cmd
-> cmd_call
-> cmdtp->cmd
```

其中，`find_cmd` 会从 `.u_boot_list` 段查找命令；找到后以 `cmd_tbl_t` 类型返回，然后通过 `cmdtp->cmd` 直接调用命令函数。

## 三、`bootz` 启动 Linux 内核过程

U-Boot 启动 Linux 内核可以使用 `bootz` 命令。这里分析 `bootz` 如何启动 Linux，以及 U-Boot 的使命如何结束。

### 3.1 `images` 全局变量

```c
bootm_headers_t images;
```

### 3.2 `do_bootz` 函数

典型启动命令：

```bash
tftp 80800000 zImage
tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb
bootz 80800000 - 83000000
```

`bootz` 命令的执行函数是 `do_bootz`。

主要流程：

```text
do_bootz
-> bootz_start
-> do_bootm_states(BOOTM_STATE_START)
-> bootm_start
-> bootz_setup
-> bootm_find_images
-> boot_get_fdt
-> bootm_disable_interrupts
-> do_bootm_states(BOOTM_STATE_OS_PREP, BOOTM_STATE_OS_FAKE_GO, BOOTM_STATE_OS_GO)
-> bootm_os_get_boot_func
-> boot_fn(BOOTM_STATE_OS_PREP, argc, argv, images)
-> boot_prep_linux
-> boot_selected_os
-> do_bootm_linux(BOOTM_STATE_OS_GO)
-> boot_jump_linux
-> announce_and_cleanup
-> kernel_entry(0, machid, r2)
```

关键步骤：

- `bootm_start` 会对 `images` 全局变量清零。
- `images->ep = 0x80800000`，表示内核入口地址。
- `bootz_setup` 判断 `zImage` 是否正确。
- `boot_get_fdt` 找到设备树，并将设备树起始地址和长度写入 `images` 的 `ft_addr` 和 `ft_len` 成员。
- `bootm_disable_interrupts` 关闭中断相关功能。
- `images.os.os = IH_OS_LINUX` 表示要启动 Linux 系统。
- `bootm_os_get_boot_func` 查找 Linux 内核启动函数，找到 `do_bootm_linux` 后赋值给 `boot_fn`。
- `boot_prep_linux` 完成启动 Linux 前的准备工作。使用设备树时，`bootargs` 会通过设备树传递给 Linux 内核。
- `boot_jump_linux` 最终跳转到 Linux 内核。

跳转内核相关代码逻辑：

```c
machid = gd->bd->bi_arch_number;

void (*kernel_entry)(int zero, int arch, uint params);
kernel_entry = (void (*)(int, int, uint))images->ep;

kernel_entry(0, machid, r2);
```

`images->ep` 为 `0x80800000`。执行 `kernel_entry(0, machid, r2)` 后，U-Boot 的最终使命完成，Linux 内核开始运行。

`zImage` 头部中的 `zi_magic` 是 `zImage` 的幻数，应该为 `0x016f2818`。前面有 9 个 32 位数据，`9 * 4 = 36`，所以从第 36 个字节开始就是 `zImage` 的幻数。

### 3.3 相关函数

- `do_bootm_states`
- `bootm_os_get_boot_func`
- `do_bootm_linux`
