# Codex Agent 源码学习路线

> 一份从浅到深的学习计划，帮助你系统理解 Codex 的 agent 设计和源码架构。

## 阶段一：全局认知

先建立整体画面，不要急着看代码细节。

1. 读 `README.md` 和 `AGENTS.md` — 了解项目定位（OpenAI 的本地 AI coding agent）和开发规范
2. 浏览 `codex-rs/Cargo.toml` workspace 定义 — 理解 ~90 个 crate 的组织方式
3. 重点关注核心 crate 的关系：`cli` → `tui` / `app-server` → `core` → `protocol`

## 阶段二：协议层 — 理解数据怎么流动

从 protocol 开始，因为它定义了所有组件之间的"语言"。

- `codex-rs/protocol/src/protocol.rs` — 核心文件，定义 Op 和 EventMsg 两个枚举

**Op（用户→系统）关键变体：**
- `UserTurn` — 最重要，携带完整上下文（cwd、审批策略、沙箱策略、模型、推理配置）
- `UserInput` — 旧版简化输入（不含 turn 上下文）
- `Interrupt` — 中断当前任务
- `InterAgentCommunication` — agent 间通信
- `OverrideTurnContext` — 更新会话级默认配置
- `ExecApproval` / `PatchApproval` — 用户审批决策
- `Shutdown` — 唯一返回 `true` 退出 submission_loop 的 Op

**EventMsg（系统→用户）关键变体：**
- `TurnStarted` / `TurnComplete` — turn 生命周期边界
- `AgentMessage` / `AgentMessageDelta` — 模型输出（完整/流式增量）
- `ExecCommandBegin` / `ExecCommandEnd` — 命令执行生命周期
- `ExecApprovalRequest` — 请求用户审批
- `GuardianAssessment` — guardian 安全评估结果
- `PatchApplyBegin` / `PatchApplyEnd` — 代码补丁应用
- `Collab*` 系列 — 多 agent 协作事件

## 阶段三：核心引擎 — Agent Loop

这是整个系统的心脏，也是最值得花时间的部分。

### 核心调用链（从入口到 LLM 交互）

```
Codex::spawn() (codex.rs:449)
  → Session::new() → 创建 session
  → tokio::spawn(submission_loop()) → 启动主循环

submission_loop() (codex.rs:4616)
  → match sub.op { ... } → 按 Op 类型分发到 handlers
  → Op::UserInput | Op::UserTurn → handlers::user_input_or_turn()

handlers::user_input_or_turn() (codex.rs:4960)
  → sess.new_turn_with_sub_id() → 创建新 turn
  → sess.steer_input() → 尝试导入活跃 turn
  → sess.spawn_task(RegularTask) → 启动新任务

RegularTask::run() (tasks/regular.rs:36)
  → 发送 TurnStarted 事件
  → loop { run_turn() } → 循环直到无 pending input

run_turn() (codex.rs:5971) — 核心主循环
  → run_pre_sampling_compact() → 预采样压缩
  → loop {
      处理 pending input（用户在模型运行时提交的新消息）
      构建 sampling_request_input（从 history 拉取完整对话）
      run_sampling_request() → 调 LLM + 处理工具调用
      如果 token 超限 → auto compact → continue
      如果 model 需要 follow up → continue
      否则 → 运行 stop hook → break
    }

run_sampling_request() (codex.rs:6760)
  → built_tools() → 构建 ToolRouter（所有可用工具的路由表）
  → build_prompt() → 从 history + tools 构建完整 prompt
  → ToolCallRuntime::new() → 创建工具执行运行时
  → try_run_sampling_request() → 真正调 LLM
  → 重试逻辑：指数退避 + WebSocket→HTTPS 降级

try_run_sampling_request() (codex.rs:7552) — 流式响应处理
  → client_session.stream() → 发起流式请求
  → loop { match stream.next() {
      ResponseEvent::OutputItemDone → handle_output_item_done() → 工具执行
      ResponseEvent::OutputTextDelta → 流式文本输出到 UI
      ResponseEvent::ReasoningSummaryDelta → 推理过程输出
      ResponseEvent::Completed → 更新 token 用量 → break
    }}
  → drain_in_flight() → 等待所有并行工具执行完成
```

### 关键文件

1. `codex-rs/core/src/codex.rs` — 最大最核心，建议分多次读
2. `codex-rs/core/src/tasks/regular.rs` — RegularTask，理解 task 抽象
3. `codex-rs/core/src/tasks/mod.rs` — SessionTask trait 和 spawn_task 逻辑
4. `codex-rs/core/src/client.rs` — ModelClient，HTTP/WebSocket 双通道
5. `codex-rs/core/src/codex_thread.rs` — 单个对话线程的公共 API
6. `codex-rs/core/src/thread_manager.rs` — 多线程/多对话管理

## 阶段四：工具系统 — Agent 的"手脚"

