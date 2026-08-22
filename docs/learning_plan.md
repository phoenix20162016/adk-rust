# ADK-Rust 深度学习计划

> 目标：从零开始，系统性地理解并掌握 ADK-Rust 框架的全部核心机制，最终能够独立扩展框架、排查生产问题、并贡献高质量代码。

---

## 学习前提

### Rust 能力要求

| 能力 | 要求程度 | 验证方式 |
|---|---|---|
| async/await + tokio | 熟练 | 能读懂 `tokio::spawn`、`select!`、`mpsc` 的用法 |
| trait object + dyn Trait | 熟练 | 能理解 `Arc<dyn Tool>` 的动态分发机制 |
| 泛型 + trait bound | 熟练 | 能读懂 `where S: Service<RoleClient> + Send + Sync + 'static` |
| derive 宏 | 了解 | 能理解 `#[derive(Serialize, Deserialize)]` 和自定义 derive |
| 错误处理 | 熟练 | 能理解 `thiserror` + `anyhow` 的分层错误策略 |
| 嵌套 async 闭包 | 了解 | 能读懂 `async move { ... }` 在 `tokio::spawn` 中的捕获语义 |

### LLM 基础知识

- 了解 LLM 的 function calling / tool use 机制
- 了解 streaming response 的基本概念
- 了解 JSON Schema 的基本结构
- 了解 OAuth2 的基本流程（HTTP transport 相关）

---

## 项目全貌

ADK-Rust 是一个由 **42 个 crate** 组成的 Rust workspace，采用严格的五层分层架构：

```
┌─────────────────────────────────────────────────────┐
│  Application    CLI / REST+A2A / ACP / Studio        │
├─────────────────────────────────────────────────────┤
│  Execution      Runner / Graph / Eval / Bench        │
├─────────────────────────────────────────────────────┤
│  Agent          LlmAgent / Sequential / Parallel     │
│                 Loop / Conditional / Custom           │
├─────────────────────────────────────────────────────┤
│  Service        Models / Tools / Sessions / Memory   │
│                 Plugin / Auth / Telemetry             │
├─────────────────────────────────────────────────────┤
│  Foundation     adk-core (Agent, Tool, Llm,          │
│                 Session, Event, Content traits)       │
└─────────────────────────────────────────────────────┘
```

**核心设计原则**：上层依赖下层，下层永远不感知上层。`adk-core` 只定义 trait 和基础类型，不包含任何具体实现。

---

## 阶段一：地基 — adk-core（第 1-3 周）

### 目标

理解框架的四根 trait 支柱，以及 Context 层次体系。

### 阅读清单

| 序号 | 文件（路径相对于项目根） | 行数 | 核心问题 |
|---|---|---|---|
| 1.1 | `adk-core/src/tool.rs` | 402 | `Tool` trait 的每个方法有什么设计意图？`is_read_only` 和 `is_concurrency_safe` 如何影响调度？ |
| 1.2 | `adk-core/src/agent.rs` | 214 | `Agent::run()` 为什么返回 `EventStream` 而不是单值？`sub_agents()` 如何支持复合模式？ |
| 1.3 | `adk-core/src/model.rs` | 576 | `Llm` trait 如何做到 provider-agnostic？`LlmRequest` 和 `LlmResponse` 包含哪些字段？ |
| 1.4 | `adk-core/src/context.rs` | 1366 | `ReadonlyContext` → `ToolContext` → `InvocationContext` 三层递进的职责边界是什么？ |
| 1.5 | `adk-core/src/event.rs` | 781 | `Event` 如何承载 token、tool call、agent transfer、state mutation 四种信号？ |
| 1.6 | `adk-core/src/types.rs` | 1217 | `Content`、`Part`、`FunctionCall`、`FunctionResponse` 的数据模型 |
| 1.7 | `adk-core/src/error.rs` | 834 | `AdkError` 的 Component + Category + Code 三元组设计 |

### 配套文档

| 文档 | 路径 |
|---|---|
| 架构设计哲学（中文） | `docs/official_docs/development/architecture.md` |
| 核心概念介绍 | `docs/official_docs/introduction.md` |
| Core 详解 | `docs/official_docs/core/core.md` |

### 练习

1. 在 `examples/` 中找最简单的 agent example（如 `examples/yaml_agent`），跑通，打断点观察 `Agent::run()` 的调用时机。
2. 画出 `ReadonlyContext` → `ToolContext` → `InvocationContext` 的继承关系图。

