https://yzddmr6.com/posts/oh-my-opencode-analyse/

# Oh My OpenCode (OMO) 深度解析

> **什么是 OMO？**
> OMO 是 OpenCode（一个终端 AI 编程工具，类似 Claude Code）的"超级插件"。它把 OpenCode 从单 agent 工具升级为**多 agent 编排平台**。可以类比为 oh-my-zsh 之于 zsh——不改变宿主核心，而是通过插件机制注入完整的 agent 体系、工具链和自动化能力。

---

## 📋 整体知识框架

```
OMO
├── 1. 整体架构          ← 四大支柱 + 插件接口
├── 2. Agent 体系        ← 希腊神话命名的多 agent 协作
│   ├── 角色清单
│   ├── 自描述系统
│   ├── 双轨主 Agent
│   ├── 三段式质量链
│   └── 构建/注册/权限
├── 3. 工具体系          ← 三层工具架构
│   ├── Hashline Edit（防幻觉编辑）
│   ├── LSP + AST-grep（代码理解）
│   ├── 任务委派中枢
│   └── MCP 集成
├── 4. 后台并发模型      ← 多 agent 并行执行
│   ├── 三级并发控制
│   ├── 任务生命周期管理
│   └── tmux 可视化
├── 5. Ralph Loop        ← 自动续跑机制
├── 6. Hook 系统         ← 消息拦截与事件处理
├── 7. 配置系统          ← 多层配置 + Model Fallback
└── 8. Claude Code 兼容层
```

---

## 1. 整体架构

### 1.1 四大支柱

插件入口 `src/index.ts` 依次创建四个核心组件：

| 组件 | 源文件 | 职责 |
|------|--------|------|
| **Managers** | `create-managers.ts` | 管理后台 agent 并发、tmux 窗格、skill MCP 服务、配置处理 |
| **Tools** | `create-tools.ts` | 注册所有工具（LSP、AST-grep、background task、session manager 等） |
| **Hooks** | `create-hooks.ts` | 注册事件钩子（Ralph Loop、上下文监控、预压缩、通知等） |
| **PluginInterface** | `plugin-interface.ts` | 将以上三者组装成 OpenCode 期望的插件接口 |

### 1.2 配置处理管线（6步串行）

`createConfigHandler()` 按固定顺序调用 6 个子处理器，**顺序有严格依赖关系**：

```
applyProviderConfig
    ↓
loadPluginComponents
    ↓
applyAgentConfig  ──── 返回 agentResult
    ↓                      ↓
applyToolConfig  ←── 需要知道哪些 agent 存在，才能设置工具权限
    ↓
applyMcpConfig
    ↓
applyCommandConfig
```

| 处理器 | 核心职责 |
|--------|---------|
| `provider-config-handler` | 提取模型上下文限制，缓存 providerID/modelID → contextLimit 映射 |
| `plugin-components-loader` | 加载第三方插件，带 10 秒超时保护 |
| `agent-config-handler` | 最复杂（约 226 行）：迁移旧版名称 → 发现 skill 来源 → 创建内置 agent → 合并配置 → 重排顺序 |
| `tool-config-handler` | 禁用原生冲突工具，为每个 agent 设置精细工具权限矩阵 |
| `mcp-config-handler` | 合并内置 MCP → 用户配置 → Claude Code MCP → 插件 MCP |
| `command-config-handler` | 14 级优先级命令合并 |

### 1.3 PluginInterface 的 9 个钩子点

OMO 不直接实现 AI 对话逻辑，而是通过钩子注入逻辑——这是典型的**中间件/拦截器模式**。

| 钩子名 | 职责 |
|--------|------|
| `tool` | 注册自定义工具 |
| `chat.params` | 拦截/修改聊天参数（temperature 等） |
| `chat.headers` | 注入 HTTP 请求头 |
| `chat.message` | 拦截/修改用户消息（最复杂） |
| `experimental.chat.messages.transform` | 变换消息历史 |
| `config` | 配置处理（上述 6 步管线） |
| `event` | 事件分发（session 生命周期、idle 检测等） |
| `tool.execute.before` | 工具执行前拦截（任务路由、ralph-loop 解析） |
| `tool.execute.after` | 工具执行后拦截（输出截断、上下文监控） |

> 💡 **新手说明**：可以把这 9 个钩子理解为"插线板"——OpenCode 的 AI 对话流程就像一根电路，OMO 在各个关键节点插入自己的逻辑，可以读取、修改、甚至拦截数据流，但不需要改动 OpenCode 的核心代码。

---

## 2. Agent 体系