理解 agent 如何执行动作。

### ToolOrchestrator 核心流程 (`tools/orchestrator.rs`)

```
ToolOrchestrator::run()
  │
  ├─ 1. Approval 阶段
  │   ├─ ExecApprovalRequirement::Skip → 直接通过（配置允许）
  │   ├─ ExecApprovalRequirement::Forbidden → 直接拒绝
  │   └─ ExecApprovalRequirement::NeedsApproval → 请求审批
  │       ├─ 如果配置了 Guardian → 路由到 Guardian 子 agent
  │       ├─ 否则 → 请求用户审批
  │       └─ Denied/Abort/TimedOut → 拒绝; Approved → 继续
  │
  ├─ 2. 首次执行（在沙箱中）
  │   ├─ sandbox_mode_for_first_attempt() → 确定沙箱类型
  │   ├─ begin_network_approval() → 网络访问审批
  │   ├─ tool.run(req, attempt, ctx) → 实际执行
  │   ├─ 成功 → 返回结果
  │   └─ SandboxDenied → 进入升级重试
  │
  └─ 3. 升级重试（sandbox escalation）
      ├─ 再次请求审批（除非已批准且无网络问题）
      └─ 以 SandboxType::None 重试执行
```

### 工具目录结构

- `tools/orchestrator.rs` — 审批→沙箱→执行→重试编排
- `tools/sandboxing.rs` — ToolRuntime trait、ApprovalStore（缓存审批决策）、SandboxAttempt
- `tools/router.rs` — ToolRouter，根据工具名路由到对应 handler
- `tools/registry.rs` — ToolHandler trait，工具注册
- `tools/network_approval.rs` — 网络访问审批（Immediate/Deferred 两种模式）
- `tools/handlers/` — 具体工具实现：
  - `shell.rs` — ShellHandler / ShellCommandHandler，命令执行
  - `apply_patch.rs` — 代码补丁应用
  - `mcp.rs` — MCP 工具调用
  - `multi_agents.rs` / `multi_agents_v2/` — 子 agent 派生
  - `dynamic.rs` — 动态工具
  - `plan.rs` — 计划生成
  - `request_permissions.rs` — 权限请求
- `tools/runtimes/` — 工具运行时实现（shell 等）

## 阶段五：多 Agent 系统

这是架构上最有设计感的部分。

### AgentControl — 控制平面 (`agent/control.rs`)

```
AgentControl {
    manager: Weak<ThreadManagerState>,  // Weak 避免循环引用
    state: Arc<AgentRegistry>,          // 共享注册表
}
```

- `spawn_agent()` → 创建新 agent 线程
  - 支持两种 fork 模式：`FullHistory`（完整历史）/ `LastNTurns(n)`（最近 N 轮）
  - 继承父 agent 的 `ShellSnapshot` 和 `ExecPolicyManager`
  - 通过 `send_input()` 向子 agent 发送初始 prompt
  - 深度限制：`depth >= config.agent_max_depth` 时禁用 SpawnCsv/Collab
- `AgentRegistry` (`agent/registry.rs`) — 跟踪存活 agent，管理 spawn slot，分配昵称
- `agent_names.txt` — 预定义的 agent 昵称列表

### Mailbox — 异步消息通信 (`agent/mailbox.rs`)

```
Mailbox (发送端)
  ├─ tx: mpsc::UnboundedSender<InterAgentCommunication>
  ├─ next_seq: AtomicU64  // 单调递增序列号
  └─ seq_tx: watch::Sender<u64>  // 通知订阅者有新消息

MailboxReceiver (接收端)
  ├─ rx: mpsc::UnboundedReceiver
  └─ pending_mails: VecDeque  // 本地缓冲队列
```

- `send()` — 发送消息 + 递增序列号 + 通知 watch 订阅者
- `drain()` — 批量取出所有 pending 消息
- `has_pending_trigger_turn()` — 检查是否有需要唤醒 session 的消息
- `trigger_turn` 标记：当子 agent 完成时，可以唤醒父 agent 开始新 turn

### 关键文件

1. `core/src/agent/control.rs` — AgentControl，spawn/message 的控制平面
2. `core/src/agent/registry.rs` — agent 注册表，深度限制防止无限递归
3. `core/src/agent/mailbox.rs` — 异步消息邮箱
4. `core/src/agent/role.rs` — agent 角色配置
5. `core/src/tools/handlers/multi_agents_v2/` — v2 多 agent 工具实现

## 阶段六：Guardian 安全机制

Codex 的安全设计是一大亮点。核心理念：fail-closed（宁可误拒不可误放）。

### 设计概览

