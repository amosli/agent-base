# 02 · AgentScope Java 版源码原理

> 源码路径：`agent_base/agentscope-java`（`2.1.0-SNAPSHOT`，GA 发布版本 2.0.3；JDK 17+）。
> 路径前缀约定：`agentscope-core/...` = `agentscope-core/src/main/java/io/agentscope/core/...`；`harness/...` = `agentscope-harness/src/main/java/io/agentscope/harness/...`。
> 姊妹篇：[01-AgentScope-Python-源码原理](01-AgentScope-Python-源码原理.md)

## 0. 一句话定性

Java 版不是 Python 版的简单移植，而是**面向企业级生产环境的重新定位**：

- `agentscope-core` = 与 Python 2.0 对齐的**内核**（ReAct 循环、事件、工具、中间件、权限、状态）。
- `agentscope-harness` = **生产运行时外壳**（workspace / 沙箱 / 子 agent / 团队 / 技能 / 记忆 / 计划模式 / 上下文工程）。
- `agentscope-service` = **Agent-as-a-Service 四平面平台**（Spring Boot 4）。

核心架构约束（`AGENTS.md:56-65`）：可复用 agent 行为放 core，高层能力放 harness，厂商代码放 extensions，**core 不得反向依赖 harness / service / 具体扩展**。

---

## 1. 多模块 Maven 工程结构

```
agentscope-parent (pom)
├─ agentscope-core             内核：ReActAgent / Msg / Event / Toolkit / Middleware / Permission / AgentState / SessionLog
├─ agentscope-harness          生产运行时：workspace / filesystem / sandbox / subagent / team / skill / memory / plan-mode
├─ agentscope-service         Agent-as-a-Service：gateway / dataplane / scheduler / controlplane(Go)
├─ agentscope-extensions       22 个子模块：model-* / rag-* / mem-* / sandbox-* / protocol / channel / spring-boot-starters
├─ agentscope-examples         documentation + jev / paw / chat / dataagent / codingagent 等完整应用
├─ agentscope-dependencies-bom 第三方版本 BOM（OTel 1.66.0 / Reactor 2025.0.2 / Jackson 2.21.1 / dashscope 2.23.2）
└─ agentscope-distribution     agentscope-all(shaded) + agentscope-bom
```

工程约定（`pom.xml`）：
- Spotless + google-java-format(AOSP)，绑定在 `compile` 阶段（`:168-213`）。
- 编译开启 **`-parameters`**（`:264-271`）——这是 `@Tool` 反射解析方法参数名的前提。
- 全局测试依赖 `reactor-test` / `mockito` / `junit-jupiter` / `okhttp3 mockwebserver`；core 发布 **test-jar** 供 harness 复用测试夹具。

### 1.1 各模块关键依赖

| 模块 | 关键依赖 | 说明 |
|---|---|---|
| core | Reactor、Jackson、victools jsonschema、networknt json-schema-validator、**MCP SDK**、OkHttp、**tree-sitter + tree-sitter-bash**、OpenTelemetry API + reactor-3.1 instrumentation | tree-sitter-bash 用于 Bash 工具 **AST 级命令安全分析** |
| harness | **仅依赖 core**（`pom.xml:33-37`）；测试期依赖 core 的 test-jar 与 `agentscope-extensions-agui` | 单向依赖，边界严格 |
| service | Spring Boot **4.0.4** + Spring Cloud 2025.1.2；`maven.deploy.skip=true` | 四平面 |
| extensions | 22 个子模块（model/rag/mem/sandbox/protocol/channel/starters…） | 厂商与协议外置 |

> 注意：core 的 POM 里有一段 `<!-- ===== Harness-merged dependencies ===== -->`（`agentscope-core/pom.xml:152`）及其后的 commons-compress / sqlite-jdbc / jedis(optional) 等，说明历史上部分 harness 相关依赖曾下沉到 core；但 `io.agentscope.core.workspace` 目前只剩一个占位 `package-info.java`（明写"will host WorkspaceBase contract"），真正的 Workspace 实现在 harness。

---

## 2. 核心抽象：Agent / AgentBase / ReActAgent

### 2.1 接口分层（`core/agent/`）

```java
public interface Agent extends CallableAgent, StreamableAgent, ObservableAgent   // Agent.java:47
```

- `CallableAgent`（`:36`）：`Mono<Msg> call(List<Msg>)` + 结构化输出重载 `call(List, Class<?>)` / `call(List, JsonNode)`。契约：**一次 call 产生且仅产生一个 terminal Msg**。
- `StreamableAgent`（`:44`）：**整个接口自 2.0.0 起 `@Deprecated(forRemoval=true)`**，返回粗粒度 `Flux<Event>`；推荐改用 `ReActAgent#streamEvents(...)` 返回 `Flux<AgentEvent>`。
- `ObservableAgent`：observe 模式，接收消息不产生回复（多智能体协作）。

