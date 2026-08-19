# AI 开发工具（Java + Web 版）需求文档

> 版本：v0.1（洞察稿）
> 日期：2026-08-19
> 范围：基于 Claude Code / opencode / openclaw / codex 四个参考实现的源码洞察，定义 Java + Web 版 AI 编程助手的能力需求与实现路径。

---

## 0. 文档目的

本文档记录「需要做什么能力」与「怎么做」，供后续设计与实施阶段参考。所有结论均来自对四个参考实现源码的实际通读，关键论点附 `file:line` 引用，可在对应源码中复核。

### 0.1 参考实现索引

| 简称 | 路径 | 语言/运行时 | 定位 |
|---|---|---|---|
| Claude Code | `D:\workspace\sourcecode\Claude-Code-main\Claude-Code-main` | TypeScript / Bun / React+Ink | Anthropic 官方 CLI 编程助手 |
| opencode | `D:\workspace\sourcecode\opencode-dev\opencode-dev` | TypeScript / Bun / Effect-TS / SolidJS | 开源 server+client 架构编程助手 |
| openclaw | `D:\workspace\sourcecode\openclaw-main\openclaw-main` | TypeScript / Node / pi-tui | 多通道 AI 网关/Agent 平台 |
| codex | `D:\workspace\sourcecode\codex-main\codex-main` | Rust / ratatui | OpenAI Codex CLI |

### 0.2 术语

- **Agent 循环**：发送消息给 LLM → 解析 tool_use → 执行工具 → 回填结果 → 继续循环直到模型不再调用工具的核心循环。
- **Op / Event**：前端→后端的操作（Op）与后端→前端的事件（Event）双向协议。
- **Part**：消息的最小流式单元（一段文本 delta、一次推理 delta、一次工具调用等），可流式落库。
- **Compaction**：上下文窗口接近溢出时，用摘要模型压缩历史对话。
- **MCP**：Model Context Protocol，外部工具服务器协议。

---

## 1. 项目概述

### 1.1 目标

构建一个 **服务端 agent + web 前端** 的 AI 编程助手：
- Agent 循环、工具执行、上下文管理、权限审批跑在 **Java 服务端**。
- 浏览器前端通过 HTTP/SSE/WebSocket 订阅事件流，展示流式输出与审批交互。
- 可在本地或远程沙箱执行文件读写、shell 命令、代码搜索等操作。

### 1.2 不做（MVP 范围外）

- 多通道消息网关（openclaw 的 WhatsApp/Telegram/Slack 桥接）。
- 语音实时对话（codex 的 Realtime API）。
- 多 agent 团队 swarm 协调（Claude Code 的 TeamCreate/SendMessage）。
- 终端 UI（TUI）——前端用 web 替代。

### 1.3 总体架构

```
┌─────────────┐   HTTP/SSE/WebSocket   ┌──────────────────────────┐
│  Web 前端   │ ←──────────────────→ │   Java Agent Server      │
│ (React/TS)  │   Op / Event 协议      │  ┌────────────────────┐  │
└─────────────┘                       │  │  Agent 主循环       │  │ ──→ LLM API
       ↑ TS 类型由 Java 生成           │  │  (Reactor Flux)     │  │
                                      │  ├────────────────────┤  │
                                      │  │  工具系统           │  │ ──→ 文件系统
                                      │  │  权限/审批          │  │ ──→ Shell (沙箱)
                                      │  │  上下文/压缩        │  │
                                      │  │  会话持久化(SQLite) │  │ ──→ MCP 服务器
                                      │  └────────────────────┘  │
                                      └──────────────────────────┘
```

**架构决策依据：**
- opencode 的 server+client 分层（Hono + SSE）证明此架构可行（`packages/opencode/src/server/server.ts`，TUI 通过 HTTP client `createOpencodeClient` 订阅 `message.updated`/`session.status`/`permission.asked` 事件，见 `run.ts:701-816`）。
- codex 的 app-server 协议（`Op`/`Event` 双向通道，`protocol/src/protocol.rs:543,1271,1289`）证明同一 agent 可同时服务 TUI/IDE/web，靠的是协议而非 UI 耦合。
- TUI 在 web 版被浏览器前端取代，但 agent 循环逻辑不变。

---

## 2. 核心能力需求

### 2.1 Agent 主循环

#### 2.1.1 能力描述

发送消息给 LLM → 解析模型返回的 tool_use → 执行工具 → 把工具结果塞回消息 → 循环直到模型不再调用工具。

#### 2.1.2 循环结构（采纳 openclaw 两层分离 + Claude Code 状态机）

**底层 `AgentLoop`（纯循环，可测试，框架无关）：**
- 采纳 openclaw 的 `runLoop`（`packages/agent-core/src/agent-loop.ts:298`）作为范本：外层 `while(true)` 处理后续轮次，内层 `while(hasMoreToolCalls || pendingMessages)` 处理工具与 steering。
- 通过钩子接口扩展，**不把策略硬编码进循环**：
  - `convertToLlm(messages)`：把内部富消息转成 provider 格式（`agent-loop.ts:547`）。
  - `beforeToolBatch` / `beforeToolCall` / `afterToolCall` / `afterToolOutcome`（`types.ts:54,111,157`）。
  - `getSteeringMessages()`：在工具检查点轮询，允许用户中途打断/重定向（`types.ts:308`，`agent-loop.ts:76-84`）。
  - `shouldStopAfterTurn` / `prepareNextTurn`（`agent-loop.ts:493`）。
  - `resolveDeferredTool`：延迟/水合工具解析（`agent-loop.ts:1223`）。

**上层 runner（重试/压缩/故障转移）：**
- 采纳 openclaw 的 `runPreparedEmbeddedLoop`（`src/agents/embedded-agent-runner/run-loop.ts:70`）：`while(true)` 重试循环，每轮 `prepareAndDispatchEmbeddedRunAttempt` → `normalizeEmbeddedRunAttempt`（分类 complete/retry）→ `recoverEmbeddedRunAttempt`（压缩/上下文恢复/模型 fallback）→ `handleEmbeddedAssistantFailure`（认证轮换/限流退避/空响应重试）。

**显式状态机（采纳 Claude Code）：**
- 用 `State` 对象 + `transition` reason 跟踪（`src/query.ts:204-217`），让恢复点成为一等的 continue 分支而非临时递归：
  - prompt-too-long 恢复（drain context collapses → reactive compact，`query.ts:1089-1166`）。
  - max_output_tokens 恢复（升级到 64k，最多 3 次多轮 resume 注入，`query.ts:1188-1252`，`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT=3`）。
  - 停止钩子阻止（`query.ts:1283-1306`）。
  - token 预算续跑（`query.ts:1308-1355`）。
  - 响应式压缩（`query.ts:799-822`）。