### 2.1 Agent 角色清单

所有 agent 以希腊神话人物命名，职责分工明确：

| Agent | 默认模型 | 温度 | 模式 | 核心职责 |
|-------|---------|------|------|---------|
| **Sisyphus** | claude-opus-4-6 | 0.1 | primary | 主编排器：意图分类、委派任务、验证结果 |
| **Hephaestus** | gpt-5.3-codex | 0.1 | primary | 自主深度执行器：端到端完成复杂任务，不中途停下 |
| **Prometheus** | claude-opus-4-6 | 0.1 | — | 战略规划师：只做计划不写代码，输出 `.sisyphus/plans/*.md` |
| **Atlas** | claude-sonnet-4-6 | 0.1 | primary | Todo 列表编排器：按波次并行调度任务执行 |
| **Oracle** | gpt-5.2 | 0.1 | subagent | 只读高智商顾问：架构决策和疑难调试 |
| **Metis** | claude-opus-4-6 | 0.3 | subagent | 规划前顾问：在 Prometheus 生成计划前做 gap 分析 |
| **Momus** | gpt-5.2 | 0.1 | subagent | 计划审查员：验证计划可执行性和引用正确性 |
| **Librarian** | glm-4.7 | 0.1 | subagent | 外部文档/代码搜索：克隆仓库、查官方文档、搜 GitHub |
| **Explore** | grok-code-fast-1 | 0.1 | subagent | 内部代码库搜索：回答"X 在哪里"类问题 |
| **Multimodal Looker** | gemini-3-flash | 0.1 | subagent | 多模态文件分析：处理 PDF/图片/图表 |
| **Sisyphus-Junior** | claude-sonnet-4-6 | 0.1 | all | 分类任务执行器，由 category 系统派生，不能再委派 task() |

> 💡 **温度参数说明**：Temperature（温度）控制 AI 输出的随机性。0.1 接近于确定性输出（适合编码、规划等需要精确的任务）；Metis 使用 0.3 是因为作为"前置分析师"，需要更多创造性来发现潜在问题和盲点。温度越高，输出越有创意也越不稳定。

### 2.2 协作拓扑

```
用户请求
    ├─→ Sisyphus（日常编排）
    │       ├─→ Explore / Librarian（后台并行搜索）
    │       ├─→ Oracle（高难度咨询，不可取消）
    │       └─→ task(category=X) → Sisyphus-Junior（执行）
    │
    ├─→ Hephaestus（深度自主执行，不中途停下）
    │
    ├─→ Prometheus（规划模式）
    │       ├─→ Metis（前置 gap 分析）
    │       └─→ Momus（计划审查）
    │
    └─→ Atlas（计划执行，按波次并行）
            └─→ task(category=X) → Sisyphus-Junior
```

### 2.3 AgentPromptMetadata 自描述系统

每个 agent 通过 `AgentPromptMetadata`（`src/agents/types.ts`）声明式描述自己的能力边界：

```typescript
export interface AgentPromptMetadata {
  category: AgentCategory        // "exploration" | "specialist" | "advisor" | "utility"
  cost: AgentCost                // "FREE" | "CHEAP" | "EXPENSIVE"
  triggers: DelegationTrigger[]  // 什么场景下应该委派给这个 agent
  useWhen?: string[]             // 详细的使用场景
  avoidWhen?: string[]           // 不应使用的场景
  keyTrigger?: string            // Phase 0 快速触发条件
  dedicatedSection?: string      // 专属 prompt 段落
}
```

`dynamic-agent-prompt-builder.ts` 在构建 Sisyphus/Hephaestus prompt 时自动聚合所有 agent 的元数据：

- `buildKeyTriggersSection()` — 从各 agent 的 `keyTrigger` 生成 Phase 0 触发器
- `buildToolSelectionTable()` — 生成工具和 agent 的成本选择表
- `buildDelegationTable()` — 从各 agent 的 `triggers` 生成委派决策表
- `buildCategorySkillsDelegationGuide()` — 生成 category + skill 的委派协议

**核心价值**：添加或移除一个 agent 时，Sisyphus 的 prompt **自动更新**。第三方插件注册的 agent 也能通过 `buildCustomAgentMetadata()` 自动融入，实现真正的开放-封闭原则。

### 2.4 双轨主 Agent：Sisyphus vs Hephaestus

#### Sisyphus——编排者

**核心理念**：委派优先，自己动手是最后选择。