```
ToolOrchestrator 发现 NeedsApproval
  → routes_approval_to_guardian() 检查是否启用 Guardian
  → review_approval_request()
    → run_guardian_review()
      │
      ├─ 1. 发送 GuardianAssessment(InProgress) 事件
      ├─ 2. 选择模型：优先 gpt-5.4，否则用父 turn 的模型
      ├─ 3. build_guardian_review_session_config() → 构建只读沙箱配置
      ├─ 4. guardian_review_session.run_review() → 运行 Guardian 子 agent
      │     ├─ 构建压缩 transcript（最近 40 条，消息 10K token，工具 10K token）
      │     ├─ 注入 guardian policy prompt
      │     ├─ 要求输出结构化 JSON（schema 约束）
      │     └─ 90 秒超时
      │
      └─ 5. 解析结果
          ├─ 成功 → parse_guardian_assessment() → Allow/Deny
          ├─ 超时 → TimedOut → 拒绝（但提示可重试）
          ├─ 解析失败 → Deny（risk=high）
          └─ 执行失败 → Deny（risk=high）
```

### Guardian 输出结构

```rust
struct GuardianAssessment {
    risk_level: GuardianRiskLevel,           // low/medium/high/critical
    user_authorization: GuardianUserAuthorization, // unknown/low/medium/high
    outcome: GuardianAssessmentOutcome,      // allow/deny
    rationale: String,                       // 决策理由
}
```

### 关键设计决策

- Guardian 自身运行在只读沙箱，`approval_policy = never`，不会触发进一步审批
- 推理 effort 设为 `Low`（快速决策，不需要深度推理）
- 支持 trunk session 复用（保持 prompt cache key 稳定）
- 并行审批时 fork trunk，避免互相阻塞
- 被拒绝后，agent 收到明确指令：不得通过变通方式绕过
- 超时后，agent 被告知"不要因超时就认为不安全，可以重试一次或请求用户指导"

### 关键常量 (`guardian/mod.rs`)

- `GUARDIAN_REVIEW_TIMEOUT` = 90 秒
- `GUARDIAN_PREFERRED_MODEL` = "gpt-5.4"
- `GUARDIAN_MAX_MESSAGE_TRANSCRIPT_TOKENS` = 10,000
- `GUARDIAN_MAX_TOOL_TRANSCRIPT_TOKENS` = 10,000
- `GUARDIAN_RECENT_ENTRY_LIMIT` = 40 条

### 关键文件

1. `guardian/mod.rs` — 模块入口，常量定义，GuardianAssessment 结构
2. `guardian/review.rs` — 审查编排，fail-closed 逻辑
3. `guardian/prompt.rs` — prompt 构建（压缩 transcript + 策略注入 + JSON schema）
4. `guardian/review_session.rs` — Guardian session 管理（trunk 复用 + fork）
5. `guardian/approval_request.rs` — 请求格式化
6. `guardian/policy.md` — Guardian 策略文档（值得一读）

## 阶段七：上层 UI 和接入层

到这里核心已经理解了，UI 层相对独立。

### 连接架构

```
┌─────────────┐     InProcessAppServerClient      ┌──────────────┐
│     TUI     │ ←── bounded in-memory channels ──→ │  App-Server  │
│  (ratatui)  │     (JSON-RPC, 零网络开销)          │ (codex-core) │
└─────────────┘                                    └──────────────┘

┌─────────────┐     RemoteAppServerClient          ┌──────────────┐
│  VS Code /  │ ←── HTTP/WebSocket/stdio ────────→ │  App-Server  │
│  Cursor     │     (JSON-RPC over transport)       │ (codex-core) │
└─────────────┘                                    └──────────────┘
```

### TUI (`codex-rs/tui/`)

- `app.rs` — 主应用状态机，处理 `AppEvent` 和 `AppCommand`
- `app_event.rs` — 键盘/鼠标/系统事件定义
- `app_server_session.rs` — 封装 `InProcessAppServerClient`，TUI 与 core 的桥梁
- `chatwidget.rs` — 消息渲染组件
- `bottom_pane/` — 输入框、审批弹窗、footer
- `render/` — 终端渲染管线
- `markdown_render.rs` — Markdown 渲染 + 语法高亮
- `diff_render.rs` — 文件变更可视化
- 基于 ratatui 框架，禁止 `print_stdout`/`print_stderr`（alternate screen 模式）

### App-Server (`codex-rs/app-server/`)

- `in_process.rs` — 进程内运行时，用内存通道替代 socket
  - `InProcessClientHandle` 提供 `request()` / `notify()` / `next_event()`
  - 背压机制：`try_send` + `WouldBlock`，审批请求永不静默丢弃
- `message_processor.rs` — JSON-RPC 消息处理器，统一处理所有 transport
- `codex_message_processor.rs` — Codex 特定的消息处理逻辑
- `transport/` — 传输层实现（WebSocket、stdio、remote control）
- `config_api.rs` — 配置读写 API
- `fs_api.rs` — 文件系统 API
- 基于 Axum 框架