### 2.2 `AgentBase`：生命周期骨架（`core/agent/AgentBase.java:92`）

- **按 session 串行化**：`callSerializationKey(RuntimeContext)` + `ConcurrentHashMap<Object, Mono<Void>> callGates`（`:113-119`）→ 同一 `(userId, sessionId)` 的调用 FIFO 串行，不同 session 并发。
- **Reactor Context 透传（不是字段！）**：三个 key——`CALL_SCOPE_KEY`（`:285`）、`RUNTIME_CONTEXT_KEY`（`:293`）、`SHUTDOWN_REQUEST_ID_KEY`（`:303`）。这是"Agent 完全无状态"的实现基础。
- **生命周期**：`call()` → `callInternal()`（`:240-266`）→ `runLifecycle`（`Mono.using` 三段式，`:313-319`）→ `runInContext`（注册 GracefulShutdown、复制入参为私有可变副本、`serializeOnKey` 门控，`:321-376`）→ `runLifecycleBody`（`Mono.usingWhen` 获取/释放 per-call scope，`:378-401`）。
- 中断：协作式，`checkInterruptedAsync()` 在检查点抛 `InterruptedException`。
- Agent ID 校验：`[A-Za-z0-9._-]+`、≤255 字符、不能全是点（`:108`）——因为 ID 会被插值进路径/命名空间。

### 2.3 `ReActAgent`（`core/ReActAgent.java:241`，332 KB 单体巨类）

**无状态化**（2.0 架构变更）：per-call 可变状态封在内部类 `CallExecution` 里，经 Reactor Context 透传；`stateCache` / `slotVersions` / `permissionEngineCache` 按 `slotKey(userId, sessionId)` 分桶（`:325-347`）。类注释明确"同一实例可并发服务多个 (userId, sessionId)"（`:235-238`）。

| 方法 | 行号 | 作用 |
|---|---|---|
| `doCallInner` | 2170 | 入口分流：无 pending tool → `coreAgent()`；有 ASKING → 权限 HITL resume；有 pending → 自动补齐/外部结果 resume |
| `coreAgent` / `resumeAgent` | 2741 / 2749 | `settleSession(executeIteration(0))` / `settleSession(acting(0))`（跳过 reasoning） |
| `settleSession` | 2755-2773 | 与 `SessionExecution.continueOrClose` 协作，决定是否需要继续排下一轮 |
| `reasoning(iter, b)` | 2790 | maxIters 判定（≥maxIters → `summarizing()`）→ 中断检查 → 组装 `GenerateOptions`/tools（含 per-call `generate_response` 结构化输出工具，`:2855-2867`）→ `MiddlewareChain.build(ON_REASONING).apply(...)` |
| `runPostReasoningPipeline` | 2956-3040 | HITL stop / `gotoReasoning` / 完成判定 / **空回复注入 reminder 再循环**（`:3022-3029`） |
| `acting(iter)` | 3455 | 取 pending tool calls；`ON_ACTING` 中间件；结果分 success/pending；`returnDirect` 短路 |
| `reasoningStream` / `actingStream` | 3055 / 3602 | 细粒度事件流 |
| `runToolBatch` | 3702 | denied（3722）/ approved（3756）分流 |
| `summaryStream` | 4555 | maxIters 超限后的总结调用 |

**结构化输出**双路径（`STRUCTURED_OUTPUT_TOOL_NAME = "generate_response"`，`:248`）：模型支持原生 `response_format` 就用 `nativeResponseFormat`；否则注入合成工具 + `ToolChoice.Specific` 强推（`:2836-2847`），最多强制重试 3 次（`:2990-3004`）。

### 2.4 与 Python 版核心差异

| 维度 | Python | Java |
|---|---|---|
| 回复 API | `reply()` / `reply_stream()` → `AsyncGenerator` | `call()` → `Mono<Msg>` / `streamEvents()` → `Flux<AgentEvent>` |
| 并发模型 | asyncio 单线程事件循环 | Reactor + `Schedulers.boundedElastic()`，按 session 门控串行 |
| 状态持有 | 对象内持有 | **完全无状态**，per-call 状态走 Reactor Context |
| 扩展点 | 7 钩子中间件 | `MiddlewareBase` 六阶段（v1 `Hook` 已 deprecated） |
| 多租户 | 隐式 | 显式文档化："同一实例可并发服务多个 (userId, sessionId)" |

---

## 3. 模型层

### 3.1 `Model` 接口（`core/model/Model.java:22`）

