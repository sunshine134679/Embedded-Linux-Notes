# Makefile基础

> 个人补充笔记

## 一、Makefile 是什么

`make` 是自动化构建工具，`Makefile` 是写给 `make` 看的构建规则文件。

当工程只有一个 `.c` 文件时，可以直接使用 `gcc` 编译；当工程包含多个源文件、头文件和库时，手动输入编译命令会很麻烦，这时就适合使用 `Makefile`。

`Makefile` 主要解决三个问题：

- 要生成什么文件。
- 生成这个文件依赖哪些文件。
- 依赖变化后，应该执行什么命令重新生成。

## 二、最基本的规则

Makefile 规则格式：

```make
目标: 依赖
	命令
```

注意：命令前面必须是 Tab，不是普通空格。

示例：

```make
hello: main.c
	gcc main.c -o hello
```

含义：

- 目标是 `hello`。
- `hello` 依赖 `main.c`。
- 如果 `main.c` 比 `hello` 新，或者 `hello` 不存在，就执行下面的编译命令。

## 三、目标、依赖、命令

| 名称 | 说明 |
| ---- | ---- |
| 目标 | 想要生成的文件或要执行的动作 |
| 依赖 | 生成目标需要用到的文件 |
| 命令 | 生成目标时执行的 Shell 命令 |

示例：

```make
main.o: main.c main.h
	gcc -c main.c -o main.o
```

如果 `main.c` 或 `main.h` 更新，`make` 会重新生成 `main.o`。

## 四、第一个目标是默认目标

直接执行：

```bash
make
```

默认会构建 Makefile 中第一个规则的目标。

通常把 `all` 放在最前面：

```make
all: app

app: main.o led.o
	gcc main.o led.o -o app
```

也可以指定目标：

```bash
make clean
make app
```

## 五、变量

变量可以减少重复。

```make
CC = gcc
CFLAGS = -Wall -O2
TARGET = app
OBJS = main.o led.o

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET)
```

常见写法：

| 写法 | 含义 |
| ---- | ---- |
| `A = xxx` | 递归展开，使用时再展开变量值 |
| `A := xxx` | 立即展开，定义时就确定变量值 |
| `A ?= xxx` | 如果变量未定义，才赋值 |
| `A += xxx` | 追加内容 |

初学时可以优先使用 `:=`，行为更直观。

## 六、自动变量

自动变量只能在规则的命令中使用。

| 自动变量 | 含义 |
| ---- | ---- |
| `$@` | 当前目标 |
| `$<` | 第一个依赖 |
| `$^` | 所有依赖，去重 |
| `$+` | 所有依赖，不去重 |
| `$?` | 比目标新的依赖 |

示例：

```make
main.o: main.c
	$(CC) $(CFLAGS) -c $< -o $@
```

等价于：

```bash
gcc -Wall -O2 -c main.c -o main.o
```

## 七、模式规则

模式规则可以批量描述同类文件如何生成。

```make
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

含义：任意 `.o` 文件都可以由同名 `.c` 文件编译得到。

例如：

```text
main.c -> main.o
led.c  -> led.o
```

## 八、伪目标 `.PHONY`

有些目标不是实际文件，而是动作，例如 `clean`。

```make
.PHONY: all clean

all: app

clean:
	rm -f *.o app
```

如果不声明 `.PHONY`，当目录中刚好存在名为 `clean` 的文件时，`make clean` 可能不会执行清理命令。

## 九、一个完整 C 工程示例

目录：

```text
project/
├── Makefile
├── main.c
├── led.c
└── led.h
```

Makefile：

```make
TARGET := app
CC := gcc
CFLAGS := -Wall -O2 -MMD -MP
OBJS := main.o led.o
DEPS := $(OBJS:.o=.d)

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(DEPS) $(TARGET)

-include $(DEPS)
```

常用命令：

```bash
make
make clean
```

## 十、嵌入式交叉编译常见写法

嵌入式开发通常不用宿主机 `gcc`，而是使用交叉编译工具链。

```make
CROSS_COMPILE ?= arm-linux-gnueabihf-
CC := $(CROSS_COMPILE)gcc
LD := $(CROSS_COMPILE)ld
OBJCOPY := $(CROSS_COMPILE)objcopy
OBJDUMP := $(CROSS_COMPILE)objdump
```

示例：

```make
TARGET := led
CROSS_COMPILE ?= arm-linux-gnueabihf-
CC := $(CROSS_COMPILE)gcc
OBJCOPY := $(CROSS_COMPILE)objcopy
CFLAGS := -Wall -O2
ASFLAGS :=
LDFLAGS := -T link.lds -nostdlib

$(TARGET).bin: $(TARGET).elf
	$(OBJCOPY) -O binary $< $@

$(TARGET).elf: start.o main.o
	$(CC) $(LDFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

%.o: %.S
	$(CC) $(ASFLAGS) -c $< -o $@
```

也可以在命令行覆盖变量：

```bash
make CROSS_COMPILE=arm-none-eabi-
```

## 十一、头文件依赖

如果只写：

```make
main.o: main.c
	$(CC) -c main.c -o main.o
```

那么 `main.h` 改了以后，`main.o` 不一定会重新编译。

简单工程可以手写头文件依赖：

```make
main.o: main.c main.h
```

较大工程通常让编译器自动生成依赖文件，例如 GCC 的 `-MMD -MP`：

```make
CFLAGS += -MMD -MP
DEPS := $(OBJS:.o=.d)

-include $(DEPS)
```

## 十二、常见易错点

- 命令前必须用 Tab，不能用空格。
- 第一个目标是默认目标，通常把 `all` 放第一。
- `clean` 这类动作目标要写进 `.PHONY`。
- 自动变量 `$@`、`$<`、`$^` 只能在命令中可靠使用。
- 链接时通常使用所有 `.o` 文件，不是只编译 `main.c`。
- 头文件变化也应该触发重新编译，否则可能出现旧目标文件。
- 交叉编译时确认 `CROSS_COMPILE`、`ARCH`、工具链路径是否正确。