### 关键设计

- TUI 和 IDE 共享同一个 App-Server 协议，行为完全一致
- In-process 模式避免了序列化/反序列化和网络开销
- `codex-app-server-protocol` crate 定义了完整的 JSON-RPC 协议（v1/v2）
- `codex-app-server-client` crate 封装了客户端逻辑（InProcess / Remote）

## 阶段八：支撑系统（按兴趣选读）

### 沙箱系统 (`codex-rs/sandboxing/`)

跨平台沙箱实现，是安全执行的基础设施。

```
SandboxType 枚举:
  ├─ MacosSeatbelt   — macOS: sandbox-exec + .sbpl 策略文件
  ├─ LinuxSeccomp    — Linux: Landlock + Seccomp（通过 codex-linux-sandbox 二进制）
  ├─ WindowsRestrictedToken — Windows: 受限令牌
  └─ None            — 无沙箱（升级重试时使用）
```

- `manager.rs` — `SandboxManager`，根据平台和策略选择沙箱类型
- `seatbelt.rs` + `*.sbpl` — macOS Seatbelt 策略（基础策略 + 网络策略）
- `landlock.rs` — Linux Landlock 文件系统隔离
- `bwrap.rs` — Linux bubblewrap 容器（可选）
- `policy_transforms.rs` — 策略转换，计算有效的文件系统/网络沙箱策略
- `get_platform_sandbox()` — 自动检测当前平台可用的沙箱类型

### MCP 系统 (`codex-rs/codex-mcp/`)

Model Context Protocol 连接管理，让 agent 能调用外部工具。

- `mcp_connection_manager.rs` — 核心管理器
  - 每个 MCP server 一个 `RmcpClient`，按 server name 索引
  - `list_all_tools()` — 聚合所有 server 的工具，返回全限定名 → Tool 的 map
  - 支持 elicitation（MCP server 向用户请求输入）
  - 支持 OAuth 认证
- `mcp_tool_names.rs` — 工具名限定（`server_name.tool_name`）
- `mcp/` — MCP 配置、server 发现、工具过滤

### 配置系统 (`codex-rs/config/`)

分层 TOML 配置，支持多级覆盖。

- `config_toml.rs` — `ConfigToml` 结构，从 `~/.codex/config.toml` 反序列化
  - 模型选择、provider、context window、审批策略、沙箱策略
  - MCP server 配置、Skills 配置、Plugin 配置
  - 支持 JSON Schema 生成（`schemars`）
- `permissions_toml.rs` — 权限配置
- `merge.rs` — 配置合并逻辑（defaults → system → user → CLI overrides）
- `constraint.rs` — 配置约束验证
- `schema.rs` — JSON Schema 导出（`just write-config-schema`）

### 其他值得了解的 crate

- `codex-rs/skills/` — 技能定义和元数据
- `codex-rs/core/src/skills/` — 技能加载、注入、依赖解析
- `codex-rs/exec/` — 非交互式执行模式
- `codex-rs/execpolicy/` — 执行策略评估（Starlark 规则引擎）
- `codex-rs/rollout/` — 事件持久化和回放
- `codex-rs/state/` — SQLite 线程状态数据库
- `codex-rs/login/` — 认证（ChatGPT OAuth、API key、device code）
- `codex-rs/otel/` — OpenTelemetry 可观测性
- `codex-rs/network-proxy/` — 网络代理管理

## 学习建议

- `codex.rs` 是最大也最核心的文件，建议分多次读，先抓主流程再看分支逻辑
- 关注几个关键设计模式：Tokio async/await、Arc/Weak 避免循环引用、Mailbox 模式、fail-closed 安全策略
- 可以用 `git log --oneline codex-rs/core/src/guardian/` 这类命令看某个模块的演进历史，理解设计决策的背景

## 附录：横切面主题

### Context 管理与 Compact 系统

agent 能持续长对话的关键机制。

**ContextManager** (`core/src/context_manager/history.rs`):
```rust
struct ContextManager {
    items: Vec<ResponseItem>,          // 有序对话历史（最旧在前）
    history_version: u64,              // 每次 compact/rollback 递增
    token_info: Option<TokenUsageInfo>, // token 用量追踪
    reference_context_item: Option<TurnContextItem>, // 上下文基线，用于 diff
}
```

**Compact 流程** (`core/src/compact.rs`):
```
run_turn() 检测 token 超限
  → run_auto_compact()
    → 把当前 history + SUMMARIZATION_PROMPT 发给 LLM
    → LLM 返回压缩摘要
    → 用摘要替换旧 history
    → 如果仍超限 → 从头部裁剪最旧 item → 重试
    → 重置 WebSocket session（因为 history 变了）
```

