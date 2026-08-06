---
title: ECU 信号映射测试分析系统架构设计 Prompt
slug: ecu-signal-mapping-analysis-architecture-prompt
published: 2026-08-07
description: 用于让大模型继续设计汽车电子 ECU 信号映射测试分析系统的软件架构，重点覆盖架构图、模块职责、LangGraph 设计和阶段实施。
image: ""
tags: [ECU, Automotive, Testing, Architecture, LangGraph, LLM]
category: 架构设计
draft: false
lang: zh-CN
---

# ECU 信号映射测试分析系统架构设计 Prompt

下面这版 prompt 适合直接复制给大模型，用来继续做架构设计。重点放在：**架构图、模块职责、分析链路、LangGraph 设计、阶段实施**。

````text
请帮我设计一个汽车电子 ECU 信号映射测试分析系统的软件架构，并输出架构图和模块职责说明。

背景：
我负责的是一个 ECU 信号映射模块，主要验证两类映射关系：
1. external_to_internal：外部 CAN 信号输入，检查 ECU 内部信号是否正确更新
2. internal_to_external：ECU 内部信号输入，检查外部 CAN 信号是否正确输出

每个测试 case 可能包含多个测试点，例如：
- 等价类
- 上边界
- 下边界
- invalid 值
- transition 值

我接手的是已有测试，所以我对失败 case 是否按需求设计正确不完全放心。
因此系统第一目标不是直接判断软件 bug，而是先判断：
“这个失败 case 本身是否符合需求？”

每次分析任务会上传三类文件：
1. Test Report：HTML 或 CSV，包含 case_id、case_name、test steps、expected、actual、status、error message、log 等
2. Requirement：半结构化需求表，包含 requirement_id、direction、source_signal、target_signal、valid_min、valid_max、default_value、mapping_rule 等；其中 mapping_rule 可能是自然语言
3. DBC：CAN signal 定义，包含外部信号名、message、scale、offset、min/max、unit、enum/value table 等

后续第二阶段可以接入：
4. Source Code / Git Diff：只有在系统判断为 probable_bug 后，才进行源码定位和 diff 分析，用于补充证据

我希望整体判断顺序是：
1. 解析 Report、Requirement、DBC
2. 将测试报告标准化为 TestCase / TestPoint
3. 将需求标准化为 Requirement
4. 将 DBC 标准化为 DbcSignal
5. 针对 failed/error case，先结合 TestCase + Requirement + DBC 判断 case 是否符合需求
6. 如果 case 不符合需求，分类为 test_case_issue
7. 如果 case 信息不足、需求解释不确定，分类为 need_human_review
8. 如果 case 符合需求，再判断失败更像环境问题、疑似真实 bug，还是需要人工校验
9. 如果是 probable_bug，第二阶段才进入源码定位和 diff 分析
10. 源码分析只作为证据增强，不作为第一阶段主链路

最终分类包括：
- test_case_issue：测试步骤、期望值、方向、信号、边界/invalid 设计不符合需求
- environment_issue：测试环境、CAN 通信、台架、信号注入、DUT 状态等问题
- probable_bug：case 合理，DBC 校验通过，环境证据不强，失败模式稳定，疑似真实软件问题
- probable_bug_with_source_evidence：源码/diff 分析后，有源码证据支持
- need_human_review：证据不足、冲突或需要人工判断

请你从软件架构角度输出方案，必须包含以下内容：

1. 整体架构图
请用 Mermaid 画一张简化但完整的架构图，体现：
- 输入层
- 解析标准化层
- Requirement/DBC 校验层
- Agent 1：Case Requirement Review
- Agent 2：Failure Cause Decision
- 可选 Agent 3：Source Evidence Review
- 条件分支
- 最终分类结果

2. 模块职责说明
请详细说明每个模块负责什么，包括但不限于：
- Upload Job / Input Manager
- Report Parser
- Requirement Parser
- DBC Parser
- Test Case Normalizer
- Requirement Matcher / Candidate Retriever
- DBC Checker
- Failure Pattern Detector
- Environment Evidence Detector
- Agent 1：Case Requirement Review Agent
- Agent 2：Failure Cause Decision Agent
- Source Locator
- Diff Analyzer
- Agent 3：Source Evidence Review Agent
- Result Aggregator
- Output Writer

3. 分析链路说明
请解释单个 failed case 从输入到最终分类的完整数据流。
重点说明：
- Report 提供“实际怎么测”
- Requirement 提供“应该怎么测”
- DBC 提供“外部 CAN 信号事实依据”
- 为什么必须先判断 case 是否合理
- 为什么只有 probable_bug 才进入源码分析

4. LangGraph 设计
请判断这个系统是否适合用 LangGraph。
如果适合，请给出：
- 推荐 graph 节点
- 条件路由逻辑
- State 中应该保存哪些核心字段
- 每个节点输入和输出的大致结构
- 如何处理多个 failed case 的批处理
- 哪些节点是规则节点，哪些节点是 LLM agent 节点

5. 规则/代码与 LLM 的分工
请明确说明：
哪些工作应该用代码/规则做：
- HTML/CSV 解析
- Requirement 表读取
- DBC 解析
- 信号硬匹配
- 数值范围校验
- scale/offset 基础计算
- 失败点分布统计
- 环境关键词检测

哪些工作适合 LLM/Agent 做：
- mapping_rule 自然语言理解
- case 是否符合需求的综合 review
- 测试步骤是否缺少前置条件
- 失败原因综合判断
- 源码片段和 diff 的证据解释

6. Token 成本控制
请说明如何控制 LLM token 成本：
- 按 failed case 调用
- 不输入整份 report
- 不输入整张 requirement 表
- 候选需求召回后只传 1-5 条
- test_points 多时只传失败点和通过点摘要
- DBC 只传相关 signal
- 源码只传候选函数片段和 diff 摘要
- 缓存相同 case + requirement 的 review 结果

7. 第一阶段 MVP
请给出第一阶段最小可落地版本：
输入：
- report.html / report.csv
- requirement.csv / requirement.xlsx
- network.dbc

输出：
- analysis_result.csv
- analysis_report.html
- summary.json

第一阶段只需要实现：
- Parser
- Normalizer
- Requirement Matcher
- DBC Checker
- Agent 1
- Agent 2
- Output Writer

暂时不做源码分析。

8. 第二阶段扩展
请给出第二阶段如何扩展：
- Source Locator
- Git Diff Analyzer
- Agent 3 Source Evidence Review
- 历史失败库
- 人工确认反馈闭环
- 规则权重优化
- 需求覆盖率/测试点覆盖率分析

9. 输出报告字段
请设计最终输出报告字段，例如：
- case_id
- case_name
- direction
- source_signal
- target_signal
- requirement_id
- dbc_signal
- case_review_result
- final_category
- confidence
- failure_pattern
- requirement_evidence
- dbc_evidence
- environment_evidence
- source_evidence
- missing_information
- recommended_action

请注意设计原则：
- 系统核心目标是先 review case 是否符合需求，而不是直接判断 bug
- Requirement 是“应该怎么测”
- Report 是“实际怎么测”
- DBC 是“外部信号事实标准”
- LLM 不应该替代所有规则判断
- 不要把整份 report、整份需求、整份源码直接丢给 LLM
- 先用规则和结构化数据缩小输入，再让 LLM 做自然语言和综合判断
- 源码分析只在 probable_bug 后触发
````
