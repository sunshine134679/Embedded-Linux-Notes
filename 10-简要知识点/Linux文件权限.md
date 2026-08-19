# 在 Windows 下使用终端管理开发板 Linux 系统时所遇到的权限问题

## 一、在 Windows 下使用终端没有权限更改开发板文件

## 背景

**环境**：Windows + WSL 宿主机（x86_64）通过 NFS 导出目录，ARM 开发板通过 NFS 挂载该目录作为根文件系统（rootfs），两边共享同一份文件。

在理解下面的权限问题之前，需要先知道 4 件事：

1. **文件系统只存 uid/gid 数字，不存用户名**
   文件属主存的是 `uid=1000` 这样的数字；`root`、`wsl`、`nobody` 这些名字是每台机器用自己的 `/etc/passwd` 本地翻译出来的，不会通过网络传过去。

2. **开发板上的 root 不是"真" root**
   NFS 默认开启 `root_squash`：开发板发出自称 root(uid 0) 的请求时，服务端会把它降级成最普通的匿名用户 `nobody`(uid 65534)。这是为了防止局域网内任意机器自称 root 从而获得全部权限。

3. **因此开发板创建的文件，属主是 nobody**
   在开发板上用 root 创建的文件/目录，落盘时属主已经变成 nobody(65534)。所以在 WSL 上看到 `nobody:nogroup` 是正常现象，表示"这个文件既不是 wsl 的，也不是 root 的"。

   > 客户端 root 创建的文件属主变为 nobody，无法修改服务器上 root 所创建的文件!!!

4. **改属主只能在宿主机（WSL）上做**

   - 开发板（busybox）上没有 `sudo`，且 `/ #` 提示符表示你已经是 root；
   - 开发板的 `/etc/passwd` 里没有 `wsl` 用户，`chown wsl:wsl` 会报 unknown user；
   - 就算用数字 `chown 1000:1000`，root 也会被 root_squash 成 nobody，而 nobody 没有 chown 权限，必然失败；
   - 所以 chown 必须在宿主机 WSL 上用 sudo 执行。

### 问题现象

在 WSL 终端里编译/创建文件时报"权限不够"：

```
/usr/bin/ld: cannot open output file a.out: 权限不够
```

**原因**：`test` 目录属主是 nobody（由开发板 root 经 root_squash 创建），而你 wsl 对它没有写权限。注意：创建新文件需要的是**所在目录的写权限**，而不是文件本身的权限。

### 解决办法

在 WSL 宿主机终端执行（会提示输入 wsl 密码）：

```bash
cd ~/Linux/nfs/rootfs
sudo chown -R wsl:wsl test
```

- `sudo`：以管理员身份执行
- `chown`：修改属主/属组
- `-R`：递归，目录及里面所有文件一起改
- `wsl:wsl`：属主改为 wsl，属组改为 wsl

执行后，`test` 目录（含 `hello.c` 等）的属主和属组都变成 wsl，之后编译就不需要 sudo 了。

### Linux 文件权限速记

```
-rw-r--r--  wsl wsl  hello.c
││ ││ ││
││ ││ └┴─ 其他用户(others)：r--
││ └┴──── 属组(group)：r--
│└┴─────── 属主(owner)：rw-
└───────── 文件类型：- 普通文件 / d 目录
```

- 权限 = 三组 × 三位：`属主(owner) / 属组(group) / 其他(others)`
- `r`=读(4)，`w`=写(2)，`x`=执行(1)，`-`=无权限
- 数字表示：`rw-r--r--` = 644，`rwxr-xr-x` = 755，`rwx------` = 700
- 目录要 `w` 才能在**里面创建/删除文件**；要 `x` 才能**进入**目录
- 改属主：`chown 属主:属组 文件`；改权限：`chmod 755 文件` / `chmod u+w 文件`

## 三、总结

1. 权限问题的根源：文件属主是 nobody（开发板 root 经 NFS root_squash 创建），而不是你 wsl。
2. 解决办法：在宿主机 WSL 上执行 `sudo chown -R wsl:wsl <目录>`。
3. 开发板上的 root 不被 NFS 信任，会被降级成 nobody；普通权限操作（chmod）在板子上可能生效，但**改属主（chown）必须在宿主机上做**。