- "错误先扣押再恢复"模式（Claude Code `query.ts:799-822`）：可恢复错误先不抛给前端，等恢复失败才暴露，防止 SDK 消费者中途终止。

**流式工具执行（采纳 Claude Code）：**
- `StreamingToolExecutor`（`src/services/tools/StreamingToolExecutor.ts`）：模型还在流式输出剩余响应时就开始执行已到达的 tool_use。对 web 的延迟收益巨大（SSE/WebSocket 天然适配）。

#### 2.1.3 退出条件（综合四家）

| 条件 | 来源 |
|---|---|
| 模型无 tool_call 且 `end_turn != false` | codex `turn.rs:519,531`；opencode `prompt.ts:1111-1130` |
| 最大步数到（`maxSteps`） | opencode `prompt.ts:1178`；Claude Code `query.ts:1705` |
| 被取消（abort signal） | openclaw `agent-loop.ts:320`；codex `turn.rs:2256` |
| 停止钩子阻止 | Claude Code `query.ts:1283`；codex `turn.rs:484-517` |
| 上下文溢出且压缩失败 | Claude Code `query.ts:646`；codex `turn.rs:1387` |
| 结构化输出已捕获 | opencode `prompt.ts:1288` |

#### 2.1.4 Java 实现要点

- `CompletableFuture` / JDK 21+ 虚拟线程做工具并发。
- `Flux<AgentEvent>`（Reactor）做流式输出，对应 opencode 的 `Stream<LLMEvent>`（`packages/opencode/src/session/llm.ts:357`）。
- 循环用显式 `State` record + switch 表达式处理 transition。

#### 2.1.5 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| 两层循环分离 | openclaw `agent-loop.ts:298` + `run-loop.ts:70` | — |
| 显式状态机 + 恢复点 | Claude Code `query.ts:204-1729` | — |
| 流式工具执行 | Claude Code `StreamingToolExecutor` | — |
| 钩子接口扩展 | openclaw `types.ts:341` | Claude Code hooks |
| steering 中途打断 | openclaw `types.ts:308` | codex `turn_input.rs:200` |

---

### 2.2 工具系统

#### 2.2.1 能力描述

定义/注册/校验/调度工具，并把工具 schema 发给模型。

#### 2.2.2 Tool 定义

采纳 codex 最干净的 `ToolSpec`（`codex-rs/tools/src/tool_definition.rs:6-13`）：

```
ToolSpec:
  name: String
  description: String
  inputSchema: JsonSchema        // 模型输入 + 运行时校验双用
  outputSchema: Optional<JsonSchema>
  deferLoading: boolean          // tool-search / 延迟加载
```

- **Schema 双用模式**（openclaw 的 typebox 范式，`src/agents/sessions/tools/read.ts:5`）：同一份 schema 既是发给模型的 JSON Schema，又是运行时参数校验器。
- Java 实现：Jackson + `com.networknt:json-schema-validator`，或 `dev.langchain4j` 的 `ToolSpecification`。
- `ToolExecutor` 接口：`CompletableFuture<ToolOutput> execute(args, ctx)`。

#### 2.2.3 工具调度

- **并发分区**（Claude Code `partitionToolCalls`，`src/services/tools/toolOrchestration.ts:91-116`）：连续的只读工具（`isConcurrencySafe`/`isReadOnly`）并行（上限 10，`getMaxToolUseConcurrency=10`），有副作用的串行。简单有效。
- **并行工具执行**（codex `FuturesOrdered`，`turn.rs:2206,2373-2375`）：`parallel_tool_calls=true` 发给模型，工具结果按完成顺序收集。Java 用 `Flux.merge` + `Flux.buffer`。

#### 2.2.4 工具集设计抉择（最大决策点）

| 方案 | 工具集 | 来源 | 评价 |
|---|---|---|---|
| A. 细粒度工具集 | Read/Write/Edit/Glob/Grep/Bash/WebFetch/Task/... | Claude Code `tools.ts:193-250`；opencode `registry.ts:101-119` | 权限细、可控、模型行为可预测；但实现工作量大 |
| B. 极简工具集 | `apply_patch`（文法工具）+ `exec_command`（shell） | codex `spec_plan.rs:892-1230` | 最简洁，文件操作全走 shell；模型自己跑 `rg`/`cat`；少踩坑 |
| C. 混合 | `apply_patch` + `exec` + Read/Edit/Grep（按需补） | — | MVP 用 B，成熟后按需补 A 的细粒度 |

**推荐：MVP 先用 B（codex 风格），成熟后补 A 的细粒度工具。**

#### 2.2.5 apply_patch 文法工具（codex 范本）

- 定义：`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:9-28`，用 `ToolSpec::Freeform` + Lark 文法（`apply_patch.lark`）。
- 实现：`codex-rs/apply-patch/src/lib.rs` → `parse_patch` → `Hunk::{AddFile, DeleteFile, UpdateFile, MoveFile}` → `apply_hunks_to_files`。
- 支持 `*** Begin Patch` / `*** Add File` / `*** Update File` / `@@ context` 标记（`apply-patch.ts:28-36` in openclaw）。
- 返回 `AppliedPatchDelta` 跟踪已提交变更用于安全回滚（`lib.rs:227-316`）。
- 一次调用可改/增/删/移多个文件，比 per-file Edit 高效。

#### 2.2.6 工具输出截断

- opencode `truncate.ts` + `tool.ts:131-144`：超大输出写临时文件，模型只拿摘要 + `outputPath` 元数据指针。保持上下文精简同时保留全量输出供前端展示。
- Claude Code 的 `maxResultSizeChars` per tool（`Tool.ts:466`），`Read`/`Edit` 设为 `Infinity`（避免循环 Read→file→Read）。
- 聚合预算 `applyToolResultBudget`（`src/utils/toolResultStorage.ts`）。

#### 2.2.7 失效工具名自修复

- opencode 用 AI SDK 的 `experimental_repairToolCall`（`session/llm.ts:296-312`）：无效工具名先尝试小写，再路由到 `invalid` 工具返回错误，让模型自纠正。

#### 2.2.8 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| ToolSpec 形状 | codex `tool_definition.rs:6` | Claude Code `Tool.ts:362` |
| Schema 双用 | openclaw typebox | Claude Code Zod→JSON Schema |
| 并发分区 | Claude Code `toolOrchestration.ts:91` | codex `FuturesOrdered` |
| apply_patch 文法工具 | codex `apply-patch/lib.rs` | openclaw `apply-patch.ts` |
| 输出截断 | opencode `truncate.ts` | Claude Code `toolResultStorage.ts` |
| 工具名自修复 | opencode `llm.ts:296` | — |

