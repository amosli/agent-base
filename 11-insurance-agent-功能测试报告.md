# 11 · insurance-agent 功能测试报告

> **TL;DR**
> - 用 **Playwright MCP**（浏览器端到端）+ curl（API 层）对 `insurance-agent` 做了功能测试，**核心链路全部通过**。
> - 覆盖：7 个工具顺序执行、**两次 HITL 停泊-恢复**、**拒绝路径**、最终结构化结论、SSE 事件流渲染、浏览器无报错。
> - 测试采用 **Mock 模型**（`-Dinsurance.mock=true`），不依赖外部 LLM，**确定性可重复**。
> - 过程中发现并修复 **4 个真实缺陷**（见 §4），其中 2 个是框架 API 使用错误、1 个是 mock 数据缺陷、1 个是 UI session 硬编码。

---

## 1. 测试环境

| 项 | 值 |
|---|---|
| 被测对象 | `agent_base/insurance-agent`（AgentScope Java **2.0.3 GA**） |
| 启动模式 | `-Dinsurance.mock=true`（脚本化 Mock 模型，无外部 LLM） |
| 启动命令 | `mvn -q compile exec:java -Dexec.mainClass=io.agentscope.demo.insurance.web.InsuranceAgentServer -Dinsurance.mock=true -Dexec.args="18080"` |
| 运行环境 | JDK 17（`/usr/lib/jvm/java-17-openjdk-amd64`）+ Maven 3.9.16 |
| UI | `http://localhost:18080/`（支持 `?session=<id>` 隔离会话） |
| 测试工具 | Playwright MCP（chromium, headless）+ curl |
| 测试日期 | 2026-10-21 |

## 2. 测试用例与结果

### 2.1 API 层（curl + SSE）

| # | 用例 | 步骤 | 预期 | 结果 |
|---|---|---|---|---|
| A1 | 健康检查 | `GET /api/health` | `{"status":"UP"}` | ✅ |
| A2 | 工具清单 | `GET /api/tools` | 9 个工具，6 ALLOW + 3 ASK | ✅ |
| A3 | 首轮对话 + 读工具 | `POST /api/chat`（报案消息） | `query_policy` 自动放行并返回保单 | ✅ |
| A4 | **HITL 停泊①** | 同上，观察事件流 | `create_claim_case` 触发 `REQUIRE_USER_CONFIRM`，Agent 停泊 | ✅ |
| A5 | **确认恢复①** | `POST /api/chat/confirm`（confirmed=true） | `USER_CONFIRM_RESULT` → 工具真正执行 → 继续 `list_required_documents` | ✅ |
| A6 | **HITL 停泊②** | 同上 | `upload_document` 再次停泊（新 replyId） | ✅ |
| A7 | **确认恢复②** | confirm（confirmed=true） | 依次执行 `check_claim_rules` → `fraud_score` → `assess_loss` → 最终结论 | ✅ |
| A8 | **拒绝路径** | confirm（confirmed=false） | 工具记 DENIED，Agent **不重试同一操作**，改为解释并询问（1 次模型调用，无 maxIters 循环） | ✅ |
| A9 | 事件序列完整性 | 全程统计 | `AGENT_START` 开始、`AGENT_END` 结束；每个 `TOOL_CALL_START` 有配对 `TOOL_RESULT_END` | ✅ |
| A10 | 未知 replyId | confirm 传假 replyId | 返回 400 + 明确错误信息 | ✅ |

### 2.2 UI 层（Playwright MCP）

| # | 用例 | 操作 | 预期 | 结果 |
|---|---|---|---|---|
| U1 | 页面加载 | `navigate /?session=uitest-final` | 标题、Mock 徽标、欢迎语渲染 | ✅ |
| U2 | 发送消息 | `fill #input` + `click #send` | 用户气泡 + Agent 流式文本增量渲染 | ✅ |
| U3 | 工具可视化 | 观察渲染 | `🔧 调用工具` + `📥 工具返回[SUCCESS]` | ✅ |
| U4 | **停泊卡片** | 观察渲染 | 黄色确认卡片出现，展示 `toolCallId/name/input` 完整 JSON | ✅ |
| U5 | **确认执行** | `click #btn-approve` | 显示"✓ 复核员已确认，恢复执行"，后续事件继续渲染 | ✅ |
| U6 | **二次停泊与确认** | 再次 `click #btn-approve` | 第二次停泊 → 确认 → 走完核赔 | ✅ |
| U7 | 最终结论 | 观察渲染 | `APPROVED / 29,385.00 元 / 依据 / 待补材料 / 下一步` 完整展示 | ✅ |
| U8 | 浏览器控制台 | `playwright_console_logs(type=error)` | **无错误** | ✅ |

**截图存证**（`~/Downloads/`）：
- `insurance-hitl-parked-2026-10-21T15-55-46-986Z.png` —— HITL 停泊状态（含待确认工具参数）
- `insurance-final-decision-2026-10-21T15-56-10-029Z.png` —— 最终理赔结论

### 2.3 关键事件序列（A7 实测，节选）