---

## 阶段二：Agent Loop 核心（第 4-7 周）

### 目标

彻底搞懂 LLM ↔ Tool 的闭环，这是整个框架的心脏（`llm_agent.rs` 有 3260 行）。

### 阅读清单

| 序号 | 文件 | 行数 | 核心问题 |
|---|---|---|---|
| 2.1 | `adk-agent/src/llm_agent.rs` LlmAgent 结构体 | ~200 | LlmAgent 持有哪些字段？模型、工具、回调如何组织？ |
| 2.2 | `adk-agent/src/llm_agent.rs` `run_codeact()` | ~400 | Agent 主循环的完整状态机：generate → dispatch tools → collect results → loop/exit |
| 2.3 | `adk-agent/src/llm_agent.rs` ToolExecutor | ~600 | 工具执行全流程：并发许可 → 确认策略 → before callbacks → execute → after callbacks |
| 2.4 | `adk-agent/src/llm_agent.rs` ToolConfirmationPolicy | ~100 | 工具确认机制：approve / deny / fingerprint 的判断逻辑 |
| 2.5 | `adk-core/src/tool_concurrency.rs` | 319 | `ToolConcurrencyManager` 的信号量实现 |
| 2.6 | `adk-agent/src/compaction.rs` | ~200 | 上下文压缩策略：何时触发、如何裁剪历史 |
| 2.7 | `adk-core/src/callbacks.rs` | ~200 | before/after tool callbacks 的执行时机和数据流 |

### 配套文档

| 文档 | 路径 |
|---|---|
| LlmAgent 详解 | `docs/official_docs/agents/llm-agent.md` |
| Callbacks 机制 | `docs/official_docs/callbacks/callbacks.md` |
| Events 机制 | `docs/official_docs/events/events.md` |

### 阅读策略

`llm_agent.rs` 是全框架最重要的文件，建议按以下顺序：

```
1. 先看结构体定义（字段、Builder 模式）
2. 再看 run_codeact() 主循环（理解状态机）
3. 然后看 ToolExecutor::execute()（理解单次工具调用）
4. 最后看 test 模块（理解预期行为）
```

### 练习

1. 找一个有多工具调用的 example，在 `ToolExecutor::execute()` 打断点，观察并发执行时序。
2. 阅读 `ToolExecutionStrategy::Auto` 的判断逻辑，画出并发/串行的决策树。

---

## 阶段三：Tool 生态 — 从声明到执行（第 8-10 周）

### 目标

理解工具如何从 MCP Server 的声明，经过层层封装，最终被 Agent 调用。

### 阅读清单

| 序号 | 文件 | 行数 | 核心问题 |
|---|---|---|---|
| 3.1 | `adk-tool/src/mcp/toolset.rs` McpToolset | ~700 | `McpToolset::tools()` 如何通过 `list_all_tools` 获取工具并封装为 `McpTool`？ |
| 3.2 | `adk-tool/src/mcp/toolset.rs` McpTool | ~300 | `McpTool::execute()` 如何通过 `call_tool_once` 调用远端 MCP Server？ |
| 3.3 | `adk-tool/src/mcp/toolset.rs` `call_tool_with_retry` | ~150 | 重连逻辑：InputRequired / Task / transport error 的处理分支 |
| 3.4 | `adk-tool/src/mcp/http.rs` | ~400 | Streamable HTTP 传输的建立，OAuth2/Bearer/ApiKey 认证 |
| 3.5 | `adk-tool/src/mcp/reconnect.rs` | ~400 | `ConnectionFactory` trait 和 `ConnectionRefresher` 的重连机制 |
| 3.6 | `adk-tool/src/mcp/manager/toolset_impl.rs` | 302 | `McpServerManager` 多服务器聚合，同名工具冲突解决（`PrefixedTool`） |
| 3.7 | `adk-devtools/src/toolset.rs` | 58 | 内置开发工具的实现方式（作为 Toolset 的参考实现） |
| 3.8 | `adk-tool/src/toolset/compose.rs` | ~200 | Toolset 组合器：`MergedToolset` / `FilteredToolset` / `PrefixedToolset` |

### 配套文档

| 文档 | 路径 |
|---|---|
| MCP 客户端 | `docs/official_docs/mcp/client.md` |
| MCP Manager | `docs/official_docs/mcp/manager.md` |
| MCP 安全 | `docs/official_docs/mcp/security.md` |
| Tools 概览 | `docs/official_docs/tools/` |