---

### 2.3 LLM 提供商集成

#### 2.3.1 能力描述

调用模型 API、流式接收、把工具 schema 发过去、重试/故障转移。

#### 2.3.2 LLMEvent 统一抽象（关键）

采纳 opencode 的关键抽象：定义自己的 `LLMEvent` 联合类型（`session/llm/ai-sdk.ts:76` `toLLMEvents`），所有 provider 适配器收敛到同一流：

```
LLMEvent =
  | TextDelta(text)
  | ReasoningDelta(text)
  | ToolCallStart(id, name)
  | ToolCallInputDelta(id, jsonDelta)
  | ToolCallEnd(id)
  | ToolResult(id, output)
  | StepStart
  | StepFinish(usage, cost)
  | Done(responseId, finishReason)
  | Error(throwable)
```

- openclaw 的 `AssistantMessageEventStream`（`packages/ai/src/utils/event-stream.ts`）同样定义了 `start/text_start/delta/end/thinking_*/toolcall_*/done/error`（`agent-loop.ts:56-70`）。
- Java 实现：sealed interface + Reactor `Flux<LLMEvent>`。

#### 2.3.3 Provider 适配器

- 至少做 **Anthropic Messages API** + **OpenAI Responses API**。
- **手写 HTTP+SSE，不依赖官方 SDK**——Java 生态的官方 SDK 质量参差，手写更可控（openclaw 范式：`packages/ai/src/providers/anthropic.ts`、`openai-responses.ts` 全是手写 HTTP/SSE 适配器，`register-builtins.ts:94-151`）。
- 用 reactor-netty 或 Java HttpClient + SSE 解析。
- Provider 懒加载（openclaw `register-builtins.ts:60-92`）：首次使用时动态初始化，立即返回同步占位流。

#### 2.3.4 请求构建

- 构建 `LLMRequest { system[], messages, tools[], params, headers }`（opencode `session/llm/request.ts:56`）。
- 系统提示分层组装：agent 提示 + `SystemPrompt.provider(model)`（按 provider 不同基础提示）+ 环境信息 + instructions + MCP + skills（opencode `session/system.ts:27-49`：GPT/Claude/Gemini/Kimi/Meta 各有不同基础提示）。
- 工具 schema 发送前做 provider 特定整形（opencode `ProviderTransform.schema`，`session/tools.ts:98`）：某些 provider 拒绝 `$schema`/`additionalProperties`，需剥离。
- 插件钩子 `chat.params` / `experimental.chat.system.transform`（opencode `request.ts:69,114`）。

#### 2.3.5 Prompt 缓存前缀策略

采纳 Claude Code 的成熟策略（`claude.ts:358-434`）：
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（`prompts.ts:114-115`）拆分"静态可缓存前缀（跨用户可缓存，`scope:'global'`）+ 动态部分"。
- 工具按内置顺序做稳定前缀（Claude Code `tools.ts:354-367`：built-ins 作为连续前缀，缓存断点稳定）。
- 1h TTL 用于合格用户（`should1hCacheTTL`，`claude.ts:393-434`）。
- Java 实现：前缀哈希策略 + `cache_control` 标记。

#### 2.3.6 粘性路由 + WebSocket 预热

采纳 codex（`client.rs:146,1954-1972`）：
- 每轮首个请求拿 `x-codex-turn-state` header，后续请求回放——降低延迟。
- WebSocket 预热：turn 开始时发一个 `generate=false` 的 `response.create`（`client.rs:14-24`），下个请求复用连接 + `previous_response_id`。
- 自动降级：WebSocket 重试耗尽后永久回退 HTTP/SSE（`client.rs:1920-1930`）。

#### 2.3.7 重试 / 故障转移

- Claude Code `withRetry.ts:170`：指数退避（`BASE_DELAY_MS=500`），529-overloaded 处理（`is529Error`），`FallbackTriggeredError` 切 fallback 模型，`CannotRetryError`。
- 非流式回退：`MAX_NON_STREAMING_TOKENS=64000`（`claude.ts:3354`），流式失败时用非流式重试（`executeNonStreamingRequest`，`claude.ts:818`）。
- codex 的 `ResponsesStreamRetryState`（`core/src/responses_retry.rs`）+ `handle_retryable_response_stream_error`（`turn.rs:1410`）。

#### 2.3.8 思考块（reasoning blocks）

- 作为一等的 `ReasoningPart` 流式展示（opencode `processor.ts:280-313`）。
- Claude Code `ThinkingConfig = {type:'adaptive'|'disabled'|{type:'enabled',budget_tokens}}`（`src/utils/thinking.ts`）。
- 思考签名是 model-bound：模型 fallback 时 `stripSignatureBlocks`（`query.ts:928`），思考块必须在整个 assistant 轨迹期间保留（`query.ts:152-163`）。
- 跨 provider 兼容：不是所有 provider 都支持 reasoning，适配器按能力降级。

#### 2.3.9 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| LLMEvent 统一抽象 | opencode `ai-sdk.ts:76` | openclaw `event-stream.ts` |
| 手写 provider 适配器 | openclaw `providers/anthropic.ts` | — |
| Prompt 缓存前缀 | Claude Code `claude.ts:358` | — |
| 粘性路由+预热 | codex `client.rs:146` | — |
| 重试/故障转移 | Claude Code `withRetry.ts:170` | codex `responses_retry.rs` |
| 思考块 | opencode `processor.ts:280` | Claude Code `thinking.ts` |
| Provider 特定提示 | opencode `system.ts:27` | — |

---

### 2.4 上下文与会话管理

#### 2.4.1 能力描述

存消息历史、检测溢出、压缩、持久化可恢复。

#### 2.4.2 持久化：SQLite-first

- **openclaw + codex 都坚持 SQLite-only 用于运行时状态**（openclaw `AGENTS.md:113-118`：所有运行时状态/缓存/队列/插件 KV 在 SQLite，JSON/JSONL 只用于导入导出/附件）。
- Java 用 JDBC + Flyway 迁移。
- 表设计（采纳 opencode 的 Part 模型）：
  - `session`：id, slug, project_id, directory, parent_id, title, agent, model, tokens, cost, summary, permission, time。
  - `message`：id, session_id, role(user/assistant), parent_id, finish_reason。
  - `part`：id, message_id, type(text/reasoning/tool/file/step-start/step-finish/patch/compaction), content(json), delta, time。
- **可持久化的流式 Part 模型**（opencode `processor.ts:278-537`）：每个 delta 落库，会话可恢复/可分叉/可重放。web 版强需求（断线重连不丢历史）。
- 旧 JSONL 文件存储仅作迁移债务，只读（openclaw `AGENTS.md:112-120`）。

