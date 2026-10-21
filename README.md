# AgentScope 研究与选型文档

对 AgentScope（Python 版 + Java 版）源码的深度研究，参照 DeepSeek Harness（dsh）原理文档，回答"如何用、有何替代、金融版是什么、怎么选型"。

源码位置：`../agentscope`（Python `2.0.11dev`，commit `402c3664`）、`../agentscope-java`（Java `2.1.0-SNAPSHOT` / GA 2.0.3，commit `2c6d0eb0`），均为 2026-10-19 的 `main`。

## 文档清单与阅读顺序

| # | 文档 | 内容 | 何时读 |
|---|---|---|---|
| 01 | [AgentScope Python 源码原理](01-AgentScope-Python-源码原理.md) | 架构与包职责、单一 `Agent` 类与 ReAct 主循环、停泊-恢复协议、中间件 7 钩子、工具系统、模型/Formatter、上下文工程、RAG、编排、运行时 | 需要理解 Python 版内部机制时 |
| 02 | [AgentScope Java 源码原理](02-AgentScope-Java-源码原理.md) | 七模块 Maven 结构、core/harness/service 分层、ReActAgent、模型 SPI、工具与三态权限、双事件系统（32 类 AgentEvent + 57 类 SessionLog）、沙箱与文件系统、与 Python 版对比矩阵 | 需要理解 Java 版内部机制时 |
| 03 | [DeepSeek Harness 原理对照与落地应用](03-DeepSeek-Harness-原理对照与落地应用.md) | dsh 原理速览（Cordis 时空可组合性、一切皆插件、三类事件域、能力 seam、日志唯一真源、工具 15 步流水线）→ 逐项对照 → **映射总表与四阶段落地路径** | **回答"如何应用"，优先级最高** |
| 04 | [同类框架对比与技术选型](04-同类框架对比与技术选型.md) | 按编排范式分派别、10 框架 × 14 维对比、决策树、组合式架构、选型必核 8 问 | 需要横向比较时 |
| 05 | [FinXScope 金融级智能体底座深度研究](05-FinXScope-金融级智能体底座深度研究.md) | 定位与体系坐标、金融四条硬约束、六层架构逐层、十大核心能力逐条（含设计思想与启发）、**与 AgentScope Java 的复用/自建映射**、2.0 演进、金融级保障、自建路线 | 关注金融场景时 |
| 06 | [技术选型建议与落地路线](06-技术选型建议与落地路线.md) | 五条选型结论、目标架构、四阶段落地路线、风险清单、三个典型场景组合 | **先看结论** |
| 07 | [内部培训大纲：AgentScope 深入原理](07-内部培训大纲-AgentScope深入原理.md) | 6 小时团队培训课程：8 个模块「Demo → 现象 → 提问 → 原理 → 源码 → 实验 → 小结」，含讲师手册、三种裁剪方案（6h 目标 / 4.5h 保底）、Demo 脚本清单 | 要做团队培训时 |
| 08 | [调试与实战手册](08-调试与实战手册.md) | 三层调试法（事件流/中间件/断点回放）、事件序列对照表、确定性回放、Prompt 调试、Token 计量、常见问题 Symptom→原因→定位 | **开发与排障时随时查** |

**建议阅读顺序**：时间紧 → `06` → `03` → 按需 `01`/`02`；要深究 → `01` → `02` → `03` → `04` → `05` → `06`。
**开发排障**：直接查 `08`（比读原理快得多）；**培训**：以 `07` 为纲，`08` 作实验素材。

## 核心结论速览

