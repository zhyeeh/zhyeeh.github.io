---
title: UDS 诊断 NRC 列表
published: 2026-08-05
description: 整理 UDS 诊断中的 Negative Response Code，包括常见 NRC 含义、分类和排查思路。
image: ""
tags: [UDS, Diagnostics, NRC, ISO14229, Automotive]
category: 汽车诊断
draft: false
lang: zh-CN
---

# UDS 诊断 NRC 列表

在 UDS（Unified Diagnostic Services）诊断里，ECU 并不会只返回一个笼统的“失败”。当请求无法被正常执行时，ECU 会返回 Negative Response，并通过 NRC（Negative Response Code）说明拒绝原因。

负响应的基本格式固定为：

```text
0x7F + 原请求 SID + NRC
```

例如：

```text
Request:
22 F1 90

Response:
7F 22 31
```

这表示客户端请求 `ReadDataByIdentifier(0x22)` 读取 DID `F190`，ECU 返回负响应，NRC 为 `0x31 RequestOutOfRange`。可能原因包括 DID 不存在、当前会话不允许读取、当前安全等级不足，或者请求参数超出了 ECU 允许范围。

NRC 不是简单的错误码表。它往往同时反映服务支持情况、诊断会话、安全访问、请求顺序、车辆条件和 ECU 内部状态。

## 负响应格式

| 字节 | 含义 | 示例 |
| --- | --- | --- |
| Byte 1 | Negative Response SID | `0x7F` |
| Byte 2 | 原请求服务 SID | `0x22` |
| Byte 3 | Negative Response Code | `0x31` |

正响应通常是在请求 SID 的基础上加 `0x40`，例如 `0x22` 的正响应 SID 是 `0x62`。负响应则统一以 `0x7F` 开头，第三个字节才是真正的拒绝原因。

## NRC 速查表

