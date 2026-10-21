# 01 · AgentScope Python 版源码原理

> **TL;DR（3 分钟版）**
> - 2.x 是彻底重构：1.x 的 `AgentBase`/`ReActAgent`/`UserAgent`、`memory`、`session`、`tracing`、`evaluate`、`MsgHub` **全部不存在或改名**（§0 有对照表，先看它，否则读代码会对不上号）。
> - 核心只有三样：**单个 `Agent` 类**（行为差异全走 4 个配置对象）+ **纯函数状态机 `_next_action`**（Reasoning/Acting/Exit 三态）+ **7 钩子中间件**（长期记忆、RAG、Tracing 全是中间件，核心类零改动）。
> - 灵魂设计是**停泊-恢复**：`reply_stream` 的入参可以是"确认结果/外部执行结果/打断事件/None"，一次 reply 跨多次调用，靠 `reply_id` 路由——这是 HITL、跨进程恢复、跨天任务的基础。
> - 面向"LLM 会犯错"做了大量防御（容错 JSON 解析、结构化输出四策略降级、空流保护、压缩渐进截断），是这套代码最值得学的工程品味。
> - **短板**：无持久化事件域（无 fork / 投影式压缩 / 逐字节重建校验）；async-only；无 evaluate 模块。

> 源码路径：`agent_base/agentscope`
> **行号基线**：版本 `2.0.11dev`（`src/agentscope/_version.py:4`），`main` 分支 **commit `402c3664`**（2026-10-19）。
> 本文所有行号基于该 commit，路径前缀统一省略为 `src/agentscope/`。
> **如何验证本文结论**：`cd agent_base/agentscope && git checkout 402c3664`，然后用 `sed -n '<行号>p' <文件>` 逐条核对；升级后行号会漂移，请以本文给出的"类名/方法名"为锚重新定位。
> 姊妹篇：[02-AgentScope-Java-源码原理](02-AgentScope-Java-源码原理.md)

## 0. 阅读须知：先纠偏 1.x 概念

AgentScope 2.x 相对 1.x 是**彻底重构**。如果带着 1.x（或 Java 1.x）的概念读代码会完全对不上号，务必先看这张对照表：

| 1.x 概念 | 2.x 实际情况 |
|---|---|
| `AgentBase` / `ReActAgent` / `UserAgent` 多个子类 | **统一收敛为单个 `Agent` 类**（`agent/_agent.py:117`），ReAct 循环内置，行为差异全部由配置对象表达 |
| `agentscope.memory` | ❌ 不存在。短期记忆 = `AgentState.context`；长期记忆 = 中间件 `middleware/_longterm_memory/` |
| `agentscope.session` | ❌ 不存在。拆为 `state/`（可序列化状态）+ `app/storage/`（服务端持久化） |
| `agentscope.tracing` | ❌ 不存在。改为中间件 `middleware/_tracing/`（OTel） |
| `agentscope.evaluate` | ❌ 不存在。`docs/roadmap.md:110-112` 明确仍在规划 |
| `KnowledgeBank` | 改名 `KnowledgeBase`（`rag/_knowledge.py:44`） |
| `sequential_pipeline` / `fanout_pipeline` / `MsgHub` | ❌ 不存在。改为 `TeamPipeline` / `GoalPipeline` / `SOPEngine` |
| `Model` 基类 | 改名 `ChatModelBase`（`model/_base.py:37`） |

2.x **新增**的模块：`app/`（FastAPI 服务）、`workspace/`（沙箱）、`realtime/`（语音）、`sop/`、`skill/`、`permission/`、`classifier/`、`event/`、`middleware/`、`state/`。

**一句话定性**：2.x 是一次「配置优于继承、中间件优于钩子、事件流优于同步调用」的重构，核心只有一个 `Agent` 类 + 一个纯函数状态机 + 一套中间件。

---

## 1. 项目事实与目录地图

### 1.1 分层结构

```
src/agentscope/（30 个顶层包）
├─ 核心层
│  ├─ agent/        _agent.py（152KB，全局最核心）· _config.py（4 个 Pydantic 配置类）· _realtime/
│  ├─ message/     Msg + 7 种 ContentBlock（多模态载体）
│  ├─ event/       32 个流式事件类型——Agent 与外界通信的唯一契约
│  ├─ state/       AgentState（短期记忆 + 权限/工具/任务上下文，可整体持久化）
│  └─ types/       ReplyFinishedReason · ErrorInfo · JSONSerializableObject
├─ 能力层
│  ├─ model/       ChatModelBase + 12 个厂商实现
│  ├─ formatter/   FormatterBase + 12 厂商各一套 Chat/MultiAgent（13 个模块、约 30 个类）
│  ├─ tool/        Toolkit · ToolBase · FunctionTool/MCPTool · _builtin/ · _task/
│  ├─ rag/         KnowledgeBase + _parser/ + _chunker/ + _vdb/
│  ├─ embedding/   EmbeddingModelBase + 4 厂商 + FileEmbeddingCache
│  ├─ mcp/         MCPClient（stateful / stateless 双模）
│  ├─ permission/  PermissionEngine + 5 种模式
│  ├─ middleware/  7 个钩子的切面系统
│  ├─ skill/       Agent Skills（Markdown + 资源目录）
│  └─ classifier/  内容安全分类器（OpenAI / JEV）
├─ 编排层
│  ├─ pipeline/    PipelineProtocol · TeamPipeline · GoalPipeline
│  └─ sop/         SOPEngine（带验证与重试预算的工作流）
└─ 运行时/外围
   ├─ app/         FastAPI 服务（18 个 router · storage · message_bus · channel · hub · scheduler）
   ├─ workspace/   8 种沙箱后端 + Offloader 协议
   ├─ console/ tui/ asr/ tts/ realtime/ credential/ _utils/ exception/
```