```java
Flux<ChatResponse> stream(List<Msg> messages, List<ToolSchema> tools, GenerateOptions options);
String getModelName();
```

能力协商（默认方法）：`supportsNativeStructuredOutput()`（`:51`）、`supportsNativeStructuredOutputWithTools()`（`:69`，部分 OpenAI 兼容厂商在 tools 存在时优先 response_format 会跳过工具调用）、`supportsToolChoiceSpecific()`（`:88`）、`getContextWindowSize()`（`:101`，供 harness 压缩中间件动态算阈值）。

### 3.2 SPI 与注册表（厂商外置）

- `ModelProvider` SPI（`core/model/spi/ModelProvider.java:22-45`）：`providerId()` / `supports(modelId)` / `create(...)`。
- `ModelRegistry`（`core/model/ModelRegistry.java:44`）解析顺序：**命名注册 → 缓存 → 用户工厂（新注册的优先）→ SPI providers**（`resolve` `:104-167`）；支持 `registerFactory(String regex, …)`。
- **2.0 起 OpenAI / Gemini / Anthropic / DashScope / Ollama 实现全部迁出 core**，改为 `agentscope-extensions-model-*`；core 只保留抽象与注册表。字符串 modelId 仍可用（`.model("dashscope:qwen-plus")`），前提是扩展在 classpath 中。

### 3.3 `Formatter`（`core/formatter/Formatter.java:55`）

泛型三元组 `TReq`（厂商请求消息）/ `TResp`（厂商响应）/ `TParams`（厂商参数 builder），保证类型安全：

```java
List<TReq> format(List<Msg> msgs);
ChatResponse parseResponse(TResp response, Instant startTime);
void applyOptions(TParams b, GenerateOptions o, GenerateOptions def);
void applyTools(TParams b, List<ToolSchema> tools);
default void applyToolChoice(TParams b, ToolChoice tc);
```

DashScope 实现含 `DashScopeChatFormatter` / `DashScopeMultiAgentFormatter` / `DashScopeConversationMerger` / `DashScopeMediaConverter` / `DashScopeResponseParser` 等。OpenAI 实现另有**兼容层** `compat/deepseek/`、`compat/glm/`、`compat/kimi/`、`compat/minimax/`——国内模型基本都有对应适配。

---

## 4. 工具系统

### 4.1 `Toolkit`（`core/tool/Toolkit.java:70`）

组合结构（`:74-83`）：`ToolGroupManager` / `ToolRegistry` / `ToolSchemaProvider` / `MetaToolFactory`（`reset_equipped_tools` 元工具）/ `McpClientManager` / `ToolSchemaGenerator` / `ToolMethodInvoker` / `ToolkitConfig` / `ToolExecutor`。

- `registerTool(Object)`（`:159`）→ 反射扫描 `getDeclaredMethods()` 上带 `@Tool` 的方法（`:187-201`）。
- `registerSchema(ToolSchema)`（`:315-317`）→ 包装为 `SchemaOnlyTool`，**外部工具**（框架不执行，返回 `TOOL_SUSPENDED` 让调用方执行）。
- 流式 Builder：`registration().tool(obj).group("g").presetParameters(Map).apply()`。

### 4.2 `@Tool` 注解（`core/tool/Tool.java:60-237`）

| 属性 | 行号 | 语义 |
|---|---|---|
| `strict()` | 97 | strict JSON schema 模式 |
| `deferLoading()` | 109 | 延迟加载定义（OpenAI Responses `tool_search`） |
| `readOnly()` | 119 | 只读工具在 `EXPLORE`/`ACCEPT_EDITS` 下自动放行 |
| `concurrencySafe()` | 129 | **默认 true**；false 时并行批次内串行化 |
| `externalTool()` | 139 | 外部执行 → `TOOL_SUSPENDED` |
| `stateInjected()` | 150 | 注入 `AgentState` 参数，且不进 schema |
| `returnDirect()` | 183 | 结果直接作为本轮最终回复，跳过下一轮 reasoning |
| `dangerousFiles()` / `dangerousDirectories()` | 192 / 201 | 敏感路径触发权限 |

### 4.3 参数校验（`core/tool/ToolValidator.java:49`）

`validateInput`（`:106-135`）用 **networknt json-schema-validator**，dialect **DRAFT_2020_12**，错误消息带 `instanceLocation` 前缀。`validateJsonOutput`（`:148-169`）做结构化输出校验，不做 optional-null 归一化（严格语义）。`requireNonBlank`（`:75-86`）在厂商返回空工具名时丢弃该调用。

### 4.4 并行调用：并发安全分区

