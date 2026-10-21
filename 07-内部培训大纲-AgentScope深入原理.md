# AgentScope 深入原理 · 团队内部培训大纲

> **TL;DR（讲师 3 分钟版）**
> - **目标**：6 小时（可裁到 4.5 小时保底），让学员能独立搭建并调试一个生产可用的 Agent，理解循环/事件流/权限/上下文/编排五大子系统，并知道团队要补什么。
> - **方法**：每个模块七步推进——**Demo → 现象 → 提问 → 原理 → 源码定位 → 课堂实验 → 小结**；白板留一张"问题墙"，模块 1 抛 4 个问题，后续逐个划掉。
> - **结构**：8 个模块，模块 2（ReAct 循环与停泊-恢复）与模块 7（对照 dsh 研讨）是重心；**故障 Demo 优先于正常 Demo**（拒绝工具、超限、上下文爆、沙箱拒绝）。
> - **裁剪**：1.5h 精华版 / 3h 半日版 / 6h 完整版，见 §6；超时先砍模块 4/5/6 的实验。
> - **交付物**：学员作业二选一（动手题/设计题），验收标准见 §0。

> 教学理念：**先 Demo，后原理**。每个模块都按「跑起来看现象 → 提出疑问 → 拆原理 → 定位源码 → 动手实验 → 小结」六步推进，避免一上来就讲抽象架构。
> 配套资料：[01 Python 源码原理](01-AgentScope-Python-源码原理.md) · [02 Java 源码原理](02-AgentScope-Java-源码原理.md) · [03 dsh 对照与落地](03-DeepSeek-Harness-原理对照与落地应用.md) · [05 FinXScope 金融详解](05-FinXScope-金融级智能体底座深度研究.md) · [06 选型与落地路线](06-技术选型建议与落地路线.md) · [08 调试与实战手册](08-调试与实战手册.md)

---

## 0. 课程信息

| 项 | 内容 |
|---|---|
| **课程名** | AgentScope 深入原理：从 Demo 到生产运行时 |
| **受众** | 后端/平台研发（4 年以上），需有 Java 或 Python 工程经验；**不需要**有 Agent 开发经验 |
| **目标** | ① 能独立搭建并调试一个生产可用的 Agent；② 理解 ReAct 循环、事件流、权限、上下文、编排五大子系统的原理与边界；③ 知道团队落地要补哪些能力、踩哪些坑 |
| **默认时长** | **6 小时目标 / 4.5 小时保底**（2 场 × 2.5h）；可裁剪为 3 小时或 1.5 小时版本（见 §6） |
| **授课形式** | 讲师演示 40% + 学员动手 40% + 研讨 20% |
| **前置准备** | 见 §1 |

### 学习目标（可验收）

课程结束后学员应能独立回答并做到：

1. 说出一次 `call()` / `reply()` 内部经历了哪几个阶段，以及在哪个阶段可以被拦截。
2. 解释"停泊-恢复"是什么，并举出触发它的三种场景（HITL、外部工具、打断）。
3. 手写一个中间件，实现"在每轮推理前注入一段提示"或"拦截某个危险工具"。
4. 说出上下文快满时框架做了什么、原文去了哪里、如何恢复。
5. 解释为什么 Java 2.0 里找不到 Python 的 `MsgHub`，以及替代方案是什么。
6. 针对自己负责的业务场景，列出落地时必补的三项能力（权限/审计/恢复）。

---

## 1. 课前准备（提前 3 天发给学员）

### 1.1 环境

| 项 | 版本 | 说明 |
|---|---|---|
| JDK | **17+** | Java 版硬要求 |
| Maven | 3.8+ | `mvn -v` 可跑 |
| Python | **3.10+** | Python 版；**异步 only，简单脚本也要 `asyncio.run()`** |
| Git | 任意 | 克隆源码 |
| 模型 Key | `DASHSCOPE_API_KEY` | 或 OpenAI/其他厂商 Key（两版都支持） |

```bash
# 源码（已在本仓库 agent_base 下，也可重新克隆）
export DASHSCOPE_API_KEY=sk-xxx

# Java：确认能编译
cd agent_base/agentscope-java && mvn -q -T1 -pl agentscope-core -am compile

# Python：安装（建议带 workspace + 长期记忆）
cd agent_base/agentscope && pip install -e ".[service]"
```

### 1.2 课前必读（30 分钟）

- 本文 §2 课程地图
- [02 Java 源码原理](02-AgentScope-Java-源码原理.md) §0~§2（知道 core/harness/service 三层是什么）
- [06 选型与落地](06-技术选型建议与落地路线.md) §1（五条结论，带着结论来听课）

### 1.3 讲师准备

- 提前把 §10 附录 C 的 Demo 脚本**全部跑通一遍**，并录屏备用（网络不稳时切录屏）。
- 准备一个"故障 Demo"：故意把 `maxIters` 设为 2、把工具标为 `concurrencySafe=false`、把上下文阈值调到 0.1，让学员**看见**异常行为。
- 每位学员确认能打印出 32 类 `AgentEvent` 的至少 6 类（模块 1 验收点）。

---

## 2. 课程地图

