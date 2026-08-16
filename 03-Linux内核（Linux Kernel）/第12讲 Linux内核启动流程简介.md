# 第12讲 Linux内核启动流程简介

> 来源：正点原子官方笔记整理

## 一、`vmlinux.lds` 简介

从 `arch/arm/kernel/vmlinux.lds` 分析 Linux 内核第一行启动代码，可以找到入口：

```asm
ENTRY(stext)
```

入口函数是 `stext`。

`Image` 和 `zImage` 是经过压缩的镜像，Linux 内核会先进行解压缩；解压缩完成后，才运行真正的 Linux 内核入口。

进入内核前要求：

1. MMU 关闭。
2. D cache 关闭。
3. I cache 无所谓。
4. `r0 = 0`。
5. `r1 = machine nr`。
6. `r2 = ATAGS` 或设备树。

## 二、Linux 入口 `stext`

主要流程：

```text
stext
-> __vet_atags
-> __create_page_tables
-> __enable_mmu
-> __turn_mmu_on
-> __mmap_switched
-> start_kernel
```

关键点：

- `__vet_atags` 验证 ATAGS 或 DTB 是否有效。使用设备树时，这里验证的就是 DTB。
- `__create_page_tables` 创建页表。
- `ldr r13, =__mmap_switched` 表示 `r13` 保存 `__mmap_switched`。
- `__enable_mmu` 使能 MMU。
- `_mmap_switched` 之后调用 `start_kernel` 启动内核。

这一段属于 Linux 内核启动第一阶段。

## 三、`__mmap_switched` 函数

`__mmap_switched` 完成 MMU 使能后的切换，并进入后续 C 语言启动流程。

## 四、`start_kernel` 函数

`start_kernel` 属于 Linux 内核启动第二阶段。

主要流程：

```text
start_kernel
-> rest_init
-> kernel_thread(kernel_init, NULL, CLONE_FS)
-> kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES)
-> cpu_startup_entry
```

关键点：

- `kernel_thread(kernel_init, NULL, CLONE_FS)` 创建 `kernel_init` 进程，也就是 init 进程，PID 为 1。
- `kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES)` 创建 `kthreadd` 进程，PID 为 2。
- `cpu_startup_entry` 进入空闲进程，主进程退化为空闲进程 `idle`。

启动开发板后，可以输入以下命令列出当前系统所有进程：

```bash
ps -A
```

## 五、`rest_init` 函数

`rest_init` 负责创建关键内核线程，并让当前 CPU 进入 idle 流程。

## 六、init 进程

`kernel_init` 主要流程：

```text
kernel_init
-> kernel_init_freeable
-> 设置标准输入、标准输出、标准错误为 console，例如 ttymxc0
-> ramdisk_execute_command = "/init"
-> 检查 /init 是否存在，存在则运行
```

U-Boot 传递给 Linux 内核的 `bootargs` 可以自定义 `init=xxx`，也就是通过命令行参数指定 init 程序。有些开发板会设置 `init=linuxrc`。

如果没有指定或前面的 init 不存在，内核会依次尝试运行：

1. `/sbin/init`
2. `/etc/init`
3. `/bin/init`
4. `/bin/sh`

由此最终引出根文件系统。