- **分阶段决策流水线**：Phase 0（Intent Gate）→ Phase 1（Codebase Assessment）→ Phase 2A/2B/2C → Phase 3（Completion）
- **Phase 0 意图分类**：Trivial / Explicit / Exploratory / Open-ended / Ambiguous，决定后续走哪条路径
- **并行是默认行为**：Explore 和 Librarian 必须用 `run_in_background=true` 启动，且必须并行发射 2-5 个
- **Oracle 的特殊地位**：唯一"永远不能取消"的 agent，即使你觉得已经有了答案也必须等 Oracle 返回
- **证据驱动的完成标准**：
  - 文件编辑 → `lsp_diagnostics` 在修改文件上无报错
  - 构建命令 → 退出码 0
  - 测试运行 → 通过
  - 委派 → 收到并验证 agent 结果
  - **NO EVIDENCE = NOT COMPLETE**

#### Hephaestus——自主执行者

**核心差异**：禁止询问，只管做。

```
// 被禁止的行为：
- "Should I proceed?" → 直接做
- "Do you want me to run tests?" → 直接跑
- 部分实现后停下 → 100% 完成或什么都不做
```

**Intent Extraction 机制**：处理每条消息前先提取"真实意图"，而不是字面意图。

**`<turn_end_self_check>` 自我约束**：prompt 末尾要求在结束每个 turn 前做四项检查，任何一项失败就不能结束。

> 💡 **设计哲学**：Sisyphus 像一个"协调型管理者"，Hephaestus 像一个"不需要管理的高级工程师"。前者适合需要多方协调的任务，后者适合端到端的深度执行任务。设计灵感来自 AmpCode 的 deep mode。

### 2.5 Prometheus 三段式质量链

Prometheus 的 prompt 拆分为 6 个语义独立的文件：

| 文件 | 职责 |
|------|------|
| `identity-constraints.ts` | 身份锁定："你是规划师，不是实现者，不写代码" |
| `interview-mode.ts` | 7 种意图类型的面谈策略 |
| `plan-generation.ts` | Metis 咨询 → gap 分类 → 摘要格式 |
| `high-accuracy-mode.ts` | Momus 审查循环 |
| `plan-template.ts` | 计划文件 Markdown 模板 |
| `behavioral-summary.ts` | 行为总结和最终约束 |

**三段式质量保证链**：
```
Metis（前置 gap 分析）→ Prometheus（计划生成）→ Momus（后置可执行性验证）
```

- Momus 有 **APPROVAL BIAS**（默认通过），只拦截真正的 blocker，避免无限修改循环
- **增量写入协议**：大型计划先写骨架，再用 Edit 分批追加任务（每批 2-4 个），解决 LLM 输出 token 限制问题

### 2.6 构建、注册与工具权限

#### Agent 工厂模式

```typescript
export type AgentFactory = ((model: string) => AgentConfig) & {
  mode: AgentMode  // "primary" | "subagent" | "all"
}
```

`createBuiltinAgents()` 执行顺序有严格依赖：
1. `fetchAvailableModels()` — 查询当前 provider 的可用模型
2. `collectPendingBuiltinAgents()` — 收集除主 agent 外的所有 agent
3. `parseRegisteredAgentSummaries()` — 解析外部插件注册的 agent
4. `maybeCreateSisyphusConfig()` / `maybeCreateHephaestusConfig()` / `maybeCreateAtlasConfig()` — **最后**构建主 agent（因为它们的动态 prompt 需要知道有哪些 subagent）

#### 工具权限矩阵

| Agent | 策略 | 被禁工具 | 设计意图 |
|-------|------|---------|---------|
| Oracle | 黑名单 | write, edit, apply_patch, task | 只读顾问，防止"顾问自己动手" |
| Librarian / Explore | 黑名单 | write, edit, apply_patch, task, call_omo_agent | 只搜索不修改，不能派生子 agent |
| Multimodal Looker | 白名单 | 只允许 read | 最严格，只能读文件 |
| Metis / Momus | 黑名单 | write, edit, apply_patch, task | 只分析/审查不执行 |
| Atlas | 黑名单 | task, call_omo_agent | 可读写文件，但不能派生 agent |
| Sisyphus-Junior | 黑名单 | task（但允许 call_omo_agent） | 可调 explore/librarian，但不能通过 task() 委派 |

> 💡 **微妙的设计**：Sisyphus-Junior 允许 `call_omo_agent` 但禁止 `task`。这意味着它可以调用 explore/librarian 做搜索，但不能像 Sisyphus 那样通过 `task()` 委派工作——这防止了无限递归委派。

---

## 3. 工具体系