#### 2.4.3 消息类型

- 采纳 openclaw 的 `AgentMessage` 扩展（`types.ts:468-480`）：标准 LLM 消息 + 自定义 harness 角色（`bashExecution`/`compactionSummary`/`branchSummary`）。
- `convertToLlm(messages)` 钩子（`types.ts:236`）在每轮调用，过滤 UI-only 消息，产出 provider 兼容格式（`agent-loop.ts:547`）。让 transcript 持有比模型所见更丰富的信息。
- opencode 的 `MessageV2` 工作层（`session/message-v2.ts`）：cursor 分页（`cursor.encode/decode`，`message-v2.ts:71`）、`filterCompactedEffect`（每轮跳过已压缩内容）、`toModelMessagesEffect`（存储 part → AI SDK `ModelMessage[]`）。

#### 2.4.4 多层上下文压缩（采纳 Claude Code，最成熟）

| 层 | 触发 | 来源 |
|---|---|---|
| 阈值自动压缩 | 达到窗口 - 13K（`AUTOCOMPACT_BUFFER_TOKENS=13000`，`autoCompact.ts:62`） | Claude Code `autoCompact.ts:72-80` |
| 微压缩 | 按工具结果粒度（`microCompact.ts`） | Claude Code |
| 缓存微压缩 | 用 API `cache_deleted_input_tokens`（`cachedMicrocompact.ts`） | Claude Code |
| 响应式压缩 | 413/media 错误时 withhold→重试（`reactiveCompact.ts`） | Claude Code |
| snip 压缩 | 历史边界 snip（`snipCompact.ts`，`HISTORY_SNIP`） | Claude Code |
| context collapse | 分阶段折叠（`contextCollapse/`），413 时 drain | Claude Code |
| 断路器 | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3`（`autoCompact.ts:70`） | Claude Code |

- 压缩用 forked agent（haiku 等小模型）跑摘要（Claude Code `compact.ts` `runForkedAgent`，`src/utils/forkedAgent.ts`）。
- 运行 PreCompact/PostCompact 钩子（Claude Code `compact.ts:55-57`）。

#### 2.4.5 保留近期预算

- opencode `compaction.ts:115-120`：`min(15000, max(2000, 25%可用上下文))` 的近期 token 原样保留，旧的摘要。
- 序列化时工具输出截断到 2000 字符（opencode `compaction.ts:51`；openclaw 同样）。

#### 2.4.6 上下文引擎抽象

- openclaw 的可插拔 Context Engine（`src/context-engine/registry.ts`）：方法 `bootstrap/maintain/ingest/ingestBatch/afterTurn/commitTurn/assemble/compact/prepareSubagentSpawn`。
- 插件可注册替代引擎（`registry.ts:35-50`）。
- 逻辑轮租约（`harness/context-engine-logical-turn.ts`）拥有引擎生命周期。
- 可选采纳：MVP 用 legacy 引擎，后期开放插件化。

#### 2.4.7 token 估算

- Claude Code `src/utils/tokens.ts`：`tokenCountWithEstimation`、`tokenCountFromLastAPIResponse`、`finalContextTokensFromLastResponse`。
- opencode `overflow.ts:10-34`：`usable = context_limit - reserved`（reserved 默认 `min(20000, maxOutputTokens)`），`isOverflow` 比较 `input+output+cache.read+cache.write` 与 `usable`。

#### 2.4.8 Git 快照做 turn 级撤销

采纳 opencode（`src/snapshot/index.ts`）：
- 在 `~/.local/share/opencode/snapshot/<projectId>/<hash>` 用**独立裸 git 仓**快照工作树。
- `track()` 返回 hash；`patch(hash)` 计算变更 diff；`restore`/`revert` 回滚。
- patch 作为 part 记录在消息流（`processor.ts:457-469`）。
- 7 天后清理。
- 很有价值的 turn 级 undo 机制，web 版前端可暴露"撤销本轮"。

#### 2.4.9 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| SQLite-first | openclaw `AGENTS.md:113` | codex `state_db.rs` |
| Part 流式落库 | opencode `processor.ts:278` | — |
| AgentMessage + convertToLlm | openclaw `types.ts:468` | — |
| 多层压缩 | Claude Code `services/compact/` | openclaw `compaction.ts` |
| 保留近期预算 | opencode `compaction.ts:115` | — |
| 上下文引擎抽象 | openclaw `context-engine/registry.ts` | — |
| Git 快照撤销 | opencode `snapshot/index.ts` | — |
| token 估算 | Claude Code `tokens.ts` | opencode `overflow.ts` |

---

### 2.5 权限模型

#### 2.5.1 能力描述

allow/deny/ask 规则，工具执行前审批。

#### 2.5.2 Ruleset 模式（采纳 opencode，最简洁）

- opencode `permission/index.ts:28-38`：规则为 `{ permission, pattern, action: "allow"|"deny"|"ask" }` 三元组数组（Ruleset）。
- `evaluate(permission, pattern, ...rulesets)`：`findLast` 通配匹配 `permission` 与 `pattern`，默认 `ask`。
- `ask(input)`（`index.ts:67-107`）：per-pattern evaluate；`deny`→`DeniedError`；`allow`→继续；`ask`→发布 `Event.Asked`，await `Deferred` 直到 `reply`。
- `reply(input)`（`index.ts:109-167`）：`reject`（同时拒绝该会话所有 pending）；`once`（succeed deferred）；`always`（加入 `approved` ruleset + 自动解析其他现在可满足的 pending）。
- 默认 ruleset（`agent.ts:119-136`）：`external_directory:"*":"ask"`、`read:"*.env":"ask"`、`doom_loop:"ask"`。
- 外部目录权限门控（`tool/external-directory.ts`）：读写工作树外文件触发 `external_directory` ask（除非 allowlist，如技能目录、临时目录）。

#### 2.5.3 审批模式（采纳 codex）

- `AskForApproval` 枚举（`protocol/src/protocol.rs`）：`Never`/`OnFailure`/`OnRequest`/`UntrustedOnly`/`Granular`。
- **当轮粘性授权**（codex `handlers/mod.rs:271-322` `apply_granted_turn_permissions`）：用户批准过的 network/filesystem glob 本轮内免再问。web 用户体验关键。
- 审批回环：会话 emit `EventMsg::ExecApprovalRequest`/`ApplyPatchApprovalRequest`/`RequestPermissions`/`ElicitationRequest`（`protocol.rs:1410-1422`），客户端回 `Op::ExecApproval { decision: ReviewDecision }`（`protocol.rs:599-606`）。

#### 2.5.4 Guardian 自动审批器（采纳 codex）

- codex `core/src/guardian/`：旁路小模型分类"该不该批准"，可自动批准/拒绝。
- 基于模型分类（Claude Code `TRANSCRIPT_CLASSIFIER`，`src/utils/permissions/yoloClassifier.ts`、`classifierDecision.ts`）+ bash 专用分类（`bashClassifier.ts`）。
- web 版价值大：减少打断，提升体验。

#### 2.5.5 沙箱

- codex 用 macOS Seatbelt（`sandbox-exec`）/ Linux Landlock+seccomp / Windows 沙箱（OS 级，Java 难复刻）。
- **web 版建议用容器/命名空间隔离**（worker pod）。
- 照搬 `SandboxPermissions::{UseDefault, RequireEscalated, WithAdditionalPermissions}` 抽象（codex `protocol/src/models.rs:51-79`）。
- BashTool 沙箱可选（Claude Code `SandboxManager`，`src/utils/sandbox/sandbox-adapter.ts`，`shouldUseSandbox`）。
- exec 审批模型（openclaw `src/infra/exec-approvals.ts`）：per-agent `mode`(allow-all/ask/deny) + allowlist 预批准可执行文件 + arg 模式；可经 Unix socket 让独立审查进程审批。

#### 2.5.6 Doom-loop 检测

- opencode `processor.ts:357-380`：同一工具同输入连续 3 次 → 强制 `doom_loop` permission ask。便宜有效。
- openclaw 的 `beforeToolBatch` 准入（`types.ts:346-350`，`agent-loop.ts:645-692`）：整批钩子可在任何调用执行前检测工具循环模式（critical-tool-loop intervention，`types.ts:68-76`），用于循环检测/恢复（`tool-loop-detection.ts`，`post-compaction-loop-guard.ts`）。

#### 2.5.7 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| Ruleset 三元组 | opencode `permission/index.ts:28` | — |
| 审批模式 + 粘性授权 | codex `AskForApproval` + `apply_granted_turn_permissions` | — |
| Guardian 自动审批 | codex `core/src/guardian/` | Claude Code `yoloClassifier.ts` |
| 沙箱抽象 | codex `SandboxPermissions` | Claude Code `SandboxManager` |
| doom-loop 检测 | opencode `processor.ts:357` | openclaw `beforeToolBatch` |
| 外部目录门控 | opencode `external-directory.ts` | — |

---

### 2.6 前后端协议（web 版特有）

#### 2.6.1 能力描述

Java 服务端 ↔ 浏览器前端的事件协议。

#### 2.6.2 Op / Event 双向协议（采纳 codex 范本）

照搬 codex 的 app-server 模式（`protocol/src/protocol.rs:543,1271,1289`）：
- codex 的 `protocol` crate 近零依赖，是范本——同一 agent 同时服务 TUI/IDE/web，靠协议而非 UI 耦合。

**Op（前端→后端）：**
| Op | 用途 | codex 参考 |
|---|---|---|
| `TurnInput` | 提交用户消息开启一轮 | `protocol.rs:543` |
| `ExecApproval { decision }` | 审批决策回环 | `protocol.rs:599-606` |
| `ApplyPatchApproval` | patch 审批 | `protocol.rs` |
| `ResolveElicitation` | MCP elicitation 回复 | `protocol.rs:617-628` |
| `Resume` | 恢复历史会话 | codex `Subcommand::Resume` |
| `Steer` | 中途打断/重定向 | openclaw `getSteeringMessages` |
| `RefreshMcpServers` | 刷新 MCP 服务器 | codex `protocol.rs:655` |

**Event（后端→前端）：**
| Event | 用途 | 参考 |
|---|---|---|
| `AgentMessageContentDelta` | 流式文本 delta | codex `turn.rs:2567` |
| `AgentReasoning` | 思考块 delta | codex `protocol.rs:1355-1361` |
| `ToolCallBegin` / `ToolCallProgress` / `ToolCallComplete` | 工具执行流 | opencode part 流 |
| `ExecApprovalRequest` / `ApplyPatchApprovalRequest` / `ElicitationRequest` | 审批请求 | codex `protocol.rs:1410-1422` |
| `SessionStatus` | 会话状态变更 | opencode `run.ts:701-816` |
| `TurnDiff` | turn 文件 diff | codex `protocol.rs:1445` |
| `TurnComplete` | 轮完成 | codex `turn.rs:531` |

#### 2.6.3 TypeScript 代码生成

- 从 Java 的 protocol record 生成 TypeScript 类型（对应 codex 用 `ts-rs`，`Cargo.toml:479`，`Subcommand::GenerateTs`，`cli/src/main.rs:1351-1361`）。
- Java 用 `typescript-generator` gradle 插件或 `cz.hcone.lib:typescript-generator`。
- 前端类型安全，协议变更编译期可见。

#### 2.6.4 传输层抽象

- `Transport` 接口：`Stdio` | `WebSocket` | `HTTP-SSE`（对应 codex `AppServerTransport::Stdio | Unix | WebSocket`，`cli/src/main.rs:559-566`）。
- web 版主用 **WebSocket**（双向，审批回环需要双向）；纯前端展示可降级 SSE。
- opencode 范式：TUI 不直接调 agent 代码，通过 Hono server + 生成 SDK client（`createOpencodeClient` from `@opencode-ai/sdk/v2`）+ SSE 事件订阅（`context/sdk.tsx`）。

#### 2.6.5 前端

- React/SolidJS + Tailwind + 事件订阅。
- 参考 opencode `packages/web`（SolidJS+Tailwind via `@solidjs/start`）或 codex app-server 客户端。
- 组件：消息流、diff 渲染、审批弹窗、命令面板、模型选择器、会话列表、权限请求对话框。

#### 2.6.6 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| Op/Event 协议 | codex `protocol.rs:543,1271,1289` | — |
| TS 代码生成 | codex `ts-rs` | — |
| 传输抽象 | codex `AppServerTransport` | opencode Hono+SDK |
| server+client 分层 | opencode `server.ts` + SDK | — |
| 前端 | opencode `packages/web` | — |

---

### 2.7 子 agent / 技能 / MCP

#### 2.7.1 子 agent（Task 工具）

采纳 openclaw 的 `sessions_spawn`（`src/agents/tools/sessions-spawn-tool.ts:321`）：
- 创建子会话，继承工作区/策略，**禁用嵌套 task 防递归**（openclaw `agent-tools.policy.ts` `SUBAGENT_TOOL_DENY_*`）。
- 深度限制（opencode `cfg.subagent_depth ?? 1`，`task.ts:111-117`）。
- 父子通过 `sessions_send`（发消息）+ `sessions_yield`（子完成回交，`sessions-yield-tool.ts:32`）+ `agents_wait`（等待）协调。
- 子会话权限派生（opencode `deriveSubagentSessionPermission`，在默认 ruleset 上 deny `todowrite` 和嵌套 `task`）。
- 子 agent 最终文本输出作为工具结果，包装在 `<task id="..." state="completed"><task_result>...</task_result></task>`（opencode `task.ts:64-79`）。
- 子 agent 复用同一个 `prompt()` 入口递归跑（opencode `ops.prompt({ sessionID: nextSession.id, ... })`，`task.ts:202-213`）。
- 后台模式（opencode `task.ts:97-102,305-308`）：`BackgroundJob.start`，完成时注入结果回父会话。
- **子 agent 上下文缓存安全**（Claude Code `src/utils/forkedAgent.ts`）：`CacheSafeParams` 在 turn 开始时冻结父的渲染系统提示字节，fork-spawn 时重调 `getSystemPrompt()` 可能偏离（GrowthBook 冷→热）导致缓存失效。

#### 2.7.2 技能即文件（三家共识）

- 技能 = `SKILL.md` + YAML frontmatter（`name`, `description`）+ 正文内容。
- 发现模式（opencode `skill/index.ts:21-25`）：`OPENCODE_SKILL_PATTERN = "{skill,skills}/**/SKILL.md"` + `EXTERNAL_SKILL_PATTERN = "skills/**/SKILL.md"`（也扫 `.claude/` 和 `.agents/` 目录）。
- **系统提示里只列"有哪些技能"**（opencode `session/system.ts:105-117` 用 verbose 描述列目录），模型用 read 工具按需加载内容（openclaw `skill-contract.ts:71-90` 的 `<available_skills>` XML 块）。
- **不要把技能做成工具**——"模型知道有什么"和"模型能做什么"分离（openclaw `wrapReadToolWithSkillContent` in `core-coding-tools.ts:167` 包装 read 工具加载技能内容）。
- codex 的技能是 TOML 定义（`codex-rs/skills/` crate），可声明 MCP 依赖（`core/src/mcp_skill_dependencies.rs` `maybe_prompt_and_install_mcp_dependencies`）。

#### 2.7.3 内置 agent 角色

采纳 opencode 的内置 agent（`agent/agent.ts:140-265`）：
| Agent | 角色 | 工具限制 |
|---|---|---|
| `build` | 默认主 agent | 全访问 |
| `general` | 子 agent，复杂多步研究 | — |
| `explore` | 子 agent，代码库搜索 | 只 grep/glob/bash/webfetch/websearch/read |
| `compaction`/`title`/`summary` | 隐藏内部 agent | 内部用 |
- 用户 agent 从配置合并覆盖（opencode `agent.ts:267-294`）。

#### 2.7.4 MCP（Model Context Protocol）

- Java 用 `io.modelcontextprotocol:mcp` Java SDK 做 client。
- 传输：Stdio / SSE / StreamableHTTP（opencode `mcp/index.ts:7-9`），OAuth（`McpOAuthProvider`，`index.ts:24`）。
- MCP 工具包装成 `mcp__<server>__<tool>` 名（Claude Code `normalizeNameForMCP`，`client.ts`），走权限 ask + 截断（opencode `McpCatalog.convertTool`，`session/tools.ts:391-490`）。
- MCP resources 暴露为 `list_mcp_resources`/`list_mcp_resource_templates`/`read_mcp_resource` 工具（动态，仅当 server 暴露 resources 时，opencode `tools.ts:139-386`）。
- MCP instructions 注入系统提示 `<mcp_instructions>` 块（opencode `session/system.ts:119-135`）。
- `ToolsChanged` 通知触发工具重解析（opencode）。
- MCP prewarm（codex `mcp_prewarm.rs`，`session/mcp_prewarm.rs`）会话开始时预热。

#### 2.7.5 Tool Search / 延迟加载（可选，后期）

- Claude Code `ToolSearchTool`（`src/tools/ToolSearchTool/`，`claude.ts:1118-1172`）：工具带 `defer_loading:true`，模型调 `tool_search` 返回 `tool_reference` 块，发现的工具后续请求纳入。
- codex 的 `ToolSpec::ToolSearch` + namespace 工具（`spec_plan.rs`）。
- `extractDiscoveredToolNames(messages)` 从历史回放发现。
- **对"接几十个 MCP 工具"场景必要，MVP 可不做。**

#### 2.7.6 参考对照

| 关注点 | 首选参考 | 次选参考 |
|---|---|---|
| 子 agent / Task | openclaw `sessions-spawn-tool.ts:321` | opencode `task.ts:81` |
| 子 agent 权限派生 | opencode `deriveSubagentSessionPermission` | openclaw `SUBAGENT_TOOL_DENY_*` |
| 子 agent 缓存安全 | Claude Code `forkedAgent.ts` | — |
| 技能即文件 | openclaw `skill-contract.ts:71` | opencode `skill/index.ts` |
| 内置 agent 角色 | opencode `agent.ts:140-265` | — |
| MCP client | opencode `mcp/index.ts` | Claude Code `services/mcp/client.ts` |
| Tool Search | Claude Code `ToolSearchTool` | codex `ToolSpec::ToolSearch` |

---

### 2.8 其他高价值模式

#### 2.8.1 Hooks 系统（采纳 Claude Code）

- Claude Code `src/utils/hooks.ts`（5022 行）：生命周期点跑用户定义命令：
  - `PreToolUse` / `PostToolUse` / `PostToolUseFailure`
  - `SessionStart` / `SessionEnd`
  - `Stop` / `StopFailure`
  - `PreCompact` / `PostCompact`
  - `SubagentStart` / `SubagentStop`
  - `PermissionDenied` / `PermissionRequest`
  - `Setup` / `ConfigChange` / `CwdChanged` / `FileChanged`
- 钩子可改工具输入（`updatedInput`）、否决权限、注入消息、请求提示。
- 配置在 settings，带 `if` 匹配器，`executeHooks` 以 JSON stdin/stdout spawn 命令。
- Java 实现：可配置的事件钩子（运行用户定义的命令/脚本）。

#### 2.8.2 Slash 命令（采纳 codex）

- codex `tui/src/slash_command.rs:12-83`：约 60 命令枚举驱动命令面板（kebab-case 序列化）。
- 注释 `// DO NOT ALPHA-SORT!`——枚举顺序即弹窗显示顺序。
- 每命令 `description()`（`slash_command.rs:87-`）：`/model` `/compact` `/review` `/rename` `/init` `/skills` `/hooks` `/mcp` `/diff` `/usage` `/quit` 等。
- opencode 的 slash 命令模板支持 `$1`/`$2`/`$ARGUMENTS` 位置替换 + `` !`cmd` `` shell 插值（`session/prompt.ts:1356-1481`），命令可作子任务跑。
- web 版移植为枚举驱动命令面板。