| # | 模块 | 时长 | 形式 | 产出 |
|---|---|---|---|---|
| 0 | 开场：为什么 Harness 决定效果 | 10 min | 讲授 | 建立共识 |
| 1 | **先看现象**：15 分钟跑通全链路 | 30 min | Demo + 动手 | 事件流直觉 |
| 2 | **拆心脏**：ReAct 循环与停泊-恢复 | 70 min | 讲授 + 源码 + 实验 | 能画出循环图 |
| 3 | 工具、权限与 HITL | 60 min | Demo + 实验 | 会写权限策略 |
| 4 | 上下文工程与记忆 | 55 min | Demo + 实验 | 会调上下文参数 |
| 5 | 多智能体编排 | 45 min | Demo + 对比 | 知道两版差异 |
| 6 | 生产运行时：Harness 与 Service | 50 min | Demo + 讲授 | 知道上线要什么 |
| 7 | 对照 dsh：我们该补什么 | 40 min | 研讨 | 落地清单 |
| 8 | 总结、考核与作业 | 20 min | 测验 + 布置 | 验收 |
| — | **合计** | **~380 min（6.3h）** | | |

> 课间各 10 min 未计入。

---

## 3. 模块详解

每个模块统一按 **Demo → 现象 → 提问 → 原理 → 源码 → 实验 → 小结** 推进。

---

### 模块 0 · 开场：为什么 Harness 决定效果（10 min）

**Demo（2 min）**：同一个问题"帮我查一下这笔交易的风险"——
1. 直接调裸模型 API（只能编一段话）；
2. 用带工具的 Agent（真的去查、去算、给出结论）。

**提问**：差距来自模型还是来自别的？

**原理（讲授要点）**：
> 智能体的业务效果不完全取决于模型能力，同样取决于**承载它运行的底座（Harness）的质量**。意图是否被精准理解、工具是否被正确调度、上下文是否被有效管理、执行过程是否可靠可控——这些"模型之外"的工程能力决定交付质量。**同一个大模型，跑在不同质量的 Harness 上，效果天差地别。**

两个佐证：
- AgentScope 官方金融用例 FinXScope 的判断（[05 §2.4](05-FinXScope-金融级智能体底座深度研究.md)）；
- DeepSeek Harness 用 135 万行 TS 只做"模型之外的运行时"（[03 §1](03-DeepSeek-Harness-原理对照与落地应用.md)）。

**小结**：本次培训不讲"怎么调提示词"，讲**模型之外那套机器**是怎么造的。

---

### 模块 1 · 先看现象：15 分钟跑通全链路（30 min）

**目标**：在讲任何原理前，让学员亲眼看到一次 Agent 运行的**完整事件流**。

**Demo 1（Java，`BasicChatExample`）**

```bash
cd agent_base/agentscope-java
export DASHSCOPE_API_KEY=sk-xxx
mvn -q -pl agentscope-examples/documentation -am exec:java \
  -Dexec.mainClass=io.agentscope.examples.documentation2.quickstart.BasicChatExample
```

**Demo 2（Java，事件流版，`AgentEventStreamExample`）** —— 本模块的重点

```bash
mvn -q -pl agentscope-examples/documentation -am exec:java \
  -Dexec.mainClass=io.agentscope.examples.documentation2.streaming.AgentEventStreamExample
```

关键代码（来自 `README_zh.md` 的 Hello AgentScope）：

```java
HarnessAgent.Builder BUILDER = HarnessAgent.builder()
        .name("assistant").agentId("assistant")
        .sysPrompt("你是一个有用的 AI 助手。")
        .model("dashscope:qwen-plus")
        .workspace(Paths.get(".agentscope/workspace"));

try (HarnessAgent agent = BUILDER.build()) {
    RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("demo").userId("alice").build();
    agent.streamEvents(new UserMessage("帮我把今天的关键点列三条。"), ctx)
            .doOnNext(event -> {
                switch (event.getType()) {
                    case TEXT_BLOCK_DELTA -> System.out.print(
                            ((TextBlockDeltaEvent) event).getDelta());
                    case TOOL_CALL_START -> System.out.println(
                            "\n[tool] " + ((ToolCallStartEvent) event).getToolCallName());
                    default -> { }
                }
            })
            .blockLast();
}
```

**Demo 3（Python，终端全功能版）**

```bash
cd agent_base/agentscope/examples/console
export DASHSCOPE_API_KEY=sk-xxx
python main.py --model qwen3.7-max --verbosity debug
```

> 该示例一次性把 workspace（内置文件工具 + 技能）、长期记忆（`AgenticMemoryMiddleware`）、Offloader 全接上了，是最贴近生产的形态。

**现象（引导学员观察）**：
- 输出是**一段一段流出**的，不是整段返回——这是事件流，不是字符串拼接。
- 能看到 `[tool] xxx` 的工具调用提示——模型在"行动"。
- 终端里可以 **Ctrl+C 打断**，打断后 Agent 还能接着聊。

**提问（写在白板上，后几个模块逐个回答）**：
1. 这些"一段一段"的东西是什么？一共有多少种？
2. 工具是谁决定调用、谁执行的？中间能不能拦下来？
3. 为什么打断之后还能继续？状态存在哪？
4. 聊到第 50 轮，前面的对话还在吗？去哪了？

**原理预告（5 min，只给概念不给细节）**：
- 32 类 `AgentEvent` 是 Agent 与外界的**唯一契约**。
- `call()` 返回 `Mono<Msg>`（只要结果）；`streamEvents()` 返回 `Flux<AgentEvent>`（要过程）。
- Python 对应 `reply()` / `reply_stream()`。

**课堂实验（10 min）**：把 `default -> {}` 改成打印**所有**事件类型，统计一次对话里出现了多少种事件。
**验收点**：至少观察到 `AgentStart`、`ModelCallStart`、`TextBlockDelta`、`ToolCallStart`、`ToolResultEnd`、`AgentEnd` 六类。

**小结**：Agent 不是一个函数，是一台**持续吐事件的机器**；后面几个模块就是把这台机器拆开。

---

### 模块 2 · 拆心脏：ReAct 循环与停泊-恢复（70 min）