关键常量：
- `COMPACT_USER_MESSAGE_MAX_TOKENS` = 20,000
- 支持两种触发：Auto（token 超限）和 Manual（用户 `/compact`）
- 两种 phase：PreTurn（turn 开始前）和 MidTurn（turn 进行中）
- `InitialContextInjection`：MidTurn 时注入到最后一条用户消息之前

### Exec Policy 规则引擎

决定哪些命令可以自动执行、哪些需要审批。

**Policy** (`execpolicy/src/policy.rs`):
```rust
struct Policy {
    rules_by_program: MultiMap<String, RuleRef>,  // 按程序名索引
    network_rules: Vec<NetworkRule>,               // 网络访问规则
    host_executables_by_name: HashMap<String, Arc<[AbsolutePathBuf]>>,
}
```

**规则匹配**:
- `PrefixRule` — 命令前缀匹配（如规则 `["git", "commit"]` 匹配 `git commit -m "xxx"`）
- `PatternToken::Single` — 精确匹配单个 token
- `PatternToken::Alts` — 多选匹配（如 `["add", "commit"]` 匹配其中任一）
- `Decision` — Allow（自动执行）/ Deny（拒绝）/ Ask（需要审批）
- 支持 heuristics fallback 作为兜底策略
- 支持运行时动态追加规则（`blocking_append_allow_prefix_rule`）

### Hooks 生命周期钩子

用户可配置的 shell 命令，在关键生命周期点触发。

**5 个钩子点** (`hooks/src/events/`):
| 钩子 | 触发时机 | 可以做什么 |
|------|---------|-----------|
| `session_start` | session 初始化时 | 注入额外上下文 |
| `user_prompt_submit` | 用户提交 prompt 后 | 拦截/修改用户输入 |
| `pre_tool_use` | 工具执行前 | 审批/拦截工具调用 |
| `post_tool_use` | 工具执行后 | 记录/审计工具结果 |
| `stop` | turn 结束时 | 注入后续指令、触发继续 |

**执行模型**:
- `HookResult::Success` — 继续
- `HookResult::FailedContinue` — 失败但继续执行后续 hook
- `HookResult::FailedAbort` — 失败并中止整个操作
- Hook payload 是 JSON 序列化的（session_id、cwd、triggered_at、hook_event）
- `ClaudeHooksEngine` 从 config layer stack 加载 hook 定义，通过 shell 执行

### Instructions 系统指令

控制 agent 行为的指令层次。

**指令来源和优先级**:
```
base_instructions (模型默认指令)
  ↓ 被覆盖
config.base_instructions (用户配置覆盖)
  ↓ 叠加
developer_instructions (开发者指令)
  ↓ 叠加
user_instructions (来自 AGENTS.md 文件)
  ↓ 叠加
skill_instructions (来自激活的 skill)
```

- `UserInstructions` — 从项目目录的 `AGENTS.md` 加载，用 fragment marker 包裹
- `SkillInstructions` — 从 skill 定义加载，包含 name、path、contents
- Fragment marker 机制允许 context diff：只发送变化的部分，而非每次全量注入

### 工具执行流程：从 LLM 响应到实际执行

完整追踪一次工具调用的生命周期。

```
try_run_sampling_request() 收到 ResponseEvent::OutputItemDone(item)
  │
  ├─ handle_output_item_done() (stream_events_utils.rs:205)
  │   → ToolRouter::build_tool_call(item) — 解析 ResponseItem 为 ToolCall
  │     ├─ FunctionCall → 检查是否 MCP 工具 → ToolPayload::Function 或 Mcp
  │     ├─ LocalShellCall → ToolPayload::LocalShell
  │     ├─ CustomToolCall → ToolPayload::Custom
  │     ├─ ToolSearchCall → ToolPayload::ToolSearch
  │     └─ 其他 → None（不是工具调用）
  │
  │   → tool_runtime.handle_tool_call(call) — 异步执行
  │     加入 FuturesOrdered<in_flight>
  │
  ├─ ToolCallRuntime::handle_tool_call() (tools/parallel.rs:55)
  │   → tokio::spawn + tokio::select! {
  │       cancelled → aborted_response（"aborted by user"）
  │       result → {
  │           并行控制：RwLock
  │             supports_parallel → read lock（可并发）
  │             !supports_parallel → write lock（独占串行）
  │           router.dispatch_tool_call_with_code_mode_result()
  │       }
  │     }
  │   → 错误转为 tool output 写回 history（模型可见），不会 panic
  │
  ├─ ToolRouter::dispatch → ToolRegistry → ToolHandler::invoke()
  │   → ToolOrchestrator::run() — 审批→沙箱→执行→重试（见阶段四）
  │
  └─ 工具结果作为 ResponseInputItem 写回 history
      → run_turn 主循环的下一次迭代将结果发给 LLM
```

### 三大核心数据结构

理解这三个结构是读懂整个系统的基础。