#### 2.8.3 分层配置（采纳 codex）

- codex `codex-rs/config/`：分层 TOML：`~/.codex/config.toml`（基础）+ `~/.codex/<profile>.config.toml`（profile v2）+ 项目级 `requirements.toml` + 云配置 + CLI `-c key=value` 覆盖。
- `ConfigLayerStack` 合并，带优先级。
- `requirements.toml` 可强制 `allow_managed_hooks_only=true`（管理员锁定）。
- strict-config 模式（`--strict-config`）未知字段报错（`cli/src/main.rs:1083`）。
- opencode 同样分层：全局（`~/.config/opencode/opencode.json`）+ 项目（`.opencode/opencode.json`）+ 远程 URL 配置，JSONC（支持注释，`jsonc-parser`），`ConfigVariable.substitute` 支持 `${ENV}`/`$ARG` 替换。
- Java 实现：JSONC 分层配置 + `ConfigLayerStack` 合并 + 环境变量替换。

#### 2.8.4 AGENTS.md 指令文件（已在用）

- 项目级指令注入系统提示，固定加载顺序（openclaw `system-prompt.ts:85-90` `CONTEXT_FILE_ORDER`：`agents.md`/`soul.md`/`identity.md`/`user.md`/`tools.md`）。
- opencode 指令文件层级（`session/instruction.ts`）：`AGENTS.md`（首选）+ `CLAUDE.md`（兼容）+ `CONTEXT.md`（废弃）；全局 + per-directory + URL 指令；per-assistant-message 注入带 claim 跟踪（`instruction.ts:70-77`）一次加载。
- codex `core/src/agents_md.rs`：从 cwd 树读 `AGENTS.md`（也读 `CLAUDE.md`/`GEMINI.md`/`.cursor/rules`，`Subcommand::Import`）；`/init` 命令创建 `AGENTS.md`。

