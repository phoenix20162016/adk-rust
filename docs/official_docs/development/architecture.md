# ADK-Rust Architecture & Design Philosophy

本文档深入剖析 ADK-Rust 的软件架构、核心设计思想与关键设计决策。

---

## 1. 整体架构：五层分层模型

ADK-Rust 采用严格的五层分层架构，依赖方向始终向下，层间通过 trait 边界解耦：

```
┌─────────────────────────────────────────────────────┐
│  Application Layer    CLI / REST+A2A / Studio (外部)  │
├─────────────────────────────────────────────────────┤
│  Execution Layer     Runner / Graph / Eval / Bench   │
├─────────────────────────────────────────────────────┤
│  Agent Layer         LlmAgent / Sequential / Parallel │
│                     Loop / Conditional / Custom      │
├─────────────────────────────────────────────────────┤
│  Service Layer       Models / Tools / Sessions / Memory│
│                     Plugin / Auth / Telemetry         │
├─────────────────────────────────────────────────────┤
│  Foundation          adk-core (Agent, Tool, Llm,     │
│                     Session, Event, Content traits)  │
└─────────────────────────────────────────────────────┘
```

**设计原则**：上层可以依赖下层，下层绝不能感知上层的存在。`adk-core` 是整个框架的"地基"——只定义 trait 和基础类型，不包含任何具体实现。

---

## 2. 核心 Trait 体系：四根支柱

### 2.1 `Agent` trait（`adk-core/src/agent.rs`）

```rust
#[async_trait]
pub trait Agent: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn sub_agents(&self) -> &[Arc<dyn Agent>];
    async fn run(&self, ctx: Arc<dyn InvocationContext>) -> Result<EventStream>;
}
```

**设计思想**：
- **统一接口**：无论是单个 LLM agent、顺序执行的 workflow，还是有向图节点，全部实现同一个 trait。Runner 不需要区分 agent 类型。
- **`sub_agents()`**：支持复合模式（Composite Pattern）。`SequentialAgent`、`ParallelAgent` 等容器型 agent 通过此方法暴露子 agent 树，使框架能遍历整个 agent 树来执行注册工具发现、权限检查等操作。
- **`supports_agent_transfer()`**：默认 `true`，但 workflow agent 覆写为 `false`。这确保多轮对话时，runner 不会跳过 workflow 的某个子 agent，而是始终从 workflow 根节点重新执行。
- **返回 `EventStream`**：不返回单一值，而是返回一个异步流，支持 token-by-token streaming、多工具调用、状态变更通知、agent 转移信号等。

### 2.2 `Tool` trait（`adk-core/src/tool.rs`）

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> Option<Value> { None }
    fn response_schema(&self) -> Option<Value> { None }
    fn is_long_running(&self) -> bool { false }
    fn is_read_only(&self) -> bool { false }
    fn is_concurrency_safe(&self) -> bool { false }
    fn required_scopes(&self) -> &[&str] { &[] }
    async fn execute(&self, ctx: Arc<dyn ToolContext>, args: Value) -> Result<Value>;
}
```

**设计思想**：
- **元数据驱动调度**：`is_read_only()` 和 `is_concurrency_safe()` 两个标记让 `ToolExecutionStrategy::Auto` 在运行时决定哪些工具可以并发执行，无需在调用方硬编码并行策略。
- **声明式权限**：`required_scopes()` 配合 `adk-auth` 的 RBAC 实现，在执行前静态检查权限，而非在 execute 内部做运行时判断。
- **Schema 自描述**：`parameters_schema()` 和 `declaration()` 方法让工具能将自身的参数契约暴露给 LLM，消除"幻影工具"（agent 在 prompt 中看到工具名但运行时找不到对应实现）。

### 2.3 `Llm` trait（`adk-core/src/model.rs`）

```rust
#[async_trait]
pub trait Llm: Send + Sync {
    fn name(&self) -> &str;
    async fn generate_content(&self, request: LlmRequest, stream: bool)
        -> Result<LlmResponseStream>;
    fn schema_adapter(&self) -> Box<dyn SchemaAdapter> { ... }
}
```

**设计思想**：
- **提供者无关**：Agent 和 Runner 只依赖 `Arc<dyn Llm>`，不感知具体提供者（Gemini、OpenAI、Anthropic、Ollama）。切换提供者只需替换 `Arc<dyn Llm>` 的实例，agent 代码无需改动。
- **SchemaAdapter**：不同 LLM 提供者对 JSON Schema 的要求不同（Gemini 不支持 `additionalProperties`，OpenAI 有 strict/non-strict 模式）。`SchemaAdapter` trait 在编译时/运行时做 provider-aware 的 schema 归一化，调用方无需关心差异。
- **统一 streaming 接口**：`LlmResponseStream` 对所有提供者返回相同的 `Stream<Item = Result<LlmResponse>>`，Runner 只需一个 loop 消费所有响应。

### 2.4 `Session` trait（`adk-core/src/context.rs`）

```rust
pub trait Session: Send + Sync {
    fn id(&self) -> &str;
    fn app_name(&self) -> &str;
    fn user_id(&self) -> &str;
    fn state(&self) -> &dyn State;
    fn conversation_history(&self) -> Vec<Content>;
}
```

**设计思想**：
- **State with scoped prefixes**：`user:`（跨会话持久）、`app:`（应用级）、`temp:`（单轮清空）——通过 key 前缀约定而非独立 API 来划分作用域，简单且易于扩展。
- **后端无关**：`adk-session` 提供 InMemory、SQLite、PostgreSQL、Redis、MongoDB、Firestore、Neo4j 共 7 种后端，全部通过 `SessionService` trait 注入。

---

## 3. 上下文层次体系：能力渐进

ADK-Rust 设计了一条清晰的上下文继承链，每一层增加一种能力，上层始终是下层的子类型：

```
ReadonlyContext          — 只读身份信息（user_id, app_name, session_id, agent_name）
    ↓ + artifacts
