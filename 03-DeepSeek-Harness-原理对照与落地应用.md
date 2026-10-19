# 03 · DeepSeek Harness 原理对照与在 AgentScope 中的落地应用

> 参照物：`dsh/`（DeepSeek Harness，`@deepseek-ai/dsh`）的 01~10 原理文档与源码分析。
> 目标：把 dsh 被验证过的工程思想，映射到 AgentScope Python / Java 上，给出可执行的落地路径。
> 前置：[01-AgentScope-Python-源码原理](01-AgentScope-Python-源码原理.md)、[02-AgentScope-Java-源码原理](02-AgentScope-Java-源码原理.md)

---

## 第一部分 · dsh 原理速览（我们要借鉴什么）

### 1.1 项目基本事实

| 项 | 值 |
|---|---|
| 定位 | 开源 agent harness：模型之外的循环、工具、会话、安全、UI 全套运行时 |
| 形态 | pnpm monorepo（~60 个插件包 + 4 个 app），~135 万行 TS |
| 底层框架 | Cordis（源码 vendor 进 `vendor/cordis`），出自论文 arXiv:2608.25512 |
| 默认模型 | `deepseek-flash` / `deepseek-v4-pro`，上下文窗 1M |
| 工程纪律 | 逐文件 100% 覆盖率 CI 门禁、录制会话回放测试、396 篇架构决策笔记、4 篇 postmortem |

### 1.2 理论基础：时空可组合性

Cordis 把动态组合分解为两个正交维度：

- **时间可组合性**：移除组件时能**完全回滚其副作用**——"可回滚效果"，`ctx.effect(fn)` 注册即撤销、`fiber.dispose()` 递归回卷。
- **空间可组合性**：声明并**响应式地**管理组件间依赖——"响应式协同效果"，`static inject = [...]`：依赖未就绪停 PENDING，依赖消失消费方随之卸载。

**为什么这对 harness 是要害**：harness 的日常操作恰好是"动态装卸能力"——换模型适配器、给某会话挂工具、热重载插件、按 agent 隔离权限。Cordis 把"加载顺序"归约为依赖声明、"行为挂载"归约为可逆副作用，框架只保证两条不变量：**依赖就绪才启动、卸载必回卷**。

### 1.3 核心思想：一切皆插件

官方第一句（docs/architecture.zh.md:11-13）：

> 产品的每一部分都是插件，包括模型适配器、工具注册表、会话日志，以及 **agent loop 本身**，因此每个都可以从配置替换。不存在需要打补丁的特权内核。

实证：`packages/core/agent-loop` 的 `ReactLoopAgent` 通过 `ctx.agents.setFactory(this)` 注册为工厂，`packages/core/agent`（契约）与 `packages/core/agent-loop`（机制）彻底分离——**循环自身是可替换插件**。

### 1.4 三类事件域：事件就是扩展点

| 域 | 载体 | 何时用 | 例 |
|---|---|---|---|
| **会话事件** | 持久日志，经 `session/event` 广播（29 个包消费） | 事实必须在重新加载后仍存在 | `user/message`、`tool/call`、`turn/start` |
| **Agent 事件** | 进程内 `agent/*`（emit/serial/waterfall） | 观察或拦截**进行中**的工作 | `agent/pre-step`、`agent/request`、`agent/turn-stopping` |
| **能力事件** | 无需导入循环即可附加到 seam | 给能力加策略/适配器 | `fs/write-intent`、`tools/pre-execute`、`telemetry/*` |

Cordis 事件 5 种分发模式：`emit`（同步广播，包容失败）、`parallel`（并行扇出）、`serial`（顺序执行，首非空胜出）、`bail`、`waterfall`（环绕中间件，不调 `next()` 即短路否决）。

**纪律**：每个事件用 `@mode` 标注模式；纯观察者必须调 `next()`；"选对事件域是大多数改动的第一个决定"。

### 1.5 能力 seam：可替换能力的三角色

**seam** = Service Definition（声明接口，拥有 `ctx.<key>`）+ Service Provider（实现）+ Consumer（通常是面向模型的工具）。

> "替换一个提供方就能改变整个产品"：文件系统与进程提供方共享同一个**执行世界**，把它们指向 SSH 远端，Bash、PTY、LSP 一并搬走，无需提供方专用 fork。

规范范例：`packages/shell`（定义 dsh-shell / 提供方 bash-local、bash-sandbox / 消费方 tool-bash）。