#### 2.8.5 turn 级 diff 追踪（采纳 codex）

- codex `core/src/turn_diff_tracker.rs`：累积本轮文件 diff 供 UI 展示。每轮创建（`turn.rs:269-271`），emit `EventMsg::TurnDiff`（`protocol.rs:1445`）。

#### 2.8.6 确定性排序（采纳 openclaw）

- openclaw `AGENTS.md:139`：map/set/registry/plugin list/file/network 结果在发给 model/tool 前确定性排序；旧 transcript 字节保留。prompt 缓存成本节省关键。

#### 2.8.7 结构化输出（采纳 opencode）

- opencode `prompt.ts:1565-1591` `createStructuredOutputTool`：用户请求 JSON-schema 输出时注入 `StructuredOutput` 工具 + `toolChoice:"required"`，模型必须调它终结。跨 provider 的 workaround。

#### 2.8.8 提供商特定转换层（采纳 opencode）

- opencode `packages/opencode/src/provider/transform.ts`：`ProviderTransform.schema`（schema 整形）、`.message`（prompt 改写）、`.options`/`.smallOptions`/`.providerOptions`/`.maxOutputTokens`——集中 provider 怪癖于一处。

---

## 3. 技术选型

| 关注点 | 选型 | 对应参考 |
|---|---|---|
| 运行时 | JDK 21+（虚拟线程做工具并发） | — |
| 异步/流式 | Project Reactor（`Flux<AgentEvent>`） | opencode Effect Stream |
| Web 框架 | Spring WebFlux 或 Helidon Níma（虚拟线程） | opencode Hono |
| 传输 | WebSocket（主）+ SSE（降级） | codex app-server |
| 协议类型 → TS | `typescript-generator` | codex `ts-rs` |
| 存储 | SQLite (JDBC) + Flyway 迁移 | openclaw/codex |
| JSON Schema | `com.networknt:json-schema-validator` | openclaw typebox 双用 |
| LLM HTTP/SSE | reactor-netty / Java HttpClient | openclaw 手写 adapter |
| MCP | `io.modelcontextprotocol:mcp` Java SDK | 三家 |
| 沙箱 | Docker/容器 worker 隔离 | codex（OS 级不可复刻） |
| Diff | `org.eclipse.jgit` + `com.github.difflib` | — |
| 配置 | JSONC 分层 | opencode |
| 构建工具 | Gradle（Kotlin DSL） | — |
| 测试 | JUnit 5 + Reactor `StepVerifier` | — |

