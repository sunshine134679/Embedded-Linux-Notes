# VSCode的语言配置问题

## 一、没法使用定义跳转或者是查找定义、声明

以下是VSCode中C语言基础服务配置文件，可在VSCode中点击右下角的编程语言标识（假如是C，右下角就会有一个C的标识；假如是asm，右下角就会有个asm的标识。），之后选择语言技术服务即可。

![image-20260821140210478](D:\学习笔记\正点原子阿尔法IMX6ULL开发板学习笔记\09-问题记录\media\关于VSCode语言配置\image-20260821140210478.png)

### 现象

- 打开 `.c` / `.cpp` 文件，**右键菜单里根本没有「转到定义」「速览定义」「查找所有引用」「转到声明」这些选项**。
- 不是"点了跳不过去"，而是**菜单项压根不存在**。
- 所有新建文件夹里的 C/CPP 文件都这样；只有内核源码工作区（`linux-nxp-ported`）正常。

### 原因

**C/C++ 扩展的 IntelliSense 引擎被禁用了。**

在 VS Code 用户设置里有一项：

```
"C_Cpp.intelliSenseEngine": "disabled",
```

C/C++ 扩展的 `package.json` 里，右键导航菜单项带了一个硬性隐藏条件：

```
!(config.C_Cpp.intelliSenseEngine =~ /^[dD]isabled$/)
```

翻译：**只要引擎是 `disabled`，扩展就不显示这些菜单项**。扩展自身的输出日志里也出现了：

```
已禁用 intelliSenseEngine
```

### 验证证据

| 检查项       | 结果                                                         |
| ------------ | ------------------------------------------------------------ |
| 扩展是否安装 | ✅ `~/.vscode-server/extensions/ms-vscode.cpptools-1.33.8-linux-x64` |
| 扩展是否启用 | ✅ 未禁用，日志显示正常激活（`onLanguage:c`）                 |
| 引擎设置     | ❌ `"C_Cpp.intelliSenseEngine": "disabled"` ← **元凶**        |
| 扩展日志     | 「已禁用 intelliSenseEngine」                                |

### 解决

把该项改回 `"default"`：

```
"C_Cpp.intelliSenseEngine": "default",
```

然后：

1. `Ctrl+Shift+P` → `Reload Window`
2. 打开 `.c` 文件，右键点函数名 → 菜单里就有「转到定义 / 速览定义 / 查找所有引用」了



## 二、`.vscode` 配置放错位置，导致子工程里 IntelliSense 不生效

### 现象

- `2_led` 文件夹里明明有 `.vscode/c_cpp_properties.json`（配置内容正确），但从上级目录打开时，`led.c` 里的 `#include <linux/init.h>` 等仍然红波浪线、跳转失效。
- 只有**单独打开 `2_led` 文件夹**时才正常。

### 原因

**`.vscode` 里的配置只对"它所在文件夹作为工作区根"时生效。**

- 从 `2_led` 打开 → 读取 `2_led/.vscode/c_cpp_properties.json` ✅
- 从 `Linux_Drivers`（上级）打开 → 读取的是 `Linux_Drivers/.vscode/...`，**`2_led/.vscode` 不被读取** ❌ → includePath 没生效 → 找不到内核头文件 → 红波浪线。

### 解决

把 `.vscode` 放到**所有驱动工程共用的根目录** `Linux_Drivers` 下：

```bash
mkdir -p /home/wsl/Linux/IMX6ULL/Linux_Drivers/.vscode
cp /home/wsl/Linux/IMX6ULL/Linux_Drivers/2_led/.vscode/c_cpp_properties.json \
   /home/wsl/Linux/IMX6ULL/Linux_Drivers/2_led/.vscode/settings.json \
   /home/wsl/Linux/IMX6ULL/Linux_Drivers/.vscode/
```

然后：

1. VS Code **重新打开 `Linux_Drivers` 文件夹**（文件 → 打开文件夹 → 选中 Linux_Drivers）
2. `Ctrl+Shift+P` → `Reload Window`
3. 打开任意驱动工程的 `.c` 文件，红波浪线消失、跳转正常

### 经验

> `.vscode` 配置要放在你**实际打开的那个工作区根目录**下；如果打算从一个根目录管理多个工程，就把配置放在那个根目录，而不是每个子工程各放一份。