| NRC | 名称 | 中文含义 | 常见原因 | 排查建议 |
| --- | --- | --- | --- | --- |
| `0x00` | positiveResponse | 正响应占位值 | 该值保留给内部实现，不应出现在负响应中 | 如果在线上报文中看到，需要检查 ECU 或解析工具 |
| `0x10` | generalReject | 通用拒绝 | ECU 拒绝请求，但没有更具体的 NRC | 优先检查是否应返回更具体的 NRC，项目中不建议滥用 |
| `0x11` | serviceNotSupported | 服务不支持 | ECU 未实现该 SID，或该寻址方式下不支持 | 确认诊断规范、服务开关、物理/功能寻址策略 |
| `0x12` | subFunctionNotSupported | 子功能不支持 | SID 支持，但 sub-function 不支持 | 检查子功能值是否在服务定义范围内 |
| `0x13` | incorrectMessageLengthOrInvalidFormat | 报文长度或格式错误 | DLC、参数数量、DID 长度、地址格式错误 | 先对照服务格式检查请求报文编码 |
| `0x14` | responseTooLong | 响应太长 | 响应数据超过传输层或 ECU 缓冲区限制 | 检查一次请求的 DID 数量、读取长度和分包能力 |
| `0x21` | busyRepeatRequest | ECU 忙，请稍后重试 | ECU 正在处理其他任务，暂时无法处理当前请求 | 客户端等待后重试，不要立即判定为永久失败 |
| `0x22` | conditionsNotCorrect | 条件不满足 | 车速、电压、点火状态、会话、ECU 内部状态不满足 | 查前置条件、环境条件和 ECU 状态机 |
| `0x24` | requestSequenceError | 请求顺序错误 | 未按规定流程调用服务，例如未请求 seed 就发送 key | 检查服务调用顺序和状态跳转 |
| `0x25` | noResponseFromSubnetComponent | 子网组件无响应 | 网关请求下游 ECU 或子网节点无响应 | 查网关路由、下游节点在线状态和网络通信 |
| `0x26` | failurePreventsExecutionOfRequestedAction | 故障阻止请求执行 | ECU 中存在故障状态，导致当前服务不能执行 | 查 DTC、故障状态和 OEM 前置条件 |
| `0x31` | requestOutOfRange | 请求超出范围 | DID/RID/地址/参数不支持，或当前条件下不可访问 | 查参数范围、会话限制、安全权限和配置表 |
| `0x33` | securityAccessDenied | 安全访问被拒绝 | 请求受保护资源，但未完成安全解锁 | 先执行 `0x27 SecurityAccess` 并确认等级正确 |
| `0x35` | invalidKey | 密钥错误 | seed-key 算法错误，key 与 seed 不匹配 | 检查算法、字节序、安全等级和计数器 |
| `0x36` | exceedNumberOfAttempts | 尝试次数超限 | 多次发送错误 key | 等待锁定解除，并修正重试策略 |
| `0x37` | requiredTimeDelayNotExpired | 延时时间未到 | 安全访问冷却时间未结束 | 等待规定时间，不要继续暴力重试 |
| `0x38` | secureDataTransmissionRequired | 需要安全数据传输 | 请求需要通过安全数据传输机制发送 | 检查是否应使用受保护通信通道 |
| `0x39` | secureDataTransmissionNotAllowed | 不允许安全数据传输 | 当前服务或状态不允许使用安全数据传输 | 检查安全传输配置和服务约束 |
| `0x3A` | secureDataVerificationFailed | 安全数据校验失败 | 安全数据认证、完整性或新鲜度校验失败 | 查签名、MAC、计数器、证书和密钥配置 |
| `0x50` | certificateVerificationFailedInvalidTimePeriod | 证书时间无效 | 证书未生效或已过期 | 检查 ECU 时间、证书有效期和时间同步 |
| `0x51` | certificateVerificationFailedInvalidSignature | 证书签名无效 | 证书签名验证失败 | 检查证书链、签名算法和根证书 |
| `0x52` | certificateVerificationFailedInvalidChainOfTrust | 证书信任链无效 | 证书链不完整或根证书不可信 | 检查证书链部署和信任锚 |
| `0x53` | certificateVerificationFailedInvalidType | 证书类型无效 | 证书用途或类型不符合服务要求 | 检查证书 profile 和用途字段 |
| `0x54` | certificateVerificationFailedInvalidFormat | 证书格式无效 | 编码格式、字段结构或长度错误 | 检查 DER/ASN.1 编码和传输拆包 |
| `0x55` | certificateVerificationFailedInvalidContent | 证书内容无效 | 证书字段内容不符合要求 | 检查 subject、issuer、扩展字段和 OEM 要求 |
| `0x56` | certificateVerificationFailedInvalidScope | 证书权限范围无效 | 证书不具备访问当前资源的权限 | 检查授权范围、角色和诊断权限 |
| `0x57` | certificateVerificationFailedInvalidCertificate | 证书无效 | 证书整体校验失败 | 结合 ECU 日志细分具体失败原因 |
| `0x58` | ownershipVerificationFailed | 所有权校验失败 | 所有权证明或绑定关系不匹配 | 检查挑战响应、绑定数据和后端授权 |
| `0x59` | challengeCalculationFailed | 挑战计算失败 | ECU 或客户端计算挑战值失败 | 检查算法输入、随机数、密钥和实现 |
| `0x5A` | settingAccessRightsFailed | 访问权限设置失败 | 权限写入或切换失败 | 检查权限状态机和持久化结果 |
| `0x5B` | sessionKeyCreationOrDerivationFailed | 会话密钥创建失败 | 密钥派生失败 | 检查密钥材料、算法套件和协商参数 |
| `0x5C` | configurationDataUsageFailed | 配置数据使用失败 | 安全配置数据无法使用 | 检查配置版本、完整性和兼容性 |
| `0x5D` | deAuthenticationFailed | 取消认证失败 | 去认证流程执行失败 | 检查认证状态和会话清理逻辑 |
| `0x70` | uploadDownloadNotAccepted | 上传/下载请求不被接受 | 下载地址、长度、格式或状态不满足 | 检查 Bootloader 下载流程和内存布局 |
| `0x71` | transferDataSuspended | 数据传输暂停 | 传输过程中 ECU 暂停接收 | 查网络稳定性、块大小、流控和 ECU 状态 |
| `0x72` | generalProgrammingFailure | 编程失败 | Flash 擦除、写入或校验失败 | 查 Flash 驱动、地址、电源和擦写流程 |
| `0x73` | wrongBlockSequenceCounter | 块序号错误 | `TransferData(0x36)` 块计数不连续 | 检查 blockSequenceCounter 是否从期望值递增 |
| `0x78` | requestCorrectlyReceived-ResponsePending | 请求已收到，等待响应 | ECU 任务耗时，例如擦除、校验、Routine 执行 | 继续等待最终响应，按 P2/P2* 处理超时 |
| `0x7E` | subFunctionNotSupportedInActiveSession | 当前会话不支持该子功能 | 子功能存在，但当前 session 不允许 | 切换扩展会话或编程会话后再试 |
| `0x7F` | serviceNotSupportedInActiveSession | 当前会话不支持该服务 | 服务存在，但当前 session 不允许 | 检查 `0x10 DiagnosticSessionControl` |
| `0x81` | rpmTooHigh | 转速过高 | 发动机转速高于允许范围 | 降低转速后重试 |
| `0x82` | rpmTooLow | 转速过低 | 发动机转速低于要求 | 满足运行条件后重试 |
| `0x83` | engineIsRunning | 发动机正在运行 | 服务要求发动机停止 | 熄火或切换到允许状态 |
| `0x84` | engineIsNotRunning | 发动机未运行 | 服务要求发动机运行 | 启动发动机后重试 |
| `0x85` | engineRunTimeTooLow | 发动机运行时间太短 | 发动机启动后的等待时间不足 | 等待 ECU 条件满足 |
| `0x86` | temperatureTooHigh | 温度过高 | ECU、冷却液或环境温度超限 | 降温后重试 |
| `0x87` | temperatureTooLow | 温度过低 | 温度低于服务要求 | 升温或满足环境条件后重试 |
| `0x88` | vehicleSpeedTooHigh | 车速过高 | 服务要求低速或静止 | 降速或停车 |
| `0x89` | vehicleSpeedTooLow | 车速过低 | 服务要求一定车速 | 满足车速条件 |
| `0x8A` | throttle/PedalTooHigh | 油门或踏板位置过高 | 踏板位置不满足服务条件 | 松开踏板或调整输入 |
| `0x8B` | throttle/PedalTooLow | 油门或踏板位置过低 | 踏板位置低于服务要求 | 调整踏板状态 |
| `0x8C` | transmissionRangeNotInNeutral | 档位不在空档 | 服务要求 N 档 | 切换到空档 |
| `0x8D` | transmissionRangeNotInGear | 档位不在行驶档 | 服务要求在档 | 切换到指定档位 |
| `0x8F` | brakeSwitchNotClosed | 刹车未踩下 | 服务要求制动踏板被踩下 | 踩下制动踏板后重试 |
| `0x90` | shifterLeverNotInPark | 换挡杆不在 P 档 | 服务要求 P 档 | 切换到 P 档 |
| `0x91` | torqueConverterClutchLocked | 液力变矩器离合器锁止 | 动力系统状态不允许执行请求 | 检查变速箱和动力系统条件 |
| `0x92` | voltageTooHigh | 电压过高 | 供电超过允许范围 | 检查电源、电池和台架供电 |
| `0x93` | voltageTooLow | 电压过低 | 供电低于允许范围 | 稳定电源后重试 |