### 关键流程图

```
MCP Server
  │ (stdio / Streamable HTTP)
  ↓
rmcp::Service<RoleClient>        ← 底层 MCP 协议客户端
  │
  ↓ list_all_tools() / call_tool()
McpToolset                       ← 实现 Toolset trait
  │
  ↓ tools() → Vec<Arc<dyn Tool>>
McpTool                          ← 实现 Tool trait
  │
  ↓ Agent.tool_map.get(name)
ToolExecutor::execute()          ← Agent Loop 中的工具调度
```

### 练习

1. 实现一个自己的 MCP 服务器（任何语言均可），用 `McpToolset` 连接，验证工具调用闭环。
2. 运行 `examples/mcp_manager`，观察 `McpServerManager` 的多服务器管理行为。

---

## 阶段四：Model 层 & Schema 适配（第 11-12 周）

### 目标

理解多 LLM 提供者如何统一接入，以及 SchemaAdapter 如何抹平差异。

### 阅读清单

| 序号 | 文件 | 核心问题 |
|---|---|---|
| 4.1 | `adk-core/src/schema_adapter.rs` | `SchemaAdapter` trait 定义了什么能力？ |
| 4.2 | `adk-core/src/schema_utils.rs` (1979行) | Schema 归一化的核心逻辑 |
| 4.3 | `adk-gemini/src/` 主要文件 | Gemini API 的 function calling 如何映射到 `Llm` trait |
| 4.4 | `adk-anthropic/src/` 主要文件 | Anthropic 的差异点（server tools、thinking blocks） |
| 4.5 | `adk-model/src/deepseek/convert.rs` | 第三方模型的接入方式参考 |

### 配套文档

| 文档 | 路径 |
|---|---|
| Gemini 集成 | `docs/official_docs/models/providers.md` |
| Anthropic 集成 | `docs/official_docs/models/anthropic.md` |
| OpenAI Responses | `docs/official_docs/models/openai-responses.md` |
| Ollama 集成 | `docs/official_docs/models/ollama.md` |

### 关键概念

**Schema 差异的核心矛盾：**

| Provider | 特殊限制 |
|---|---|
| Gemini | 不支持 `additionalProperties`，required 必须显式列出 |
| OpenAI | strict 模式下有额外约束，function name 有长度限制 |
| Anthropic | tool description 有长度限制，input_schema 格式略有不同 |
| 本地模型 | 可能完全不支持 JSON Schema，需要 text-based tool calling |

`SchemaAdapter` 在每次 `generate_content` 调用前，对 tools 列表做 provider-aware 的 schema 变换。

---

## 阶段五：Session、Memory & Context（第 13-14 周）

### 目标

理解状态如何跨轮次持久化，以及 memory 如何为 Agent 提供长期记忆。

### 阅读清单

| 序号 | 文件 | 核心问题 |
|---|---|---|
| 5.1 | `adk-session/src/` 主要文件 | `SessionService` trait 的 7 种后端实现 |
| 5.2 | `adk-memory/src/` 主要文件 | Memory trait 的抽象：短期/长期/项目级记忆 |
| 5.3 | `adk-core/src/shared_state.rs` (305行) | 跨 agent 的共享状态读写机制 |
| 5.4 | `adk-core/src/identity.rs` (651行) | Identity 如何在 agent 间传递 |

### 配套文档

| 文档 | 路径 |
|---|---|
| Session 概念 | `docs/official_docs/sessions/` |
| Memory 概念 | `docs/official_docs/memory/concepts.md` |
| Memory 后端 | `docs/official_docs/memory/backends.md` |
| Memory 工具与 Agent | `docs/official_docs/memory/tools-and-agents.md` |

### 关键概念

**State 作用域约定（key 前缀）：**

| 前缀 | 作用域 | 生命周期 |
|---|---|---|
| `user:` | 用户级 | 跨所有会话持久 |
| `app:` | 应用级 | 同一应用所有用户可见 |
| `temp:` | 临时 | 单轮对话后清空 |

---

## 阶段六：Graph 编排（第 15-18 周）

### 目标

掌握有向图工作流，这是框架最复杂的部分（`adk-graph` 有 10379 行代码）。

### 阅读清单