**目标**：能画出循环图，说出每一步的决策依据，并解释"停泊-恢复"。

#### 2.1 Demo（10 min）

- 让 Agent 执行一个**必须多轮工具调用**的任务：`"读一下 /data/harness/README.md，然后总结三条要点"`（用 `examples/console`，或 Java `ToolStreamingExample`）。
- 打开 `--verbosity debug`（Python）/ 事件全打印（Java），观察 `step` 的重复出现。

**现象**：模型调用 → 工具 → 模型调用 → 工具 → … → 最终答复。这个"思考-行动-观察"的循环就是 ReAct。

#### 2.2 原理（30 min）

**① 两个基本单位**（对齐 dsh 的定义，便于跨框架理解）：
- **step（步骤）** = 一次模型请求 + 它触发的工具执行
- **turn（轮次）** = 零或多个 step，从领取输入开始，到不再欠任何工作为止

**② 循环的骨架**（两版一致，只是语言载体不同）

```
reasoning（思考）
  ├─ 组装请求：系统提示 + 上下文 + 工具 schema
  ├─ 中间件 ON_REASONING 拦截（可改写/注入/拒绝）
  └─ 模型流式返回 → 转成事件块
acting（行动）
  ├─ 取出本轮 tool calls
  ├─ 并发安全分区 → 执行
  └─ 结果回填上下文
判定下一步：继续？还是结束？
```

**③ 判定是纯函数状态机**（这是最值得讲的设计）

Python `Agent._next_action()`（`agent/_agent.py:3495`）返回三者之一：`Reasoning` / `Acting` / `Exit`。docstring 明写 *"Read-only: all side effects are performed by the caller"*。

判定顺序（讲师逐条讲，每条配一个"什么时候发生"的例子）：

| 顺序 | 条件 | 结果 | 现实例子 |
|---|---|---|---|
| 1 | 有待执行的工具调用 | `Acting` | 模型刚说"我要读文件" |
| 2 | 有 awaiting（ASKING/SUBMITTED）调用 | `Exit`（**停泊**） | 等用户点"允许" |
| 3 | 结构化输出已满足 / 未满足 | `Exit(COMPLETED)` / 继续 Reasoning | 要求返回 JSON |
| 4 | 有最终答复 | `Exit` | 模型说完了 |
| 5 | **到达 maxIters** | 注入"最后一次总结机会"提示 + 禁用工具 | **优雅降级，不是硬截断** |
| 6 | 默认 | `Reasoning` | 继续想 |

> **易错点**：第 5 条很多人以为是"跑不动了就报错"，实际是给模型一次**不带工具**的收尾机会——这是产品体验的分水岭。

**④ 停泊-恢复协议**（本模块第二个重点）

Python 的 `reply_stream` 入参是一个联合类型（`_agent.py:289-295`）：

```python
inputs: Msg | list[Msg]
      | UserConfirmResultEvent        # 续：用户确认结果（HITL）
      | UserInterruptEvent            # 续：用户打断
      | ExternalExecutionResultEvent  # 续：外部执行结果
      | None                          # 续：从上次断点续跑
```

**含义：一次 reply 可以跨多次调用。** Agent 可以"停泊"在等待人工确认或外部执行的状态，`reply_stream` 直接返回；外部拿到结果后再调用一次，用 `reply_id` 路由回去继续。

Java 对应：`doCallInner`（`ReActAgent.java:2170`）分流——无 pending → `coreAgent()`；有 ASKING → 权限 HITL resume；有 pending → 自动补齐或外部结果 resume。

**为什么这是灵魂设计**：它让 Agent 可以**跨进程、跨 HTTP 请求、跨天恢复**。没有它，"人工审批"就只能阻塞线程，生产上不可行。

#### 2.3 源码定位（10 min，带着学员打开 IDE）

| 想看什么 | Python | Java |
|---|---|---|
| 循环主干 | `agent/_agent.py:1035` `_reply_impl` | `ReActAgent.java:2790` `reasoning` / `:3455` `acting` |
| 状态机判定 | `agent/_agent.py:3495` `_next_action` | `ReActAgent.java:2956` `runPostReasoningPipeline` |
| 入口分流 | `agent/_agent.py:288` `reply_stream` | `ReActAgent.java:2170` `doCallInner` |
| 轮次收尾 | `agent/_agent.py:1283-1304` finally | `ReActAgent.java:2755` `settleSession` |
| 超限总结 | `agent/_agent.py:3701-3715` | `ReActAgent.java:4555` `summaryStream` |

#### 2.4 课堂实验（15 min）

1. **看循环**：在 `reasoning` / `acting` 各打一条日志，跑一个多步任务，数一下 iter。
2. **触发降级**：把 `maxIters` 设为 2，观察模型是否被强制收尾（应看到"最后一次总结机会"的提示效果，而不是报错）。
3. **触发停泊**：注册一个需要审批的工具（模块 3 会讲），运行到审批时**不响应**，另起一个进程传入确认结果——体会"跨调用恢复"。
4. **思考题**：如果要让 Agent 中途"睡一晚上第二天继续"，需要哪些东西已经具备？（答：reply_id 路由 + 状态持久化；模块 6 讲）

#### 2.5 小结

> 循环 = 纯函数状态机 + 副作用实现分离；停泊-恢复 = 把"等待"建模成"reply 结束但可续"。这两点决定了 Agent 能不能上生产。

---

### 模块 3 · 工具、权限与 HITL（60 min）

**目标**：会注册工具、会配权限策略、能解释一次工具调用的完整生命周期。

#### 3.1 Demo（10 min）