OMO 的工具体系分为三层：
- **原生工具**（`src/tools/`）：直接与 AI agent 交互
- **功能模块**（`src/features/`）：提供后台 agent、tmux、context injector 等复杂功能
- **MCP 集成**（`src/mcp/`）：外部服务连接

### 3.1 完整工具清单

| 工具目录 | 核心职责 | 关键设计 |
|---------|---------|---------|
| `ast-grep/` | AST 感知代码搜索/替换 | 支持 25 种语言，元变量 `$VAR`/`$$$`，空结果智能提示 |
| `lsp/` | LSP 集成（6 个工具） | 4 层架构，双重诊断源 |
| `hashline-edit/` | 行哈希精确编辑 | CID 哈希防幻觉，bottom-up 应用 |
| `interactive-bash/` | tmux 命令执行 | 子命令黑名单，超时保护 |
| `background-task/` | 后台任务 CRUD | 创建/查看/取消 |
| `delegate-task/` | 任务委派中枢 | category 路由 + subagent 直接调用 |
| `call-omo-agent/` | 内置 agent 调用 | explore/librarian，同步/异步 |
| `look-at/` | 多模态文件分析 | 创建子 session 调用 multimodal-looker |
| `session-manager/` | 历史 session 操作 | list/read/search/info，60s 超时保护 |
| `skill/` + `skill-mcp/` | Skill 执行与 MCP 代理 | 加载 Skill 内容注入 prompt |
| `task/` | 任务 CRUD | create/get/list/update + todo-sync |
| `glob/` + `grep/` | 文件搜索与内容搜索 | 内置 ripgrep 自动下载 |

### 3.2 Hashline Edit：防幻觉的精确编辑

**核心问题**：AI 可能基于过时的文件内容生成编辑指令，传统精确文本匹配会静默失败或错误编辑。

**解决方案**：

```
文件内容（带 CID 哈希）：
1#ZP  import React from 'react'
2#MQ  
3#VK  export default function App() {
4#WR    return <div>Hello</div>
5#SN  }

编辑操作必须携带正确的 LINE#ID 锚点：
replace_lines(3#VK, 5#SN, newContent)

如果哈希不匹配 → 拒绝编辑 + 提示重新读取文件
```

- **CID 哈希字符集**：`ZPMQVRWSNKTXJBYH`（16 个字符）
- **2 字符哈希**：16² = 256 种组合，对单个文件的行数碰撞概率极低
- **4 种操作**：`set_line`、`replace_lines`、`insert_after`、`replace`
- **bottom-up 应用**：从底部向上应用编辑，保持行号引用稳定

### 3.3 LSP 集成

> 💡 **什么是 LSP？** Language Server Protocol 把编辑器功能拆分为客户端（Agent）和服务器（了解代码结构的"专家"）。Agent 发 JSON-RPC 消息询问 Server，Server 回答"这个函数在哪里定义"、"修改这个变量会影响哪些地方"等问题。

**LSP 4 层架构**：

```
LSPClient（高层 API：openFile/definition/references/diagnostics）
  └── LSPClientConnection（JSON-RPC 连接管理）
       └── lsp-client-transport.ts（stdio 传输层）
            └── lsp-process.ts（LSP 服务器进程管理）
```

**6 个 LSP 工具**：

| 工具 | 功能 | LSP 方法 |
|------|------|---------|
| `lsp_goto_definition` | 跳转到定义 | `textDocument/definition` |
| `lsp_find_references` | 查找引用 | `textDocument/references` |
| `lsp_symbols` | 文档/工作区符号 | `textDocument/documentSymbol` |
| `lsp_diagnostics` | 获取诊断信息 | 双重诊断源（pull + push） |
| `lsp_prepare_rename` | 重命名预检 | `textDocument/prepareRename` |
| `lsp_rename` | 执行重命名 | `textDocument/rename` |

**关键设计细节**：
- **文档版本追踪**：`documentVersions` Map 维护每个文件的版本号
- **内容去重**：`lastSyncedText` Map 缓存上次同步文本，避免冗余通知
- **双重诊断源**：先尝试 pull 模式，失败回退到 push 模式
- **服务器自动管理**：自动发现并安装 LSP 服务器

### 3.4 AST-grep

> 💡 **什么是 AST-grep？** 传统 grep 基于字符串匹配，容易被空格和换行干扰。AST-grep 基于代码的语法结构（抽象语法树），能"读懂"代码逻辑，精准搜索函数调用、变量定义等语法单元，不受格式影响。