`ToolkitConfig` 默认 `parallel = true`（`:127`），默认 5 分钟超时、1 次尝试。`ToolExecutor.executeAll`（`core/tool/ToolExecutor.java:415/435`）：

- 顺序模式：`Flux.concat(monos)`（`:450-464`）。
- **并行模式带"并发安全分区"**（`:466-494`）：连续的 `concurrencySafe` 工具用 `Flux.mergeSequential`（**保序**）并发跑；不安全工具各自单独成串行 slot，最后 `Flux.concat(chunks)`。这是比"全并行"更精细的设计。
- 基础设施：`subscribeOn(Schedulers.boundedElastic())`（`:566`）、`timeout`（`:573`）、`Retry.backoff(...).jitter(0.5)`（`:587`，排除 `ActionObservationException` / `SessionLogException`）。

### 4.5 权限引擎（`core/permission/PermissionEngine.java:47`）

三张规则表 allow / deny / ask。求值顺序（`:30-41`）：deny → ask → **工具自身 `ToolBase#checkPermissions`（bypass 免疫）** → allow → `PermissionMode.BYPASS` → 默认 ASK（`DONT_ASK` 下转 DENY）。ASK 决策会附带 `tool.generateSuggestions(input)`（`:152`）。

### 4.6 MCP（`core/tool/McpClientManager.java`，20.97 KB）

`registerMcpClient(wrapper[, enableTools][, disableTools][, groupName][, presetParameters][, toolNamePrefix])`（`:73-159`）：支持工具白/黑名单、分组、**预设参数（如 API key 不外泄给模型）**、工具名前缀。Harness 侧另有声明式注册：`harness/agent/tools/McpServerRegistrar.java` + `ToolsConfigLoader.java`，由 `workspace/tools.json` 驱动。

---

## 5. 状态、记忆与上下文工程

### 5.1 三层状态

`core/state/AgentState.java:57`：`session_id` / `user_id` / `summary` / `context` / `reply_id` / `cur_iter` / `shutdown_interrupted` / `permission_context` / `tool_context` / `tasks_context` / `plan_mode_context`。子状态：`ToolContextState` / `TaskContextState` / `PlanModeContextState` / `PermissionContextState`。

持久化：`AgentStateStore` + `InMemoryAgentStateStore` + `JsonFileAgentStateStore`；**乐观并发** `saveIfVersion` + `ConflictPolicy`；ReActAgent 侧维护 `slotVersions` 与 `stateConflictCount`（`:333-340`），CAS 保存 `persistAgentStateCas`（`:608`）。

> v1 的 `Memory` 接口（`core/memory/Memory.java:35`）已 `@Deprecated(forRemoval=true, since="2.0.0")`，javadoc 明确"Conversation context is now held on `AgentState.getContext()`"。

### 5.2 Harness 双层长期记忆（`harness/agent/memory/MemoryConfig.java:52`）

三级流水线（javadoc `:30-43`）：

1. **Flush**：从对话窗口抽取长期事实 → `memory/YYYY-MM-DD.md`（只追加）。`MemoryFlushManager` + `MemoryFlushMiddleware`。
2. **Consolidation**：周期性把 daily ledger 合并进 `MEMORY.md`（默认 `consolidationMaxTokens=4000`、`consolidationMinGap=30min`、`dailyFileRetentionDays=90`）。
3. **Compaction summary**：对话前缀蒸馏成一条 summary（走 `CompactionConfig` 的 prompt）。

触发策略 `FlushTrigger`：`ALWAYS` / `NEVER` / `THROTTLED(minGap)`。

工作区常量（`harness/agent/workspace/WorkspaceConstants.java:23-37`）：`AGENTS.md`、`MEMORY.md`、`tools.json`、`memory/`、`skills/`、`knowledge/` + `KNOWLEDGE.md`、`rules/`、`agents/`、`sessions/`、`tasks/`，默认根 `.agentscope/workspace`。

### 5.3 上下文工程：`HarnessContextBuilder`（`harness/agent/context/HarnessContextBuilder.java:53`）

"Single final request compiler. Instance is stateless and safe to share between sessions."

- `prepare(...)`（`:109-154`）：每次模型调用前编译最终请求，产出 `ContextManifest` 写入 RC；**失败也产出 `validation="build_failed"` 的 manifest**。
- 可插拔来源 `ContextSources`（`context/ContextSources.java:30-149`）：`withSource(id, source, options)` / `withInstruction(id, content)` / `withTaskContext(...)`；每个来源可配 `timeout` 与 `SourceFailurePolicy.OMIT`（且 required block 不允许来自 OMIT 源）。

### 5.4 压缩与大结果卸载