---

## 4. 实施计划

### 4.1 阶段划分

| 阶段 | 内容 | 产出 | 参考实现 |
|---|---|---|---|
| P1 | 协议层：`Op`/`Event` 定义 + TS 生成 + WebSocket 传输 | 打通"前端发消息/收流式文本" | codex `protocol` |
| P2 | 最小循环：`apply_patch` + `exec` 两工具 + 一个 provider | 跑通"改文件" | codex |
| P3 | SQLite 持久化 + Part 流式落库 | 会话可恢复 | opencode |
| P4 | 权限 + 审批回环 | 安全可控 | opencode + codex |
| P5 | 上下文压缩（多层） | 长会话不溢出 | Claude Code |
| P6 | 子 agent / 技能 / MCP | 扩展能力 | openclaw/opencode |
| P7 | Hooks / 命令面板 / 配置分层 | 可配置可扩展 | Claude Code/codex |
| P8 | Provider 扩展 + 缓存 + 预热 | 多模型 + 低延迟 | Claude Code/codex |

### 4.2 MVP 验收标准

1. 前端发消息 → 服务端流式返回 assistant 文本（SSE/WebSocket）。
2. 模型可调用 `apply_patch` 改文件、`exec` 跑命令，前端展示 diff 与输出。
3. 高危操作触发审批弹窗，用户 allow/deny 后继续。
4. 会话历史持久化，刷新/重连不丢。
5. 至少接通一个 LLM provider（Anthropic 或 OpenAI）。

### 4.3 风险点

- **沙箱**：codex 的 OS 级沙箱（Seatbelt/Landlock）Java 无法直接复刻，需用容器隔离，但容器有启动开销与文件系统映射复杂度。
- **provider SDK**：Java 生态 LLM SDK 成熟度不如 TS（Vercel `ai` SDK 有 20+ provider），手写适配器工作量大但更可控。
- **prompt 缓存**：Anthropic 的 `cache_control` 语义复杂（breakpoint/TTL/scope），照搬 Claude Code 策略需仔细实现。
- **流式工具执行**：模型流式输出中途就执行工具，涉及部分解析与并发控制，实现复杂度高。

---

## 5. 参考实现速查表

### 5.1 关键文件对照