| 序号 | 文件 | 行数 | 核心问题 |
|---|---|---|---|
| 6.1 | `adk-graph/src/graph.rs` | 863 | Graph 的编译、验证、边路由定义 |
| 6.2 | `adk-graph/src/node.rs` | 1043 | Node 类型：AgentNode、ActionNode、SubgraphNode |
| 6.3 | `adk-graph/src/executor.rs` | 1373 | 执行引擎：并发限制、超时、重试 |
| 6.4 | `adk-graph/src/delta.rs` | 1412 | Delta Checkpoint：增量状态保存 |
| 6.5 | `adk-graph/src/checkpoint.rs` | 505 | Checkpoint 策略：SQLite 持久化 |
| 6.6 | `adk-graph/src/deferred.rs` | 501 | Deferred Node：延迟执行节点 |
| 6.7 | `adk-graph/src/cache.rs` | 501 | Node 结果缓存 |
| 6.8 | `adk-graph/src/time_travel.rs` | 576 | 时间旅行：历史状态回放 |
| 6.9 | `adk-agent/src/workflow/` | — | SequentialAgent / ParallelAgent / ConditionalAgent |

### 配套文档

| 文档 | 路径 |
|---|---|
| Graph Agent 概览 | `docs/official_docs/agents/graph-agents.md` |
| Workflow Agent | `docs/official_docs/agents/workflow-agents.md` |
| Multi-Agent | `docs/official_docs/agents/multi-agent.md` |

### 必跑 Examples

| Example | 学习重点 |
|---|---|
| `examples/graph_resilient_research` | 基础图执行、重试、超时 |
| `examples/graph_subgraph_claims` | SubgraphNode 嵌套 |
| `examples/graph_goto_routing` | `with_goto` 动态路由 |
| `examples/functional_workflow` | 声明式 vs 命令式编排 |
| `examples/delta_checkpoints` | 增量 Checkpoint 恢复 |
| `examples/node_timeouts` | 节点超时处理 |
| `examples/deferred_nodes` | 延迟执行节点 |
| `examples/parallel_shared_state` | 并行节点的共享状态 |

---

## 阶段七：Runner & Server 层（第 19-20 周）

### 目标

理解 Agent 如何被外部调用，以及对外服务的暴露方式。

### 阅读清单

| 序号 | 文件 | 核心问题 |
|---|---|---|
| 7.1 | `adk-runner/src/` 主要文件 | Runner 如何将 input → agent.run() → output 串联？ |
| 7.2 | `adk-server/src/` 主要文件 | REST + A2A 接口如何暴露 Agent？ |
| 7.3 | `adk-acp/src/` 主要文件 | ACP 协议的实现：session 管理、mode 切换 |
| 7.4 | `awp-types/src/` + `adk-awp/src/` | A2A/AWP 协议类型定义 |

### 配套文档

| 文档 | 路径 |
|---|---|
| Runner 详解 | `docs/official_docs/core/runner.md` |
| Server 部署 | `docs/official_docs/deployment/server.md` |
| A2A 入门 | `docs/official_docs/a2a/getting-started.md` |
| ACP 客户端 | `docs/official_docs/acp/client.md` |
| ACP 服务端 | `docs/official_docs/acp/server.md` |

---

## 阶段八：专项能力 — 按需选修（第 21-26 周）

### 方向 A：实时语音 Agent（2 周）

| 文件 | 核心内容 |
|---|---|
| `adk-realtime/src/` | WebSocket 连接、音频流、function calling over voice |
| `adk-audio/src/` | 音频编解码、格式转换 |
| `examples/realtime_voice` | 实时语音交互 |
| `examples/realtime_tools` | 语音中的工具调用 |
| `examples/desktop_audio` | 桌面音频集成 |
| `docs/official_docs/realtime/` | 实时 Agent 文档 |

### 方向 B：桌面自动化 / Computer Use（2 周）

| 文件 | 核心内容 |
|---|---|
| `adk-computer-use/src/` | 屏幕截图、鼠标键盘控制、approval digest |
| `examples/desktop_agent` | 桌面 Agent 示例 |
| `docs/official_docs/computer-use/` | Computer Use 文档 |

### 方向 C：浏览器控制（2 周）

| 文件 | 核心内容 |
|---|---|
| `adk-browser/src/` | 浏览器自动化、页面交互 |
| 相关 examples | — |

### 方向 D：RAG 检索增强（2 周）

| 文件 | 核心内容 |
|---|---|
| `adk-rag/src/` | 向量存储、文档切分、检索 |
| `examples/knowledge_graph_agent` | 知识图谱 Agent |