- `CompactionMiddleware`（`harness/agent/middleware/CompactionMiddleware.java:65`）：`onReasoning` 阶段触发。流程：flush 长期记忆 → **原生 session log 保留完整对话** → LLM 蒸馏前缀 → 替换 `AgentState.context` → 重建 `ReasoningInput`。动态阈值 = `model.getContextWindowSize() - reserved`，模型不报窗口则回退 `FALLBACK_TRIGGER_TOKENS`。另有 `overflowRecovery()` 在同一执行内重试上下文溢出。
- `ToolResultEvictionMiddleware`：>80K 字符的工具结果落盘 + 占位符。

---

## 6. 多智能体编排

### 6.1 重要发现：Pipeline / MsgHub 已被删除

`docs/v2/zh/docs/change-log.md:42`：

> `io.agentscope.core.pipeline.*`（`Pipeline`、`Pipelines`、`SequentialPipeline`、`FanoutPipeline`、`MsgHub`）→ middleware + 子 agent + event stream

即 **Java 2.0 没有 Python 版 `TeamPipeline` / `GoalPipeline` / `MsgHub` 的等价物**，改为「子 agent + 团队 + 事件流 + middleware」的运行时编排。这是两版最大的编排差异。

### 6.2 子 Agent（`harness/agent/subagent/`，29 个文件）

- 声明式：`SubagentDeclaration`（26.31 KB）、`AgentSpecLoader`（从 `workspace/subagents/*.md` 加载）。
- 执行：`DefaultAgentManager`、`SubagentFactory`、`RemoteSubagentStub`。
- 远程协议：`protocol/RemoteEventCodec`、`RemoteAgentEvent`、`RemotePendingConfirm`。
- 任务：`WorkspaceTaskRepository`（45.41 KB）、`AgentProtocolTaskClient`（Agent Protocol / SSE）。
- 中间件 `SubagentsMiddleware`（`middleware/SubagentsMiddleware.java:77`，35.87 KB）：暴露 `agent_spawn` / `agent_send` / `agent_list` + `task_output` / `wait_async_results` / `task_cancel` / `task_list`；每次 `onAgent` 从工作区文件系统重载声明，支持 per-user 隔离。
- 子 agent 上下文 prompt（`HarnessAgentBuilderSupport.java:85-113`）：明确"你是叶子 worker，不能再 spawn 子 agent"。

### 6.3 团队（`harness/agent/team/`）

`TeamsMiddleware`（`middleware/TeamsMiddleware.java:57`）：注入 AgentTeams 上下文、暴露 `TeamTool`；通过 `BY_SESSION` / `BY_MEMBER` 两个静态表把 TeamEvent 路由到会话，并用 `TeamWakeups` 支持按成员名唤醒。

### 6.4 消息总线与异步工具

`harness/agent/bus/`：`MessageBus`、`WorkspaceMessageBus`、`AsyncToolRegistry`。`InboxMiddleware` 每轮 reasoning 前排空会话 inbox；`AsyncToolMiddleware` 把超时工具转后台，结果经 inbox 回投。

---

## 7. 【重点】agentscope-harness 模块

### 7.1 定位

POM 描述（`agentscope-harness/pom.xml:31`）："Enhanced agent runtime with workspace context, subagent orchestration, memory management, and backend abstraction"。

README_zh.md:186 一句话最精炼：

> **HarnessAgent** 在 ReActAgent 之上通过 Middleware 与 Toolkit 叠加工程基础设施，**核心推理循环原样保留**，能力按需叠加。

### 7.2 `HarnessAgent`（`harness/agent/HarnessAgent.java:191`，143 KB）

```java
public class HarnessAgent implements Agent, AutoCloseable {
    private final ReActAgent delegate;   // :195  组合而非继承
}
```

所有 `call` / `streamEvents` 委托给 `delegate`，外层用 `wrappedCall` / `wrappedStream` 包沙箱生命周期（`:801-989`）。**无状态 & 并发**（`:186-189`）："stateless between calls and safe to use as a singleton serving multiple users/sessions concurrently"。

叠加的 8 项能力（javadoc `:173-184`）：workspace 上下文加载（AGENTS.md / MEMORY.md / KNOWLEDGE.md）、可插拔文件系统、子 agent 编排、技能加载 + 自学习闭环、记忆 flush + 压缩前消息卸载、`tools.json`（MCP + allow/deny 过滤）、Plan Mode、上下文溢出紧急压缩。

### 7.3 20 个 Harness 中间件（全实现 `HarnessRuntimeMiddleware`）