### 1.2 依赖设计：轻内核 + 插件化

`pyproject.toml:53-194`：核心依赖**只有 28 个**，其余全部走 optional extras（`model-gemini`、`vdb-qdrant`、`workspace-docker`、`memory-mem0`、`service`…）。配合惰性导入（provider 的重试异常在方法内 `import openai`；`AsyncSQLAlchemyStorage` 用模块级 `__getattr__` 惰性加载），实现"装什么有什么、不装不报错"。

---

## 2. 核心抽象：Agent / Msg / AgentState / Event

### 2.1 Agent：单一类承载全部生命周期

`agent/_agent.py:117`

```python
class Agent:
    def __init__(self, name, system_prompt, model, toolkit=None,
                 middlewares=None, state=None, offloader=None,
                 model_config=None, context_config=None,
                 react_config=None, injection_config=None): ...
```

**构造函数里最关键的一件事**（`_agent.py:218-240`）：把 middleware **按实现的钩子预先分桶**，运行时零反射：

```python
self._reply_middlewares = [_ for _ in middlewares if _.is_implemented("on_reply")]
self._reasoning_middlewares = [_ for _ in middlewares if _.is_implemented("on_reasoning")]
self._check_permission_middlewares = ...
self._acting_middlewares / _model_call_middlewares / _system_prompt_middlewares / _compress_context_middlewares
```

`is_implemented` 的实现很巧妙（`middleware/_base.py:55-66`）：直接比较 `getattr(MiddlewareBase, hook)` 与 `getattr(type(self), hook)` 的**对象身份**；未重写的基类方法会 `raise RuntimeError`（`_base.py:96-99`），因此"是否实现"在静态上就是可判定的。

### 2.2 生命周期：三个公开方法

| 方法 | 位置 | 语义 |
|---|---|---|
| `reply_stream(...)` | `_agent.py:288` | 主入口，`AsyncGenerator[AgentEvent \| Msg, None]` |
| `reply(...)` | `_agent.py:332` | 消费掉所有事件，只返回最终 `Msg` |
| `observe(msgs)` | `_agent.py:381` | 只把外部消息写入 context，**不触发推理** |
| `compress_context(...)` | `_agent.py:386` | 手动触发上下文压缩 |

注意 2.x **没有 `print()` 方法**：1.x 的 `agent.print()` 被「事件流 + 渲染器」取代（`console/_console.py:105` 的 `launch_console`、`tui/_launcher.py` 的 `launch_tui`）。这是核心与表现层彻底解耦的直接体现——`Agent` 完全不知道输出是什么。

### 2.3 停泊-恢复：`reply_stream` 的联合类型入参

这是 2.x 最重要的接口设计（`_agent.py:289-295`）：

```python
inputs: Msg | list[Msg]
      | UserConfirmResultEvent        # 续：用户确认结果（HITL）
      | UserInterruptEvent            # 续：用户打断
      | ExternalExecutionResultEvent  # 续：外部执行结果
      | None                          # 续：从上一次状态断点续跑
```

**一次 reply 可以跨多次调用**：agent 可以"停泊（park）"在等待人工确认或外部执行的状态，`reply_stream` 直接返回；外部拿到结果后再调用一次，用 `reply_id` 路由回去继续。这是整个框架异步性、可恢复性、跨进程能力的基石。

### 2.4 `_reply_impl`：ReAct 主循环（`_agent.py:1035`）

1. **Step 1** `_check_incoming_event(event)`（`_agent.py:1098`）：判断是否处于"等待外部事件"状态。
2. **Step 2**（`:1105-1122`）分叉：`is_awaiting` → `_handle_incoming_event()` 落盘结果；否则 `_handle_incoming_messages()` + 新建 `ReplyContext`（新 `reply_id`、`cur_iter=0`）+ yield `ReplyStartEvent`。
3. **结构化输出工具挂载**（`:1126-1132`）：先 `remove_tool(_GenerateStructuredOutput.name)`，若本次 reply 要求结构化输出再 `add_tool`——保证 schema 每次都是最新的。
4. **Step 3**（`:1142`）无限循环，核心状态机：