### 方向 E：安全与护栏（1 周）

| 文件 | 核心内容 |
|---|---|
| `adk-guardrail/src/` | 输入/输出检查、内容过滤 |
| `adk-sandbox/src/` | 安全沙箱执行 |
| `docs/official_docs/sandbox/` | Sandbox 文档 |

### 方向 F：评测与基准（1 周）

| 文件 | 核心内容 |
|---|---|
| `adk-eval/src/` | Agent 评测框架 |
| `adk-bench/src/` | 性能基准测试 |
| `examples/eval_showcase` | 评测示例 |
| `docs/official_docs/evaluation/` | 评测文档 |

### 方向 G：Coding Agent（2 周）

| 文件 | 核心内容 |
|---|---|
| `adk-code/src/` | 代码执行环境（Python embedded runtime） |
| `adk-codeact-monty/src/` | Monty 运行时 |
| `adk-devtools/src/` | 文件读写、glob、grep、bash 工具 |
| `examples/coding_agent` | Coding Agent 完整示例 |
| `docs/official_docs/coding-agent/` | Coding Agent 文档套件 |

### 方向 H：Plugin 系统（1 周）

| 文件 | 核心内容 |
|---|---|
| `adk-plugin/src/` | Plugin trait、生命周期钩子 |
| `examples/plugin_system` | Plugin 示例 |
| `docs/official_docs/core/plugins.md` | Plugin 文档 |

---

## 学习节奏建议

```
第 1-3 周    ▓▓▓░░░░░░░  阶段一：adk-core 基础
第 4-7 周    ▓▓▓▓▓░░░░░  阶段二：Agent Loop 核心（重难点，多分配时间）
第 8-10 周   ▓▓▓▓░░░░░░  阶段三：Tool 生态
第 11-12 周  ▓▓░░░░░░░░  阶段四：Model 层
第 13-14 周  ▓▓░░░░░░░░  阶段五：Session & Memory
第 15-18 周  ▓▓▓▓▓░░░░░  阶段六：Graph 编排（重难点）
第 19-20 周  ▓▓░░░░░░░░  阶段七：Runner & Server
第 21-26 周  ▓▓▓▓▓▓░░░░  阶段八：专项选修（按兴趣选 2-3 个方向）
```

**总计约 26 周（6-7 个月）**

---

## 推荐学习资料

### 框架内文档（优先阅读）

| 文档 | 路径 | 说明 |
|---|---|---|
| 架构设计哲学 | `docs/official_docs/development/architecture.md` | 全局视角，五层架构 + 四根 trait 支柱 |
| Quick Start | `docs/official_docs/quickstart.md` | 5 分钟跑通第一个 Agent |
| LlmAgent 详解 | `docs/official_docs/agents/llm-agent.md` | Agent Loop 的设计文档 |
| MCP 全套文档 | `docs/official_docs/mcp/` | client / manager / security / testing |
| Graph Agent | `docs/official_docs/agents/graph-agents.md` | 图编排的入门文档 |
| Coding Agent | `docs/official_docs/coding-agent/index.md` | Coding Agent 的完整文档套件 |
| Memory 概念 | `docs/official_docs/memory/concepts.md` | Memory 体系的概念解释 |
| Session 概念 | `docs/official_docs/sessions/` | Session 管理的设计 |

### 框架外参考资料

#### Rust 基础

