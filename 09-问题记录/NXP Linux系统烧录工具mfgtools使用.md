# NXP mfgtools使用注意事项

>  由于mfgtools版本较老,所以在像Windows 11这种新系统上面使用可能会出现很多问题



## 1. 烧录的时候USB却没有二次识别出来（过程中USB会掉线一次）

- **表现**：`mfgtools`使用途中一直卡在 `no device connect` 
- **原因和解决**：
  1. `VMware`没有关闭 -> 彻底关闭`VMware`
  2. 开发板连接的是插在电脑上的扩展坞 -> 使用电脑自带的原生USB接口
  3. 不要使用双TypeC线去接着电脑自带的TypeC接口,协议兼容有可能出现问题 -> 在电脑上就直接插电脑的USB接口
  4. `firmware` 目录中混用了自己的 `U-Boot / zImage / DTB`，导致临时烧录系统的 USB Gadget/UTP 无法正常枚举 -> **`firmware` 目录全部使用官方 MFGTool 原版文件，自己的 `U-Boot / zImage / DTB / rootfs` 只放到 `files` 目录中用于最终烧录**。（神人Windows 再换回官方的文件，烧录成功之后，再换回自己的又能烧录成功，反正如果一直卡在 no device connected，可以先用官方的 firmware 下面的那些文件，如果能成功启动了，再试试把文件换回自己的，能不能启动）



## 2. 莫名其妙出现以下报错：

`"Boot" body="BootStrap" error, file="D:\NXP_mfgtool\L4.1.15_2.0.0-ga_mfg-tools\mfgtools-with-rootfs\mfgtools\Profiles\linux\OS Firmware\firmware\u-boot-imx6ull14x14evk_emmc.imx`

是狗日的 Windows 的 USB 驱动出问题了，重启电脑即可



