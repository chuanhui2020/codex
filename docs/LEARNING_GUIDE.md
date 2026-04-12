# Codex Agent 源码学习路线

> 一份从浅到深的学习计划，帮助你系统理解 Codex 的 agent 设计和源码架构。

## 阶段一：全局认知

先建立整体画面，不要急着看代码细节。

1. 读 `README.md` 和 `AGENTS.md` — 了解项目定位（OpenAI 的本地 AI coding agent）和开发规范
2. 浏览 `codex-rs/Cargo.toml` workspace 定义 — 理解 ~90 个 crate 的组织方式
3. 重点关注核心 crate 的关系：`cli` → `tui` / `app-server` → `core` → `protocol`

## 阶段二：协议层 — 理解数据怎么流动

从 protocol 开始，因为它定义了所有组件之间的"语言"。

- `codex-rs/protocol/src/` — Event 和 Op 的定义
- 搞清楚 `Op`（用户操作）和 `Event`（系统事件）这两个核心枚举，后面所有模块都围绕它们转

## 阶段三：核心引擎 — Agent Loop

这是整个系统的心脏，也是最值得花时间的部分。

1. `codex-rs/core/src/codex.rs`（~309KB）— 从 `Codex::spawn()` 入口开始，理解：
   - Session 初始化流程
   - Turn 生命周期（用户输入 → 模型调用 → 工具执行 → 响应）
   - 事件发射机制
2. `codex-rs/core/src/client.rs` — ModelClient，LLM API 交互层
   - HTTP 和 WebSocket 双通道
   - 流式响应处理
3. `codex-rs/core/src/codex_thread.rs` — 单个对话线程的公共 API
4. `codex-rs/core/src/thread_manager.rs` — 多线程/多对话管理

## 阶段四：工具系统 — Agent 的"手脚"

理解 agent 如何执行动作。

1. `codex-rs/core/src/tools/orchestrator.rs` — 工具编排核心：审批 → 沙箱选择 → 执行 → 重试
2. `codex-rs/core/src/tools/handlers/` 目录下逐个看：
   - `shell.rs` — shell 命令执行
   - `apply_patch.rs` — 代码补丁
   - `mcp.rs` — MCP 工具调用
   - `multi_agents.rs` / `multi_agents_v2.rs` — 子 agent 派生
3. `codex-rs/core/src/tools/sandboxing.rs` — 沙箱策略和权限管理
4. `codex-rs/core/src/tools/network_approval.rs` — 网络访问审批

## 阶段五：多 Agent 系统

这是架构上最有设计感的部分。

1. `codex-rs/core/src/agent/control.rs` — AgentControl，agent 的控制平面（派生、消息传递）
2. `codex-rs/core/src/agent/registry.rs` — agent 注册表，深度限制防止无限递归
3. `codex-rs/core/src/agent/mailbox.rs` — 异步消息邮箱，agent 间通信机制
4. 回头看 `multi_agents_v2.rs`，理解 fork 模式（FullHistory vs LastNTurns）

## 阶段六：Guardian 安全机制

Codex 的安全设计是一大亮点。

1. `codex-rs/core/src/guardian/review.rs` — 审查编排
2. `codex-rs/core/src/guardian/prompt.rs` — guardian prompt 构建（压缩上下文 + 策略注入）
3. `codex-rs/core/src/guardian/review_session.rs` — guardian 作为子 agent 运行，90 秒超时，fail-closed
4. `codex-rs/core/src/guardian/approval_request.rs` — 请求格式化

关键设计理念：guardian 超时 = 拒绝执行，宁可误拒不可误放。

## 阶段七：上层 UI 和接入层

到这里核心已经理解了，UI 层相对独立。

- `codex-rs/tui/` — 基于 ratatui 的终端 UI
- `codex-rs/app-server/` — 基于 Axum 的 HTTP/WebSocket 服务，给 VS Code 等 IDE 用

## 阶段八：支撑系统（按兴趣选读）

- `codex-rs/config/` + `core/src/config/` — 分层配置系统
- `codex-rs/core/src/skills/` — 技能加载和依赖解析
- `codex-rs/sandboxing/` — 跨平台沙箱实现（Linux landlock、macOS、Windows）
- `codex-rs/exec/` + `codex-rs/execpolicy/` — 执行策略评估
- `codex-rs/codex-mcp/` — MCP 协议实现

## 学习建议

- `codex.rs` 是最大也最核心的文件，建议分多次读，先抓主流程再看分支逻辑
- 关注几个关键设计模式：Tokio async/await、Arc/Weak 避免循环引用、Mailbox 模式、fail-closed 安全策略
- 可以用 `git log --oneline codex-rs/core/src/guardian/` 这类命令看某个模块的演进历史，理解设计决策的背景

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