**Session** (`codex.rs:825`) — 会话级状态，生命周期 = 整个对话：
```rust
struct Session {
    conversation_id: ThreadId,           // 唯一会话 ID
    tx_event: Sender<Event>,             // 事件发送通道（→ UI）
    state: Mutex<SessionState>,          // 可变状态（history、config 等）
    active_turn: Mutex<Option<ActiveTurn>>, // 当前活跃 turn
    mailbox: Mailbox,                    // agent 间通信（发送端）
    mailbox_rx: Mutex<MailboxReceiver>,  // agent 间通信（接收端）
    guardian_review_session: GuardianReviewSessionManager,
    services: SessionServices,           // 所有共享服务
    features: ManagedFeatures,           // 功能开关（session 内不变）
    // ...
}
```

**TurnContext** (`codex.rs:866`) — turn 级状态，生命周期 = 单次 turn：
```rust
struct TurnContext {
    sub_id: String,                      // turn 唯一 ID
    config: Arc<Config>,                 // 配置快照
    model_info: ModelInfo,               // 模型信息
    provider: ModelProviderInfo,         // 提供商信息
    cwd: AbsolutePathBuf,               // 工作目录
    approval_policy: Constrained<AskForApproval>,  // 审批策略
    sandbox_policy: Constrained<SandboxPolicy>,    // 沙箱策略
    reasoning_effort: Option<ReasoningEffortConfig>,
    collaboration_mode: CollaborationMode,
    tools_config: ToolsConfig,           // 工具配置
    dynamic_tools: Vec<DynamicToolSpec>, // 动态工具
    session_telemetry: SessionTelemetry, // 遥测
    // ...
}
```

**SessionServices** (`state/service.rs:32`) — 共享服务注册表：
```rust
struct SessionServices {
    model_client: ModelClient,                    // LLM 通信
    auth_manager: Arc<AuthManager>,               // 认证
    models_manager: Arc<ModelsManager>,            // 模型目录
    exec_policy: Arc<ExecPolicyManager>,           // 执行策略
    mcp_connection_manager: Arc<RwLock<McpConnectionManager>>, // MCP
    agent_control: AgentControl,                   // 多 agent 控制
    hooks: Hooks,                                  // 生命周期钩子
    rollout: Mutex<Option<RolloutRecorder>>,        // 持久化
    tool_approvals: Mutex<ApprovalStore>,           // 审批缓存
    guardian_rejections: Mutex<HashMap<...>>,       // Guardian 拒绝记录
    network_proxy: Option<StartedNetworkProxy>,     // 网络代理
    skills_manager: Arc<SkillsManager>,            // 技能管理
    plugins_manager: Arc<PluginsManager>,           // 插件管理
    analytics_events_client: AnalyticsEventsClient, // 分析
    session_telemetry: SessionTelemetry,           // 遥测
    // ...
}
```

### ModelClient — LLM 通信层

Codex 与 OpenAI API 交互的核心，支持 WebSocket 和 HTTP 双通道。

**两层抽象**:
```
ModelClient (session 级别)
  ├─ auth_manager — 认证管理
  ├─ provider — 模型提供商信息
  ├─ conversation_id — 会话 ID
  ├─ cached_websocket_session — WebSocket 连接缓存
  └─ disable_websockets — 降级标记

ModelClientSession (turn 级别)
  ├─ websocket_session — 当前 WebSocket 连接 + 上次请求/响应
  └─ turn_state: OnceLock<String> — sticky routing token
```

**stream() 双通道流程**:
```
stream(prompt, model_info, ...)
  ├─ WebSocket 启用？
  │   ├─ 是 → stream_responses_websocket()
  │   │   ├─ 支持增量请求（只发 diff，不重发完整 history）
  │   │   ├─ 成功 → 返回 ResponseStream
  │   │   └─ FallbackToHttp → 降级到 HTTP
  │   └─ 否（已降级或不支持）
  └─ stream_responses_api() → HTTP SSE 流式请求
```

**关键设计**:
- `x-codex-turn-state` header：服务端返回，客户端在同一 turn 内回传，实现 sticky routing
- `window_generation`：compact 后递增，触发 WebSocket session 重置
- 重试耗尽后永久降级到 HTTP（`force_http_fallback`），不再尝试 WebSocket
- WebSocket 连接在 turn 间缓存复用（`cached_websocket_session`）

### Prompt 构建

`build_prompt()` (`codex.rs:6719`) 组装发给 LLM 的完整请求：

```rust
Prompt {
    input: Vec<ResponseItem>,       // 完整对话历史（从 ContextManager 拉取）
    tools: Vec<ToolSpec>,           // 模型可见的工具规格（从 ToolRouter 获取）
    parallel_tool_calls: bool,      // 是否支持并行工具调用（模型能力决定）
    base_instructions: BaseInstructions, // 系统指令
    personality: Personality,       // 人格设定
    output_schema: Option<Value>,   // 可选的 JSON Schema 约束最终输出
}
```