| 资料 | 说明 |
|---|---|
| [The Rust Programming Language](https://doc.rust-lang.org/book/) | Rust 官方教程，重点看 Chapter 17 (Trait Objects) 和 Chapter 16 (Fearless Concurrency) |
| [Rust by Example — Traits](https://doc.rust-lang.org/rust-by-example/trait.html) | trait 的快速参考 |
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | async 运行时基础，重点看 `spawn`、`select!`、`mpsc` |
| [Async Book](https://rust-lang.github.io/async-book/) | async/await 的深入理解 |

#### LLM Agent 概念

| 资料 | 说明 |
|---|---|
| [Anthropic: Building Effective Agents](https://docs.anthropic.com/en/docs/build-with-claude/agent-patterns) | Agent 设计模式（orchestration、routing、parallelization） |
| [OpenAI: Function Calling Guide](https://platform.openai.com/docs/guides/function-calling) | Function calling 的基本机制 |
| [LangChain Agent Concepts](https://python.langchain.com/docs/concepts/agents/) | Agent 概念的通用解释（框架无关） |
| [Google DeepMind: Agent白皮书](https://deepmind.google/discover/blog/agent-whitepaper/) | Agent 系统的学术视角 |

#### MCP 协议

| 资料 | 说明 |
|---|---|
| [MCP 官方规范](https://spec.modelcontextprotocol.io/) | MCP 协议的完整规范 |
| [MCP 架构文档](https://modelcontextprotocol.io/specification/2025-03-26/basic/architecture) | MCP 的架构设计 |
| [rmcp crate 文档](https://docs.rs/rmcp/) | ADK-Rust 底层使用的 MCP Rust SDK |

#### A2A 协议

| 资料 | 说明 |
|---|---|
| [A2A 协议规范](https://google.github.io/A2A/) | Agent-to-Agent 协议的官方文档 |

#### Rust 设计模式

| 资料 | 说明 |
|---|---|
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | Builder、Newtype、Typestate 等模式 |
| [Comprehensive Rust — Google](https://google.github.io/comprehensive-rust/) | Google 出品的 Rust 训练课程 |

---

## 验证理解的里程碑

### 第 3 周末

- [ ] 能画出 `adk-core` 四根 trait 的依赖关系图
- [ ] 能解释 `EventStream` 为什么是异步流而不是单一返回值
- [ ] 能说明三层 Context 各自能访问什么数据

### 第 7 周末

- [ ] 能画出 Agent Loop 的完整状态机图
- [ ] 能解释 `ToolExecutor` 中并发工具调用的判断逻辑
- [ ] 能说清 before/after callbacks 的执行时机

### 第 10 周末

- [ ] 能画出 MCP 工具从声明到执行的完整调用链
- [ ] 能解释 `McpToolset` 如何处理连接断开和重连
- [ ] 能实现一个简单的 `Tool` trait 实现

### 第 14 周末

- [ ] 能解释 `SchemaAdapter` 如何抹平不同 LLM provider 的差异
- [ ] 能说明 `user:` / `app:` / `temp:` 三种 state 前缀的语义

### 第 18 周末

- [ ] 能画出 Graph 的执行模型（并发限制、超时、重试）
- [ ] 能解释 Delta Checkpoint 如何实现增量状态保存
- [ ] 能实现一个简单的 `Agent` trait 实现

### 第 20 周末

- [ ] 能独立搭建一个 REST 服务，暴露一个自定义 Agent
- [ ] 能解释 ACP 协议的 session 管理机制

### 第 26 周末（最终）

- [ ] 能独立排查生产环境中的 Agent 执行问题
- [ ] 能为框架贡献新功能（新 Tool、新 Model provider、新 Agent 类型）
- [ ] 能设计并实现一个完整的多 Agent 系统

---

## 每周学习节奏

| 日 | 活动 | 时间 |
|---|---|---|
| 周一-周三 | 阅读指定源码文件，做笔记 | 每天 1-2 小时 |
| 周四 | 运行对应 example，加断点调试 | 2 小时 |
| 周五 | 做练习，写小型验证代码 | 2 小时 |
| 周末 | 复习笔记，整理学习心得 | 1-2 小时 |

**每周约 10-12 小时**

---

## 附录：关键文件速查

| 概念 | 文件路径 |
|---|---|
| Agent trait 定义 | `adk-core/src/agent.rs` |
| Tool trait 定义 | `adk-core/src/tool.rs` |
| Llm trait 定义 | `adk-core/src/model.rs` |
| Context 层次 | `adk-core/src/context.rs` |
| Event 系统 | `adk-core/src/event.rs` |
| Agent Loop 主循环 | `adk-agent/src/llm_agent.rs` |
| 工具并发控制 | `adk-core/src/tool_concurrency.rs` |
| MCP Toolset | `adk-tool/src/mcp/toolset.rs` |
| MCP 重连 | `adk-tool/src/mcp/reconnect.rs` |
| MCP 多服务器管理 | `adk-tool/src/mcp/manager/toolset_impl.rs` |
| MCP HTTP 传输 | `adk-tool/src/mcp/http.rs` |
| Graph 核心 | `adk-graph/src/graph.rs` |
| Graph 执行引擎 | `adk-graph/src/executor.rs` |
| Delta Checkpoint | `adk-graph/src/delta.rs` |
| DevTools 工具集 | `adk-devtools/src/toolset.rs` |
| Schema 适配器 | `adk-core/src/schema_adapter.rs` |
| 错误体系 | `adk-core/src/error.rs` |