```python
while True:
    next_action = self._next_action(final_msg)      # 纯函数，只读
    match next_action:
        case Exit(exit_msg=..., exit_events=...):   # :1149
        case Reasoning(hint=..., tool_choice=...): # :1178
        case Acting(tool_calls=...):                # :1214
```

**中间件可以吞掉 `ReplyEndEvent` 强制再来一轮**（`:1141`, `:1159-1176`）是这里最精巧的设计：

```python
made_progress = True
...
if self._receive_reply_end:
    yield exit_msg; return
if not made_progress:
    raise RuntimeError("A middleware swallowed the ReplyEndEvent twice without any reasoning/acting in between...")
made_progress = False
final_msg = None
continue
```

即中间件可截获终止事件不放行，强制 agent 再走一轮 ReAct（例如"还没拿到结构化输出，再试一次"），框架用 `made_progress` 检测忙循环。

### 2.5 `_next_action`：纯函数状态机（`_agent.py:3495`）

返回 `Acting` / `Reasoning` / `Exit` 三者之一（`agent/_utils.py:26-43`）。docstring 明写 "Read-only: all side effects are performed by the caller"（`:3499`）。判定顺序：

1. **有待执行的工具调用？**（`:3507-3529`）→ `Acting`。`:3515-3525` 的过滤逻辑：已有 result 的跳过；`PENDING` 的只有在**没有** awaiting 调用时才执行（避免并发批次里被去重挂起的调用抢跑）。
2. **有 awaiting（ASKING/SUBMITTED）调用？**（`:3531-3543`）→ `Exit(exit_events=None)`，即**停泊**。
3. **结构化输出**：`required and satisfied` → `Exit(COMPLETED)`（`:3553`）；`required and not satisfied` → 继续 `Reasoning` 并注入 hint（`:3575-3648`）；超过 `max_iters + structured_output_grace_iters` → `Exit(EXCEED_MAX_ITERS)`。
4. **有 final_msg** → `Exit`（`:3655-3697`），按 `cur_iter > max_iters` 判定 `COMPLETED` 还是 `EXCEED_MAX_ITERS`。
5. **`cur_iter == max_iters`** → 注入"最后一次总结机会"的 hint 并 `tool_choice=ToolChoice(mode="none")`（`:3701-3715`）——**优雅降级**：不是硬截断，而是给模型一次不带工具地收尾的机会。
6. 默认 → `Reasoning()`。

### 2.6 `_reasoning_impl`：模型调用与事件转换（`_agent.py:1698`）

- yield `ModelCallStartEvent`（`:1721`）→ `_prepare_model_input()` → `_call_model()`
- 用 `block_ids` 字典做**流式块状态追踪**（`:1736-1741`），把每个 chunk 转成 `TextBlock*Event` / `ThinkingBlock*Event` / `ToolCall*Event` / `DataBlock*Event`
- 结束时补发所有未闭合块的 `EndEvent`（`:1768-1787`）
- **空流保护**（`:1790-1796`）：未收到 `is_last=True` 抛 `RuntimeError`
- **纯 thinking 不算终态**（`:1819-1822`）：避免 reasoning 模型只输出 thinking 就被当成最终答复

### 2.7 消息模型 `Msg`：块化的多模态

`message/_base.py:70`

```python
class Msg(BaseModel):
    name: str
    content: list[ContentBlock]
    role: Literal["user", "assistant", "system"]
    id / metadata / created_at / usage
    finished_at / finished_reason / structured_output / error
```

7 种 `ContentBlock`（`message/_block.py:219-235`）：`TextBlock` / `ThinkingBlock` / `HintBlock` / `ToolCallBlock` / `ToolResultBlock` / `DataBlock`。多模态在 `DataBlock.source`（`_block.py:90`）：`Base64Source` 或 `URLSource`。

**`Msg.append_event()`（`_base.py:243`）是事件流 → 消息的"物化"函数**，是服务端 SSE 重建的关键。三处工程细节：

- **base64 分段拼接**（`:339-351`）：每个 delta 是独立带 padding 的 base64，直接字符串拼接会损坏字节流，所以 decode → 拼 bytes → 再 encode。
- **`TOOL_RESULT_END` 顺带把配对的 `ToolCallBlock` 置为 `FINISHED`**（`:463-469`）：保证 SSE 重建的 `reply_msg` 与 `agent.state.context` 一致。
- **`USER_CONFIRM_RESULT` 只处理 `ASKING` 状态的调用**（`:487`）：丢弃迟到的、已被打断解决的陈旧结果。

### 2.8 状态：`AgentState`（`state/_state.py:209`）

整个对象是 Pydantic `BaseModel`，`model_dump()` / `model_validate()` 即可持久化：

```python
class AgentState(BaseModel):
    session_id: str
    summary: str | list[...] = ""              # 压缩后的长期摘要
    context: list[Msg] = []                     # 未压缩的对话
    reply_context: ReplyContext                 # reply_id / cur_iter / structured_schema / output
    permission_context: PermissionContext
    tool_context: ToolContext                   # 文件读缓存 LRU + activated_groups
    tasks_context: TaskContext
    middle_context: dict[str, Any]              # 给中间件跨 reply 存数据
```