工具规格来自 `ToolRouter::model_visible_specs()`，延迟加载的 dynamic tools 会被过滤。

### Rollout 持久化

对话历史的持久化机制，支持 session 恢复和离线检查。

**RolloutRecorder** (`rollout/src/recorder.rs`):
- JSONL 格式，每行一个 `RolloutItem`（ResponseItem / EventMsg / SessionMeta / TurnContext）
- 通过 `mpsc` channel 异步写入，不阻塞主循环
- 文件路径：`~/.codex/sessions/rollout-{timestamp}-{uuid}.jsonl`
- 支持 Create（新建）和 Resume（从已有文件恢复）
- 配合 SQLite state db（`codex-rs/state/`）存储线程元数据
- `EventPersistenceMode` 控制哪些事件需要持久化

可以用 `jq` 直接检查 rollout 文件：
```bash
jq -C . ~/.codex/sessions/rollout-*.jsonl
```

### 启动流程：从 CLI 到 Session 就绪

完整的启动链路，帮助理解各层如何串联。

```
codex (binary)
  │
  ├─ cli/src/main.rs: main()
  │   → arg0_dispatch_or_else()  // 支持多二进制名（codex-linux-sandbox 等）
  │   → cli_main()
  │       → MultitoolCli::parse()  // clap 解析命令行
  │       → match subcommand {
  │           None → run_interactive_tui()
  │                   → codex_tui::run_main()
  │                     → 初始化 tracing、terminal 检测
  │                     → InProcessAppServerClient::start()  // 启动进程内 app-server
  │                     → App::new() → 进入 TUI 事件循环
  │
  │           Exec → codex_exec::run_main()     // 非交互模式
  │           AppServer → codex_app_server::run_main_with_transport()  // IDE 模式
  │           McpServer → codex_mcp_server::run_main()
  │           Login/Logout → 认证流程
  │         }
  │
  ├─ app-server/src/in_process.rs: start()
  │   → MessageProcessor::new()  // JSON-RPC 消息处理器
  │   → initialize/initialized 握手
  │   → 返回 InProcessClientHandle
  │
  ├─ 用户发送第一条消息 (Op::UserTurn)
  │   → app-server 转发到 core
  │   → Codex::spawn() (core/src/codex.rs:449)
  │       → 加载 config、auth、models、skills、plugins、MCP
  │       → Session::new()
  │       → tokio::spawn(submission_loop())  // 启动主循环
  │       → 返回 CodexSpawnOk { codex, thread_id }
  │
  └─ submission_loop 开始接收 Op
      → 进入阶段三描述的 agent loop
```

### Ghost Snapshot — Undo 系统

基于 git 的文件系统快照，让 agent 的修改可以一键撤销。

**工作流程**:
```
Turn 开始
  → GhostSnapshotTask (tasks/ghost_snapshot.rs)
    → spawn_blocking { create_ghost_commit() }
    → 创建一个不在任何分支上的 git commit（"ghost commit"）
    → 将 ResponseItem::GhostSnapshot { ghost_commit } 写入 history
    → 超过 240 秒发出警告（可能有大文件需要 .gitignore）

用户执行 /undo
  → UndoTask (tasks/undo.rs)
    → 从 history 中倒序查找最近的 GhostSnapshot
    → spawn_blocking { restore_ghost_commit() } — 恢复文件系统状态
    → 从 history 中移除该 snapshot item
    → 发送 UndoCompleted 事件
```

**关键设计**:
- Ghost commit 不污染 git 分支历史
- 快照在后台异步创建，不阻塞 turn 执行
- 支持 `ghost_snapshot.disable_warnings` 配置
- 大文件检测：报告被忽略的大文件和目录

### 模板系统 — Agent 的"灵魂"

`core/templates/` 目录下的模板定义了 Codex 的行为和人格。

**模型指令** (`model_instructions/gpt-5.2-codex_instructions_template.md`):
- 开头："You are Codex, a coding agent based on GPT-5"
- `{{ personality }}` 占位符 — 运行时注入人格模板
- 详细的格式化规则（GitHub Markdown、不用 emoji、不用嵌套列表、文件引用格式）
- 编辑约束（默认 ASCII、不 revert 用户改动、不用 `git reset --hard`、非交互 git）
- Plan tool 指导（简单任务跳过、不做单步计划）
- 前端任务特殊指导（避免"AI slop"、大胆设计、不要紫色偏好）

**人格模板** (`personalities/`):
- `friendly` — 温暖、鼓励、协作，优化团队士气，"NEVER curt or dismissive"
- `pragmatic` — 务实、高效、严谨，"quiet joy"，承认好工作但避免 cheerleading