### 1.6 轮次流程（turn flow）

```text
turn/start                                ← 持久会话事件（先落账再走）
  claim:领取 next-step 全部 + (轮边界)一条 next-turn
  组装提示词片段 + 工具 schema,投影运行时上下文
  → agent/pre-step (waterfall)            ← 插件可改写/拒绝进入的消息批次
    step/start
    agent/request (waterfall) → prepareCall()   ← 解析路由;此阶段取消则系统提示词与用户消息都不提交
    reconcile 系统提示词(surface 节点 0),追加 user/message
    从日志派生并冻结模型历史(deriveMessages + deepFreeze)
    流式请求 → llm/stream (waterfall) → agent/assistant-stream(实时帧,进程内)
      chunk*
    结算:assistant/message(成功,内嵌紧凑流)| assistant/attempt(失败/取消,log-only)
    tool/call* → tools/pre-execute → tools/execute → tools/post-execute → tool/result*
    step/end
  → agent/turn-stopping (serial)          ← 收尾检查点,监听器可 steer 反对
turn/end {reason: completed|aborted|blocked|error|max-tokens}
```

**一个 step = 一次模型请求 + 它调用的工具；一个 turn = 零或多个 step。**

### 1.7 哲学基石：会话日志是唯一真源

三段论（docs/architecture.zh.md:123-131）：

1. **会话日志是模型所见上下文的来源**：`deriveMessages()` 从日志投影出模型历史；每个请求都可由日志逐字节重建（循环内置不变式检查）。
2. **模型可见即已记录**：任何要进入模型视野的内容必须是会话事件。
3. **投影而非截断**：压缩不删日志，而是用 surface `replace` 把区间遮蔽成摘要节点——历史永远无损可重放。

奠基决策（2026-06-11）**否决**了"可变消息数组 + 事件仅作通知"：日志即状态后，状态与日志的分歧在结构上不可能。派生性质：**fork = 取日志前缀 + 打标；resume = 重放日志 + 补齐中断轮次；UI = 订阅 session/event 折叠渲染；审计与回放免费获得。**

### 1.8 工具系统 15 步流水线

`ToolRuntime.execute()`（`packages/core/tools/src/index.ts:1369`），拆成供并行调度器使用的三段 prepare/dispatch/finalize：

1. 实体化（lossless-JSON 快照 + deepFreeze + 不透明关联 token）
2. PTC 折叠判定（策略之前确定性终止）
3. `tools/pre-execute` waterfall → `allow | deny | cancel | ask`（**有意不允许改写 arguments**）
4. 审批 ask → **无人应答即 fail-closed**（可区分"人说不"与"没人可问"）
5. 单调守卫（**只返回拒绝理由，没有 allow 结果**——监听器顺序不可能把拒绝翻回允许）
6. `tools/execute` around waterfall（超时/重试/指标，信号熔合）
7. 工具体 execute → 结果过 output.schema + render
8. fs 写意图门禁（变异类工具，落盘前）
9. `projectContent`（在 post-execute 之前）
10. `tools/post-execute` waterfall（accept / block / 附加上下文）
11. 注册表外层规范化兜底
12. `finalizeContent`（对失败结果也恰好执行一次）
13. `tools/result` 同步通知（冻结 + lossless-JSON）
14. 会话事件 `tool/call` + `tool/result`
15. `additionalContexts` FIFO → 下一步 user/message

**取消语义**：主体未启动 → `ABORTED_BEFORE_DISPATCH`；已启动 → 等完全停稳后把成功结果替换为 `ABORTED`。**任何插件失败都塌缩为结构化结果而非中止轮次。**

### 1.9 执行世界：fs + subprocess = 一个世界

决策笔记 2026-07-28：`ctx.fs` + `ctx.subprocess` 共同定义一个执行世界——共同挂载的提供方必须描述相同的路径命名空间、可执行文件、进程与终端会话；上层能力（bash 执行器、PTY、LSP）只消费这两个接口。SSH 家族把执行世界搬到远端，**Harness 本体留在本地**。

### 1.10 13 条设计思想（dsh/01 §13）

组合即补丁序列 · 日志即上下文投影即视图 · 契约与机制分离 · 事件分域 waterfall 即权威 · 先落账再执行 · fail-closed 安全观 · 呈现与执行分离（PTC） · 执行世界可移植 · 工程纪律即架构。

---

## 第二部分 · 逐项对照：dsh vs AgentScope

### 2.1 总览对照表