1. **AgentScope 2.x 是彻底重构**：Python 版收敛为单一 `Agent` 类 + 4 个配置对象 + 7 钩子中间件；Java 版删除了 Pipeline/MsgHub，改为"子 agent + 团队 + 事件流"。带 1.x 概念读代码会完全对不上。
2. **Java 版不是简单移植**：`agentscope-harness` 是"ReActAgent 之上的生产运行时"（组合而非继承，核心循环不动，能力通过 20 个中间件叠加），另有 `agentscope-service` 四平面平台。定位是"让 agent 在多租户、多副本、长期运行、可中断可恢复、安全隔离的生产环境里跑"。
3. **Java 版两个短板**：RAG 与长期记忆在 v2 处于推进期（v1 全部 `@Deprecated(forRemoval)`，实战路线见 [02 §5.5](02-AgentScope-Java-源码原理.md)）；无 Pipeline/MsgHub 等价物。
4. **Python 版一个短板**：无持久化事件域，因此 fork / 投影式压缩 / 逐字节重建校验都不具备，可用中间件自建。
5. **dsh 最值得借鉴的三条纪律**：日志即状态（fork/resume/审计/回放结构性免费）、扩展点显式声明权力边界（观察/否决/改写）、失败一律塌缩为结构化结果而非中止轮次。落到 AgentScope 均可用中间件完成，**不需要 fork 核心代码**。
6. **金融领域的研究对象是 FinXScope**（阿里云新金融团队基于 AgentScope Java 的金融级智能体底座；官方用例文档在 `agentscope-java/docs/v2/zh/blogs/usecases/finxscope.md`）：六层架构 + 十大核心能力，其中"共享执行底座"、"确定性流程工具化（MountableResource）"、"Skill 级超时"、"UserProfile 作为策略层"最值得借鉴；"点金"是其上层平台产品（本地化部署开放 AgentScope 金融版源码）。
7. **选型一句话**：基座选 AgentScope（Java 优先）；编排用其原生能力，不引入第二个运行时；检索短板用外部检索层补齐；金融场景照 FinXScope 六层架构做三级权限、全链路审计与可观测。
8. **调试先看事件流，别先打断点**：事件序列是 Agent 的"栈追踪"，正常序列固定、偏离即异常（对照表见 [08 §2.3](08-调试与实战手册.md)）。

## 术语表（跨文档统一口径）

> 同一概念在多篇文档出现时，以此表为准；正文中用〔见术语表〕引用。

| 术语 | 含义 |
|---|---|
| **停泊-恢复（park & resume）** | Agent 因等待人工确认/外部执行而提前结束 reply，之后用 `reply_id` 路由回去继续。是 HITL 与跨进程恢复的基础 |
| **step / turn** | step = 一次模型请求 + 它触发的工具执行；turn = 零或多个 step，从领取输入到不再欠任何工作 |
| **能力 seam** | 可替换能力的三角色：Service Definition（声明接口）+ Service Provider（实现）+ Consumer（通常是面向模型的工具）。来自 dsh |
| **表面（surface）** | dsh 中"模型可见的消息流"由日志投影而来；压缩用 `replace` 遮蔽区间而非删除日志 |
| **事件域** | dsh 的三类：会话事件（持久）/ agent 事件（进程内，观察或拦截）/ 能力事件（附加到 seam） |
| **双事件系统** | Java 版特有：32 类 `AgentEvent`（前端渲染/HITL，进程内）+ 57 类 `SessionLog`（持久化，租约/CAS/inbox，面向多副本） |
| **共享执行底座** | FinXScope 的核心设计：OneAgent 与 Multi-Agent 共用同一 `AgentProcessEngine`，保证权限/审计/执行管理一致 |
| **MountableResource** | FinXScope 把工作流封装成 Tool 挂给自主 Agent 的机制（"确定性流程工具化"） |
| **三层记忆** | FinXScope 口径：STM（滑动窗口+压缩）/ LTM（PG+pgvector）/ UserProfile（结构化金融属性层，策略而非记忆） |
| **中间件洋葱** | AgentScope 的扩展模型：6~7 个钩子，观察型只放行、改写型才动内容 |
| **fail-closed** | 审批/沙箱等安全服务缺失时一律拒绝，而非静默放行 |
| **文档即契约** | FinXScope 的技能定义方式：SKILL.md 描述触发条件与参数契约，人类与模型都可读 |
| **Formatter** | 把统一 Msg 转成厂商 API 请求的适配层（Python 12 厂商 × Chat/MultiAgent；Java 泛型三元组） |

## 文档来源与可信度

- **源码级结论**（01、02）：来自本地源码阅读，附文件路径与行号；**行号基线已锚定 commit**（Python `402c3664` / Java `2c6d0eb0`，均 2026-10-19），可用各文档开头的"如何验证本文结论"逐条复核。未逐文件通读 `ReActAgent.java`（332KB）等巨类，只读了关键路径。
- **dsh 对照**（03 第一部分）：来自本仓库 `dsh/` 已有的 01~04 原理文档（其本身基于 dsh 源码 v0.1.7-rc.2）。
- **行业信息**（04、05）：来自 AgentScope 官方仓库用例文档与公开横评（**数据时点 2026-10**），市场数据（客户数、下载量）为公开报道口径；04 的星级为相对判断；05 主体为官方文档口径，推断处已标注〔推断〕。
