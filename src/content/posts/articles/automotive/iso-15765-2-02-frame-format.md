---
title: 2. ISO 15765-2 详解：帧格式、分包与流控
slug: iso-15765-2-02-frame-format
published: 2026-08-07
description: 逐字节拆解 ISO-TP 的 Single Frame、First Frame、Consecutive Frame 和 Flow Control，理解 BS、STmin、SN 的作用。
image: ""
tags: [ISO15765, ISO-TP, DoCAN, FlowControl, Automotive]
category: 汽车诊断
draft: false
lang: zh-CN
---

# 2. ISO 15765-2 详解：帧格式、分包与流控

ISO 15765-2 最核心的内容就是四类 N_PDU：**Single Frame、First Frame、Consecutive Frame、Flow Control**。掌握这四类帧，就能读懂大多数 DoCAN 诊断日志。

下面以 Classical CAN 的 8 字节数据区为主说明。CAN FD 下的思路一致，只是单帧和多帧中的可用载荷长度会发生变化。

## 1. PCI 字段是什么

ISO-TP 会在 CAN 数据区开头放一个或多个字节，用来描述当前帧类型、长度、序号或流控参数。这个字段通常叫 **PCI**，即 Protocol Control Information。

PCI 的高 4 bit 通常用来表示帧类型：

| 高 4 bit | 类型 | 名称 |
| --- | --- | --- |
| `0x0` | SF | Single Frame |
| `0x1` | FF | First Frame |
| `0x2` | CF | Consecutive Frame |
| `0x3` | FC | Flow Control |

看到日志时，先看第一个字节的高半字节，基本就能判断当前帧类型。

## 2. Single Frame

Single Frame 用于一帧装得下的诊断消息。

```text
02 10 03 00 00 00 00 00
```

拆解如下：

| 字节 | 含义 |
| --- | --- |
| `02` | Single Frame，应用层数据长度为 2 |
| `10 03` | UDS 数据 |
| `00 00 00 00 00` | 填充字节 |

这里的 `02` 表示后面有效 UDS 数据长度为 2 字节。它不是 UDS SID。

对于普通 UDS 请求，很多都是 Single Frame：

```text
10 03       进入扩展会话
11 01       ECU Reset
22 F1 90    读取 VIN DID
3E 00       Tester Present
```

实际 CAN 帧里会在这些 UDS 数据前加上 SF PCI。

## 3. First Frame

当诊断消息超过 Single Frame 能承载的长度时，发送方先发 First Frame。

```text
10 14 62 F1 90 31 32 33
```

拆解如下：

| 字节 | 含义 |
| --- | --- |
| `10 14` | First Frame，总长度为 `0x014`，即 20 字节 |
| `62` | UDS 正响应 SID，对应请求 `0x22` |
| `F1 90` | DID |
| `31 32 33` | 响应数据的前 3 字节 |

First Frame 的前 12 bit 用于表示完整诊断消息长度。Classical CAN 下，First Frame 通常还能携带 6 字节应用层数据。

重点是：**First Frame 不是完整响应，它只是长消息的第一段**。

## 4. Flow Control

接收方收到 First Frame 后，需要回复 Flow Control，告诉发送方是否可以继续、一次能发多少帧、连续帧之间至少间隔多久。

常见 Flow Control：

```text
30 00 00 00 00 00 00 00
```

拆解如下：

| 字节 | 含义 |
| --- | --- |
| `30` | Flow Control，FS = Continue To Send |
| `00` | BS = 0 |
| `00` | STmin = 0 ms |
| 后续字节 | 填充 |

Flow Status 常见值：

| FS | 含义 |
| --- | --- |
| `0x0` | Continue To Send，继续发送 |
| `0x1` | Wait，请等待 |
| `0x2` | Overflow/Abort，缓冲区溢出或中止 |

BS 是 Block Size。如果 `BS = 0`，表示发送方可以一直发到整条消息结束，中间不用再等新的 Flow Control。

STmin 是 Separation Time minimum，表示 Consecutive Frame 之间的最小间隔。

## 5. Consecutive Frame

收到允许继续的 Flow Control 后，发送方开始发 Consecutive Frame。

```text
21 34 35 36 37 38 39 41
22 42 43 44 45 46 47 48
23 49 4A 4B 4C 4D 4E 4F
```

拆解如下：

| 字节 | 含义 |
| --- | --- |
| `21` | Consecutive Frame，序号 SN = 1 |
| `22` | Consecutive Frame，序号 SN = 2 |
| `23` | Consecutive Frame，序号 SN = 3 |

SN 是 Sequence Number，范围是 `0x0` 到 `0xF`，递增到 `0xF` 后回到 `0x0`。

接收方会用 SN 检查是否丢帧、乱序或重复。如果期望 `0x22`，却收到 `0x23`，通常就会判定传输层错误。

## 6. BS 与 STmin 的实战含义

Flow Control 中最容易影响刷写和大数据读取性能的是 BS 和 STmin。

### BS = 0

```text
30 00 00
```

表示不分块，发送方可以连续发完所有 CF。这个配置吞吐更高，但接收方缓冲和处理能力要足够。

### BS = 8

```text
30 08 00
```

表示发送方最多连续发送 8 个 CF，之后必须等待接收方再发一个 Flow Control。

这在 ECU 缓冲较小、网关转发压力较大或 Bootloader 写入较慢时很常见。

### STmin = 10 ms

```text
30 00 0A
```

表示连续帧之间至少间隔 10 ms。发送太快可能导致 ECU 接收不过来，发送太慢则会影响刷写效率。

## 7. 常见日志阅读方法

读一段 ISO-TP 日志时，可以按这个顺序：

1. 看 CAN ID，确认方向是 Tester 到 ECU，还是 ECU 到 Tester
2. 看首字节高半字节，判断 SF、FF、CF、FC
3. 如果是 SF，看低半字节长度
4. 如果是 FF，看总长度
5. 如果是 FC，看 FS、BS、STmin
6. 如果是 CF，看 SN 是否连续
7. 最后再把应用层 UDS 数据拼出来

不要一开始就按 UDS 服务解析所有字节。很多看似奇怪的 `10`、`21`、`30`，其实是 ISO-TP 控制信息。

## 8. 参考

- [ISO 15765-2 官方页面](https://www.iso.org/standard/84211.html)
- [Linux Kernel ISO-TP documentation](https://docs.kernel.org/networking/iso15765-2.html)
- ISO 14229 UDS 与具体 OEM 诊断规范