| 中间件 | 职责 |
|---|---|
| `WorkspaceContextMiddleware` | 注入 AGENTS.md / MEMORY.md / KNOWLEDGE.md |
| `SubagentsMiddleware` | 子 agent 工具 + 后台任务摘要注入 |
| `TeamsMiddleware` | AgentTeams 上下文 + `TeamTool` + 唤醒桥接 |
| `CompactionMiddleware` | 对话压缩（onReasoning） |
| `ToolResultEvictionMiddleware` | 大工具结果落盘 |
| `MemoryFlushMiddleware` / `MemoryMaintenanceMiddleware` | 长期记忆抽取 / 合并维护 |
| `HarnessSkillMiddleware` / `SkillUsageMiddleware` / `SkillCuratorMiddleware` | 动态技能加载 / 用量统计 / 后台策展 |
| `PlanModeMiddleware` | 只读规划态强制 |
| `SandboxLifecycleMiddleware` | 沙箱 acquire/release |
| `InboxMiddleware` / `AsyncToolMiddleware` | inbox 排空 / 超时工具转后台 |
| `AtPathExpansionMiddleware` / `ContextConventionsMiddleware` | `@path` 展开 / 上下文约定 |
| `AgentTraceMiddleware` | 执行追踪日志 |
| `DynamicSubagentsMiddleware` | 动态子 agent |

> 注意：`.middleware(...)` 注册的**自定义中间件跑在所有 Harness 内置中间件之前**（`docs/v2/zh/docs/harness/architecture.md:61-67`）。

### 7.4 沙箱（`harness/agent/sandbox/`，50 个文件）

契约：`Sandbox` / `SandboxClient` / `SandboxManager` / `SandboxLease` / `SandboxIsolationKey` / `SandboxExecutionGuard`（分布式锁）/ `SessionSandboxStateStore`。工作区投影：`WorkspaceSpec` / `WorkspaceProjectionApplier` / `layout/`（BindMount / Dir / File / GitRepo / LocalDir / LocalFile）。快照：`snapshot/`（Local / Remote / `OssSnapshotSpec`）。实现：`impl/docker/DockerSandbox`（30.52 KB）；云沙箱扩展 `{agentrun, daytona, e2b, kubernetes}`。

隔离粒度 `IsolationScope`（`harness/agent/IsolationScope.java:53-126`）：`SESSION` / `USER`(默认) / `AGENT` / `GLOBAL`；既是沙箱状态 key，也是远端文件系统命名空间前缀。

### 7.5 文件系统（`harness/agent/filesystem/`）

`AbstractFilesystem` / `CompositeFilesystem`（23.17 KB）/ `OverlayFilesystem` / `ProjectAwareOverlay` / `RoutedSandboxFilesystem` / `BakedContextFilesystem` + `local/` `remote/` `sandbox/`。

`WorkspaceManager`（`harness/agent/workspace/WorkspaceManager.java:97`）双层读架构（`:66-79`）：**读**先查 `AbstractFilesystem`，非空则用，否则回退本地磁盘；**写**全部走 `AbstractFilesystem`；**列举**取并集去重。

### 7.6 分布式（`harness/agent/DistributedStore.java:74`）

一站式接口：`agentStateStore()` / `baseStore()` / `sandboxSnapshotSpec()` / `sandboxExecutionGuard()` / `messageBus()` / `asyncToolRegistry()`。实现由扩展提供：Redis / Jdbc / Mongo / Oss / Cos。

---

## 8. 双事件系统（Java 版最容易被忽略的设计）

Java 版有**两套**事件机制，这是它与 Python 版、与 DeepSeek harness 的关键差异。

### 8.1 运行时事件流（给前端渲染 / HITL）

`core/event/AgentEventType.java:40-91`：**32 个**类型常量，每个带 `@JsonAlias` 兼容旧名，`fromValue` 兜底 legacy 映射（`:114-137`）。对应 32 个 `AgentEvent` 子类：`AgentStart/End/Result`、`ModelCallStart/End`、Block（Text/Thinking/Data）Start/Delta/End、`ToolCall*`、`ToolResult*`、`ExceedMaxIters`、`RequireUserConfirm`、`RequireExternalExecution`、`UserConfirmResult`、`ExternalExecutionResult`、`RequestStop`、`SubagentExposed`、`HintBlock`、`AllToolsDenied`、`Custom`。

### 8.2 持久化事件日志（真正的 event sourcing）

`core/session/SessionLog.java:22`——**阻塞式持久化 SPI**（注释：runtime integration schedules it on boundedElastic）：

```java
Head(seq, epoch, owner, leaseUntil, commitId)                 // :29
Writer(owner, epoch)                                          // :31
acquire(owner, lease) / renew / release / sealWriter           // :70-83  带租约的写入者选举（≈ 分布式锁/fencing）
commit(writer, batchId, expectedSeq, events)                  // :85      乐观并发 CAS
readAfter(after, limit) / scan(after, through)                 // :87/:42  stable-prefix scan
inbox()                                                        // :24-27   持久化输入日志，独立于执行写入者
```

