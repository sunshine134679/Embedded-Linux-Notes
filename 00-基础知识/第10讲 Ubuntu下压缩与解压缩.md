# 第10讲 Ubuntu下压缩与解压缩

> 来源：正点原子官方笔记整理

## 一、Linux 下常用压缩格式

Linux 下常见压缩扩展名：

- `.tar`
- `.tar.bz2`
- `.tar.gz`

## 二、Windows 下安装 7-Zip

Linux 下很多文件以 `.bz2`、`.gz` 结尾。如果需要在 Windows 下处理这些压缩文件，可以安装 7-Zip。

## 三、`gzip` 压缩工具

`gzip` 负责压缩和解压 `.gz` 格式文件。

```bash
gzip xxx
gzip -d xxx.gz
```

对文件夹递归处理：

```bash
gzip -r xxx
gzip -rd xxx.gz
```

`gzip` 虽然可以递归处理文件夹，但不提供打包功能，只是对文件夹中的每个文件分别压缩。

## 四、`bzip2` 压缩工具

`bzip2` 与 `gzip` 类似，负责压缩和解压 `.bz2` 格式文件。

```bash
bzip2 -z xxx
bzip2 -d xxx.bz2
```

## 五、`tar` 打包工具

`tar` 常用参数：

| 参数 | 作用 |
| ---- | ---- |
| `-f` | 使用归档文件或 ARCHIVE 设备 |
| `-c` | 创建新归档 |
| `-x` | 从归档中解出文件 |
| `-j` | 使用 `bzip2` 压缩格式 |
| `-z` | 使用 `gzip` 压缩格式 |
| `-v` | 打印命令执行过程 |

只打包和解包：

```bash
tar -vcf test.tar test
tar -vxf test.tar
```

`tar` 可以在打包和解包的同时调用 `gzip` / `bzip2`，实现类似 Windows 下 WinRAR 的压缩归档功能。

### 5.1 `.tar.bz2` 压缩和解压

```bash
tar -vcjf xxx.tar.bz2 xxx
tar -vxjf xxx.tar.bz2
```

### 5.2 `.tar.gz` 压缩和解压

```bash
tar -vczf xxx.tar.gz xxx
tar -vxzf xxx.tar.gz
```

## 六、其他格式压缩和解压

### 6.1 `.rar` 格式

先安装 `rar`：

```bash
sudo apt-get install rar
```

常用命令：

```bash
rar a xxx.rar xxx
rar x xxx.rar
```

### 6.2 `.zip` 格式

压缩：

```bash
zip -rv xxx.zip xxx
```

解压：

```bash
unzip xxx.zip
```