- 支持 **25 种语言**
- **元变量模式**：`$VAR`（单节点）、`$$$`（多节点）
- **智能提示**：空结果时给出修正建议（AI 经常写出不完整的 AST 模式，智能提示大幅降低失败率）
- **二进制自动下载**

### 3.5 delegate-task 任务委派中枢

`createDelegateTask()` 支持：

- **category 路由**：指定 category → 自动使用 Sisyphus-Junior + 对应模型配置
- **subagent 直接调用**：指定 `subagent_type` → 直接使用该 agent
- **session 续写**：通过 `session_id` 继续已有 session，保持完整上下文
- **同步/异步执行**：`run_in_background=true` 返回 task_id，`false` 等待结果
- **Skill 注入**：`load_skills` 参数加载 Skill 内容注入到 agent system prompt

### 3.6 MCP 集成

三个内置 MCP 服务器（全部 remote HTTP/SSE 连接）：

| MCP | 服务地址 | 用途 | 认证 |
|-----|---------|------|------|
| websearch | mcp.exa.ai（默认）/ mcp.tavily.com | 通用网络搜索 | API key |
| context7 | mcp.context7.com | 库文档查询 | 可选 API key |
| grep_app | mcp.grep.app | 跨开源仓库代码搜索 | 无需认证 |

三个 MCP 覆盖了 AI 编程助手的三个核心外部信息需求：最新信息（websearch）、API 用法（context7）、实现参考（grep_app）。

---

## 4. 后台 Agent 并发模型

> 💡 **核心创新**：让 AI agent 能像人类开发者一样"开多个终端窗口并行干活"。由两个核心模块协作：逻辑层（`background-agent/`）和可视化层（`tmux-subagent/`）。

### 4.1 架构时序

```
用户发起后台任务
        ↓
BackgroundManager.createTask()
        ↓
ConcurrencyManager.acquire()  ← 检查并发槽是否可用
        ↓（槽可用）
创建 OpenCode session
        ↓
TmuxSessionManager 创建对应 tmux pane（可视化）
        ↓
fire-and-forget 发送 prompt
        ↓
监听 session.idle 事件 / 轮询兜底
        ↓
tryCompleteTask() 双路径汇聚
        ↓
批量通知父 session
```

### 4.2 三级并发控制

`ConcurrencyManager`（`src/features/background-agent/concurrency.ts`）：

```
查找优先级：模型级 → Provider 级 → 全局默认

示例：
  anthropic/claude-sonnet-4-5 → 最多 3 个并发
  anthropic（所有模型）      → 最多 5 个并发
  全局默认                   → 5 个并发
```

**settled-flag 防 double-resolution**：当 `cancelWaiters()` 和 `release()` 同时操作同一个队列条目时，settled-flag 防止 Promise 被 resolve 又被 reject。

### 4.3 BackgroundManager 任务生命周期

**状态机**：
```
pending → running → completed
                 → error
                 → cancelled
                 → interrupt
```

**双重完成检测**（两条路径互为备份）：
1. **事件驱动**：监听 `session.idle` 事件，经过 5 秒最小运行时间校验
2. **轮询兜底**：`pollRunningTasks()` 定期检查所有 session 状态

两条路径通过 `tryCompleteTask()` 汇聚，用状态检查实现原子性防止重复完成：

```typescript
private async tryCompleteTask(task: BackgroundTask): Promise<boolean> {
  if (task.status !== "running") return false  // 防止竞态
  task.status = "completed"                     // 原子标记
  // 释放并发槽 BEFORE 任何异步操作，防止槽泄漏
  if (task.concurrencyKey) {
    this.concurrencyManager.release(task.concurrencyKey)
  }
}
```

**输出验证防误完成**：`validateSessionHasOutput()` 在标记完成前验证 session 确实有 assistant 输出（检查 text/reasoning/tool/tool_result 多种 part 类型）。

**批量通知机制**：
- 单个完成 → 静默通知（`noReply: true`）
- 全部完成 → 汇总通知并触发父 session 响应

**过期清理**：30 分钟 TTL 自动清理，进程退出时 abort 所有 running session。

### 4.4 TmuxSessionManager 与 QDEU 架构

> 💡 **为什么不用 Worker Threads？** 因为 OMO 不是自己运行 agent，而是通过 OpenCode 的 session API 创建新会话。每个后台 agent 就是一个独立的 OpenCode session，完全复用了 OpenCode 的基础设施。

`TmuxSessionManager` 遵循 **Query-Decide-Execute-Update** 模式：