| 维度 | DeepSeek Harness (dsh) | AgentScope Python | AgentScope Java |
|---|---|---|---|
| **扩展机制** | Cordis 插件 + 可逆效果 + patch 层叠加 | 中间件（7 钩子）+ Toolkit 动态注册 | MiddlewareBase 六阶段 + 20 个 harness 中间件 |
| **agent loop 可替换** | ✅ 显式（`setFactory`，契约包与机制包分离） | ❌ 单一 `Agent` 类内置 | ❌ 单一 `ReActAgent`（但 harness 用**组合**包一层） |
| **会话日志 = 唯一真源** | ✅ 强不变式（逐字节重建校验） | ⚠️ 有 `AgentState`（可序列化），但**模型历史不是从日志投影** | ✅ `SessionLog` 持久化事件日志（租约/CAS/inbox），但**模型上下文仍来自 `AgentState.context`** |
| **事件分域** | ✅ 三类域（会话/agent/能力）+ 5 种分发模式 | ⚠️ 32 类 `AgentEvent`（运行时流），无持久化域 | ✅ 双事件系统：32 类 `AgentEvent`（UI）+ 57 类 SessionLog 事件（持久） |
| **能力 seam 三角色** | ✅ 显式（Definition/Provider/Consumer），fs/subprocess/sandbox/shell 四族 | ⚠️ 有 workspace 8 后端，但非统一 seam 三分 | ✅ filesystem / sandbox / model / 存储 均有抽象 + 多实现 |
| **工具流水线** | ✅ 15 步固定轨道 + 3 waterfall + 单调守卫 | ⚠️ Toolkit + 权限引擎，层数较少 | ✅ 三态权限 + 并发安全分区 + 沙箱矩阵 |
| **HITL** | ✅ approval seam（缺位即拒绝） | ✅ 停泊-恢复（`RequireUserConfirmEvent`） | ✅ 一等公民 + 分布式挂起持久化 |
| **上下文压缩** | ✅ 投影式（surface replace，日志无损） | ⚠️ 替换式（`AgentState.context` 被摘要替代，原文丢失）+ Offloader 落盘 | ⚠️ 同样替换式，但**原生 session log 保留完整对话** |
| **fork / 重放** | ✅ 一等公民（fork seed + 合成闭合器） | ❌ 无 | ⚠️ 有 `SessionRecovery.reconcile` / `SessionProjection`，非一等公民 |
| **执行世界可移植** | ✅ fs+subprocess 共享身份，SSH 整体搬移 | ⚠️ workspace 8 后端（Docker/E2B/K8s…），偏"沙箱"而非"执行世界" | ✅ AbstractFilesystem + 沙箱，namespace 前缀隔离 |
| **可观测性** | 遥测插件 + 事件日志 | OTel 中间件 | OTel + reactor instrumentation + AgentTrace |
| **工程纪律** | 100% 覆盖门禁 + 回放测试 + 396 篇决策笔记 + postmortem | 有 roadmap、CI | Spotless 强制 + JaCoCo + StepVerifier + AGENTS.md 边界约束 |

### 2.2 关键差异解读

**① 日志真源的强度不同。** dsh 是"模型历史从日志投影"，这是结构性保证；AgentScope 是"状态对象可序列化"，`AgentState.context` 本身就是模型历史。AgentScope 的做法更简单直接（Pydantic/POJO 直接 dump），但失去了 dsh 的三项免费能力：逐字节重建校验、fork、投影式压缩（日志无损）。

**② 扩展点的权力边界显式程度不同。** dsh 用事件模式显式声明"这个扩展点有多大权力"（emit 不能否决 / serial 可否决 / waterfall 可改写）；AgentScope 的中间件是统一的洋葱模型，没有"只观察不可否决"与"可改写"之分。

**③ 执行世界的抽象层次不同。** dsh 的 `fs + subprocess` 是**最小充分条件**——换提供方即整体远程化；AgentScope 的 workspace/沙箱偏"给工具一个隔离环境"，未把"终端身份/进程身份"作为一组共享抽象。

**④ AgentScope 反而更强的地方**：结构化输出的四策略降级阶梯、并发工具的 sentinel 完整性保证、工具组懒加载、双层长期记忆（daily ledger + MEMORY.md）、Plan Mode、技能自学习闭环、Agent-as-a-Service 平台。这些都是 dsh 未覆盖或覆盖较浅的。

---