`0x80` 以后很多 NRC 属于更具体的条件不满足类；`0xF0` 到 `0xFE` 通常可用于车辆制造商自定义。实际项目里一定要以 OEM 诊断规范、ECU 诊断需求文档和当前 ISO 14229 版本为准。

## 常见 NRC 分类

### 报文格式类

最典型的是 `0x13 incorrectMessageLengthOrInvalidFormat`。

常见触发场景：

- DID 应为 2 字节，但请求中少发或多发了字节
- `RoutineControl` 的 RID 或参数长度不符合定义
- `RequestDownload` 的 `DataFormatIdentifier` 或 `AddressAndLengthFormatIdentifier` 编码错误
- 传输层长度正确，但应用层字段数量不符合服务格式
- 把 `suppressPosRspMsgIndicationBit` 当成普通子功能值处理

排查这类 NRC 时，先不要怀疑 ECU 逻辑，应该先逐字节对照服务格式。

### 服务支持类

`0x11`、`0x12`、`0x7E`、`0x7F` 很容易混淆。

| NRC | 区别 |
| --- | --- |
| `0x11` | ECU 不支持这个服务 |
| `0x12` | 服务支持，但这个子功能不支持 |
| `0x7F` | 服务支持，但当前会话不支持 |
| `0x7E` | 子功能支持，但当前会话不支持 |