```
1. QUERY:   queryWindowState()    → 获取 tmux 实际 pane 状态（唯一真实来源）
2. DECIDE:  decideSpawnActions()  → 纯函数决定操作（spawn/close/replace）
3. EXECUTE: executeActions()      → 执行 tmux 操作
4. UPDATE:  sessions.set()        → 仅在 tmux 确认成功后更新内部缓存
```

**三种 pane 操作**：
- `spawn`：分割新 pane
- `close`：关闭已完成 pane
- `replace`：用新 session 替换旧 pane（避免频繁开关）

**两层协作**：BackgroundManager 创建 session 后通知 TmuxSessionManager 创建对应 pane，200ms 延迟确保 tmux pane 在 prompt 发送前就绪。

---

## 5. Ralph Loop 自动续跑

> 💡 **什么是 Ralph Loop？** 当 agent 完成一轮工作后，如果任务还没完成，自动注入 continuation prompt 让它继续干。解决了 LLM 上下文窗口限制导致复杂任务无法一次完成的问题。

### 5.1 状态机

```
idle
  ↓（用户启动）
running
  ├─→（completion_promise 检测到）→ completed
  ├─→（达到最大迭代次数）→ max_reached
  ├─→（用户手动停止）→ stopped
  └─→（错误）→ recovering → running（自动恢复）
```

### 5.2 两条启动路径

1. **Skill 命令路径**：用户通过 `/ralph-loop` 或 `/ulw-loop` 命令启动
2. **消息模板路径**：`chat-message.ts` 检测消息文本中的 Ralph Loop 模板标记，解析 `<user-task>` 标签

参数：`prompt`、`--max-iterations`、`--completion-promise`

### 5.3 completion_promise 双通道检测

```
用户指定完成承诺（如 --completion-promise="all tests pass"）
    ↓
优先：Transcript 文件扫描（快速，无网络开销）
    ↓（失败时）
回退：Session Messages API（准确但慢）
```

**`<promise>DONE</promise>` 标签机制**：让 agent 能明确表达"任务已完成"，避免无限循环。

### 5.4 续写 Prompt 构建

`continuation-prompt-builder.ts` 构建包含以下内容的续写 prompt：
- 当前迭代进度（如 "Iteration 3/10"）
- 原始用户任务描述
- 完成承诺格式提醒
- 上下文信息（之前的工作进展）

### 5.5 Ultrawork 变体

`ulw-loop` 是 Ralph Loop 的增强版，在续写提示前触发 Ultrawork 模型覆盖：

- 检测消息中是否包含 "ultrawork"/"ulw" 关键词
- 通过 `scheduleDeferredModelOverride()` 在 microtask 中修改 SQLite，切换到更强大的模型
- 巧妙之处：TUI 底栏仍显示原始模型，但 API 调用使用覆盖后的模型（利用了 OpenCode 读取配置和发送 API 请求之间的时间差）

---

## 6. Hook 系统与消息拦截

### 6.1 Hook 创建结构

```
createHooks()
  ├── createCoreHooks()
  │     ├── createSessionHooks()     // 会话生命周期
  │     ├── createToolGuardHooks()   // 工具执行守卫
  │     └── createTransformHooks()   // 消息变换
  ├── createContinuationHooks()      // 续写/自动化
  └── createSkillHooks()             // 技能相关
```

**安全机制**：`safeCreateHook()` 包装器确保单个 hook 创建失败不影响整个插件启动。

### 6.2 事件处理顺序（18 个 hook）

```
autoUpdateChecker → claudeCodeHooks → backgroundNotificationHook → sessionNotification
→ todoContinuationEnforcer → unstableAgentBabysitter → contextWindowMonitor
→ directoryAgentsInjector → directoryReadmeInjector → rulesInjector → thinkMode
→ anthropicContextWindowLimitRecovery → agentUsageReminder → categorySkillReminder
→ interactiveBashSession → ralphLoop → stopContinuationGuard → compactionTodoPreserver
→ atlasHook
```

**顺序说明**：监控类 hook 先执行 → 内容注入类 → 控制流类。`stopContinuationGuard` 必须在 `ralphLoop` 之后，确保用户主动停止时不自动恢复。

### 6.3 Synthetic Idle 去重

```typescript
const recentSyntheticIdles = new Map<string, number>()
const recentRealIdles = new Map<string, number>()
const DEDUP_WINDOW_MS = 500
```

**背景**：`session.status(idle)` 和 `session.idle` 可能在 500ms 内重复触发。去重机制确保同一个 idle 事件只被处理一次。

### 6.4 关键 Hook 说明