CallbackContext          — 添加 Artifacts 访问
    ↓ + function_call_id + actions + memory
ToolContext              — 工具执行上下文（回调、记忆搜索、权限）
    ↓ + agent + session + run_config + end_invocation + secrets
InvocationContext        — 完整 agent 执行上下文
```

**设计思想**：
- **最小权限原则**：工具只能访问 `ToolContext` 提供的能力，不能直接操作 session 或触发 invocation 终止。Agent 只能访问 `InvocationContext`。权限在类型系统中被强制执行。
- **能力保留（Capability Preservation）**：派生上下文（workflow agent 包装父上下文时）必须转发所有能力方法。框架通过合规测试套件验证：无论 agent 运行在 `LoopAgent`、`ParallelAgent` 还是带 skills 注入的环境中，cancellation、secrets、scopes、shared_state、artifacts 和 memory 都能到达子 agent。

---

## 4. 异步流式架构：EventStream

```rust
pub type EventStream = Pin<Box<dyn Stream<Item = Result<Event>> + Send>>;
```

**设计思想**：
- **拉取式（Pull-based）**：消费者按需从 stream 中拉取事件，天然背压（backpressure），无需额外的 channel 缓冲。
- **复合事件模型**：单个 `Event` 可以携带文本内容、工具调用请求、工具执行结果、状态变更（`state_delta`）、agent 转移信号、工具确认请求等，是一个统一的"通信总线"。
- **可组合性**：workflow agent 可以将子 agent 的 stream 合并（sequential 拼接、parallel 合并），对外暴露单一 stream，调用方无需感知内部结构。

---

## 5. 图编排系统：adk-graph

`adk-graph` 实现了类 LangGraph 的有向图工作流：

```
┌─────┐     fetch      ┌───────────┐     END
│START├───────────────→│transform  ├───────────→
└─────┘                └─────┬─────┘
                             │
                        tools│(cycle back)
                             ↓
                       ┌───────────┐
                       │  LLM Node  │←─ conditional edge
                       └───────────┘
```

**关键设计**：
| 特性 | 设计意图 |
|------|---------|
| **Checkpoint 机制** | 每个节点执行后持久化状态，故障时从最近 checkpoint 恢复，不重复已完成的步骤 |
| **子图（SubgraphNode）** | 图作为节点嵌入另一个图，每个子图有独立的 checkpoint 线程，实现层次化编排 |
| **`with_goto` / `with_goto_parent`** | 运行时动态路由——节点可以在执行时决定下一个节点，无需预先声明边 |
| **`ctx.run_node_with`** | 命令式子节点调用——用于工作单元大小在运行时才确定的场景 |
| **per-node RetryPolicy & Timeout** | 节点级容错，而非全局重试；超时控制防止单个节点阻塞整个图 |
| **RetentionPolicy** | 控制 checkpoint 保留策略（保留最近 N 个/按时间保留），防止数据库无限增长 |
| **Human-in-the-Loop 中断** | 图可以在任意节点前/后暂停，等待人工审批，状态在暂停期间被持久化 |
| **Functional API** | `#[entrypoint]` / `#[task]` 宏将 async 函数转换为图节点，自动插入 checkpoint |

