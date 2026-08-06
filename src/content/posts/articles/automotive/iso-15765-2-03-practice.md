---
title: 3. ISO 15765-2 详解：UDS 实战、超时与排查
slug: iso-15765-2-03-practice
published: 2026-08-07
description: 结合 UDS 读 DID、刷写和诊断日志，整理 ISO-TP 的超时参数、常见问题和排查思路。
image: ""
tags: [ISO15765, ISO-TP, UDS, Diagnostics, Automotive]
category: 汽车诊断
draft: false
lang: zh-CN
---

# 3. ISO 15765-2 详解：UDS 实战、超时与排查

前两篇分别讲了 ISO 15765-2 的定位和四类帧格式。这一篇更偏工程实战：如何把 ISO-TP 放回 UDS 场景里看，遇到超时、乱序、流控异常时该从哪里查。

## 1. UDS 与 ISO-TP 的边界

实战里最重要的一点是分层。

UDS 关注的是服务语义：

```text
10 03       进入扩展会话
22 F1 90    读取 VIN
27 01       请求 seed
34 ...      请求下载
36 ...      传输数据
37 ...      退出传输
```

ISO-TP 关注的是传输过程：

```text
02          Single Frame 长度
10 14       First Frame，总长度 20
21          Consecutive Frame，SN = 1
30 00 00    Flow Control，允许继续发送
```

排查问题时，要先判断问题发生在哪一层。UDS 返回 `7F 22 31` 是应用层负响应；First Frame 后没有收到 Flow Control，则是传输层问题。

## 2. 读取 VIN 的完整例子

请求：

```text
Tester -> ECU:
03 22 F1 90 00 00 00 00
```

含义：

1. `03`：ISO-TP Single Frame，UDS 数据长度 3
2. `22 F1 90`：UDS ReadDataByIdentifier，读取 VIN

响应：

```text
ECU -> Tester:
10 14 62 F1 90 4C 56 53

Tester -> ECU:
30 00 00 00 00 00 00 00

ECU -> Tester:
21 41 42 43 44 45 46 47
22 48 49 4A 4B 4C 4D 4E
```

含义：

1. ECU 用 First Frame 声明完整响应长度为 `0x14`
2. Tester 用 Flow Control 允许继续发送
3. ECU 用 Consecutive Frame 发送剩余数据
4. Tester 重组后再交给 UDS 层解析

如果诊断工具只显示最终 UDS 响应，你可能看不到这些传输层细节；但抓 CAN 原始日志时，这些字段都会出现。

## 3. 刷写场景为什么更容易暴露 ISO-TP 问题

普通诊断读写 DID 通常数据量较小，而刷写会大量使用多帧传输。

典型刷写流程：

```text
10 02    进入编程会话
27 xx    安全访问
34 ...   RequestDownload
36 ...   TransferData
37 ...   RequestTransferExit
31 ...   RoutineControl 校验
11 01    ECU Reset
```

其中 `TransferData(0x36)` 经常携带较大的数据块。一个数据块会被 ISO-TP 拆成 First Frame 和大量 Consecutive Frame。

刷写时常见问题包括：

1. 发送方没有等待 Flow Control 就继续发 CF
2. STmin 没有遵守，连续帧发得太快
3. Block Size 计数错误，发满一个 block 后没有等待新的 FC
4. CF 序号 SN 跳变、重复或回绕处理错误
5. ECU 处理慢，诊断工具的超时设置过短

这类问题表面上可能显示为刷写失败，但根因往往在传输层。

## 4. 常见超时参数

ISO-TP 实现里经常会看到这些 N_xx 定时参数。不同项目、工具链和栈实现命名可能略有差异，但思路相近。

| 参数 | 常见含义 |
| --- | --- |
| N_As | 发送方开始发送 N_PDU 的超时 |
| N_Ar | 接收方接收 N_PDU 的超时 |
| N_Bs | 发送方等待 Flow Control 的超时 |
| N_Br | 接收方发送 Flow Control 的相关时间 |
| N_Cs | 发送方发送 Consecutive Frame 的相关时间 |
| N_Cr | 接收方等待 Consecutive Frame 的超时 |