| Hook | 功能 |
|------|------|
| `context-window-monitor` | 跟踪 token 使用量，区分显示限制（1M）和实际限制（200K 或 1M） |
| `preemptive-compaction` | 在上下文快满之前主动触发压缩 |
| `tool-output-truncator` | 根据模型上下文窗口大小动态决定截断阈值 |
| `session-recovery` | 检测可恢复错误（如 API 超时），自动发送 "continue" 恢复会话 |
| `anthropic-context-window-limit-recovery` | 处理 Anthropic API 上下文超限，实现多种恢复策略 |
| `session-notification` | 跨平台通知（macOS / Windows / Linux） |

### 6.5 消息拦截链

```
variant 解析
    ↓
stopContinuationGuard
    ↓
keywordDetector
    ↓
claudeCodeHooks
    ↓
autoSlashCommand
    ↓
noSisyphusGpt / noHephaestusNonGpt（模型兼容性检查）
    ↓
startWork
    ↓
ralphLoop
    ↓
ultraworkModelOverride（链末端执行）
```

### 6.6 Context Injector

**优先级系统**：

```typescript
const PRIORITY_ORDER: Record<ContextPriority, number> = {
  critical: 0, high: 1, normal: 2, low: 3,
}
```

**注入方式**：
- `chat.message` hook：在输出的 text part 前插入上下文
- `experimental.chat.messages.transform` hook：在最后一条 user message 中插入 `synthetic: true` 的 part（在 UI 中隐藏）

`synthetic: true` 标记既保证了 agent 能看到必要信息，又不污染用户的对话界面。

### 6.7 Claude Code Hooks 兼容层

提供与 Claude Code 原生 hooks 的兼容，支持 5 种处理器：
- `experimental.session.compacting` — 压缩前处理
- `chat.message` — 消息拦截
- `tool.execute.before` — 工具执行前
- `tool.execute.after` — 工具执行后
- `event` — 事件处理

---

## 7. 配置系统与 Model Fallback

### 7.1 多层配置合并

**配置来源（优先级从高到低）**：
1. 项目级：`.opencode/oh-my-opencode.json[c]`
2. 用户级：`~/.config/opencode/oh-my-opencode.json[c]`
3. 内置默认值

**两级容错**：
1. 完整验证：`OhMyOpenCodeConfigSchema.safeParse()` 尝试完整解析
2. 部分加载：验证失败时逐字段尝试，有效的保留、无效的跳过（一个配置项的错误不会导致整个配置失效）

**合并规则**：
- `agents` / `categories` / `claude_code`：`deepMerge()` 递归合并（带原型链污染防护）
- `disabled_*` 数组：Set 去重后并集合并
- 其他字段：简单覆盖

### 7.2 双层 Model Fallback

**两套独立机制，有意为之**：
- **安装时静态降级**：基于用户声明的订阅状态做静态配置，简单且确定性强
- **运行时动态降级**：基于实际可用模型做动态降级

**运行时动态降级四级优先级管道**：

```
1. UI 选择 / 用户覆盖  → 直接返回，不做可用性检查（信任用户意图）
2. Category 默认      → fuzzy match 可用模型集；若为空，检查 provider 是否连接
3. Fallback Chain     → 遍历链中每个 provider，fuzzy match；支持跨 provider 模糊匹配
4. 系统默认           → 兜底模型
```

**解析结果携带 `provenance` 字段**：标记模型来源（`override` / `category-default` / `provider-fallback` / `system-default`），便于调试。

### 7.3 Agent 模型降级链

| Agent | 首选模型 | 降级路径 |
|-------|---------|---------|
| sisyphus | claude-opus-4-6 (max) | → kimi-k2p5 → glm-5 → big-pickle |
| hephaestus | gpt-5.3-codex (medium) | 无降级（需要特定 provider） |
| oracle | gpt-5.2 (high) | → gemini-3-pro → claude-opus-4-6 |
| prometheus | claude-opus-4-6 (max) | → gpt-5.2 → kimi → gemini-3-pro |
| librarian | gemini-3-flash | → minimax-m2.5-free → big-pickle |
| explore | grok-code-fast-1 | → minimax-m2.5-free → claude-haiku → gpt-5-nano |

### 7.4 Category 模型偏好

| Category | 首选模型 | 选择理由 |
|---------|---------|---------|
| visual-engineering | gemini-3-pro | 视觉能力 |
| ultrabrain | gpt-5.3-codex xhigh | 最强推理 |
| deep | gpt-5.3-codex medium | 深度执行 |
| artistry | gemini-3-pro | 创意生成 |
| quick | claude-haiku-4-5 | 速度优先 |
| unspecified-low | claude-sonnet-4-6 | 通用低成本 |
| unspecified-high | claude-opus-4-6 max | 通用高质量 |
| writing | kimi-k2p5 | 长文写作 |