---

## 6. Feature-Gated 模块化：编译时组合

ADK-Rust 使用 Cargo feature flags 实现零成本抽象：

```toml
[features]
minimal = ["agents", "models", "gemini", "runner", "sessions"]  # 默认
standard = ["minimal", "openai", "anthropic", "tools", "memory", ...]
enterprise = ["standard", "realtime", "browser", "rag", "payments"]
full = ["enterprise", "audio", "code", "sandbox"]
```

**设计思想**：
- **分层不是上限**：`minimal + audio` 是合法的，无需升级到 `full`。Feature 是可组合的，不是互斥的。
- **编译时间优化**：默认只编译 Gemini provider，构建时间从数分钟降到数十秒。需要 OpenAI 时加一个 feature flag 即可。
- **二进制体积**：嵌入式部署（如边缘设备）可以只编译 `minimal`，RSS 只有约 15MB。
- **Feature-gated module pattern**：
```rust
#[cfg(feature = "my-feature")]
pub mod my_module;
#[cfg(feature = "my-feature")]
pub use my_module::MyType;
```

---

## 7. 插件系统：生命周期钩子

```rust
#[async_trait]
pub trait EnhancedPlugin: Send + Sync {
    fn name(&self) -> &str;
    fn priority(&self) -> i32;  // 数字越小越先执行
    // 工具/模型拦截、前后置钩子...
}
```

**设计思想**：
- **优先级管道**：多个插件按 `priority` 排序执行，每个插件可以短路后续插件（如 guardrail 拒绝请求）。
- **PluginContext 共享状态**：插件之间通过 `PluginContext` 共享状态，而非全局变量。
- **实际应用**：`adk-retry-reflect` 插件拦截工具失败，注入反思提示，实现指数退避和熔断器模式——无需修改 agent 或工具代码。

---

## 8. 类型化身份体系：防 ID 混淆

```rust
AppName      — 应用名（TryFrom<&str>，校验非空、非null、≤512字节）
UserId       — 用户 ID（同上）
SessionId    — 会话 ID（同上）
InvocationId — 调用 ID（同上）
```

**设计思想**：
- **类型安全**：`AppName` 和 `UserId` 虽然底层都是字符串，但类型不同，编译器阻止参数顺序错误。
- **三层分离**：
  - `RequestContext` — 谁在认证（Auth identity）
  - `AdkIdentity` — 哪个会话（Session identity: app + user + session）
  - `ExecutionIdentity` — 哪次调用（Execution identity: + invocation + branch + agent）
- **边界验证**：在 HTTP 入口处用 `TryFrom<&str>` 解析一次，后续全部使用类型化 ID，不再重复校验。

---

## 9. 工具授权：四层防御

评估顺序（从外到内）：

```
RBAC（adk-auth）
  ↓ pass
BeforeToolCallback（编程式拦截）
  ↓ pass
ToolConfirmationPolicy（HITL 确认）
  ↓ approved
execute()
  ↓
AfterToolCallback（后置审计）
```

**设计思想**：
- **组合而非替代**：四层机制可以任意组合。一个工具可以同时受 RBAC 权限控制 + 编程式拦截 + HITL 确认。
- **HITL 事件模型**：`ToolConfirmationPolicy::PerTool` 暂停执行，发出 `ToolConfirmationRequest` 事件，等待 `Approve/Deny` 响应——跨 CLI 和 Web Server 统一工作。

---

## 10. 持久化与多后端设计

### Session 后端

| 后端 | Feature | 适用场景 |
|------|---------|---------|
| InMemory | 默认 | 测试、本地开发 |
| SQLite | `sqlite` | 单机部署、边缘设备 |
| PostgreSQL | `postgres` | 生产多实例 |
| Redis | `redis` | 高吞吐、TTL 支持 |
| MongoDB | `mongodb` | 灵活 schema |
| Firestore | `firestore` | GCP 生态 |
| Neo4j | `neo4j` | 图关系查询 |

### Memory 后端（6 种）

向量搜索 + 语义记忆，支持项目级隔离（`search_in_project`）。