- Java：`tool/ToolCallingExample`、`tool/PermissionContextExample`、`tool/ToolGroupExample`
- 现场把某个工具设为 `dangerousFiles`，触发**审批弹窗**；分别演示"允许"和"拒绝"两条路径。

**现象**：
- 拒绝后，Agent **没有崩**，而是收到一条"被拒绝"的结果，然后自己换了个办法（或向用户解释）——这就是"失败塌缩为结构化结果"。
- 允许后可以配置"本次会话不再询问"（规则沉淀）。

#### 3.2 原理（25 min）

**① 工具调用六步生命周期**

```
① 可用性检查（组是否激活 / 工具是否存在）
② 参数解析（容错 JSON）→ JSON Schema 校验
③ 权限决策（三态：ALLOW / ASK / DENY）
④ 执行（并发安全分区）
⑤ 结果截断 + 超限落盘
⑥ 回填上下文
```

**② 三态权限引擎**（Java `PermissionEngine.java:30-41`）

求值顺序：`deny → ask → 工具自身 checkPermissions（bypass 免疫）→ allow → BYPASS 模式 → 默认 ASK`。

> **单调性**：这条链上"监听器顺序不可能把拒绝翻回允许"。这是安全设计的核心，也是我们要照搬的纪律之一（[03 §3.1](03-DeepSeek-Harness-原理对照与落地应用.md)）。

**③ 并发安全分区**（最容易被忽略的工程细节）

- Python：`_batch_tool_calls`（`_agent.py:2096`）按 `is_concurrency_safe` 切批次；并发批次用 **sentinel** 保证事件流不被截断（`_agent.py:2186`）。
- Java：`ToolExecutor:466-494`——连续的 `concurrencySafe` 工具用 `Flux.mergeSequential`（**保序**）并发跑；不安全工具各自串行。

> 提问：为什么"保序"重要？（答：模型期望工具结果顺序与调用顺序一致；乱序会让模型困惑甚至出错。）

**④ MCP 接入**：两版都是一等公民。Java `McpClientManager` 支持白黑名单、分组、**预设参数（API key 不外泄给模型）**、命名前缀；harness 侧还能用 `workspace/tools.json` 声明式注册，`McpClientRegistry` 运行时动态生效。

#### 3.3 源码定位（5 min）

| 想看什么 | Python | Java |
|---|---|---|
| 工具注册 | `tool/_toolkit.py:66` | `core/tool/Toolkit.java:70`、`Tool.java:60-237` |
| 权限决策 | `permission/_engine.py:17` | `core/permission/PermissionEngine.java:47` |
| 单调用生命周期 | `agent/_agent.py:2426` | `ReActAgent.java:3702` `runToolBatch` |
| 并发分区 | `agent/_agent.py:2096/2186` | `core/tool/ToolExecutor.java:466-494` |
| 结果截断落盘 | `agent/_agent.py:2634-2686` | `ToolResultEvictionMiddleware` |

#### 3.4 课堂实验（15 min）

1. 写两个工具，一个 `concurrencySafe=true` 一个 `false`，让模型一次调用多个，观察执行顺序与耗时差异。
2. 把某工具加入 `dangerousFiles`，分别走"拒绝"路径，观察 Agent 的自我纠正行为，并讨论：**拒绝信息该怎么写，模型才能真的改对？**（提示：参考 Python 的 `_json_loads_with_repair`，它在错误里附可执行复现代码）
3. （进阶）用 MCP 接一个外部服务，然后**故意停掉该服务**，观察框架行为：Python 是 warn 跳过（不拖垮整个 reply），讨论这种 fail-open 在你的场景里是否可接受、要不要改成 fail-closed。

#### 3.5 小结

> 工具系统的三条纪律：**权限链单调（拒绝不可翻回允许）**、**并发保序且事件流完整**、**失败塌缩为结构化结果而非中止轮次**。

---

### 模块 4 · 上下文工程与记忆（55 min）

**目标**：理解上下文为什么会爆、爆了会发生什么、原文去哪了、长期记忆怎么跨会话。

#### 4.1 Demo（10 min）

- Java：`harness/memory/MemoryCompactionExample`、`harness/context/ContextStatePersistenceExample`
- Python：用 `examples/console` 连续聊 30 轮，或故意喂一个超长文件让它读。

**现象**：
- 聊到一定轮次，响应变慢/成本上升，最终触发压缩。
- 超大工具结果被替换成一段提示 + 一个文件路径（而不是消失）。
- 关掉程序再打开，Agent 还记得上次说过的偏好（长期记忆）。

#### 4.2 原理（25 min）

**① 三层数据**（对照 FinXScope 的三层记忆，[05 §4.2](05-FinXScope-金融级智能体底座深度研究.md)）

| 层 | Python 实现 | Java 实现 |
|---|---|---|
| **STM 短期** | `AgentState.context`（未压缩）+ `summary`（压缩后）；阈值 `trigger_ratio=0.8` | 同左 + `CompactionMiddleware`（`onReasoning` 触发），阈值 = 模型窗口 - reserved |
| **LTM 长期** | 中间件三选一：`AgenticMemory`（Markdown 文件即记忆）/ `Mem0` / `ReMe` | harness 双层：daily ledger（`memory/YYYY-MM-DD.md`）→ 合并进 `MEMORY.md` |
| **UserProfile** | 无原生，自建 | 无原生，自建（结构化属性层，注入系统提示） |

**② 压缩的完整流程**（Python `_agent.py:490`，逐条讲防御性设计）：