- `append_context()`（`:298`）：**一次 reply 聚合为一条 assistant 消息**（`context[-1].id == reply_id` 就续写），让 reply 与消息一一对应，`reply_id` 即消息 id。
- `get_awaiting_tool_calls()`（`:345`）/ `has_awaiting_tool_calls`（`:328`）：停泊判定。
- **向后兼容迁移器**（`:226-241`）：旧版本把 `reply_id`/`cur_iter` 存在顶层，`_migrate_legacy_reply_fields` 在 `model_validator(mode="before")` 里搬进 `reply_context`，配合 property 转发——老存档能直接加载。
- `ToolContext.read_file_cache`（`:32-172`）：完整 LRU + 字节上限缓存（`max_cache_files=100`、`max_cache_bytes=25000`，mtime 判失效）。

### 2.9 长期记忆 = 中间件（三种实现）

三者都遵循同一模式：**`on_reply` 启动异步检索任务 → `on_reasoning` 消费结果**，记忆检索不阻塞主流程：

| 实现 | 位置 | 特点 |
|---|---|---|
| `AgenticMemoryMiddleware` | `middleware/_longterm_memory/_agentic_memory/_middleware.py:359` | **Markdown 文件即记忆**；`MEMORY.md` 注入系统提示（`:513`），用 LLM 做相关性选择 |
| `Mem0Middleware` | `_mem0/_middleware.py` | 对接 Mem0 服务 |
| `ReMeMiddleware` | `_reme/_middleware.py` | 对接 ReMe |

---

## 3. 中间件：唯一的扩展点

`middleware/_base.py:13`，7 个钩子（6 个洋葱 + 1 个变换器）：`on_reply` / `on_reasoning` / `on_acting` / `on_model_call` / `on_check_permission` / `on_system_prompt` / `on_compress_context`。

长期记忆、RAG、Tracing、TTS、模型路由、预算控制**全部**是中间件——核心类一行都不用改。这是 2.x 的灵魂设计之一。

---

## 4. 工具系统

### 4.1 `Toolkit`：唯一注册中心（`tool/_toolkit.py:66`）

- **工具组（ToolGroup）机制**（`tool/_tool_group.py:10`）：工具按组组织，`basic` 永远激活，其他组由 agent 调用 `ResetTools` 元工具动态激活/停用（lazy tool loading）。`check_tool_available`（`:556`）会区分"组未激活"和"根本不存在"两种错误。
- **MCP 不可达不拖垮对话**（`:524-537`）：`list_tools()` 失败只 `logger.warning` 跳过该 MCP。

### 4.2 Schema 与校验：三层防线

1. **容错解析** `_json_loads_with_repair`（`_utils/_common.py:95-212`）：先 `json.loads`；失败用 `json_repair` 带 schema 修复；拒绝 NaN/Infinity；拒绝"删字段式"修复（`:167-170`，因为 `additionalProperties:false` 下的删字段是改写而非类型修复）。失败时在错误消息里**附一段可执行的 Python 复现代码**给 LLM 看（`:200-212`）——非常 agent-friendly 的错误设计。
2. **显式校验**（`_agent.py:2492-2497`）：`jsonschema.validate` 失败 → `AgentOrientedException`。
3. **异常二分**（`exception/_base.py`）：`AgentOrientedException`（喂回 agent 自我纠正）vs `DeveloperOrientedException`（抛给开发者）。

`ToolChoice`（`tool/_types.py:178-203`）：`mode` 支持 `"auto"/"none"/"required"` 或**具体工具名**。文档明确建议用 `mode=<tool_name>` 而非 `tools=[...]`——后者会改变 schema 列表从而使 prompt cache 失效。

### 4.3 并行工具调用：按安全性自动分批

`_batch_tool_calls`（`_agent.py:2096-2134`）依据 `tool.is_concurrency_safe` 把调用切成连续的批次序列。两档执行器：

- `_execute_sequential_tool_calls`（`:2136`）：for 循环逐个，遇 HITL/INTERRUPTED 立即 break。
- `_execute_concurrent_tool_calls`（`:2186`）——全文件最讲究的实现：

```python
sentinel = object()
queue: Queue = Queue()
kept_rules: list[PermissionRule] = []     # 批次内确认去重共享累加器

async def _run_all():
    results = await asyncio.gather(*[self._into_queue(tc, queue, kept_rules) for tc in tool_calls],
                                   return_exceptions=True)
    await queue.put(sentinel)             # sentinel 在 gather 之后才入队
    return results
```

- **sentinel 保证事件流完整性**（`:2271-2273`）：gather 返回后才放哨兵，消费者收不到半截流。
- **失败不取消兄弟任务**（`:2262-2263`）：最后聚合成 `ExceptionGroup`。
- **取消时先 cancel 再 flush**（`:2285-2302`）：避免孤儿任务，同时把已入队事件冲刷给调用方后再 raise。
- **确认去重**（`:2246-2252`）：并发批次里前一个确认已覆盖本次调用则保持 `PENDING` 不再弹窗；但 **safety ASK 永不去重**（`:2539-2542`）。