**设计思想**：
- 所有后端通过 trait 注入，核心代码不依赖任何具体数据库。
- 加密会话（`EncryptedSession<S>`）使用 AES-256-GCM + key rotation，在后端透明层之上叠加。

---

## 11. 协议生态：MCP / A2A / ACP / AWP

| 协议 | crate | 作用 |
|------|-------|------|
| MCP | `adk-tool`（`mcp` feature） | 连接外部工具服务器（rmcp 3.1），支持 tools/resources/prompts/elicitation/tasks |
| A2A | `adk-server` | Agent-to-Agent 通信协议 v1.0.0，11 个 JSON-RPC 操作 |
| ACP | `adk-acp` | Agent Client Protocol，连接 Claude Code / Codex / Kiro CLI 作为工具 |
| AWP | `adk-awp` | Agentic Web Protocol，Web 发现、信任等级、支付意图 |

**设计思想**：
- **协议 crate 零内部依赖**：`awp-types` 没有任何 `adk-*` 依赖，确保协议类型可以被独立使用。
- **McpServerManager**：动态管理多个本地 MCP 进程的生命周期、健康检查和自动重启。

---

## 12. 性能设计

| 设计决策 | 效果 |
|---------|------|
| Tokio async 运行时 | 亚毫秒级 agent loop 开销（568 μs vs LangGraph 1,228 ms） |
| `EventStream` 拉取式 | 天然背压，无需额外 channel |
| Feature-gated 编译 | `minimal` 配置 RSS 约 15MB |
| SchemaCache | 避免每次 LLM 调用重新序列化工具 schema |
| `debug = 1` + incremental | 开发构建快速重编译 |
| `sccache` 缓存 | CI 和本地共享编译缓存 |
| wild linker（Linux） | 加速链接阶段 |

---

## 13. 错误处理哲学

```rust
AdkError {
    component: ErrorComponent,   // Model / Session / Tool / ...
    category: ErrorCategory,     // RateLimited / NotFound / Unauthorized / ...
    code: String,                // "model.openai.rate_limited"
    message: String,             // 人类可读描述
    retry_hint: Option<RetryHint>,
    details: Option<ErrorDetails>,
}
```

**设计思想**：
- **结构化而非字符串**：错误可以被程序化检查（`err.is_retryable()`），可以生成 HTTP 响应（`err.to_problem_json()`），可以用于 OpenTelemetry span 属性。
- **Crate 内部用 thiserror，边界转换为 AdkError**：每个 crate 有本地错误类型，在 crate 边界用 `From<CrateLocalError> for AdkError` 统一。

---

## 14. 发布与依赖拓扑

42 个可发布的 crate 被分成 8 个发布层级（Tier），按依赖顺序发布：

```
Tier 1: adk-core, awp-types, adk-rust-macros, adk-telemetry, ...
  ↓
Tier 2: adk-session, adk-memory, adk-gemini, adk-browser, ...
  ↓
Tier 3: adk-model, adk-graph, adk-realtime, adk-code, ...
  ↓
Tier 4: adk-agent, adk-runner, adk-tool, adk-audio
  ↓
Tier 5: adk-server, adk-eval, adk-acp, ...
  ↓
Tier 8: adk-rust（umbrella，始终最后发布）
```

**设计思想**：
- `adk-rust`（umbrella crate）始终最后发布，确保消费者只依赖一个 crate 即可获得所有功能。
- Internal dev-dependencies 用 path-only 声明（无 version），防止发布顺序死锁。

---

## 15. 总结：设计哲学清单

| 哲学 | 体现 |
|------|------|
| **Trait-based polymorphism** | Agent / Tool / Llm / Session / Toolset 全部是 trait |
| **Stream-first** | EventStream 贯穿所有执行路径 |
| **Type-safe boundaries** | 类型化 ID、context 层次、SchemaAdapter |
| **Feature-gated composition** | 4 层 tier + 独立 feature 可任意组合 |
| **Provider-agnostic** | 切换 LLM 只替换 Arc<dyn Llm> |
| **Backend-agnostic** | Session / Memory 可选 6-7 种后端 |
| **Defense in depth** | 工具授权四层机制 |
| **Composite Pattern** | Agent 树 + 子图嵌套 |
| **Checkpoint & Durable Resume** | 图工作流故障恢复 |
| **Zero-cost abstraction** | Feature flags 在编译时消除未使用代码 |
| **Structured errors** | AdkError 带 component/category/code，可程序化处理 |
| **Capability preservation** | 派生 context 必须转发所有能力方法 |