| 关注点 | Claude Code | opencode | openclaw | codex |
|---|---|---|---|---|
| CLI 入口 | `src/entrypoints/cli.tsx` | `packages/opencode/src/index.ts` | `src/entry.ts` | `codex-rs/cli/src/main.rs:1040` |
| Agent 循环 | `src/query.ts:241` (`queryLoop`) | `src/session/prompt.ts:1081` (`runLoop`) | `agent-core/agent-loop.ts:298` (`runLoop`) | `core/src/session/turn.rs:153` (`run_turn`) |
| 流式处理器 | `src/services/api/claude.ts:752` | `src/session/processor.ts:627` | `agent-loop.ts:530` (`streamAssistantResponse`) | `core/src/client.rs:1861` (`stream`) |
| 工具定义 | `src/Tool.ts:362` | `src/tool/tool.ts:151` (`define`) | `agent-core/types.ts:549` | `tools/src/tool_definition.rs:6` |
| 工具注册 | `src/tools.ts:193` (`getAllBaseTools`) | `src/tool/registry.ts:91` | `src/agents/agent-tools.ts:1020` | `core/src/tools/spec_plan.rs:892` |
| 权限 | `src/utils/permissions/permissions.ts` | `src/permission/index.ts:28` | `src/agents/tool-policy.ts` | `protocol/src/protocol.rs` (`AskForApproval`) |
| 上下文压缩 | `src/services/compact/` | `src/session/compaction.ts` | `src/agents/compaction.ts` | `core/src/compact.rs:111` |
| 会话持久化 | `src/utils/sessionStorage.ts` (JSONL) | `src/session/session.ts` + `message-v2.ts` (SQLite) | `sessions/session-manager.ts` (SQLite) | `codex-thread-store` + `state_db.rs` (SQLite) |
| LLM 编排 | `src/services/api/claude.ts` | `src/session/llm.ts:357` (`stream`) | `packages/ai/src/providers/` | `core/src/client.rs` |
| 子 agent | `src/tools/AgentTool/runAgent.ts` | `src/tool/task.ts:81` | `src/agents/tools/sessions-spawn-tool.ts:321` | `core/src/tools/handlers/multi_agents.rs` |
| 技能 | `src/skills/` | `src/skill/index.ts` | `src/skills/loading/skill-contract.ts` | `codex-rs/skills/` |
| MCP | `src/services/mcp/client.ts` | `src/mcp/index.ts` | `src/agents/agent-bundle-mcp-*.ts` | `core/src/mcp.rs` |
| 快照/撤销 | — | `src/snapshot/index.ts` (git) | — | — |
| 配置 | `src/utils/settings.ts` | `src/config/config.ts` | `~/.openclaw/openclaw.json` | `codex-rs/config/` |
| 指令文件 | `src/utils/processUserInput/` | `src/session/instruction.ts` | `system-prompt.ts:85` | `core/src/agents_md.rs` |

### 5.2 工具集对照

| 工具 | Claude Code | opencode | openclaw | codex |
|---|---|---|---|---|
| Read | FileReadTool | read | read | (走 exec: cat) |
| Write | FileWriteTool | write | write | (走 apply_patch Add) |
| Edit | FileEditTool | edit | edit | — |
| Glob | GlobTool | glob | find | (走 exec: fd) |
| Grep | GrepTool | grep | grep | (走 exec: rg) |
| Bash/Shell | BashTool | shell | bash/exec | exec_command |
| apply_patch | — | apply_patch (GPT only) | apply_patch | apply_patch (文法工具) |
| WebFetch | WebFetchTool | webfetch | web_fetch | — |
| WebSearch | WebSearchTool | websearch | web_search | web_search (hosted) |
| Task/Subagent | AgentTool | task | sessions_spawn | multi_agent_v1/v2 |
| Todo | TodoWriteTool + v2 | todowrite | — | update_plan |
| Skill | SkillTool | skill | (read 工具加载) | (技能即文件) |
| Question | AskUserQuestionTool | question | ask_user | request_user_input |
| LSP | LSPTool | lsp | — | — |

### 5.3 架构模式采纳矩阵

| 模式 | 采纳 | 来源 | 理由 |
|---|---|---|---|
| server+client 分层 | ✅ | opencode | web 版天然需要 |
| Op/Event 协议 + TS 生成 | ✅ | codex | 前后端类型安全 |
| 两层循环（底层+runner） | ✅ | openclaw | 可测试 + 策略可换 |
| 显式状态机 + 恢复点 | ✅ | Claude Code | 恢复路径一等公民 |
| 流式工具执行 | ✅ | Claude Code | 延迟收益 |
| LLMEvent 统一抽象 | ✅ | opencode | provider 可换 |
| 手写 provider 适配器 | ✅ | openclaw | 可控 |
| 多层压缩 + 断路器 | ✅ | Claude Code | 最成熟 |
| Part 流式落库 | ✅ | opencode | 可恢复可分叉 |
| AgentMessage + convertToLlm | ✅ | openclaw | transcript 富于模型所见 |
| Git 快照撤销 | ✅ | opencode | turn 级 undo |
| Ruleset 权限 | ✅ | opencode | 最简洁 |
| 审批模式 + 粘性授权 | ✅ | codex | UX |
| Guardian 自动审批 | ✅ | codex | 减少打断 |
| doom-loop 检测 | ✅ | opencode | 便宜有效 |
| apply_patch 文法工具 | ✅ (MVP) | codex | 极简 |
| 细粒度工具集 | ⬜ (后期) | Claude Code/opencode | 权限更细 |
| 技能即文件 | ✅ | openclaw/opencode | 三家共识 |
| 子 agent 权限派生 | ✅ | opencode | 防递归 |
| Hooks 系统 | ✅ (后期) | Claude Code | 扩展机制 |
| 命令面板 | ✅ | codex | UX |
| 分层配置 | ✅ | codex/opencode | 可配置 |
| Tool Search 延迟加载 | ⬜ (后期) | Claude Code/codex | MCP 多时需要 |
| Prompt 缓存前缀 | ✅ (后期) | Claude Code | 成本 |
| 粘性路由+预热 | ✅ (后期) | codex | 延迟 |

---

## 6. 待确认问题

> 仅在实施阶段遇到且仓库/参考实现无法回答时，再向用户确认。

1. 部署形态：单机本地运行，还是多租户云端服务？（影响沙箱与存储设计）
2. 认证：是否需要接入 OAuth 登录（如 codex 的 ChatGPT 登录、Claude Code 的 Anthropic OAuth）？
3. 默认 provider：Anthropic 优先还是 OpenAI 优先？是否需要支持本地模型（Ollama/LMStudio）？
4. 前端框架：React 还是 SolidJS？（opencode 用 SolidJS，生态相对小；React 生态大）

---

## 修订记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-08-19 | v0.1 | 初稿，基于四参考实现源码洞察 |
