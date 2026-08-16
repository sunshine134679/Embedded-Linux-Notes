# 第7讲 Ubuntu软件安装

> 来源：正点原子官方笔记整理

## 一、什么是软件

软件是运行在操作系统上的程序或工具。Ubuntu 下的软件可以通过应用商店、包管理器、安装包或源码编译等方式安装。

## 二、Ubuntu 软件安装方法

### 2.1 通过 App Store 安装

通过图形化应用商店安装，适合常见桌面软件。

### 2.2 使用 APT 工具安装

`apt` 是 Ubuntu 常用包管理工具，可以自动下载、配置和安装软件。安装软件通常需要 `sudo` 权限。

```bash
sudo apt-get install 软件包名
```

### 2.3 使用 `.deb` 软件包安装

使用 `dpkg` 安装 `.deb` 软件包：

```bash
sudo dpkg -i xxxx.deb
```

Windows 下常见安装包是 `.exe`，Ubuntu 下常见安装包是 `.deb`。

### 2.4 下载源码后编译安装

常见流程：

```bash
make
sudo make install
```

### 2.5 其他安装方法

部分软件会提供 `.run` 安装文件，例如 Qt 安装包。