### 4.4 单工具调用全生命周期（`_agent.py:2426`）

`check_tool_available` → 解析 + 校验 → `_check_permission`（中间件洋葱）→ 三分支：

- `ASK`/`PASSTHROUGH` → 置 `ASKING`，yield `RequireUserConfirmEvent`，**return**（停泊）
- `DENY` → `_handle_error_tool_call(state=DENIED)`
- `ALLOW` → 置 `ALLOWED`；若 `is_external_tool` 则置 `SUBMITTED` + yield `RequireExternalExecutionEvent` + return（注释 `:2601-2604`：状态必须在 yield 之前更新，因为外层收到事件会立即 break）

最后**工具结果截断 + 卸载**（`:2634-2686`）：超出部分若有 `offloader` 就写入工作区并追加 `<<<TRUNCATED>>>` + 文件路径提醒。

### 4.5 MCP 集成（`mcp/_mcp_client.py:33`）

- stateful（显式 connect/close，STDIO 强制）/ stateless（HTTP，每次临时建 session）双模。
- **工具命名净化**（`_adapters.py:240-253`）：`re.sub(r"[^a-zA-Z0-9_-]", "x", tool.name)`，用 `"x"` 而非 `"_"` 以避免与 `__` 分隔符冲突；最终名 `mcp__{mcp_name}__{sanitized_tool}`。
- `MCPTool`（`_adapters.py:195`）：完整保留 `$defs`/`anyOf`/`oneOf`（否则 `$ref` 无法解析）；`is_concurrency_safe = False`（保守默认）；read-only → ALLOW，否则 ASK。
- `_RESERVED_HEADERS`（`_mcp_client.py:84-86`）：禁止 `connection/content-length/host/transfer-encoding` 被客户端覆盖，带 RFC 7230 正则校验。

### 4.6 权限引擎（`permission/_engine.py:17`）

5 种模式 + `PermissionRule` + 每个工具自定义 `match_rule`/`generate_suggestions`（Bash 提 `git commit:*` 前缀，Read/Write 提 `src/**` glob）。`dangerous_files`/`dangerous_directories` 大小写不敏感 + realpath 解析符号链接（防止 `/tmp → /private/tmp` 绕过）。

---

## 5. 模型层与 Formatter

### 5.1 `ChatModelBase`（`model/_base.py:37`）

三个可覆写的能力声明钩子：`_get_retryable_exceptions()`（子类**惰性 import SDK 异常**）、`_get_disable_thinking_kwargs()`、`_get_structured_output_fallback_exceptions()`。`list_models()` 从子类源文件同级的 `_models/*.yaml` 加载 ModelCard。

`__call__`（`:182`）的重试 + 流归一化要点：

- **空 content 的 chunk 不对外暴露**（`:278-279`）：吸收 OpenAI 兼容 API 只带 usage 的尾包元数据，但不产生可见空 delta。
- `CancelledError` 时补发累积结果并置 `INTERRUPTED`（`:283-285`）。
- `finally` 里 await 底层 `aclose()`（`:289-293`），注释明确"要等 provider 清理完再返回"。

`count_tokens`（`:377`）默认启发式：`len(utf8_bytes) / 4`，每个 `DataBlock` 固定 2000 token。

### 5.2 结构化输出：四策略降级阶梯（`model/_base.py:465`）

```python
strategies = (
    ("forced",   {},              ToolChoice(mode="generate_structured_output")),  # 强制工具调用
    ("auto",     {},              ToolChoice(mode="auto")),
    *((("no_think", disable_kwargs, forced_tc),) if disable_kwargs else ()),       # 关思考+强制
    ("none",     {},              None),
)
```

每种策略内 `max_retries+1` 次重试；**可重试异常 → 重试同策略**（换策略对限流没用）；`StructuredOutputError` → **换下一个策略**。机制是构造一个名为 `generate_structured_output` 的假工具，把 `model_json_schema()` 作为其 parameters 强制模型调用（`:603-733`）。

### 5.3 重试层次（三层叠加）

SDK provider 级 → `ChatModelBase.__call__` → `Agent._call_model`（`_agent.py:3274`，外加 fallback 模型一级）。`ModelConfig.max_retries` 默认 0 正是为了避免两层重试叠加。

### 5.4 Formatter：Msg → 厂商 API dict

`FormatterBase`（`formatter/_formatter_base.py:20`）**是 `BaseModel`**（不是 ABC），因此天然可序列化/可配置。三个通用能力：

- `convert_tool_result_to_string`（`:105-187`）：**把多模态工具结果降级为文本**，用 block 自身 id 作标识符（保证重复格式化结果稳定、prompt cache 友好）；不支持的写成 `<<<TRUNCATED>>>` 式提醒。
- `_group_messages`（`:189-232`）：把消息流切成 `tool_sequence` / `agent_message` 两组——这是 Chat 与 MultiAgent 两种 formatter 分化的基础。