1. 先卸载旧图（超过 `max_image_num`）
2. 上下文为空却超阈值 → **直接报错**（说明系统提示本身就太长，压不动）
3. reserve 太大导致无可压缩内容 → **降级到 0 再拆一次**
4. 压缩请求本身超长 → **逐个丢弃最老的待压缩消息**
5. 失败兜底 → 保留旧 summary
6. 中断保护 → `asyncio.shield` 保证"摘要已生成但上下文未替换"的中间态不残留
7. **把压缩调用的 usage 记到 context 尾部**（否则 token 账目会漏）

> 这是全课程**最能体现工程成熟度**的一段：每一条都是"LLM 会犯错 / 环境会出意外"的防御。

**③ 投影 vs 替换**（承上启下，引出模块 7）

- 两版都是**替换式**：压缩后原文不在上下文里了（Java 保留原生 SessionLog；Python 靠 Offloader 落盘）。
- dsh 是**投影式**：日志永不被删，压缩只是把区间"遮蔽"成摘要节点，可随时展开。
- 提问：你的业务需要"事后追溯某轮到底说了什么"吗？如果需要，这就是必补项。

**④ 运行时状态注入用 HintBlock 而非改系统提示**——**理由是为了保住 prompt cache**。这是一个"看似小事、实际省钱"的设计。

#### 4.3 课堂实验（15 min）

1. 把 `trigger_ratio` 调到 0.1，观察压缩被提前触发，并检查压缩后 Agent 是否还记得关键信息。
2. 构造一个 >80K 字符的工具返回，观察落盘与提示替换，然后让 Agent 去读那个文件——验证"信息没丢，只是换了位置"。
3. 开启长期记忆，跨两次进程问"上次我说偏好什么"，观察召回。
4. （可选）对比压缩前后一次请求的 token 数与耗时，量化收益。

#### 4.4 小结

> 上下文工程 = **在"信息完整"与"成本可控"之间做可恢复的折中**。三条实践：能落盘就不丢、能保 cache 就不改系统提示、每次降级都要有兜底。

---

### 模块 5 · 多智能体编排（45 min）

**目标**：知道两版编排的差异，能选对方案，并理解"共享执行底座"的价值。

#### 5.1 Demo（10 min）

- Java：`harness/subagent/SubagentStreamingExample`、`harness/async/AsyncToolExample`
- Python：`TeamPipeline` / `GoalPipeline` 示例

**现象**：主 Agent 把任务分给子 Agent，子 Agent 的事件**实时转发**到主流程；子 Agent 有自己的上下文（看不到主 Agent 的对话）。

#### 5.2 原理（20 min）

**① 最大的坑：Java 2.0 没有 Pipeline / MsgHub**

`agentscope-java/docs/v2/zh/docs/change-log.md:42` 明确：`io.agentscope.core.pipeline.*`（`Pipeline`、`SequentialPipeline`、`FanoutPipeline`、`MsgHub`）已删除，改为 **middleware + 子 agent + event stream**。

> 讲师务必强调：**不要拿 Python 的概念去 Java 代码里找**，这是团队最容易浪费半天的地方。

**② Python 的三种编排**

| 机制 | 做法 | 适用 |
|---|---|---|
| `TeamPipeline` | 给 leader 注册 `TeamAssign` **外部工具**（成员名是 schema enum），leader 调用 → pipeline 截获事件 → 真跑成员 → 结果作为 tool result 回灌 | 分派式协作 |
| `GoalPipeline` | executor + verifier 闭环，`fail` 则把反馈喂回 executor | 需要质量校验的任务 |
| `SOPEngine` | 有序步骤 + 每步 `max_attempts`；**步骤间只传递 `<handover>` 包装的产出** | 标准作业流程 |

**③ Java 的等价物**：`SubagentsMiddleware`（`agent_spawn` / `agent_send` / `agent_list` + 任务工具）+ `TeamsMiddleware` + 消息总线/Inbox + 异步工具转后台。

**④ "共享执行底座"为什么重要**（引 FinXScope 的实践）

> OneAgent 与 Multi-Agent 共享同一套执行引擎，因此**权限校验、审计留痕、执行管理**在切换模式时保持一致——不会因为换了编排模式就丧失保障能力。由此才能把工作流封装成 Tool 挂到自主 Agent 上（`MountableResource`）。

#### 5.3 课堂实验（10 min）

1. spawn 一个子 Agent，观察事件转发与主子上下文隔离。
2. 让子 Agent 执行超过超时阈值的任务，观察它是否转后台、结果如何回投（Inbox + `wait_async_results`）。
3. 讨论：你的业务需要"确定流程"还是"自主探索"？能否两者混合（主流程自主 + 关键子流程用固定工作流）？

#### 5.4 小结

> 编排选型：**流程确定用 SOP/工作流，开放探索用 ReAct，混合场景用 MountableResource**。两版 API 不同但思想一致。

---

### 模块 6 · 生产运行时：Harness 与 Service（50 min）

**目标**：知道从"能跑"到"能上线"之间差什么。

#### 6.1 Demo（10 min）

- `harness/workspace/WorkspaceSandboxExample`：工具在隔离沙箱里执行。
- `agui/AguiExampleApplication`：前端通过 AG-UI / SSE 消费事件流。
- 状态持久化：`ContextStatePersistenceExample`——**换一个进程/实例继续同一会话**。

#### 6.2 原理（25 min）

**① Java 三层架构**（核心图，务必讲透）

