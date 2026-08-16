# 第11讲 Linux内核顶层Makefile分析

> 来源：正点原子官方笔记整理

## 一、Linux 顶层 Makefile 基本流程

架构变量示例：

```make
hdr-arch = arm
```

## 二、`make xxx_defconfig` 过程

顶层规则：

```make
%config: scripts_basic outputmakefile FORCE
	$(Q)$(MAKE) $(build)=scripts/kconfig $@
```

`scripts_basic` 目标：

```make
scripts_basic:
	$(Q)$(MAKE) $(build)=scripts/basic
```

其中 `build` 定义在 `scripts/Kbuild.include`：

```make
build := -f $(srctree)/scripts/Makefile.build obj
```

所以两条关键命令可以理解为：

```make
$(Q)$(MAKE) $(build)=scripts/basic
$(Q)$(MAKE) $(build)=scripts/kconfig $@
```

展开后：

```bash
make -f ./scripts/Makefile.build obj=scripts/basic
```

这一步会生成：

```text
fixdep
bin2c
```

第二步：

```bash
make -f ./scripts/Makefile.build obj=scripts/kconfig xxx_defconfig
```

继续展开：

```bash
scripts/kconfig/conf --defconfig=arch/arm/configs/%_defconfig Kconfig
```

最终生成 `.config`。

## 三、`make` 过程

通过 `make` 或 `make all` 编译 Linux 内核。默认目标：

```make
_all:

_all: all

all: vmlinux
```

`vmlinux` 目标：

```make
vmlinux: scripts/link-vmlinux.sh $(vmlinux-deps) FORCE
```

条件构建项：

```make
ifdef CONFIG_HEADERS_CHECK
	$(Q)$(MAKE) -f $(srctree)/Makefile headers_check
endif

ifdef CONFIG_SAMPLES
	$(Q)$(MAKE) $(build)=samples
endif

ifdef CONFIG_BUILD_DOCSRC
	$(Q)$(MAKE) $(build)=Documentation
endif

ifdef CONFIG_GDB_SCRIPTS
	$(Q)ln -fsn `cd $(srctree) && /bin/pwd`/scripts/gdb/vmlinux-gdb.py
endif
```

最后链接：

```make
+$(call if_changed,link-vmlinux)
```

`vmlinux-deps` 相关变量：

```make
vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT) $(KBUILD_VMLINUX_MAIN)
KBUILD_LDS := arch/$(SRCARCH)/kernel/vmlinux.lds
```

展开后：

```make
KBUILD_LDS := arch/arm/kernel/vmlinux.lds
```

也就是说，`arch/arm/kernel/vmlinux.lds` 是链接脚本。

初始化和主体部分：

```make
KBUILD_VMLINUX_INIT := $(head-y) $(init-y)
export KBUILD_VMLINUX_MAIN := $(core-y) $(libs-y) $(drivers-y) $(net-y)
```

头文件目标：

```make
head-y := arch/arm/kernel/head.o
```

初始化目录：

```make
init-y := init/
init-y := $(patsubst %/, %/built-in.o, $(init-y))
init-y = init/built-in.o
```

驱动和网络目录：

```make
drivers-y := drivers/ sound/ firmware/
net-y     := net/
```

转换为 `built-in.o`：

```make
drivers-y := $(patsubst %/, %/built-in.o, $(drivers-y))
net-y     := $(patsubst %/, %/built-in.o, $(net-y))
```

展开后：

```make
drivers-y = drivers/built-in.o sound/built-in.o firmware/built-in.o
net-y     = net/built-in.o
```

库目录：

```make
libs-y := lib/
libs-y := arch/arm/lib/ $(libs-y)
libs-y := arch/arm/lib/ lib/
```

继续转换：

```make
libs-y1 := $(patsubst %/, %/lib.a, $(libs-y))
libs-y2 := $(patsubst %/, %/built-in.o, $(libs-y))
libs-y  := $(libs-y1) $(libs-y2)
```

展开后：

```make
libs-y := arch/arm/lib/lib.a lib/lib.a arch/arm/lib/built-in.o lib/built-in.o
```

所有 `.o` 和 `.a` 库文件准备好后，就可以链接生成 `vmlinux`。

最终链接命令：

```make
+$(call if_changed,link-vmlinux)
```

展开后可理解为：

```bash
/bin/bash scripts/link-vmlinux.sh arm-linux-gnueabihf-ld -EL -p --no-undefined -X --pic-veneer --build-id
```

## 四、`built-in.o` 文件编译生成过程

`vmlinux-deps` 示例：

```make
vmlinux-deps = arch/arm/kernel/vmlinux.lds arch/arm/kernel/head.o init/built-in.o usr/built-in.o arch/arm/vfp/built-in.o arch/arm/vdso/built-in.o arch/arm/kernel/built-in.o arch/arm/mm/built-in.o arch/arm/common/built-in.o arch/arm/probes/built-in.o arch/arm/net/built-in.o arch/arm/crypto/built-in.o arch/arm/firmware/built-in.o arch/arm/mach-imx/built-in.o kernel/built-in.o mm/built-in.o fs/built-in.o ipc/built-in.o security/built-in.o crypto/built-in.o block/built-in.o arch/arm/lib/lib.a lib/lib.a arch/arm/lib/built-in.o lib/built-in.o drivers/built-in.o sound/built-in.o firmware/built-in.o net/built-in.o
```

目录计算：

```make
vmlinux-dirs := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
	$(core-y) $(core-m) $(drivers-y) $(drivers-m) \
	$(net-y) $(net-m) $(libs-y) $(libs-m)))
```

展开后：

```make
vmlinux-dirs = init usr arch/arm/vfp arch/arm/vdso arch/arm/kernel arch/arm/mm arch/arm/common arch/arm/probes arch/arm/net arch/arm/crypto arch/arm/firmware arch/arm/mach-imx kernel mm fs ipc security crypto block drivers sound firmware net arch/arm/lib lib
```

构建规则：

```make
$(vmlinux-dirs): prepare scripts
	$(Q)$(MAKE) $(build)=$@
```

展开示例：

```bash
make -f ./scripts/Makefile.build obj=init
```

默认目标为 `__build`，规则可简化为：

```make
__build: $(builtin-target) $(lib-target) $(extra-y) $(subdir-ym) $(always)
	@:
```

`cmd_link_o_target`：

```make
cmd_link_o_target = $(if $(strip $(obj-y)),\
	$(LD) $(ld_flags) -r -o $@ $(filter $(obj-y), $^) \
	$(cmd_secanalysis),\
	rm -f $@; $(AR) rcs$(KBUILD_ARFLAGS) $@)
```

`built-in.o` 目标：

```make
builtin-target := $(obj)/built-in.o
```

规则：

```make
$(builtin-target): $(obj-y) FORCE
	$(call if_changed,link_o_target)
```

等价理解：

```make
$(obj)/built-in.o: $(obj-y) FORCE
	$(call if_changed,link_o_target)
```

也就是说，`built-in.o` 由当前目录下的 `obj-y` 链接生成。

## 五、`make zImage` 过程

前面主要分析的是生成 `vmlinux`，但实际启动常用的是 `zImage`。

ARM 启动镜像目标：

```make
BOOT_TARGETS = zImage Image xipImage bootpImage uImage
```

依赖关系：

```make
$(BOOT_TARGETS): vmlinux
	$(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
```

对 `zImage` 来说：

```make
zImage Image xipImage bootpImage uImage: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
```

展开后可理解为：

```bash
make -f ./scripts/Makefile.build obj=arch/arm/boot MACHINE=arch/arm/boot/zImage
```
