---
title: 1. ISO 15765-2 详解：DoCAN 与 ISO-TP 是什么
slug: iso-15765-2-01-overview
published: 2026-08-07
description: 从汽车诊断分层、CAN 报文长度限制和 UDS 的关系入手，理解 ISO 15765-2 为什么是 DoCAN 通信的核心传输协议。
image: ""
tags: [ISO15765, DoCAN, ISO-TP, UDS, Automotive]
category: 汽车诊断
draft: false
lang: zh-CN
---

# 1. ISO 15765-2 详解：DoCAN 与 ISO-TP 是什么

ISO 15765-2 的正式名称是 **Road vehicles — Diagnostic communication over Controller Area Network (DoCAN) — Part 2: Transport protocol and network layer services**。在实际项目里，它更常被叫作 **ISO-TP**，也就是跑在 CAN 或 CAN FD 上的诊断传输协议。

如果说 UDS 定义了“诊断服务能做什么”，比如读 DID、清 DTC、刷写、例程控制，那么 ISO 15765-2 解决的是另一个问题：**当一条诊断消息超过一帧 CAN 能装下的数据长度时，应该怎么拆包、发包、流控和重组**。

## 1. 为什么需要 ISO 15765-2

Classical CAN 的数据区最多 8 字节。即使使用 CAN FD，诊断消息也经常需要统一的传输层规则，因为应用层并不应该关心底层一帧最多能放多少字节。

例如读取 VIN：

```text
Request:
22 F1 90
```

请求本身很短，但响应可能是 17 字节 VIN，再加上服务 ID 和 DID 字段，完整响应长度会超过 Classical CAN 单帧容量。

如果没有 ISO-TP，上层诊断工具就必须自己约定：

1. 第一帧怎么声明总长度
2. 后续帧怎么排序
3. 接收方缓冲区满了怎么办
4. 连续帧之间要不要限速
5. 丢帧或乱序时如何处理

ISO 15765-2 正是把这些规则标准化。

## 2. 它在诊断协议栈里的位置

可以把汽车诊断通信理解成几层：

```text
应用层       ISO 14229 UDS
传输/网络层  ISO 15765-2 ISO-TP
数据链路层   ISO 11898 CAN / CAN FD
物理层       CAN physical layer
```

UDS 报文里的 `10 03`、`22 F1 90`、`19 02`、`31 01` 是应用层内容。ISO-TP 报文里的 Single Frame、First Frame、Consecutive Frame、Flow Control 是传输层内容。

一个常见误区是把 UDS 和 ISO-TP 混在一起看。例如：

```text
02 10 03 00 00 00 00 00
```

这里的 `02` 不是 UDS 服务，而是 ISO-TP 的单帧长度字段；`10 03` 才是 UDS 的 DiagnosticSessionControl 请求。

## 3. DoCAN 是什么

DoCAN 是 **Diagnostic communication over CAN** 的缩写，即“基于 CAN 的诊断通信”。ISO 15765 系列定义了 DoCAN 相关内容，其中第 2 部分重点规定传输协议和网络层服务。

在工程里，说“这个 ECU 走 DoCAN 诊断”，通常意味着：

1. 底层总线是 CAN 或 CAN FD
2. 传输层使用 ISO 15765-2
3. 应用层常见为 UDS ISO 14229
4. 请求和响应通过诊断 CAN ID 区分

常见物理寻址示例：

```text
Tester -> Engine ECU: 0x7E0
Engine ECU -> Tester: 0x7E8
```

其中 `0x7E0` 和 `0x7E8` 是 CAN ID，不是 UDS 服务号。

## 4. ISO-TP 解决的核心问题

ISO-TP 的目标可以压缩成一句话：

**让上层诊断协议可以发送一条完整消息，而不用关心这条消息最终被拆成多少个 CAN 帧。**

它主要处理这些事情：

1. 短消息用 Single Frame 一帧发完
2. 长消息用 First Frame 声明总长度
3. 接收方用 Flow Control 控制发送节奏
4. 发送方用 Consecutive Frame 发送后续数据
5. 接收方根据序号重组完整诊断消息

所以，当你在诊断日志里看到 `10 xx`、`21 xx`、`22 xx`、`30 xx xx` 这类字节时，不能只按 UDS 去解释。它们很可能是 ISO-TP 的 PCI 字段。

## 5. 一个完整的长报文过程

假设 Tester 请求读取一个较长 DID，ECU 的响应超过单帧容量，典型流程如下：

```text
Tester -> ECU: 03 22 F1 90 00 00 00 00
ECU    -> Tester: 10 14 62 F1 90 31 32 33
Tester -> ECU: 30 00 00 00 00 00 00 00
ECU    -> Tester: 21 34 35 36 37 38 39 41
ECU    -> Tester: 22 42 43 44 45 46 47 48
ECU    -> Tester: 23 49 4A 4B 4C 4D 4E 4F
```

可以拆开看：

1. `03 22 F1 90`：Tester 发送 Single Frame，请求读取 DID `F190`
2. `10 14 ...`：ECU 发送 First Frame，声明总长度为 `0x14`
3. `30 00 00`：Tester 发送 Flow Control，允许 ECU 继续发送
4. `21`、`22`、`23`：ECU 发送 Consecutive Frame，序号依次递增

最终诊断工具把这些帧重新拼回一条完整 UDS 响应。

## 6. 学习 ISO 15765-2 的切入点

如果刚开始接触 ISO-TP，建议按这个顺序理解：

1. 先分清 CAN ID、ISO-TP PCI、UDS SID 三个层次
2. 再掌握 SF、FF、CF、FC 四类帧
3. 接着理解 BS、STmin、SN、N_As、N_Bs、N_Cs 等参数
4. 最后结合 UDS 服务，比如 `0x22`、`0x34`、`0x36`，看真实诊断日志

后面两篇会继续展开帧格式、流控参数和实战排查。

## 7. 参考

- [ISO 15765-2 官方页面](https://www.iso.org/standard/84211.html)
- [Linux Kernel ISO-TP documentation](https://docs.kernel.org/networking/iso15765-2.html)
- ISO 14229 UDS 与具体 OEM 诊断规范