```
┌──────────────────────────────────────────┐
│ agentscope-service（四平面，Spring Boot 4）│
│  gateway / controlplane(Go) / dataplane / scheduler │
└───────────────────┬──────────────────────┘
                    │ 依赖（单向）
┌───────────────────▼──────────────────────┐
│ agentscope-harness：HarnessAgent          │
│  workspace / fs / sandbox / subagent / team │
│  skill / memory / plan-mode / context      │
└───────────────────┬──────────────────────┘
                    │ 依赖（单向，core 不反向依赖）
┌───────────────────▼──────────────────────┐
│ agentscope-core：ReActAgent / Msg / Event  │
│  Model / Formatter / Toolkit / Middleware │
│  Permission / AgentState / SessionLog     │
└───────────────────▲──────────────────────┘
                    │ SPI 实现
┌───────────────────┴──────────────────────┐
│ agentscope-extensions：model-* / rag-* / mem-* / sandbox-* │
└──────────────────────────────────────────┘
```

**一句话**：`HarnessAgent` 在 `ReActAgent` 之上通过 Middleware 与 Toolkit 叠加工程基础设施，**核心推理循环原样保留**，能力按需叠加。

**② HarnessAgent 的组合而非继承**（`HarnessAgent.java:195` `private final ReActAgent delegate`）——这是可维护性的关键：核心循环不动，20 个中间件各管一摊。

**③ 双事件系统**（最容易讲错的地方，务必对照讲）

| 用途 | 载体 | 特点 |
|---|---|---|
| 前端渲染 / HITL | 32 类 `AgentEvent` | 进程内，实时，不持久 |
| 恢复 / 审计 | `SessionLog`（57 类事件） | **持久化 append-only，带租约 + epoch + fencing + CAS commit + 独立 inbox** |

> 为什么持久化日志要带"租约/epoch/fencing"？因为它是**多副本**场景下的写入者选举——这是单进程 Agent 框架不会有的复杂度，也是 Java 版定位"企业级"的证据。

**④ 无状态与多租户**：`ReActAgent.java:235-238` 明确"同一实例可并发服务多个 (userId, sessionId)"；per-call 状态走 **Reactor Context 透传而非字段**。

#### 6.3 课堂实验（10 min）

1. 起两个进程/实例，用同一 `sessionId` 轮流发消息，验证状态恢复。
2. 打开沙箱示例，让 Agent 执行一次危险命令（如写工作区外文件），观察拒绝与升权提示。
3. 用 AG-UI 示例看前端如何只靠标准事件流渲染（换后端模型/工具，前端零改动）。

#### 6.4 小结

> 从"能跑"到"能上线"差四件事：**隔离（沙箱）、恢复（持久化日志）、多租户（无状态）、可观测（OTel + 指标 + 审计）**。Java 版这四项都已有内建能力。

---

### 模块 7 · 对照 DeepSeek Harness：我们该补什么（40 min · 研讨）

**目标**：把外部最佳实践转化为团队的落地清单。

#### 7.1 讲授（15 min）：dsh 的三条纪律

1. **日志即状态**：`deriveMessages()` 从日志投影模型历史；每个请求可由日志逐字节重建（内置不变式检查）。派生能力——fork、resume、审计、回放**全部免费**。
2. **扩展点显式权力分级**：`emit`（通知，不能否决）/ `serial`（可否决）/ `waterfall`（可改写）。"选对事件域是大多数改动的第一个决定"。
3. **失败塌缩为结构化结果**：任何插件失败都变成工具的结构化返回值，**绝不中止轮次**。

补充两点可对照的工程观：**fail-closed**（审批 seam 缺位即拒绝）、**执行世界可移植**（fs + subprocess 共享身份，换提供方即整体远程化）。

#### 7.2 对照（10 min）：我们有什么、缺什么

| dsh 能力 | AgentScope Java | 我们要不要补 |
|---|---|---|
| 会话日志 = 唯一真源 | 有 SessionLog，但模型上下文仍读 `AgentState.context` | **建议补**：加一层"从日志投影"的校验 |
| 事件权力分级 | 中间件统一洋葱，无分级约定 | **建议补**：团队约定哪些中间件只观察、哪些可改写 |
| 失败塌缩 | 已有（异常二分/结构化错误） | 已有，保持 |
| fork / 重放 | 有 `SessionRecovery`/`SessionProjection`，非一等公民 | 按需 |
| 执行世界可移植 | filesystem 有抽象，**subprocess 未纳入同一抽象** | 若做远程编码场景需补 |
| PTC（run_code 上下文经济学） | 无 | 一般不补（用 Offloader 已够） |

> 详细映射表见 [03 §3.1](03-DeepSeek-Harness-原理对照与落地应用.md)。

#### 7.3 研讨（15 min）：分组输出落地清单

按学员负责的业务分组，每组输出一张表：

| 我们的场景 | 必补能力 | 用 AgentScope 的什么实现 | 工作量 |
|---|---|---|---|

参考维度（来自 [06 §4](06-技术选型建议与落地路线.md) 风险清单）：Java RAG/长期记忆空窗、无 Pipeline、并发取消半截流、压缩丢原文、fail-open 权限、单体巨类维护、供应商绑定。

---

### 模块 8 · 总结、考核与作业（20 min）

**课堂测验（10 min，6 题，见 §0 学习目标）**——建议口答 + 画循环图。

**课后作业（二选一，一周内提交）**：

1. **动手题**：用 `HarnessAgent.Builder` 搭一个带 3 个业务工具 + 审批策略 + 长期记忆的 Agent，能跨进程恢复会话，并给出一次完整调用的事件流截图。
2. **设计题**：针对自己负责的业务场景，写一份 1 页落地方案——选 Python 还是 Java、编排方式、权限策略、上下文预算、审计与恢复方案、风险与对策。

**验收标准**：能回答 §0 的 6 个问题 + 作业通过评审。

---

## 4. 讲师手册：每个模块的"必讲"与"可略"

