# 第11讲 BSP工程管理实验

> 来源：正点原子官方笔记整理

## 一、BSP 工程管理原理

BSP 工程管理的目的，是把代码模块化整理，让同一属性的文件放到对应目录中。

整理步骤：

1. 新建所需文件夹，将同一属性的文件放入对应文件夹。
2. 修改 `clk`、`led`、`delay` 驱动，创建对应驱动文件并放入对应目录。
3. 根据新的驱动文件修改 `main.c`。

VSCode 头文件路径设置：

1. 创建 `.vscode` 目录。
2. 打开 C/C++ 配置器。
3. 生成 `.vscode/c_cpp_properties.json`。
4. 在配置文件中添加头文件搜索路径。

## 二、Makefile 编写

Makefile 中指定头文件路径需要使用 `-I`。

例如：

```make
-I bsp/clk
```

不要把头文件本身写成包含路径，例如下面这种写法不合适：

```make
-I bsp/clk/bsp_clk.h
```

Makefile 通过变量准备要编译的源文件、目标文件、头文件路径等原材料。

静态模式规则格式：

```make
<targets ...>: <target-pattern>: <prereq-patterns ...>
```

示例：

```make
$(OBJS): obj/%.o : %.S
```

含义：将所有 `.S` 文件编译为 `.o`，并存放到 `obj` 目录。