## 第三部分 · 落地应用：把 dsh 思想映射到 AgentScope

### 3.1 映射总表

| dsh 概念 | Python 版落地手段 | Java 版落地手段 |
|---|---|---|
| **能力 seam（Definition/Provider/Consumer）** | 用 Protocol/ABC 定义接口（如 `workspace/` 的 Offloader），Provider 走 extras，Consumer 是内置工具 | 已有天然对应：`AbstractFilesystem` / `Sandbox` / `Model` + `extensions-*` 实现。**建议照抄三角色命名规范** |
| **一切皆插件 / 组合即补丁序列** | 中间件（7 钩子）即是插件；配置对象（4 个 Config）即是 patch 层 | `HarnessAgent.Builder` 就是 patch 层；`.middleware(...)` 跑在内置中间件**之前**——等价于"后注册的 patch 覆盖前层" |
| **三类事件域** | 32 类 `AgentEvent` = 运行时域；**缺持久化域**——建议自建：把 `AgentEvent` 追加到自己的 session log，并用 `Msg.append_event()` 反向物化 | ✅ 已有双事件系统，直接用 `SessionLog`。**只需补上"模型历史从日志投影"这一步** |
| **会话日志唯一真源** | ⚠️ 需自建：用一个 `on_reply` / `on_reasoning` 中间件把事件落盘，并在 resume 时重放。核心难点是**没有 `deriveMessages()`** | ⚠️ 已有 `SessionLog`，但模型上下文仍读 `AgentState.context`。建议：保留 `AgentState` 作热态，SessionLog 作冷态真相源 + 恢复依据（**已是官方设计**） |
| **waterfall 权力分级** | 用中间件实现：观察型中间件**绝不吞事件**；改写型中间件才可截获 `ReplyEndEvent`（框架已支持，且有 `made_progress` 忙循环保护） | 六阶段 `MiddlewareBase`：同左，约定 `ON_REASONING` 可改写、`ON_MODEL_CALL` 只观察 |
| **单调守卫（只拒绝不允许）** | 权限引擎已近似（`PermissionEngine` 求值顺序 deny → ask → 工具自身 → allow）。建议**明确禁止中间件把 DENY 翻回 ALLOW** | ✅ `PermissionEngine.java:30-41` 已是单调顺序，加一条代码评审规则即可 |
| **fail-closed 安全观** | 审批中间件缺失 → 一律 DENY；MCP 不可达已有 warn 降级（可按需改为 fail-closed） | 审批 seam 缺位即拒绝；`Sandbox` 不可用 → `SANDBOX_UNAVAILABLE` fail-closed |
| **工具流水线固定轨道** | 现有：可用性检查 → 解析 → 校验 → 权限 → 执行 → 结果截断。**建议补齐三段 waterfall**：pre（可 deny/ask，不改参数）/ around（超时重试）/ post（可 block + 附加上下文） | `ToolExecutor` 已有 around（timeout + Retry.backoff），**缺 pre/post waterfall**——用 `ToolMiddlewareBase` 补（`tool/_base.py` 对应 Java 的 `ToolMiddleware`） |
| **执行世界可移植** | 用 `workspace/` + builtin 工具族（Bash/Read/Write/Edit/Glob/Grep）统一走 Offloader 抽象；把"路径命名空间 + 进程"作为一组配置整体切换 | ✅ 最接近：`AbstractFilesystem` 已支持 local/remote/sandbox/composite + `IsolationScope` 命名空间前缀。**建议把 subprocess 也纳入同一抽象**（目前终端身份未统一） |
| **投影而非截断（压缩）** | 现有是替换式 + Offloader。建议：开启 offloader 落盘（已有 `<<<TRUNCATED>>>` + 路径提醒），并保留一份原始事件日志 | ✅ `CompactionMiddleware` 明确保留原生 session log——**已符合"日志无损"精神**，只需暴露"查看原始对话"能力 |
| **先落账再执行** | `tool/call` 应在执行前落盘。建议在 pre-execute 中间件里 append | 同左，用 `SessionRecorder` |
| **实时帧与持久结算分离** | ✅ 已是：`Msg.append_event()` 物化 vs `AgentEvent` 实时流 | ✅ 已是：32 类 `AgentEvent` 与 SessionLog 分离 |

### 3.2 推荐落地路径（Java 版，因为 harness 模块已最接近）

**阶段一：把"会话日志真源"补强（1~2 周）**