**Orchestrator 模板** (`agents/orchestrator.md`) — 多 agent 编排指令：
- "Prefer multiple sub-agents to parallelize your work"
- "When sub-agents are running, wait for them before yielding"
- "Your only role becomes to coordinate them"
- 用户更新规范：长时间工作时发 heads-down note，恢复时总结发现

**Compact prompt** (`compact/prompt.md`):
- 极简指令："Create a handoff summary for another LLM that will resume the task"
- 要求：进度、决策、约束、下一步、关键数据

**Guardian Policy** (`guardian/policy.md`) — 4 大风险类别：
| 类别 | 风险等级 | 示例 |
|------|---------|------|
| Data Exfiltration | high/critical | 向不可信目标发送私有数据、凭证 |
| Credential Probing | high | 从浏览器 profile 提取 token/cookie |
| Persistent Security Weakening | high/critical | 修改安全设置、过宽权限 |
| Destructive Actions | high/critical | 删除数据、force push protected branch |

关键规则：sandbox retry 本身不算可疑；本地文件操作通常 low risk；git 操作只影响用户自己的 feature branch 时是 medium。

### Feature Flags 系统

集中式功能开关，控制整个系统的行为。

**生命周期**: `UnderDevelopment` → `Experimental` → `Stable` → `Deprecated` → `Removed`

**关键 Feature 枚举** (`features/src/lib.rs`):
- `GhostCommit` — ghost snapshot undo 系统（Stable）
- `ShellTool` — shell 工具（Stable）
- `GuardianApproval` — Guardian 自动审批
- `MultiAgentV2` — v2 多 agent 路由
- `Collab` — 协作工具
- `CodexHooks` — 生命周期钩子
- `Apps` / `Plugins` — 应用和插件系统
- `RealtimeConversation` — 实时语音对话
- `ResponsesWebsockets` / `ResponsesWebsocketsV2` — WebSocket 传输

**Features 结构体**:
```rust
struct Features {
    enabled: BTreeSet<Feature>,           // 启用的功能集合
    legacy_usages: BTreeSet<LegacyFeatureUsage>, // 旧版兼容追踪
}
```

- Session 内不变（`ManagedFeatures` 包装）
- 支持 config TOML、CLI `--enable/--disable`、legacy 别名
- `Experimental` 阶段的功能可通过 TUI `/experimental` 菜单切换

### 关键设计模式总结

| 模式 | 用途 | 示例 |
|------|------|------|
| `Arc<T>` + `Weak<T>` | 共享所有权 + 避免循环引用 | AgentControl → ThreadManagerState |
| `async_channel` | 跨 task 通信 | submission_loop 的 tx_sub/rx_sub |
| `watch::channel` | 状态广播 | AgentStatus、Mailbox 序列号 |
| `CancellationToken` | 协作式取消 | turn 中断、guardian 超时 |
| `FuturesOrdered` | 并行工具执行 + 有序结果 | try_run_sampling_request 的 in_flight |
| fail-closed | 安全默认 | Guardian 超时/解析失败 = 拒绝 |
| sandbox escalation | 渐进式权限 | 沙箱拒绝 → 请求审批 → 无沙箱重试 |
| trunk + fork | session 复用 | Guardian review session 缓存 |
| prefix cache 友好 | 性能优化 | compact 从头部裁剪、history 追加 |

## 架构总览

```
┌───────────────────────────────────────────────────┐
│                 User Interface Layer              │
│   ┌────────────┐         ┌─────────────────────┐  │
│   │    TUI     │         │  App-Server (HTTP/WS)│  │
│   │ (ratatui)  │         │  (Axum)              │  │
│   └─────┬──────┘         └──────────┬───────────┘  │
└─────────┼───────────────────────────┼──────────────┘
          └─────────────┬─────────────┘
                        │
         ┌──────────────▼──────────────┐
         │      Core Session (codex.rs)│
         │  ┌────────────────────────┐ │
         │  │  Codex (Orchestrator)  │ │
         │  └────────────────────────┘ │
         │  ┌────────────────────────┐ │
         │  │  ModelClient (LLM API) │ │
         │  └────────────────────────┘ │
         │  ┌────────────────────────┐ │
         │  │  ToolOrchestrator      │ │
         │  │  (Approval → Sandbox   │ │
         │  │   → Execute → Retry)   │ │
         │  └────────────────────────┘ │
         │  ┌────────────────────────┐ │
         │  │  Guardian (Sub-Agent)  │ │
         │  │  (Risk Assessment)     │ │
         │  └────────────────────────┘ │
         │  ┌────────────────────────┐ │
         │  │  Agent System          │ │
         │  │  (Control / Registry   │ │
         │  │   / Mailbox)           │ │
         │  └────────────────────────┘ │
         └──────────────────────────────┘
                        │
         ┌──────────────▼──────────────┐
         │     External Services       │
         │  - OpenAI Responses API     │
         │  - MCP Servers              │
         └─────────────────────────────┘
```