实战里最常遇到的是：

1. **N_Bs 超时**：First Frame 发出后，迟迟收不到 Flow Control
2. **N_Cr 超时**：Flow Control 发出后，迟迟收不到后续 Consecutive Frame
3. **STmin 违规**：CF 间隔小于接收方要求

## 5. Flow Control 异常怎么查

如果发送 First Frame 后没有收到 Flow Control，可以按这个顺序查：

1. CAN ID 是否配置反了
2. 请求方向和响应方向是否正确
3. 接收方是否支持该寻址方式
4. ECU 或 Tester 是否正确识别 First Frame
5. 接收方缓冲区是否不足
6. 网关是否过滤了 Flow Control
7. N_Bs 超时时间是否设置过短

如果收到的是：

```text
31 00 00
```

这表示 Flow Status 为 Wait。发送方应该等待，而不是继续发送 CF。

如果收到的是：

```text
32 00 00
```

这表示 Overflow 或 Abort。通常需要检查接收方缓冲能力、报文总长度、寻址方式和项目配置。

## 6. Consecutive Frame 异常怎么查

CF 问题通常从 SN 入手。

正常序号：

```text
21 ...
22 ...
23 ...
24 ...
...
2F ...
20 ...
21 ...
```

SN 从 `1` 开始递增，到 `F` 后回绕到 `0`。

异常示例：

```text
21 ...
23 ...
```

这里缺了 `22`，接收方通常会判断为丢帧或乱序。

再比如：

```text
21 ...
21 ...
```

这可能是重复帧、重发策略错误或日志采集重复导致，需要结合时间戳和 CAN 控制器状态判断。

## 7. 和 UDS NRC 的关系

ISO-TP 成功传完，并不代表 UDS 服务一定成功。

例如：

```text
Tester -> ECU:
03 22 F1 99 00 00 00 00

ECU -> Tester:
03 7F 22 31 00 00 00 00
```

这说明传输层没有问题，ECU 正常收到了请求，也正常返回了 UDS 负响应。真正的问题是应用层：DID `F199` 超出范围，或者当前会话/安全等级不允许访问。

相反，如果 First Frame 发出后没有 Flow Control，或者 CF 序号错误，那还没到 UDS 服务逻辑，应该先查 ISO-TP。

## 8. 排查清单

遇到 DoCAN 通信异常时，可以按这个顺序定位：

1. 确认 CAN ID、波特率、CAN FD 配置和 DLC 是否正确
2. 确认寻址方式是否匹配，比如 Normal、Extended、Mixed
3. 判断报文是 SF、FF、CF 还是 FC
4. 多帧时检查 First Frame 总长度是否正确
5. 检查 Flow Control 的 FS、BS、STmin
6. 检查 CF 的 SN 是否连续并正确回绕
7. 检查 N_Bs、N_Cr 等超时参数
8. 传输层无误后，再解析 UDS SID、DID、RID 和 NRC

## 9. 小结

ISO 15765-2 在项目里很少单独出现，但它几乎藏在每一次 DoCAN 诊断背后。只要涉及长响应、DID 批量读取、DTC 扩展信息、Bootloader 下载或远程诊断，ISO-TP 的分包、流控和超时都会直接影响稳定性。

工程排查时，最有效的习惯是：**先分层，再定位**。先确认 ISO-TP 是否把完整 UDS 消息可靠送达，再判断 UDS 服务本身是否被 ECU 接受。

## 10. 参考

- [ISO 15765-2 官方页面](https://www.iso.org/standard/84211.html)
- [Linux Kernel ISO-TP documentation](https://docs.kernel.org/networking/iso15765-2.html)
- ISO 14229 UDS 与具体 OEM 诊断规范
