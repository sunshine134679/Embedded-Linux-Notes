# CMake基础

> 个人补充笔记

## 一、CMake 是什么

`CMake` 是跨平台构建系统生成工具。它本身通常不直接编译源码，而是根据 `CMakeLists.txt` 生成底层构建系统文件，例如 Makefile、Ninja 工程文件、Visual Studio 工程文件等。

可以简单理解为：

```text
CMakeLists.txt -> CMake -> Makefile / Ninja / VS 工程 -> 编译器
```

CMake 解决的问题：

- 用统一写法描述工程。
- 自动生成不同平台的构建文件。
- 管理源文件、头文件、库、编译选项和安装规则。
- 更适合中大型 C / C++ 工程。

## 二、最小 CMake 工程

目录：

```text
project/
├── CMakeLists.txt
└── main.c
```

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.16)

project(hello C)

add_executable(hello main.c)
```

构建命令：

```bash
cmake -S . -B build
cmake --build build
```

说明：

| 命令 | 作用 |
| ---- | ---- |
| `cmake -S . -B build` | 配置工程，源码目录是当前目录，构建目录是 `build` |
| `cmake --build build` | 调用底层构建工具开始编译 |

建议始终使用单独的 `build` 目录，避免构建产物污染源码目录。

## 三、常用基础命令

### 3.1 `cmake_minimum_required`

指定项目需要的最低 CMake 版本。

```cmake
cmake_minimum_required(VERSION 3.16)
```

### 3.2 `project`

声明项目名称和语言。

```cmake
project(my_app C CXX)
```

常见语言：

- `C`
- `CXX`
- `ASM`

### 3.3 `add_executable`

生成可执行文件。

```cmake
add_executable(app main.c led.c)
```

### 3.4 `add_library`

生成库。

```cmake
add_library(bsp STATIC led.c delay.c)
```

常见库类型：

| 类型 | 说明 |
| ---- | ---- |
| `STATIC` | 静态库 |
| `SHARED` | 动态库 |
| `INTERFACE` | 只传递头文件路径、编译选项等，不生成实际库文件 |

## 四、目标式写法

现代 CMake 推荐围绕 target 写配置，而不是到处修改全局变量。

### 4.1 添加头文件路径

```cmake
target_include_directories(app
	PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

可见性：

| 关键字 | 含义 |
| ---- | ---- |
| `PRIVATE` | 只当前目标使用 |
| `PUBLIC` | 当前目标使用，链接它的目标也会继承 |
| `INTERFACE` | 当前目标自己不用，只传递给链接它的目标 |

### 4.2 添加编译选项

```cmake
target_compile_options(app
	PRIVATE
		-Wall
		-O2
)
```

### 4.3 添加宏定义

```cmake
target_compile_definitions(app
	PRIVATE
		DEBUG
)
```

等价于给编译器传：

```text
-DDEBUG
```

### 4.4 链接库

```cmake
target_link_libraries(app
	PRIVATE
		bsp
)
```

## 五、一个稍完整的 C 工程示例

目录：

```text
project/
├── CMakeLists.txt
├── include/
│   └── led.h
└── src/
    ├── main.c
    └── led.c
```

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.16)

project(app C)

add_executable(app)

target_sources(app
	PRIVATE
		src/main.c
		src/led.c
)

target_include_directories(app
	PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/include
)

target_compile_options(app
	PRIVATE
		-Wall
		-O2
)
```

构建：

```bash
cmake -S . -B build
cmake --build build
```

运行：

```bash
./build/app
```

## 六、Debug 和 Release

单配置生成器，例如 Unix Makefiles、Ninja，常用：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

常见构建类型：

| 类型 | 说明 |
| ---- | ---- |
| `Debug` | 带调试信息，优化较少 |
| `Release` | 优化较高，适合发布 |
| `RelWithDebInfo` | 优化并保留调试信息 |
| `MinSizeRel` | 尽量减小体积 |

注意：Visual Studio 这类多配置生成器通常在 build 阶段选择配置。

```bash
cmake --build build --config Debug
```

## 七、变量和缓存变量

普通变量：

```cmake
set(SRC_LIST src/main.c src/led.c)
```

使用变量：

```cmake
add_executable(app ${SRC_LIST})
```

缓存变量可以从命令行传入：

```bash
cmake -S . -B build -DENABLE_LOG=ON
```

CMake 中定义选项：

```cmake
option(ENABLE_LOG "Enable log output" ON)

if(ENABLE_LOG)
	target_compile_definitions(app PRIVATE ENABLE_LOG)
endif()
```

## 八、CMake 和 Makefile 的关系

CMake 不是 Makefile 的简单替代品。

| 工具 | 角色 |
| ---- | ---- |
| Makefile | 直接描述构建规则，给 `make` 执行 |
| CMake | 生成底层构建系统文件，可以生成 Makefile，也可以生成 Ninja 或其他工程文件 |

常见流程：

```bash
cmake -S . -B build -G "Unix Makefiles"
cmake --build build
```

这里 CMake 会生成 Makefile，再由 `make` 完成实际编译。

如果使用 Ninja：

```bash
cmake -S . -B build -G Ninja
cmake --build build
```

## 九、嵌入式交叉编译基础

交叉编译时，通常使用 toolchain file 指定编译器和目标系统。

示例：`toolchain-arm.cmake`

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)
```

配置时指定：

```bash
cmake -S . -B build-arm -DCMAKE_TOOLCHAIN_FILE=toolchain-arm.cmake
cmake --build build-arm
```

裸机工程还可能需要指定：

- 链接脚本。
- 启动文件 `start.S`。
- `-nostdlib` 等编译或链接选项。
- `objcopy` 生成 `.bin` 文件。
- 如果目标是裸机程序，`CMAKE_SYSTEM_NAME` 通常更适合写成 `Generic`；如果目标运行 Linux 用户态程序，再写 `Linux`。

## 十、生成 `.bin` 的思路

嵌入式裸机常需要从 ELF 生成 BIN。

```cmake
cmake_minimum_required(VERSION 3.16)

project(led C ASM)

add_executable(led start.S main.c)

target_link_options(led
	PRIVATE
		-T${CMAKE_CURRENT_SOURCE_DIR}/link.lds
)

add_custom_command(TARGET led
	POST_BUILD
	COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:led> ${CMAKE_CURRENT_BINARY_DIR}/led.bin
)
```

如果 `CMAKE_OBJCOPY` 没有自动设置，可以在 toolchain file 中指定：

```cmake
set(CMAKE_OBJCOPY arm-linux-gnueabihf-objcopy)
```

## 十一、常见易错点

- `CMakeLists.txt` 文件名大小写要正确。
- 推荐使用独立 `build` 目录，不要在源码目录里直接生成构建文件。
- 改了 toolchain file 后，最好删除旧 `build` 目录重新配置。
- 优先使用 `target_include_directories`、`target_link_libraries`、`target_compile_options` 这类目标式命令。
- 不要把头文件路径写进 `target_link_libraries`。
- `CMAKE_BUILD_TYPE` 对 Unix Makefiles / Ninja 有效，对 Visual Studio 这类多配置生成器不按同样方式工作。
- CMake 配置阶段和编译阶段不同：`cmake -S -B` 是配置，`cmake --build` 才是编译。
