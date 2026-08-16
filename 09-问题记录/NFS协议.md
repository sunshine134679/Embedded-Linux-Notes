# NFS 根文件系统挂载失败（error -6 / ENXIO）

> 平台：正点原子 i.MX6ULL（NXP 内核 4.1.15），主机 Ubuntu 24.04（nfs-utils 2.6.4）
> 时间：2026-08-16

## 1. 现象

内核启动正常、网卡正常、静态 IP 已配置，但挂载 NFS 根文件系统失败：

```
fec 20b4000.ethernet eth0: Link is Up - 100Mbps/Full
IP-Config: Complete:
     device=eth0, hwaddr=00:04:9f:04:d2:35, ipaddr=192.168.5.55, ...
     bootserver=192.168.5.5, rootserver=192.168.5.5, rootpath=
VFS: Unable to mount root fs via NFS, trying floppy.
VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6
```

当时的 bootargs：

```
bootargs=console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.5.5:/home/wsl/Linux/nfs/rootfs ip=192.168.5.55:192.168.5.5:192.168.5.1:255.255.255.0::eth0:off
```

## 2. 缘由（根本原因）

**bootargs 未指定 NFS 版本 → 老内核默认用 NFSv2 → 现代 NFS 服务端不支持 NFSv2 → 挂载失败。**

完整链条：

1. **内核默认参数是 NFSv2 + UDP**：源码 `fs/nfs/nfsroot.c` 中写死
   ```c
   #define NFS_DEF_OPTIONS  "vers=2,udp,rsize=4096,wsize=4096"
   ```
   bootargs 里 `nfsroot=` 没写 `vers=` / `proto=`，就走这个默认值。

2. **服务端不支持 NFSv2**：主机（Ubuntu 24.04，nfs-utils 2.6.4）的 nfsd 只注册了 v3/v4：
   ```
   $ rpcinfo -p localhost | grep 100003
   100003    3   tcp   2049  nfs
   100003    4   tcp   2049  nfs
   100003    3   udp   2049  nfs
   ```
   （没有任何 v2。NFSv2 太古老，现代 nfs-utils 已移除。）

3. **错误码翻译**：NFSv2 客户端把服务器返回的 `NFSERR_NXIO` 映射成 `-ENXIO`（源码 `fs/nfs/nfs2xdr.c`）：
   ```c
   { NFSERR_NXIO,  -ENXIO }   // = -6
   ```
   于是日志出现 `error -6`。

> 排除项：网络通（IP 已拿到）、路径对（nfsroot 与 `/etc/exports` 导出一致）、权限没问题（`no_root_squash`）、服务端运行中（rpcbind/mountd/nfsd 均 active）。问题只在**协议版本**。

## 3. 解决办法

U-Boot 里给 `nfsroot` 显式指定 NFSv3（建议同时用 TCP）：

```bash
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.5.5:/home/wsl/Linux/nfs/rootfs,vers=3,tcp rw ip=192.168.5.55:192.168.5.5:192.168.5.1:255.255.255.0::eth0:off'
saveenv
boot
```

关键就是 `nfsroot=...,vers=3,tcp` 这一段（只写 `,vers=3` 也行，服务端 v3 的 UDP 也在）。

## 4. 排查思路（以后遇到同类问题）

1. 先看 U-Boot：`printenv bootargs` —— `nfsroot=` 有没有写版本？路径和 `/etc/exports` 对不对？
2. 主机上看服务端状态：
   ```bash
   systemctl status nfs-kernel-server
   rpcinfo -p localhost | grep 100003     # 看服务端支持哪些 NFS 版本
   exportfs -v                            # 看导出是否生效
   ```
3. 主机上手动自测挂载（排除服务端问题）：
   ```bash
   sudo mount -t nfs 192.168.5.5:/home/wsl/Linux/nfs/rootfs /mnt/test
   ```
4. 判断 errno：`error -6` = ENXIO（服务器应答不支持），`-110` = 超时（多半网络不通/防火墙），`-13` = 权限拒绝。

## 5. 避坑

- **内核 4.1.15 的 nfsroot 默认 NFSv2**，新系统上必须显式 `vers=3`。
- nfsroot 路径必须和 `/etc/exports` 实际导出的目录一致（这里是 `/home/wsl/Linux/nfs/rootfs`，不是教程里的 `/home/wsl/Linux/IMX6ULL/rootfs`）。
- 本 rootfs 还缺 `/etc` 目录，NFS 挂载成功后会卡在 busybox init 阶段，需要补 inittab / init.d/rcS 等（另记）。