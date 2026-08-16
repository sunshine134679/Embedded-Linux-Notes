# GCC版本所带来的问题

> 使用的Linux内核版本为老版本(NXP 4.1.15，2015 年的老代码)，内核的GCC版本较老(小于10)。但使用的Ubuntu的GCC版本教心（大于10），会有很多改动，导致内核编译失败。



## 1. `HOST_EXTRACFLAGS`变量(GCC版本10上和10下不一致)

- **问题**：Linux系统编译失败

- **改动**：GCC版本大于10的话编译本源码的时候需要在编译脚本或者顶层`Makefile`文件中指定变量`HOST_EXTRACFLAGS=-fcommon`(建议加上 `export`)
- **缘由**：GCC 10 开始，`-fcommon` 的默认值被反转为 `-fno-common`