`core/session/SessionEventTypes.java:24-82`：**57 种**事件名，含 `run/*`、`turn/*`、`step/*`、`input/*`、`inbox/*`、`message/*`、`context/*`、`request/*`、`model/*`、`tool/*`、`action/*`、`interaction/*`、`compaction/*`、`task/*`、`plan/*`、`permission/*`、`verification/*`、`subagent/*`、`recovery/*`、`migration/*`、`presentation/*`、`state/checkpoint`、`state/restored`。

支撑类：`JournalSessionLog`、`SessionRecorder`、`SessionProjection`、`SessionRecovery`、`SessionInbox`、`SessionViews`、`SessionTranscriptExport`。

---

## 9. agentscope-service：Agent-as-a-Service 四平面

| 平面 | 职责 |
|---|---|
| **Gateway** | 公共入口、认证、API 路由；不持有业务状态与 Agent 执行 |
| **Control Plane**（Go 实现） | Agent 目录、绑定、发布、Invocation、公共事件、任务协调 |
| **Dataplane** | Managed Harness Runtime、事件日志、**SSE**、Turn Lease、HITL、Work Queue |
| **Scheduler** | Channel、Cron、外发任务、自托管 Hands Workers |

技术栈：Spring Boot **4.0.4** + Spring Cloud 2025.1.2；dataplane 用 `spring-boot-starter-webflux`。端口：Gateway 18080 / Control Plane 8081 / Dataplane 8082 / Scheduler 8083。

**三种执行模型**：`managed`（Service 托管 Harness）、`external-application`（接入已有 AgentScope / LangChain / ADK / Claude Agent SDK 应用，ASDP 协议）、`hosted-runtime`（复用 Codex / Claude Code 等 Coding Agent）。

服务端能力：会话与运行控制器（`AgentScopeContractController`、`DataSessionApiController`、`AgentSessionTurnsController`、`AgentSessionEventsController`、`AgentSessionCheckpointsController`、`AgentSessionArtifactsController`、`AgentSessionBudgetController`、`AgentSessionWebhooksController` …）、SSE 流式、分布式部署（Helm Chart / Docker Compose）、多租户与权限（`AgentAclService`、`JwtService`、`VaultCredentialDto`）、Agent 装配（`HarnessAgentBuildService`，39.62 KB）。

---

## 10. 工程特性

- **响应式**：Project Reactor（BOM `2025.0.2`）；`Mono<Msg>` / `Flux<AgentEvent>` / `Flux<ChatResponse>`；默认 `Schedulers.boundedElastic()`；全链路保留 `doOnCancel`、`Mono.usingWhen`。`AGENTS.md:75` 禁止阻塞响应式事件循环线程。
- **序列化**：Jackson 2.21.1；`AgentEventType` 的 `@JsonAlias` 兼容；victools 生成 schema + networknt 校验。版本资源用两个互斥的 resource block 避免 unfiltered 覆盖 filtered，并加 antrun 断言文件中不含 `$`。
- **可观测性**：OpenTelemetry 1.66.0 + **`opentelemetry-reactor-3.1` instrumentation**（把 Reactor 链路纳入 trace context）；`OtelTracingMiddleware`；`StateStoreActionObserver`；`AgentTraceMiddleware`（默认开启）。
- **Spring 集成**：13 个 `agentscope-spring-boot-starters`（a2a / admin / agui / anthropic / chat-completions-web / dashscope / gemini / jev / nacos / ollama / openai / openai-official）；`AguiWebFluxHandler`（SSE）、`ChatCompletionsController`（OpenAI 兼容 `/v1/chat/completions`）、`A2aJsonRpcController`。
- **优雅停机**：`shutdown/`（`GracefulShutdownManager`、`GracefulShutdownMiddleware`、`PartialReasoningPolicy`）；停机中断后重试自动去重输入。
- **Bash 安全**：tree-sitter-bash 做 **AST 级命令安全分析**。
- **测试**：JUnit 5（6.0.2）+ Mockito + **reactor-test StepVerifier** + testcontainers；core 发布 test-jar 供 harness 复用 `MockModel` / `MockToolkit`；JaCoCo 覆盖率。

---

## 11. 与 Python 版特性对比矩阵

### 11.1 已对齐

ReAct 循环、细粒度事件流（32 种，明确"与 Python 2.0 对齐"）、结构化输出（原生 + 合成工具双路径）、工具 + JSON Schema（**Draft 2020-12 强校验**）、并行工具调用、MCP、HITL 一等公民、技能体系、计划（Plan Mode）、子 agent、多模态（`DataBlock` 统一 base64/URL）、多模型厂商（含 DeepSeek/GLM/Kimi/MiniMax 兼容层）。