例如同样是请求 `DiagnosticSessionControl(0x10)`：

```text
Request:
10 03

Response:
7F 10 12
```

这通常表示 `0x10` 服务存在，但 `0x03` 这个子功能不支持。

如果响应是：

```text
7F 10 7E
```

则含义更接近：这个子功能存在，但当前会话下不允许切换或使用。

### 条件与范围类

`0x22 conditionsNotCorrect` 和 `0x31 requestOutOfRange` 是项目里最高频、也最容易争议的两个 NRC。

可以先这样理解：

- `0x22` 更偏“ECU 当前状态不允许执行”
- `0x31` 更偏“请求参数本身不在允许范围”

举例：

```text
读取某个 DID:
- DID 根本不存在：更可能返回 0x31
- DID 存在，但当前电压过低：更可能返回 0x22
- DID 存在，但默认会话不可读：可能是 0x31，也可能是 0x7F，取决于项目规范
```

实际落地时，不同 OEM 对这两个 NRC 的边界定义可能不完全一样。测试用例应以项目诊断规范为准，不能只按个人理解判定 ECU 错误。

### 安全访问类

安全访问相关 NRC 主要集中在 `0x33`、`0x35`、`0x36`、`0x37`。

典型流程：

```text
请求受保护服务
-> ECU 返回 0x33 securityAccessDenied
-> 客户端执行 0x27 请求 seed
-> 客户端根据 seed 计算 key
-> key 错误时返回 0x35 invalidKey
-> 错误次数超限后返回 0x36 exceedNumberOfAttempts
-> 冷却时间内继续请求返回 0x37 requiredTimeDelayNotExpired
```

排查 seed-key 问题时，建议优先确认：

- 安全等级是否匹配，例如请求的是 level 1 seed，却发送了 level 3 key
- seed 和 key 的字节序是否一致
- 算法输入是否包含额外常量、VIN、序列号或会话信息
- 失败计数器和延时计时器是否按规范清除

### 编程下载类

Bootloader 或刷写流程里，常见 NRC 包括 `0x70`、`0x71`、`0x72`、`0x73` 和 `0x78`。

典型刷写流程：

```text
10 02 进入编程会话
27 xx 安全解锁
34 RequestDownload
36 TransferData
37 RequestTransferExit
31 xx RoutineControl 校验
11 01 ECU Reset
```

常见判断：

- `0x70`：`RequestDownload` 阶段就不接受，重点查地址、长度、格式和下载状态
- `0x71`：传输过程中暂停，重点查网络、流控、块大小和 ECU 内部状态
- `0x72`：编程失败，重点查 Flash 擦写、校验、电压和地址映射
- `0x73`：块序号错误，重点查 `TransferData` 的 blockSequenceCounter
- `0x78`：请求已收到但还在处理，客户端应继续等待最终响应

## 高频 NRC 深入说明

### 0x13 incorrectMessageLengthOrInvalidFormat

`0x13` 通常说明 ECU 已经识别出服务，但请求报文结构不符合该服务的格式要求。

比如 `ReadDataByIdentifier(0x22)` 中 DID 必须按 2 字节组织：

```text
正确:
22 F1 90

错误:
22 F1
```

又比如多个 DID 连续读取时：

```text
正确:
22 F1 90 F1 91

错误:
22 F1 90 F1
```

排查建议：

1. 先确认 SID 和 sub-function 是否正确
2. 再按服务定义逐字段核对长度
3. 检查是否有可选参数被错误省略或错误添加
4. 检查诊断工具是否在传输层或应用层做了额外拼接

### 0x22 conditionsNotCorrect

`0x22` 不是“报文写错”，而是 ECU 判断当前状态不适合执行请求。

常见条件包括：

- 当前诊断会话不满足
- 未完成安全解锁
- 车速不为 0
- 供电电压过低或过高
- 发动机状态不满足
- ECU 初始化未完成
- 当前存在会阻止执行的 DTC 或内部故障