| 模块 | 必讲（不能省） | 时间不够可略 |
|---|---|---|
| 0 | Harness 决定效果 | 两个佐证的细节 |
| 1 | 事件流是全课程的入口直觉 | Python 的 console 示例（只演示 Java） |
| 2 | **状态机判定顺序 + 停泊-恢复** | sentinel / shield 等内部细节 |
| 3 | 权限链单调性 + 失败塌缩 | MCP 预设参数细节 |
| 4 | 压缩的防御性设计（7 条中的前 4 条） | UserProfile 自建细节 |
| 5 | **Java 无 Pipeline 这个坑** + 共享执行底座 | Python 三种 Pipeline 的细节差异 |
| 6 | 三层架构图 + 双事件系统 + 无状态 | 四平面各平面的细节 |
| 7 | 三条纪律 + 落地清单研讨 | fork/PTC |

---

## 5. 教学节奏建议

- **每 45 分钟必须有一次动手或提问**，纯讲授不要超过 15 分钟。
- 每个模块结束前用一句话小结（已提供），并要求学员**复述**一遍。
- 白板保留一张"问题墙"（模块 1 提出的 4 个问题），回答一个划掉一个，课程结束时全部划完——这是"由 demo 到原理"的闭环感来源。
- 故障 Demo 的效果远好于正常 Demo：**故意让它失败**（拒绝工具、超限、上下文爆、沙箱拒绝），学员印象最深。

---

## 6. 三种裁剪方案

| 版本 | 目标时长 / 保底 | 保留模块 | 适用 |
|---|---|---|---|
| **精华版** | 1.5 h（无保底，讲不完就到 8 收尾） | 0 + 1 + 2（只讲状态机与停泊）+ 8 | 管理层/非直接开发人员科普 |
| **半日版** | 3 h / 2.5 h | 0 + 1 + 2 + 3 + 8（4/5/6 各压成 10 min 概览） | 全体研发 |
| **完整版** | **6 h 目标 / 4.5 h 保底** | 全部 8 模块 | 平台/架构组、即将落地的团队 |

**完整版超时时的降级顺序（按此砍，不动模块 2 与 7）**：
1. 先砍各模块的「课堂实验」中最末一项（保留 2 项）；
2. 再砍模块 4 的实验 4、模块 5 的实验 3、模块 6 的实验 3（各留 2 项）；
3. 再砍模块 6 的 §6.2 原理讲授中的"双事件系统细节"（保留三层架构图与无状态两点）；
4. 最后压模块 7 研讨从 15 min 到 10 min（落地清单改为课后作业）。

> 时间预算经验：每个模块的"实验"环节平均会超出标称时长 20~30%（学员环境与网络问题），**排课时请按目标时长 × 1.15 预留**。
>
> 若要讲 Python 版为主，把模块 2/3/4 的源码定位换成 Python 列即可（本文已双列），模块 6 换成 Python 的 `app/` FastAPI 服务（`create_app`，18 个 router）。调试手法可参考 [08 调试与实战手册](08-调试与实战手册.md)。

---

## 7. 附录 A：源码速查表（发给学员当"地图"）

### Java（路径前缀 `agentscope-core/src/main/java/io/agentscope/core/`）

```
ReActAgent.java:241        类   :248 generate_response  :325-347 状态分桶
                           :2170 doCallInner  :2741 coreAgent  :2755 settleSession
                           :2790 reasoning    :2956 postReasoning  :3455 acting
                           :3055 reasoningStream  :3602 actingStream  :3702 runToolBatch  :4555 summaryStream
agent/Agent.java:47        agent/AgentBase.java:92（:113-119 串行门，:285-304 Context keys）
state/AgentState.java:57
event/AgentEventType.java:40-91（32 类）
session/SessionLog.java:22-88（租约/CAS/inbox）
session/SessionEventTypes.java:24-82（57 类）
model/Model.java:22        model/ModelRegistry.java:44      model/spi/ModelProvider.java:22
formatter/Formatter.java:55
tool/Toolkit.java:70       tool/Tool.java:60-237    tool/ToolValidator.java:49
tool/ToolExecutor.java:466-494（并发安全分区）
permission/PermissionEngine.java:47（:30-41 求值顺序）
```
harness（前缀 `agentscope-harness/src/main/java/io/agentscope/harness/`）：
```
agent/HarnessAgent.java:191（:195 delegate，:1662-2438 Builder）
agent/context/HarnessContextBuilder.java:53
agent/memory/MemoryConfig.java:52
agent/middleware/CompactionMiddleware.java:65 · SubagentsMiddleware.java:77 · TeamsMiddleware.java:57
agent/workspace/WorkspaceManager.java:97     agent/DistributedStore.java:74
```

### Python（前缀 `src/agentscope/`）

```
agent/_agent.py:117 Agent  :288 reply_stream  :1035 _reply_impl  :3495 _next_action
                    :1698 _reasoning_impl  :2096 _batch_tool_calls  :2186 并发执行
                    :2426 _execute_tool_call  :490 _compress_context_impl  :1365 _inject_runtime_state
agent/_config.py:51 ContextConfig   :362 ReActConfig   :415 ModelConfig
message/_base.py:70 Msg  :243 append_event
state/_state.py:209 AgentState
tool/_toolkit.py:66 Toolkit   tool/_types.py:178 ToolChoice   tool/_adapters.py:195 MCPTool
permission/_engine.py:17
model/_base.py:37 ChatModelBase  :465 结构化输出四策略
formatter/_formatter_base.py:20
rag/_knowledge.py:44 KnowledgeBase    middleware/_rag.py:626 RAGMiddleware
pipeline/_team_pipeline.py:110   pipeline/_goal_pipeline.py:55   sop/_engine.py:24
middleware/_base.py:13（7 钩子）      app/_app.py:113 create_app
```