### 11.2 Java 领先

| 能力 | 依据 |
|---|---|
| Agent 完全无状态 + 显式多租户 | `ReActAgent.java:235-238`、`change-log.md:126-139` |
| **持久化事件溯源日志**（租约 + epoch + fencing + CAS commit + 独立 inbox 日志） | `SessionLog.java:22-88`、`SessionEventTypes.java:24-82` |
| 三态权限引擎 + 敏感路径 + 建议规则 | `PermissionEngine.java:30-178` |
| 分布式会话恢复（Redis/MySQL/PG/OSS/COS/Mongo） | `DistributedStore.java:62-72` |
| 多形态沙箱矩阵 + 快照 + 分布式执行守卫 | `sandbox/` 50 文件 |
| Agent-as-a-Service 四平面平台 | `agentscope-service/` |
| A2A / AG-UI / Agent Protocol / OpenAI 兼容端点 | extensions-protocol、starters |
| Spring Boot 4 starter 生态（13 个） | extensions/spring-boot-starters |
| Bash 命令 AST 级安全分析 | core pom 120-130 |
| 乐观并发状态保存 + 冲突策略 | `ConflictPolicy`、`persistAgentStateCas` |
| 工具结果落盘卸载（>80K 字符） | `ToolResultEvictionMiddleware` |

### 11.3 Java 缺失 / 落后（选型时必须知道）

| 能力 | 状态 | 证据 |
|---|---|---|
| **Pipeline / MsgHub / Fanout** | **已删除，无等价物** | `change-log.md:42` |
| **RAG / Knowledge（v2 架构）** | **推进中**；v1 全部 `@Deprecated(forRemoval=true)`，javadoc 写"integrate retrieval at the application layer" | `Knowledge.java:30-33`、`change-log.md:238-242` |
| **长期记忆（core 层）** | **推进中**；`LongTermMemory` 等全废弃，实际实现在 harness 与 extensions-mem | `Memory.java:30-34`、`change-log.md:244-248` |
| **TTS / Realtime** | **已删除** 14 个文件 | `change-log.md:43,47` |
| 异步/远程子 agent 事件流 | **未实现**（`EventSource.taskId` 为保留字段） | `EventSource.java:51-53,70-84` |
| `AgentEvent` 中子 agent 来源通道 | **v2 roadmap**，当前只用 v1 `EventSource` | `EventSource.java:104-109` |

---

## 12. 关键文件速查

```
core/agent/Agent.java:47                    CallableAgent:36  StreamableAgent:44(deprecated)
core/agent/AgentBase.java:92                108(ID正则) 113-119(串行门) 285-304(Context keys) 313-401(生命周期)
core/ReActAgent.java:241                    248(generate_response) 325-347(状态分桶) 2170(doCallInner)
                                           2741(coreAgent) 2755(settleSession) 2790(reasoning) 2956(postReasoning)
                                           3055(reasoningStream) 3455(acting) 3602(actingStream) 4555(summaryStream)
core/state/AgentState.java:57
core/event/AgentEventType.java:40-91        32 类
core/session/SessionLog.java:22-88          租约/CAS/inbox
core/session/SessionEventTypes.java:24-82   57 类
core/model/Model.java:22                    ModelRegistry:44  ModelProvider SPI:22
core/formatter/Formatter.java:55
core/tool/Toolkit.java:70                   Tool.java:60-237  ToolValidator.java:49  ToolExecutor.java:466-494
core/permission/PermissionEngine.java:47
harness/agent/HarnessAgent.java:191        195(delegate) 801-989(call/stream) 1662-2438(Builder)
harness/agent/context/HarnessContextBuilder.java:53
harness/agent/memory/MemoryConfig.java:52
harness/agent/middleware/CompactionMiddleware.java:65
harness/agent/workspace/WorkspaceManager.java:97
harness/agent/DistributedStore.java:74
harness/agent/IsolationScope.java:53
docs/v2/zh/docs/change-log.md              27-260 迁移与废弃清单
docs/v2/zh/docs/harness/architecture.md    27-59 能力矩阵与三层状态
```

**阅读路径建议**：`ReActAgent.java`（332 KB）/`AgentSpawnTool.java`（87 KB）/`HarnessAgent.java`（143 KB）是三个巨类，理解成本最高，建议按上表行号导航，先读 `doCallInner → reasoning → acting` 主干，再按模块深入 harness 中间件。

---

> 下一篇：[03-DeepSeek-Harness-原理对照与落地应用](03-DeepSeek-Harness-原理对照与落地应用.md)