```
AGENT_START → USER_CONFIRM_RESULT
→ TOOL_RESULT_END(upload_document, SUCCESS)
→ MODEL_CALL_START → TEXT_BLOCK_DELTA("材料已收到，开始核赔流程…")
→ TOOL_CALL_START(check_claim_rules) → TOOL_RESULT_END(SUCCESS)
→ TOOL_CALL_START(fraud_score)        → TOOL_RESULT_END(SUCCESS)
→ TOOL_CALL_START(assess_loss)        → TOOL_RESULT_END(SUCCESS)
→ MODEL_CALL_START → TEXT_BLOCK_DELTA(理赔结论…) → MODEL_CALL_END → AGENT_RESULT → AGENT_END
```

与 [10 §1.3 设计的时序图](10-insurance-agent-保险理赔Demo.md) 一致。

---

## 3. 与设计文档的对照

| 设计声明（[10](10-insurance-agent-保险理赔Demo.md)） | 实测 |
|---|---|
| 读工具 ALLOW 自动放行 | ✅ `query_policy` 等 6 个无需确认直接执行 |
| 写工具 ASK 触发停泊 | ✅ `create_claim_case` / `upload_document` 均停泊 |
| 停泊后经 `METADATA_CONFIRM_RESULTS` 恢复 | ✅ 两次恢复均成功 |
| 拒绝后工具以 DENIED 回喂、Agent 调整 | ✅ 见 A8 |
| 最终输出结构化结论 | ✅ 文本形态完整（结构化 `ClaimDecision` 走 `call(msgs, Class, ctx)` 重载，HTTP 层未启用，见 §5） |

---

## 4. 测试中发现并修复的缺陷（4 个）

> 这一节是本次测试**最有价值的产出**——功能测试暴露了 4 个静态审查看不出的问题。

| # | 缺陷 | 根因 | 修复 |
|---|---|---|---|
| 1 | **工具参数全部为空**，`query_policy` 报 `未找到所需属性 policyNo`，Agent 无限重试至 maxIters | `ToolUseBlock` 只设了 `input`，**没设 `content`**（原始 JSON 串）。框架从 `content` 解析参数 | 改为 `ToolUseBlock.builder().input(args).content(JSON).build()`，与框架自带 `MockModel`（`agentscope-core/src/test/.../MockModel.java:234-252`）的构造方式一致 |
| 2 | **`ConfirmResult` 编译失败** | GA `2.0.3` 只有 2/3 参构造，**没有带 `reason` 的 4 参版本**（本地 SNAPSHOT 才有） | 按 2.0.3 API 改用 2 参构造，并在 README/文档标注版本差异 |
| 3 | **李四保单"意外失效"**：保单到期日 2026-06-01 早于 mock 事故时间 2026-09-18，`check_claim_rules` 会判 FAIL | mock 数据未随当前日期（2026-10）维护 | `MockPolicyStore` 到期日改为 `2027-06-01` |
| 4 | **UI sessionId 硬编码**：无法隔离测试会话，重启后可能恢复旧停泊状态 | `const SESSION_ID = "ui-session-1"` | 支持 `?session=<id>` URL 参数 |

另有一个**剧本行为缺陷**（非框架问题）：mock 被拒绝后仍重复调用同一工具直至 maxIters。已修复：mock 检测到 `DENIED` 后改为"解释 + 询问用户"，不再重试（A8 用例）。

---

## 5. 未覆盖项与后续建议

| 未覆盖 | 原因 | 建议 |
|---|---|---|
| **真实 LLM 链路** | 环境无 DashScope key | 有 key 后去掉 `-Dinsurance.mock=true` 重跑 U1–U7 |
| **结构化输出 API** | HTTP 层只走事件流 | 加一个 `POST /api/chat/decision` 端点，用 `call(msgs, ClaimDecision.class, ctx)` 返回 JSON |
| **保单失效路径**（王五 EXPIRED） | mock 剧本固定用有效保单 | 给 mock 剧本加第二个场景分支 |
| **DENY 规则** | 权限只配了 ALLOW/ASK | 加一条 deny 规则用例（如禁用 `fraud_score`） |
| **多租户并发** | 单会话顺序测试 | 用两个 session 并发发消息，验证 `callGates` 串行与事件不交叉 |
| **中断/取消** | 未测 | 客户端断开 SSE，验证 `doOnCancel` 与状态一致性 |
| **Playwright 断言化** | 本次以人工观察 + 文本验证为主 | 把 U1–U8 写成 Playwright 脚本（`expect(page.locator(...)).toContainText(...)`），接入 CI |

---

## 6. 结论

> insurance-agent 的**核心机制（ReAct 循环、工具调用、三态权限、HITL 停泊-恢复）在 API 层与 UI 层均验证通过**，事件序列与设计文档一致，浏览器无报错。
> Mock 模式让测试**不依赖外部 LLM、确定性可重复**，适合作为回归基线；有真实 key 后需补测真实模型链路与结构化输出端点。

---

> 相关：[10 insurance-agent 设计与架构](10-insurance-agent-保险理赔Demo.md) · [08 调试与实战手册](08-调试与实战手册.md)（Mock 模型即其"确定性回放"的实践） · [insurance-agent 源码](../insurance-agent/)