例如某个 Routine 要求车辆静止、发动机停止、电压正常。如果车辆仍在行驶中，即使请求报文完全正确，ECU 也可以返回 `0x22`。

### 0x31 requestOutOfRange

`0x31` 更关注请求中的资源或参数是否在允许范围内。

常见场景：

- 请求了不存在的 DID
- 请求了不支持的 RID
- 请求的内存地址非法
- 请求长度超过 ECU 允许范围
- 参数值不在枚举或数值范围内
- 当前会话或安全等级下该资源不可访问

例如：

```text
Request:
22 F1 99

Response:
7F 22 31
```

这通常说明 DID `F199` 不在当前可读取范围内。可能是 DID 没有实现，也可能是当前会话、安全等级或配置状态不允许访问。

### 0x78 requestCorrectlyReceived-ResponsePending

`0x78` 很容易被测试脚本误判为失败，但它的真实含义是：ECU 已经正确收到请求，只是最终结果还没准备好。

典型场景：

- Flash 擦除
- Flash 写入
- CRC 校验
- 签名校验
- 大数据读取
- `RoutineControl` 执行耗时任务

例如：

```text
Request:
31 01 FF 00

Pending Response:
7F 31 78

Final Response:
71 01 FF 00
```

客户端处理建议：

1. 收到 `0x78` 后不要立即判失败
2. 进入等待最终响应的状态
3. 使用 P2/P2* 计时参数控制超时
4. 最终可能收到正响应，也可能收到另一个最终负响应
5. 超过 P2* 或项目规定上限后，再判定超时

## 实战排查流程

收到 NRC 后，可以按这个顺序查：

1. 看 `0x7F` 后面的原请求 SID，确认是哪条服务失败
2. 判断 NRC 属于格式类、支持类、条件类、安全类还是编程类
3. 如果是 `0x11`、`0x12`、`0x7E`、`0x7F`，先查服务表和当前 session
4. 如果是 `0x33`、`0x35`、`0x36`、`0x37`，先查 SecurityAccess 流程
5. 如果是 `0x22`，查车辆状态、ECU 状态、电压、车速、点火状态和前置条件
6. 如果是 `0x31`，查 DID、RID、地址、参数范围、会话限制和安全权限
7. 如果是 `0x78`，检查客户端等待策略、P2/P2* 和总超时设置
8. 如果是 `0x72` 或 `0x73`，重点查 Bootloader 下载流程和块序号

## 三个典型例子

### DID 不支持

```text
Request:
22 F1 99

Response:
7F 22 31
```

解释：请求读取 DID `F199`，ECU 返回 `0x31`。该 DID 可能不存在，也可能当前会话或安全等级不允许读取。

### 未解锁直接写 DID

```text
Request:
2E F1 90 12 34 56 78

Response:
7F 2E 33
```

解释：`WriteDataByIdentifier(0x2E)` 常用于写入受保护数据。ECU 返回 `0x33`，说明需要先完成安全访问。

### Routine 正在执行

```text
Request:
31 01 FF 00

Response:
7F 31 78

Final Response:
71 01 FF 00
```

解释：ECU 收到 RoutineControl 请求后需要时间处理，所以先返回 `0x78`，随后返回最终正响应。

## 小结

NRC 的价值不在于背表，而在于定位问题。实际调试时，应把 NRC 放回 UDS 上下文里理解：当前是什么服务、处于什么 session、是否完成安全解锁、请求参数是否有效、ECU 当前状态是否满足。

当你能从 `7F xx yy` 快速推导出“下一步该查哪里”，NRC 才真正从错误码变成了调试线索。

## 参考

- [py-uds Diagnostic Message 文档](https://uds.readthedocs.io/en/v3.1.1/pages/knowledge_base/diagnostic_message.html)
- [py-uds Time Parameters 文档](https://uds.readthedocs.io/en/latest/pages/knowledge_base/time_parameters.html)
- [Automotive Wiki: ISO 14229](https://www.automotive.wiki/index.php/ISO_14229)
- ISO 14229-1 / UDS 规范体系。具体项目仍应以 OEM 诊断规范和 ECU 需求文档为准。
