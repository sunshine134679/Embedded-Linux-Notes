# 第22讲 RTC实验

> 来源：正点原子官方笔记整理

## 一、RTC 原理详解

6U 内部自带 RTC 外设，准确来说是 SRTC。6U 和 6ULL 的 RTC 内容在 SNVS 章节。

6U 的 RTC 分为：

| 类型 | 说明 |
| ---- | ---- |
| SNVS_LP / SRTC | 低功耗 RTC，掉电后可配合纽扣电池保持 |
| SNVS_HP / RTC | 高功耗 RTC，掉电后数据会丢失，即使用纽扣电池也没用 |

因此实际使用中必须使用 LP，也就是 SRTC。

SNVS 章节有些内容和加密有关，需要与 NXP 签订 NDA 协议才可以获取完整资料。

如果做产品，建议使用外置 RTC 芯片，例如 `PCF8563`。

RTC 类似定时器，外接 32.768 kHz 晶振后开始计时。RTC 使用寄存器保存计数值。

使用流程：

1. 打开 RTC。
2. RTC 开始工作。
3. 读取 RTC 计数寄存器获取时间。
4. 或向 RTC 计数器写入时间值，用于调整时间。

关键寄存器：

| 寄存器 | 位 | 作用 |
| ---- | ---- | ---- |
| `SNVS_HPCOMR` | `bit31` | 置 1 后，所有软件都可以访问 SNVS 所有寄存器 |
| `SNVS_HPCOMR` | `bit8` | 和安全相关，可置 1，也可以不置 1 |
| `SNVS_LPCR` | `bit0` | 置 1 后启动 SRTC |
| `SNVS_LPSRTCMR` | `bit14:0` | RTC 计数寄存器高位 |
| `SNVS_LPSRTCLR` | - | RTC 计数寄存器低位 |

6U 的 RTC 从以下时间开始计数：

```text
1970-01-01 00:00:00
```

## 二、时间乱码的问题

### 2.1 问题

按照 6U 参考手册读取 `SRTC` 的 `LPSRTCMR` 和 `LPSRTCLR` 获取时间值时，发现时间错误。

手册说明：

- `LPSRTCMR` 是 SRTC 高 15 bit。
- `LPSRTCLR` 是 SRTC 低 32 bit。
- RTC 计数器是 47 bit。

### 2.2 解决方法

实际使用中：

- `LPSRTCMR` 作为 SRTC 计数器的高 15 位。
- `LPSRTCLR bit31:15` 作为 SRTC 计数器的低 17 位。

也就是说，SRTC 计数器实际按 32 位处理，而不是 47 位。