12 个厂商各一套 Chat/MultiAgent 两套（另有 OpenAI Response 变体与各厂商的 Converter/Merger 辅助类，`formatter/` 下共 13 个模块、约 30 个类）。以 OpenAI 为例：`OpenAIMultiAgentFormatter`（`_openai_formatter.py:427`）把多个 agent 的消息**折叠成一段 `<history>` 文本**（`f"{msg.name}: {block.text}"`，`:520`），工具序列仍委托 `OpenAIChatFormatter`。

---

## 6. 上下文工程（2.x 最重的工程投入之一）

### 6.1 `ContextConfig`（`agent/_config.py:51`）

`trigger_ratio=0.8`（上限 0.9）、`reserve_ratio=0.1`、`context_buffer_ratio=0.2`；关系校验在 `_validate_configs`（`_agent.py:246-282`）。

`compression_prompt`（`_config.py:82-114`）是一份很值得学习的 prompt：要求把相对时间转成绝对时间、把"这个文件/上面那个"改成绝对路径/符号名/PR 号、记录 in-flight 的后台工具及其 id——**因为摘要之后原文就没了**。

### 6.2 `_compress_context_impl`（`_agent.py:490`）的防御性设计

1. `_limit_context_images`（`:792`）先按 `max_image_num` 卸载旧图
2. 上下文为空却超阈值 → `RuntimeError`（系统提示本身太长，无法压缩，`:537-540`）
3. **reserve_ratio 过大导致无可压缩内容时的降级**（`:553-568`）：把 reserve 降到 0 再拆一次
4. **压缩请求本身超长时的渐进截断**（`:646-685`）：逐个丢弃最老的待压缩消息，直到估算低于阈值
5. 失败兜底：`compression_fallback_to_truncation=True` 时保留旧 summary
6. 中断保护（`:785-790`）：`asyncio.shield` + 补等，保证"summary 已生成但 context 尚未替换"的中间态不残留
7. **把压缩调用的 usage 记到 context 尾部**（`:760-778`）：压缩调用不产生 model call event，否则 token 账目会漏

### 6.3 运行时状态注入：`HintBlock` 而非改系统提示

`_inject_runtime_state`（`_agent.py:1365`）用 `HintBlock` 注入，**理由是保住 prompt cache**（`:1376-1378`）。注入是**持久化**的（让 agent 有时间感）。四个维度：时间（带 `<timezone>`）/ 未完成任务 / 上下文长度 / 重复工具错误（连续失败 `tool_retries_limit` 次则注入"停止重试"提醒，`_config.py:272-286`）。

### 6.4 Offloader：有限上下文 vs 完整信息的折中

`workspace/` 提供 `LocalWorkspace` + 8 个远端后端（Docker / E2B / Daytona / K8s / OpenSandbox / AppleContainer / Bubblewrap）。`Offloader` 协议把被截断的内容和旧上下文落到沙箱文件系统，并在提示里给出路径。

---

## 7. RAG（`rag/`）

三层数据模型：`原始文件 --Parser--> Section --Chunker--> Chunk --Embedding--> VectorRecord --> VectorStore`。

- `Section`（`rag/_document.py:31`）是解析器的"自然边界"单位（PDF 按页、PPTX 按 slide、Markdown 按顶层标题）。**`Chunker` 绝不跨 Section 合并**（`:35-37`），防止格式结构泄漏到下游。
- `KnowledgeBase`（`rag/_knowledge.py:44`）刻意做"薄"：类 docstring 写明它是"单一算法真相源"，解析/切分/凭据/维度策略/持久化全在上一层（服务端 `KnowledgeBaseManagerBase`）。
  - `ensure_collection()`（`:160`）幂等 + 记忆化，首次成功后不再多一次往返。
  - `search()`（`:190-273`）：非多模态模型时静默丢弃 `DataBlock` 查询；批量 embed 后并发检索；按 `(document_id, chunk_index)` 去重保留最高分（该组合键能 survive 重建索引，block UUID 不能）。
  - `insert_document()`（`:279-356`）**metadata 三级优先级**：`metadata_filter`（安全边界，最高）> chunk 自带 > `document_metadata`——恶意/有 bug 的 parser 无法把记录重新绑定到别的 scope。多租户隔离即由此实现。
- **检索如何接入 Agent**：通过 `RAGMiddleware`（`middleware/_rag.py:626`），两种模式：`agentic`（默认，给 agent 一个 `_SearchKnowledgeTool`，让 LLM 自己决定何时检索）/ `static`（每次输入都检索后注入 hint）。可选 `rerank_model` 做 LLM 重排，失败保留向量序。
- 向量库后端：Qdrant / MilvusLite / MongoDB / Elasticsearch。

---

## 8. 多智能体编排

### 8.1 `PipelineProtocol`（`pipeline/_base.py:15`）