---

## 8. 附录 B：常见问题预判（讲师提前准备答案）

| 学员可能问 | 参考答案 |
|---|---|
| 为什么 Python 版是 async-only？ | 流式与并发模型调用的统一抽象需要；代价是简单脚本也要 `asyncio.run()` |
| 为什么 Java 版没有 MsgHub？ | 2.0 重构删除，改"子 agent + 团队 + 事件流"（`change-log.md:42`） |
| 压缩后原文真的没了吗？ | Java 保留原生 SessionLog；Python 靠 Offloader 落盘并留路径提示。若业务需追溯原文，要自建持久化事件域 |
| Agent 实例能不能复用？ | Java 可以（无状态 + per-call Context 透传），同一实例可服务多 session；Python 的 `Agent` 持有 state，按 session 使用 |
| 模型切换要改多少？ | 只换 `Formatter` + 模型实现；Python 有 24 套 Formatter，Java 有 DeepSeek/GLM/Kimi/MiniMax 兼容层 |
| RAG 能用吗？ | Python 成熟（`KnowledgeBase` + `RAGMiddleware`）；**Java v2 仍在推进**，v1 全部废弃，重度依赖需外挂检索层 |
| 和 Dify/低码怎么选？ | 低码适合验证与 ≤10 步稳定流程；复杂编排、跨会话状态、自主规划、金融级保障必须高码 |
| 我们要不要上 agentscope-service？ | 多团队/多 Agent/需要统一管控与观测时上；单场景应用直接引 `agentscope-harness` 即可 |

---

## 9. 附录 C：Demo 脚本清单（可复制）

```bash
# ===== 通用 =====
export DASHSCOPE_API_KEY=sk-xxx
cd agent_base/agentscope-java
MVN="mvn -q -pl agentscope-examples/documentation -am exec:java"
PKG=io.agentscope.examples.documentation2

# ===== 模块 1：跑通 + 看事件流 =====
$MVN -Dexec.mainClass=$PKG.quickstart.BasicChatExample
$MVN -Dexec.mainClass=$PKG.streaming.AgentEventStreamExample
$MVN -Dexec.mainClass=$PKG.streaming.ToolStreamingExample
$MVN -Dexec.mainClass=$PKG.quickstart.UserIsolatedMultiTurnsExample   # 多用户隔离

# ===== 模块 3：工具与权限 =====
$MVN -Dexec.mainClass=$PKG.tool.ToolCallingExample
$MVN -Dexec.mainClass=$PKG.tool.PermissionContextExample
$MVN -Dexec.mainClass=$PKG.tool.ToolGroupExample

# ===== 模块 3/7：中间件 =====
$MVN -Dexec.mainClass=$PKG.middleware.CustomizedMiddlewareExample
$MVN -Dexec.mainClass=$PKG.middleware.ModelCallMiddlewareExample
$MVN -Dexec.mainClass=$PKG.middleware.SystemPromptMiddlewareExample

# ===== 模块 4：上下文与记忆 =====
$MVN -Dexec.mainClass=$PKG.harness.memory.MemoryCompactionExample
$MVN -Dexec.mainClass=$PKG.harness.context.ContextStatePersistenceExample
$MVN -Dexec.mainClass=$PKG.state.StateAutoSaveExample

# ===== 模块 5：编排与异步 =====
$MVN -Dexec.mainClass=$PKG.harness.subagent.SubagentStreamingExample
$MVN -Dexec.mainClass=$PKG.harness.async.AsyncToolExample
$MVN -Dexec.mainClass=$PKG.harness.planmode.PlanModeAutoExample

# ===== 模块 6：生产运行时 =====
$MVN -Dexec.mainClass=$PKG.harness.workspace.WorkspaceSandboxExample
$MVN -Dexec.mainClass=$PKG.harness.workspace.WorkspaceLocalShellExample
$MVN -Dexec.mainClass=$PKG.harness.skill.SkillCompositionExample
$MVN -Dexec.mainClass=$PKG.agui.AguiExampleApplication        # Web SSE，浏览器打开
```

Python 版：

```bash
cd agent_base/agentscope
pip install -e ".[service]"
export DASHSCOPE_API_KEY=sk-xxx

# 终端全功能示例（workspace + 内置工具 + 长期记忆 + Offloader）
python examples/console/main.py --model qwen3.7-max --verbosity debug

# 完整服务部署（Redis + Qdrant + IM 通道 + Hub + 调度）
python examples/agent_service/main.py
```

---

## 10. 培训后跟进

1. **一周内**：作业评审（§模块 8）。
2. **两周内**：把研讨输出的落地清单合并进 `/data/harness` 工程规范（建议新增 `14-Agent工程/`：工具命名、权限分级、事件词汇、上下文预算、审计要求），同步到 `amosli/harness` 仓库。
3. **一个月内**：挑一个真实场景走完 [06 §3](06-技术选型建议与落地路线.md) 的阶段 1（可用），用真实事件流做一次复盘分享。
4. **长期**：建立"决策笔记"制度——每项架构决策留一篇 note（借鉴 dsh 的 396 篇 `.agents/notes`），让"为什么这么做"不随人员流失。

---

> 相关：[README](README.md) · [01 Python 源码原理](01-AgentScope-Python-源码原理.md) · [02 Java 源码原理](02-AgentScope-Java-源码原理.md) · [03 dsh 对照与落地](03-DeepSeek-Harness-原理对照与落地应用.md) · [06 选型与落地路线](06-技术选型建议与落地路线.md)