### 7.5 fuzzyMatchModel 匹配策略

```
优先级：精确匹配 > 精确 model ID 匹配 > 最短匹配

特性：
- 大小写不敏感
- 版本号标准化（claude-opus-4-6 ↔ claude-opus-4.6）
- 子串匹配
```

### 7.6 配置迁移系统

安全机制：
- `_migrations` 字段记录防止重复迁移
- 迁移前创建带时间戳的 `.bak` 备份
- 内存回写确保即使文件写入失败也能生效

迁移类型：
- **Agent 名称**：`omo/OmO` → `sisyphus`，`OmO-Plan` → `prometheus`
- **Hook 名称**：`anthropic-auto-compact` → `anthropic-context-window-limit-recovery`
- **模型版本**：`claude-opus-4-5` → `claude-opus-4-6`
- **字段迁移**：`omo_agent` → `sisyphus_agent`

### 7.7 Skill 系统六源发现

按优先级去重：

```
opencode-project (.opencode/skills/)          ← 最高优先级
  > opencode-global (~/.config/opencode/skills/)
  > project-claude (.claude/skills/)
  > project-agents (.agents/skills/)
  > user-claude (~/.claude/skills/)
  > user-agents (~/.agents/skills/)            ← 最低优先级
```

**内置 5 个技能**：playwright / playwright-cli / agent-browser（三选一）、frontend-ui-ux、git-master、dev-browser

---

## 8. Claude Code 兼容层

> 💡 **设计意图**：让用户从 Claude Code 迁移到 OpenCode + OMO 时零成本复用已有配置。OMO 不只是 OpenCode 的插件，还能加载 Claude Code 生态的资源。

### 8.1 四个 Loader

| Loader | 职责 |
|--------|------|
| `claude-code-plugin-loader` | 顶层入口，扫描 node_modules 中的 Claude Code 插件包，带 10 秒超时保护 |
| `claude-code-command-loader` | 加载 Markdown 命令文件（4 个位置，OpenCode 优先于 Claude Code） |
| `claude-code-agent-loader` | 从 `~/.claude/agents/` 和 `.claude/agents/` 加载 agent 定义 |
| `claude-code-mcp-loader` | 从 4 个位置加载 `.mcp.json` 配置，转换格式，处理环境变量 |

### 8.2 资源路径对照

| 资源类型 | Claude Code 路径 | OpenCode 路径 | 优先级 |
|---------|----------------|--------------|--------|
| 命令（用户级） | `~/.claude/commands/` | `~/.config/opencode/command/` | OpenCode > Claude Code |
| 命令（项目级） | `{cwd}/.claude/commands/` | `{cwd}/.opencode/command/` | OpenCode > Claude Code |
| Agent | `~/.claude/agents/` / `{cwd}/.claude/agents/` | — | 直接加载 |
| MCP | `~/.claude.json` / `~/.claude/.mcp.json` 等 | — | 直接加载 |
| Skill | `~/.claude/skills/` / `{cwd}/.claude/skills/` | `~/.config/opencode/skills/` 等 | OpenCode > Claude Code |

---

## 附：八大创新亮点总结

| 创新点 | 解决的问题 | 核心机制 |
|--------|-----------|---------|
| **AgentPromptMetadata 自描述系统** | 添加新 agent 需要手动修改主 agent prompt | 声明式元数据 + 自动聚合，实现开放-封闭原则 |
| **Hashline Edit** | AI 基于过时内容编辑导致静默失败 | 行级 CID 哈希，哈希不匹配则拒绝编辑 |
| **Ralph Loop completion_promise** | AI 说"done"但实际未完成 | 用户定义完成标准 + 双通道检测 |
| **双轨主 Agent** | 不同任务需要不同行为模式 | Sisyphus（委派优先）vs Hephaestus（自主优先） |
| **三级并发控制** | 不同模型 rate limit 不同 | 模型级 → Provider 级 → 全局默认 |
| **QDEU 架构** | tmux 内部缓存与实际状态不一致 | 以 tmux 实际状态为唯一真实来源，纯函数决策 |
| **Prometheus 三段式质量链** | AI 生成的计划可能不可执行 | Metis（前置）→ Prometheus → Momus（后置） |
| **Context Injector Synthetic Part** | hook 需要注入上下文但不应显示在 UI | `synthetic: true` 标记，agent 可见用户不可见 |