```python
class PipelineProtocol(Protocol):
    def reply_stream(self, inputs: ...) -> AsyncGenerator[AgentEvent | Msg, None]:
```

注意（`:15-22`）：`reply_stream` 声明为**普通 `def` 返回 async generator** 而非 `async def`——因为 `async def` 在这里既不能被 `Agent` 满足，也不能被任何 pipeline 满足。因为 pipeline 与 agent 实现同一协议，pipeline 可直接插到 `launch_console` 和 `SOPEngine` 的 step 里。

### 8.2 `TeamPipeline`：Leader + 工具调用式派活（`_team_pipeline.py:110`）

给 leader 注册一个 `TeamAssign` **外部工具**（`_TeamAssign`，`:51`，`is_external_tool = True`，成员名是 schema 里的 enum）。leader 调用它 → agent 发 `RequireExternalExecutionEvent` → **pipeline 截获该事件**（`:254-258`）真正跑 member → 把 member 的最终答复作为 tool result 回灌给 leader。

调度语义：
- **同轮不同成员并发，同一成员串行**（`:296-299`）：`groups.setdefault(member, []).append(tool_call)`，每成员一个 task + `asyncio.gather`。
- 跨 HITL 暂停也保持顺序：成员停泊后 `run_group` 里 `break`（`:336-337`）。
- 结果按 leader 的调用顺序回填（`:373-375`）。
- `reset_members=True`（默认）：leader 的 reply 结束后清空成员 context。

### 8.3 `GoalPipeline`：执行者 + 验证者闭环（`_goal_pipeline.py:55`）

executor 带 `_ExecutionReport` 跑 → verifier 带 `_VerificationResult`（`pass`/`fail`/`impossible`）验证 → `fail` 则把 message 作为 feedback 喂回 executor。**迭代预算放在实例上**（`self._iters`），注释明确"so a HITL resume does not restart the budget"。

### 8.4 `SOPEngine`：带验证与预算的工作流（`sop/_engine.py:24`）

逐 step 推进，`AWAITING`（停泊）时**直接 return，不留挂起状态**（`:145-148`）。`_handover`（`:155-174`）**步骤之间只传递包装成 `<handover from="...">` 的前一步产出**——"一个步骤读的是前驱的叙述，不是它的文件或对话"，很好的上下文隔离。

---

## 9. 运行时

### 9.1 Tracing（OTel）

`middleware/_tracing/_trace.py:117` 的 `TracingMiddleware`，三钩子。零开销短路（未调用 `setup_tracing` 直接透传）。**OTel context 的 attach/detach 在同一 `__anext__` 步内完成**（`:165-171`）——注释引用 issue #2076：async generator 可能从另一个 asyncio task 关闭，否则 token 会在错误的 context 里 detach。

### 9.2 会话状态持久化

核心侧 `AgentState` 直接序列化；服务端侧 `app/storage/_base.py:32` 的 `StorageBase`（约 70+ 方法）有 `RedisStorage`（同步导出）与 `AsyncSQLAlchemyStorage`（惰性加载）两个实现。

### 9.3 错误处理

结构化错误分类（`types/_reply.py:19-43`）：`ErrorType` 有 9 种（AUTHENTICATION / PERMISSION / RATE_LIMIT / INVALID_REQUEST / UPSTREAM / CONNECTION / INTERNAL / SETUP / UNKNOWN），挂在 `ReplyEndEvent.error` 与 `Msg.error` 上。

### 9.4 实时控制：打断路径

1. `UserInterruptEvent` 进入（`:1084-1091`）：若 `has_awaiting_tool_calls` 则置 `ReplyEndEvent(INTERRUPTED)` 并**直接 return**（不进 ReAct 循环）。
2. `asyncio.CancelledError` 路径（`:1271-1281`）。
3. `finally`（`:1283-1304`）：`_close_unfinished_tool_calls()`（`:970`）为所有未完成调用补发 `ToolResultEndEvent(INTERRUPTED)`，最后发兜底 `AssistantMsg(interruption_message)`。

### 9.5 部署：`create_app`（`app/_app.py:113`）

18 个 router（`app/_router/__init__.py:3-19`）：agent / chat / session / model / embedding_model / tts / realtime / mcp / skill / knowledge_base / workspace / sop / schedule / channel / hub / credential / health 等。

生态组件：`InMemoryMessageBus` / `RedisMessageBus`；通道 `DingTalkChannel` / `DiscordChannel` / `FeishuChannel`；Hub `GitHubMCPHub` / `ClawSkillHub`；`CollectionPerKbManager`（每个知识库一个 collection，允许任意 embedding 维度）；`apscheduler` 驱动的 schedule_router。

启动（`examples/agent_service/main.py:168-177`）：`uvicorn.run(...)`，注释说明 Windows 上禁用 reload（会强制 SelectorEventLoop，而内置工具依赖 spawn 子进程）。

### 9.6 实时语音（`realtime/`）