1. 确认 `SessionHistoryMode` 使用 **EVENT_LOG** 模式（而非 LEGACY 的 `AgentStateStore`）。
2. 补一个 `on_reply`/`on_reasoning` 中间件，把每次模型请求的**完整输入**（系统提示 + 工具 schema + messages）以 log-only 事件落盘，形成 dsh 的 `request/header` 等价物。
3. 建立**重放校验**：从 SessionLog 重建一次请求，与实际请求做逐字节比对（对齐 dsh 的 invariant.ts）。这是"日志即真源"从口号变成保证的关键一步。

**阶段二：把工具流水线补成固定轨道（2~3 周）**

4. 用 `ToolMiddleware` 实现 pre / around / post 三段：
   - pre：可 `deny`/`ask`，**不可改写 arguments**（对齐 dsh 的有意设计）
   - around：超时 + 指数退避重试 + 指标（`ToolExecutor` 已有 Retry.backoff，补指标）
   - post：可 `block`（把结果变成纠正性反馈）+ 附加上下文
5. 审批 seam 缺位即 DENY（fail-closed），并把"人说不"与"没人可问"分成两种可区分的拒绝理由。
6. 单调守卫：权限判定链只返回拒绝理由，不提供 allow 通道。

**阶段三：上下文与执行世界（3~4 周）**

7. 压缩改为"投影式"：保留 SessionLog 原文，`AgentState.context` 只存"表面"（已部分具备），并暴露"展开原始区间"能力。
8. 把 subprocess/终端纳入与 filesystem 同一组抽象（共享命名空间与身份），实现"换提供方即整体远程化"。
9. Offloader / `ToolResultEviction` 已有，确保落盘后的路径提醒进入提示词。

**阶段四：纪律（长期）**

10. 借鉴 dsh 的**工程纪律即架构**：录制会话回放测试（用 `MockModel` + 真实 SessionLog 回放）、决策笔记制度（每项架构决策留一篇 note）、postmortem。AgentScope Java 已有 `AGENTS.md` 边界约束与 JaCoCo，补齐"覆盖率门禁 + 回放测试"即可。

### 3.3 如果选 Python 版，优先级调整

Python 版缺持久化事件域，因此**阶段一要重做**：

1. 自建 `SessionLog`：一个 `on_reply` + `on_reasoning` + `on_acting` 三钩子中间件，把事件追尾写 JSONL（`_utils` 已有 id/timestamp 工厂可注入，便于确定性回放）。
2. 用 `Msg.append_event()` 做反向物化——已有实现且处理了 base64 分段拼接等坑，**直接复用**。
3. 其余（工具三段流水线、fail-closed、投影式压缩）与 Java 版路径一致，用中间件实现即可，**不需要改核心类**。

> 关键判断：Python 版因为中间件是唯一扩展点且核心类封闭，**上述所有增强都能在不 fork 源码的前提下完成**——这正是 2.x 中间件设计的价值。

### 3.4 不要照搬的三件事

1. **不要照搬 Cordis 的可逆效果体系**。Python/Java 都没有 fiber + 可逆副作用，硬套会与 asyncio/Reactor 的生命周期模型打架。用"中间件 + 配置对象"已经能达到 80% 的效果。
2. **不要照搬 PTC（`run_code`）**。dsh 的 PTC 是为"上下文经济学"设计的（中间工具结果被沙箱程序消化），需要独立沙箱运行时；AgentScope 用 Offloader/Eviction 落盘 + 路径提醒已经解决大部分问题，成本更低。
3. **不要强求 100% 覆盖率门禁**。dsh 有 135 万行 TS 的体量和配套录制回放基础设施才撑得住；AgentScope 接入项目建议先做"核心路径回放测试 + 增量覆盖率门禁"。

---

## 第四部分 · 一句话结论

> **dsh 最值得借鉴的不是某一个模块，而是三条纪律**：① 日志即状态（让 fork/resume/审计/回放变成结构性免费能力）；② 扩展点显式声明权力边界（观察/否决/改写三类，用事件模式编码）；③ 失败一律塌缩为结构化结果而非中止轮次。
>
> 落到 AgentScope：**Java 版选 harness 模块为基座**（已有 SessionLog、沙箱、文件系统抽象，只需补"模型历史从日志投影"与"工具三段流水线"）；**Python 版用中间件自建持久化事件域**，其余同理。两者都不需要 fork 核心代码。

---

> 下一篇：[04-同类框架对比与技术选型](04-同类框架对比与技术选型.md)