`RealtimeAgent`（`agent/_realtime/_agent.py:92`）三层生命周期：Agent 拥有 model session 与 state；Transport 由创建者拥有（可断线重连而不丢模型会话）；`reply_stream` 借用两者。带服务端 VAD（启用时关掉 provider 自带 turn detection，"exactly one source decides turn boundaries"）、`TurnAggregator`、`_barge_in`、`TurnMetrics`。

---

## 10. 工程亮点与可借鉴点

1. **单一 Agent 类 + 配置对象化**：行为差异全部由 `ModelConfig`/`ContextConfig`/`ReActConfig`/`InjectionConfig` 表达，对比 1.x 的类爆炸——"配置优于继承"。
2. **中间件作为唯一扩展点**：7 个钩子，构造时预分桶，`is_implemented` 用对象身份比较零成本判定。
3. **停泊-恢复协议取代同步阻塞**：HITL / 外部执行 / 打断都建模为"reply 结束但事件流可续"，靠 `reply_id` 路由，agent 可跨进程、跨 HTTP 请求、跨天恢复。
4. **纯函数状态机**：`_next_action` 只读，副作用全在 `_reply_impl`，使状态迁移可测试、可推理、可被中间件干预。
5. **异常二分**：喂回 LLM 自我纠正 vs 抛给程序员；错误文本里甚至附可执行复现代码。
6. **面向"LLM 会犯错"的防御性设计**：容错 JSON 解析 + 拒绝"删字段式修复" + 结构化输出四策略降级 + 空流保护 + 纯 thinking 不算终态 + 压缩请求超长时渐进丢最老消息 + MCP 不可达只 warn。
7. **Prompt cache 友好**：运行时状态用 `HintBlock`；强制单工具用 `mode=<name>` 而非 `tools=[name]`；多模态提升用 block 自身 id 作标识符。
8. **并发安全语义内建**：`is_concurrency_safe` 自动分批；sentinel 保证事件流完整；`ExceptionGroup` 聚合失败；取消时 flush-then-raise。
9. **安全纵深**：权限 5 模式 + 敏感路径 realpath 解析 + MCP 默认禁状态注入 + 保留 HTTP 头校验。
10. **可选依赖 + 惰性导入**：28 个核心依赖，其余全走 extras。
11. **向后兼容的显式处理**：`AgentState` 的 `mode="before"` 迁移器 + property 转发；废弃字段主动 `warnings.warn` 并自动搬运。
12. **UI 契约内建于数据模型**：中间件 `Parameters` 字段全带 `title`/`description`，前端直接 `model_json_schema()` 渲染；同时用 `SkipJsonSchema` 把 prompt 契约类字段从 UI 隐藏，防止"逐 session prompt 漂移"。
13. **可观测性不牺牲正确性**：OTel attach/detach 严守同一 `__anext__` 步；`asyncio.shield` 保证压缩原子性。

---

## 11. 关键文件速查

```
agent/_agent.py          Agent(117) reply_stream(288) reply(332) observe(381)
                         _reply(900) _reply_impl(1035) _next_action(3495)
                         _reasoning_impl(1698) _acting_impl(2768)
                         _execute_tool_call(2426) _batch_tool_calls(2096)
                         _execute_concurrent_tool_calls(2186) _check_permission(2335)
                         _call_model(3274) _inject_runtime_state(1365)
                         _compress_context_impl(490) _handle_incoming_event(1926)
agent/_config.py         ContextConfig(51) InjectionConfig(195) ReActConfig(362) ModelConfig(415)
message/_base.py         Msg(70) append_event(243) UserMsg(541) AssistantMsg(594)
state/_state.py          AgentState(209) ReplyContext(182) ToolContext(32) append_context(298)
tool/_toolkit.py         Toolkit(66) call_tool(225) _get_available_tools(473)
tool/_types.py           RegisteredTool(24) get_tool_schema(56) ToolChoice(178)
tool/_adapters.py        FunctionTool(37) MCPTool(195)
model/_base.py           ChatModelBase(37) __call__(182) count_tokens(377) generate_structured_output(465)
formatter/_formatter_base.py  FormatterBase(20) convert_tool_result_to_string(105) _group_messages(189)
rag/_knowledge.py        KnowledgeBase(44) search(190) insert_document(279)
pipeline/_team_pipeline.py    TeamPipeline(110) _TeamAssign(51) _run_assignments(284)
pipeline/_goal_pipeline.py    GoalPipeline(55)
sop/_engine.py           SOPEngine(24) _handover(155)
middleware/_base.py      MiddlewareBase(13) is_implemented(55)
middleware/_rag.py       RAGMiddleware(626)
app/_app.py              create_app(113)
_utils/_common.py        _json_loads_with_repair(95) _flatten_json_schema(317)
```

**阅读路径建议**：`_reply_impl`（`:1035`）是唯一能同时触达 event / state / tool / model / middleware / permission / context 七个子系统的地方，以它为主干串起全文。

---

> 下一篇：[02-AgentScope-Java-源码原理](02-AgentScope-Java-源码原理.md)
