https://yzddmr6.com/posts/oh-my-opencode-analyse/

# Oh My OpenCode (OMO) — Deep Dive

> **What is OMO?**
> OMO is a "super plugin" for OpenCode (a terminal AI coding tool similar to Claude Code). It upgrades OpenCode from a single-agent tool into a **multi-agent orchestration platform**. Think of it as oh-my-zsh for zsh — it doesn't change the host core, but injects a complete agent system, toolchain, and automation capabilities via a plugin mechanism.

---

## 📋 Overall Knowledge Framework

```
OMO
├── 1. Architecture           ← Four pillars + plugin interface
├── 2. Agent System           ← Greek-mythology-named multi-agent collaboration
│   ├── Agent roster
│   ├── Self-description system
│   ├── Dual-track primary agents
│   ├── Three-stage quality chain
│   └── Build / register / permissions
├── 3. Tool System            ← Three-layer tool architecture
│   ├── Hashline Edit (anti-hallucination editing)
│   ├── LSP + AST-grep (code comprehension)
│   ├── Task delegation hub
│   └── MCP integration
├── 4. Background Concurrency ← Parallel agent execution
│   ├── Three-level concurrency control
│   ├── Task lifecycle management
│   └── tmux visualization
├── 5. Ralph Loop             ← Auto-continuation mechanism
├── 6. Hook System            ← Message interception & event handling
├── 7. Configuration System   ← Multi-layer config + Model Fallback
└── 8. Claude Code Compat Layer
```

---

## 1. Overall Architecture

### 1.1 Four Pillars

The plugin entry `src/index.ts` creates four core components in sequence:

| Component | Source File | Responsibility |
|-----------|-------------|----------------|
| **Managers** | `create-managers.ts` | Background agent concurrency, tmux panes, skill MCP services, config processing |
| **Tools** | `create-tools.ts` | Registers all tools (LSP, AST-grep, background task, session manager, etc.) |
| **Hooks** | `create-hooks.ts` | Registers event hooks (Ralph Loop, context monitoring, pre-compaction, notifications) |
| **PluginInterface** | `plugin-interface.ts` | Assembles the above three into the plugin interface OpenCode expects |

### 1.2 Config Processing Pipeline (6 Serial Steps)

`createConfigHandler()` calls 6 sub-handlers in a fixed order. **The order has strict dependency requirements:**

```
applyProviderConfig
    ↓
loadPluginComponents
    ↓
applyAgentConfig  ──── returns agentResult
    ↓                         ↓
applyToolConfig  ←── needs to know which agents exist to set tool permissions
    ↓
applyMcpConfig
    ↓
applyCommandConfig
```

| Handler | Core Responsibility |
|---------|---------------------|
| `provider-config-handler` | Extracts model context limits, caches providerID/modelID → contextLimit mappings |
| `plugin-components-loader` | Loads third-party plugins with 10-second timeout protection |
| `agent-config-handler` | Most complex (~226 lines): migrates legacy names → discovers skill sources → creates built-in agents → merges configs → reorders |
| `tool-config-handler` | Disables conflicting native tools, sets fine-grained tool permission matrices per agent |
| `mcp-config-handler` | Merges built-in MCPs → user config → Claude Code MCPs → plugin MCPs |
| `command-config-handler` | 14-level priority command merge |

### 1.3 PluginInterface's 9 Hook Points

OMO doesn't implement AI conversation logic directly — it injects logic via hooks. This is a classic **middleware/interceptor pattern**.

| Hook | Responsibility |
|------|---------------|
| `tool` | Register custom tools |
| `chat.params` | Intercept/modify chat params (temperature, etc.) |
| `chat.headers` | Inject HTTP request headers |
| `chat.message` | Intercept/modify user messages (most complex) |
| `experimental.chat.messages.transform` | Transform message history |
| `config` | Config processing (the 6-step pipeline above) |
| `event` | Event dispatch (session lifecycle, idle detection, etc.) |
| `tool.execute.before` | Pre-execution interception (task routing, ralph-loop parsing) |
| `tool.execute.after` | Post-execution interception (output truncation, context monitoring) |

> 💡 **Beginner note**: Think of these 9 hooks as a "power strip." OpenCode's AI conversation flow is the circuit, and OMO plugs into key points to read, modify, or intercept the data flow — without touching OpenCode's core code.

---

## 2. Agent System

### 2.1 Agent Roster

All agents are named after Greek mythology figures, with clear division of responsibilities:

| Agent | Default Model | Temperature | Mode | Core Responsibility |
|-------|--------------|-------------|------|---------------------|
| **Sisyphus** | claude-opus-4-6 | 0.1 | primary | Main orchestrator: intent classification, task delegation, result validation |
| **Hephaestus** | gpt-5.3-codex | 0.1 | primary | Autonomous deep executor: completes complex tasks end-to-end without stopping midway |
| **Prometheus** | claude-opus-4-6 | 0.1 | — | Strategic planner: plans only, never writes code, outputs to `.sisyphus/plans/*.md` |
| **Atlas** | claude-sonnet-4-6 | 0.1 | primary | Todo list orchestrator: schedules task execution in parallel waves |
| **Oracle** | gpt-5.2 | 0.1 | subagent | Read-only high-intelligence advisor: architecture decisions and difficult debugging |
| **Metis** | claude-opus-4-6 | 0.3 | subagent | Pre-planning advisor: gap analysis before Prometheus generates a plan |
| **Momus** | gpt-5.2 | 0.1 | subagent | Plan reviewer: validates plan executability and reference correctness |
| **Librarian** | glm-4.7 | 0.1 | subagent | External doc/code search: clone repos, check official docs, search GitHub |
| **Explore** | grok-code-fast-1 | 0.1 | subagent | Internal codebase search: answers "where is X" questions |
| **Multimodal Looker** | gemini-3-flash | 0.1 | subagent | Multimodal file analysis: handles PDF/images/charts |
| **Sisyphus-Junior** | claude-sonnet-4-6 | 0.1 | all | Category task executor, derived by the category system, cannot re-delegate via task() |

> 💡 **Temperature explained**: Temperature controls the randomness of AI output. 0.1 is near-deterministic (good for coding and planning). Metis uses 0.3 because as a "pre-analysis consultant" it needs more creativity to surface potential blind spots. Higher temperature = more creative but less stable.

### 2.2 Collaboration Topology

```
User request
    ├─→ Sisyphus (everyday orchestration)
    │       ├─→ Explore / Librarian (background parallel search)
    │       ├─→ Oracle (high-difficulty consulting, non-cancellable)
    │       └─→ task(category=X) → Sisyphus-Junior (execution)
    │
    ├─→ Hephaestus (deep autonomous execution, never stops midway)
    │
    ├─→ Prometheus (planning mode)
    │       ├─→ Metis (pre-emptive gap analysis)
    │       └─→ Momus (plan review)
    │
    └─→ Atlas (plan execution, parallel waves)
            └─→ task(category=X) → Sisyphus-Junior
```

### 2.3 AgentPromptMetadata Self-Description System

Each agent declaratively describes its own capability boundaries via `AgentPromptMetadata` (`src/agents/types.ts`):

```typescript
export interface AgentPromptMetadata {
  category: AgentCategory        // "exploration" | "specialist" | "advisor" | "utility"
  cost: AgentCost                // "FREE" | "CHEAP" | "EXPENSIVE"
  triggers: DelegationTrigger[]  // scenarios where this agent should be delegated to
  useWhen?: string[]             // detailed use cases
  avoidWhen?: string[]           // scenarios to avoid
  keyTrigger?: string            // Phase 0 fast-trigger condition
  dedicatedSection?: string      // dedicated prompt section
}
```

`dynamic-agent-prompt-builder.ts` auto-aggregates all agent metadata when building Sisyphus/Hephaestus prompts:

- `buildKeyTriggersSection()` — generates Phase 0 triggers from each agent's `keyTrigger`
- `buildToolSelectionTable()` — generates a cost/selection table for tools and agents
- `buildDelegationTable()` — generates a delegation decision table from each agent's `triggers`
- `buildCategorySkillsDelegationGuide()` — generates the category + skill delegation protocol

**Core value**: When you add or remove an agent, Sisyphus's prompt **updates automatically**. Third-party plugin agents are also auto-integrated via `buildCustomAgentMetadata()` — true Open/Closed Principle in practice.

### 2.4 Dual-Track Primary Agents: Sisyphus vs Hephaestus

#### Sisyphus — The Orchestrator

**Core philosophy**: Delegate first, act yourself as a last resort.

- **Phased decision pipeline**: Phase 0 (Intent Gate) → Phase 1 (Codebase Assessment) → Phase 2A/2B/2C → Phase 3 (Completion)
- **Phase 0 intent classification**: Trivial / Explicit / Exploratory / Open-ended / Ambiguous — determines which path to take
- **Parallelism is the default**: Explore and Librarian must be launched with `run_in_background=true`, and 2–5 must be fired in parallel
- **Oracle's special status**: The only agent that can "never be cancelled" — even if you think you have the answer, you must wait for Oracle to respond
- **Evidence-driven completion criteria**:
  - File edit → `lsp_diagnostics` clean on changed files
  - Build command → exit code 0
  - Test run → pass
  - Delegation → agent result received and verified
  - **NO EVIDENCE = NOT COMPLETE**

#### Hephaestus — The Autonomous Executor

**Core difference**: No asking for permission, just act.

```
// Forbidden behaviors:
- "Should I proceed?" → JUST DO IT
- "Do you want me to run tests?" → RUN THEM
- Stopping after partial implementation → 100% OR NOTHING
```

**Intent Extraction mechanism**: Before processing each message, extract the "true intent" rather than the literal text.

**`<turn_end_self_check>` self-constraint**: The prompt requires four checks before ending each turn; any failure prevents the turn from ending.

> 💡 **Design philosophy**: Sisyphus is like a "coordination manager," Hephaestus like a "senior engineer who needs no management." The former suits multi-stakeholder coordination tasks, the latter deep end-to-end execution. Inspired by AmpCode's deep mode.

### 2.5 Prometheus Three-Stage Quality Chain

Prometheus's prompt is split into 6 semantically independent files:

| File | Responsibility |
|------|----------------|
| `identity-constraints.ts` | Identity lock: "YOU ARE A PLANNER. YOU DO NOT WRITE CODE." |
| `interview-mode.ts` | Interview strategy for 7 intent types |
| `plan-generation.ts` | Metis consulting → gap classification → summary format |
| `high-accuracy-mode.ts` | Momus review loop |
| `plan-template.ts` | Plan file Markdown template |
| `behavioral-summary.ts` | Behavioral summary and final constraints |

**Three-stage quality assurance chain**:
```
Metis (pre-emptive gap analysis) → Prometheus (plan generation) → Momus (post-hoc executability validation)
```

- Momus has **APPROVAL BIAS** (default: approve), only blocks genuine blockers to avoid infinite revision loops
- **Incremental write protocol**: Large plans first write a skeleton, then use Edit to append tasks in batches (2–4 per batch) — adapts to LLM output token limits

### 2.6 Build, Register, and Tool Permissions

#### Agent Factory Pattern

```typescript
export type AgentFactory = ((model: string) => AgentConfig) & {
  mode: AgentMode  // "primary" | "subagent" | "all"
}
```

`createBuiltinAgents()` has strict dependency order:
1. `fetchAvailableModels()` — query available models from the current provider
2. `collectPendingBuiltinAgents()` — collect all agents except the primary ones
3. `parseRegisteredAgentSummaries()` — parse externally registered agents
4. `maybeCreateSisyphusConfig()` / `maybeCreateHephaestusConfig()` / `maybeCreateAtlasConfig()` — **last**: build primary agents (because their dynamic prompts need to know which subagents are available)

#### Tool Permission Matrix

| Agent | Strategy | Blocked Tools | Design Intent |
|-------|----------|--------------|---------------|
| Oracle | Blocklist | write, edit, apply_patch, task | Read-only advisor — prevent "advisor acts itself" |
| Librarian / Explore | Blocklist | write, edit, apply_patch, task, call_omo_agent | Search only, cannot spawn child agents |
| Multimodal Looker | Allowlist | only read | Strictest — file read only |
| Metis / Momus | Blocklist | write, edit, apply_patch, task | Analyze/review only, no execution |
| Atlas | Blocklist | task, call_omo_agent | Can read/write files, but cannot spawn agents |
| Sisyphus-Junior | Blocklist | task (but allows call_omo_agent) | Can call explore/librarian, but cannot delegate via task() |

> 💡 **Subtle design**: Sisyphus-Junior allows `call_omo_agent` but blocks `task`. It can call explore/librarian for searching, but cannot delegate work via `task()` the way Sisyphus can — preventing infinite recursive delegation.

---

## 3. Tool System

OMO's tool system has three layers:
- **Native tools** (`src/tools/`): direct interaction with AI agents
- **Feature modules** (`src/features/`): complex features like background agents, tmux, context injector
- **MCP integration** (`src/mcp/`): external service connections

### 3.1 Full Tool List

| Tool Directory | Core Responsibility | Key Design |
|---------------|---------------------|------------|
| `ast-grep/` | AST-aware code search/replace | 25 languages, metavariables `$VAR`/`$$$`, smart empty-result hints |
| `lsp/` | LSP integration (6 tools) | 4-layer architecture, dual diagnostic sources |
| `hashline-edit/` | Line-hash precise editing | CID hash anti-hallucination, bottom-up application |
| `interactive-bash/` | tmux command execution | Subcommand blocklist, timeout protection |
| `background-task/` | Background task CRUD | Create/view/cancel |
| `delegate-task/` | Task delegation hub | Category routing + direct subagent calls |
| `call-omo-agent/` | Built-in agent calls | explore/librarian, sync/async |
| `look-at/` | Multimodal file analysis | Creates child session for multimodal-looker |
| `session-manager/` | Historical session operations | list/read/search/info, 60s timeout protection |
| `skill/` + `skill-mcp/` | Skill execution & MCP proxy | Loads skill content injected into prompt |
| `task/` | Task CRUD | create/get/list/update + todo-sync |
| `glob/` + `grep/` | File search & content search | Built-in ripgrep with auto-download |

### 3.2 Hashline Edit: Anti-Hallucination Precise Editing

**Core problem**: AI may generate edit instructions based on stale file content — traditional exact-text matching will silently fail or incorrectly edit.

**Solution**:

```
File content (with CID hashes):
1#ZP  import React from 'react'
2#MQ  
3#VK  export default function App() {
4#WR    return <div>Hello</div>
5#SN  }

Edit operations must include the correct LINE#ID anchor:
replace_lines(3#VK, 5#SN, newContent)

Hash mismatch → reject edit + prompt to re-read the file
```

- **CID hash character set**: `ZPMQVRWSNKTXJBYH` (16 characters)
- **2-character hash**: 16² = 256 combinations, collision probability is very low for any single file's line count
- **4 operations**: `set_line`, `replace_lines`, `insert_after`, `replace`
- **Bottom-up application**: edits applied from bottom to top, keeping line number references stable

### 3.3 LSP Integration

> 💡 **What is LSP?** Language Server Protocol splits editor capabilities into a client (the Agent) and a server (the "expert" that understands code structure). The Agent sends JSON-RPC messages asking the Server questions like "where is this function defined" or "which places reference this variable."

**LSP 4-layer architecture**:

```
LSPClient (high-level API: openFile/definition/references/diagnostics)
  └── LSPClientConnection (JSON-RPC connection management)
       └── lsp-client-transport.ts (stdio transport layer)
            └── lsp-process.ts (LSP server process management)
```

**6 LSP tools**:

| Tool | Function | LSP Method |
|------|----------|------------|
| `lsp_goto_definition` | Jump to definition | `textDocument/definition` |
| `lsp_find_references` | Find all references | `textDocument/references` |
| `lsp_symbols` | Document/workspace symbols | `textDocument/documentSymbol` |
| `lsp_diagnostics` | Get diagnostics | Dual diagnostic sources (pull + push) |
| `lsp_prepare_rename` | Pre-check rename | `textDocument/prepareRename` |
| `lsp_rename` | Execute rename | `textDocument/rename` |

**Key design details**:
- **Document version tracking**: `documentVersions` Map maintains version numbers per file
- **Content deduplication**: `lastSyncedText` Map caches last synced text, avoids redundant notifications
- **Dual diagnostic sources**: tries pull mode first, falls back to push mode on failure
- **Server auto-management**: auto-discovers and installs LSP servers

### 3.4 AST-grep

> 💡 **What is AST-grep?** Traditional grep uses string matching and can be disrupted by whitespace and newlines. AST-grep operates on code syntax structure (Abstract Syntax Tree), "reading" the code logic to precisely find function calls, variable definitions, etc., regardless of formatting.

- Supports **25 languages**
- **Metavariable patterns**: `$VAR` (single node), `$$$` (multiple nodes)
- **Smart hints**: provides correction suggestions on empty results (AI often writes incomplete AST patterns; smart hints significantly reduce failure rates)
- **Binary auto-download**

### 3.5 delegate-task: Task Delegation Hub

`createDelegateTask()` supports:

- **Category routing**: specify category → auto-uses Sisyphus-Junior + corresponding model config
- **Direct subagent calls**: specify `subagent_type` → uses that agent directly
- **Session continuation**: use `session_id` to continue an existing session with full context
- **Sync/async execution**: `run_in_background=true` returns a task_id, `false` waits for result
- **Skill injection**: `load_skills` parameter loads skill content injected into the agent system prompt

### 3.6 MCP Integration

Three built-in MCP servers (all remote HTTP/SSE connections):

| MCP | Service URL | Purpose | Auth |
|-----|-------------|---------|------|
| websearch | mcp.exa.ai (default) / mcp.tavily.com | General web search | API key |
| context7 | mcp.context7.com | Library documentation lookup | Optional API key |
| grep_app | mcp.grep.app | Cross-open-source-repo code search | No auth needed |

These three MCPs cover the three core external information needs of an AI coding assistant: latest information (websearch), API usage (context7), and implementation references (grep_app).

---

## 4. Background Agent Concurrency Model

> 💡 **Core innovation**: Lets AI agents work like human developers who "open multiple terminal windows and work in parallel." Two core modules collaborate: the logic layer (`background-agent/`) and the visualization layer (`tmux-subagent/`).

### 4.1 Architecture Sequence

```
User triggers background task
        ↓
BackgroundManager.createTask()
        ↓
ConcurrencyManager.acquire()  ← checks if concurrency slot is available
        ↓ (slot available)
Create OpenCode session
        ↓
TmuxSessionManager creates corresponding tmux pane (visualization)
        ↓
Fire-and-forget prompt send
        ↓
Listen to session.idle events / polling fallback
        ↓
tryCompleteTask() — two-path convergence
        ↓
Batch notification to parent session
```

### 4.2 Three-Level Concurrency Control

`ConcurrencyManager` (`src/features/background-agent/concurrency.ts`):

```
Lookup priority: model-level → provider-level → global default

Example:
  anthropic/claude-sonnet-4-5 → max 3 concurrent
  anthropic (all models)      → max 5 concurrent
  global default              → 5 concurrent
```

**settled-flag prevents double-resolution**: When `cancelWaiters()` and `release()` operate on the same queue entry simultaneously, the settled-flag prevents a Promise from being both resolved and rejected.

### 4.3 BackgroundManager Task Lifecycle

**State machine**:
```
pending → running → completed
                 → error
                 → cancelled
                 → interrupt
```

**Dual completion detection** (two paths as mutual backups):
1. **Event-driven**: listens for `session.idle` events, validated by a 5-second minimum runtime check
2. **Polling fallback**: `pollRunningTasks()` periodically checks all session statuses

Both paths converge at `tryCompleteTask()`, using state checks for atomicity to prevent duplicate completions:

```typescript
private async tryCompleteTask(task: BackgroundTask): Promise<boolean> {
  if (task.status !== "running") return false  // prevent race conditions
  task.status = "completed"                     // atomic mark
  // Release concurrency slot BEFORE any async operations, prevent slot leak
  if (task.concurrencyKey) {
    this.concurrencyManager.release(task.concurrencyKey)
  }
}
```

**Output validation prevents false completion**: `validateSessionHasOutput()` verifies that the session actually has assistant output before marking complete (checks text/reasoning/tool/tool_result part types).

**Batch notification**:
- Single completion → silent notification (`noReply: true`)
- All complete → summary notification + trigger parent session response

**Expiry cleanup**: 30-minute TTL auto-cleanup, all running sessions aborted on process exit.

### 4.4 TmuxSessionManager and QDEU Architecture

> 💡 **Why not Worker Threads?** Because OMO doesn't run agents itself — it creates new sessions via OpenCode's session API. Each background agent is an independent OpenCode session that fully reuses OpenCode's infrastructure.

`TmuxSessionManager` follows the **Query-Decide-Execute-Update** pattern:

```
1. QUERY:   queryWindowState()    → get actual tmux pane state (single source of truth)
2. DECIDE:  decideSpawnActions()  → pure function deciding actions (spawn/close/replace)
3. EXECUTE: executeActions()      → execute tmux operations
4. UPDATE:  sessions.set()        → update internal cache only after tmux confirms success
```

**Three pane operations**:
- `spawn`: split a new pane
- `close`: close a completed pane
- `replace`: replace an old pane with a new session (avoids frequent open/close)

**Two-layer collaboration**: BackgroundManager notifies TmuxSessionManager after creating a session; 200ms delay ensures the tmux pane is ready before the prompt is sent.

---

## 5. Ralph Loop Auto-Continuation

> 💡 **What is Ralph Loop?** When an agent finishes a round of work but the task isn't complete yet, it automatically injects a continuation prompt to keep going. Solves the problem of complex tasks that can't be completed in one LLM context window.

### 5.1 State Machine

```
idle
  ↓ (user starts)
running
  ├─→ (completion_promise detected) → completed
  ├─→ (max iterations reached) → max_reached
  ├─→ (user stops manually) → stopped
  └─→ (error) → recovering → running (auto-recovery)
```

### 5.2 Two Startup Paths

1. **Skill command path**: user starts via `/ralph-loop` or `/ulw-loop` command
2. **Message template path**: `chat-message.ts` detects Ralph Loop template markers in message text, parses `<user-task>` tags

Parameters: `prompt`, `--max-iterations`, `--completion-promise`

### 5.3 completion_promise Dual-Channel Detection

```
User specifies completion promise (e.g. --completion-promise="all tests pass")
    ↓
Primary: Transcript file scan (fast, no network overhead)
    ↓ (on failure)
Fallback: Session Messages API (accurate but slower)
```

**`<promise>DONE</promise>` tag mechanism**: Lets the agent explicitly signal "task complete," avoiding infinite loops.

### 5.4 Continuation Prompt Construction

`continuation-prompt-builder.ts` builds continuation prompts containing:
- Current iteration progress (e.g. "Iteration 3/10")
- Original user task description
- Completion promise format reminder
- Context information (previous work progress)

### 5.5 Ultrawork Variant

`ulw-loop` is the enhanced version of Ralph Loop, triggering Ultrawork model override before the continuation prompt:

- Detects whether the message contains "ultrawork"/"ulw" keywords
- Uses `scheduleDeferredModelOverride()` to modify SQLite in a microtask, switching to a more powerful model
- Clever trick: the TUI status bar still shows the original model, but API calls use the overridden model (exploits the time gap between OpenCode reading the config and sending the API request)

---

## 6. Hook System and Message Interception

### 6.1 Hook Creation Structure

```
createHooks()
  ├── createCoreHooks()
  │     ├── createSessionHooks()      // session lifecycle
  │     ├── createToolGuardHooks()    // tool execution guards
  │     └── createTransformHooks()    // message transforms
  ├── createContinuationHooks()       // continuation/automation
  └── createSkillHooks()              // skills
```

**Safety mechanism**: `safeCreateHook()` wrapper ensures a single hook creation failure doesn't affect the entire plugin startup.

### 6.2 Event Handling Order (18 Hooks)

```
autoUpdateChecker → claudeCodeHooks → backgroundNotificationHook → sessionNotification
→ todoContinuationEnforcer → unstableAgentBabysitter → contextWindowMonitor
→ directoryAgentsInjector → directoryReadmeInjector → rulesInjector → thinkMode
→ anthropicContextWindowLimitRecovery → agentUsageReminder → categorySkillReminder
→ interactiveBashSession → ralphLoop → stopContinuationGuard → compactionTodoPreserver
→ atlasHook
```

**Order rationale**: monitoring hooks first → content injection → control flow. `stopContinuationGuard` must come after `ralphLoop` to ensure user manual stops aren't auto-resumed.

### 6.3 Synthetic Idle Deduplication

```typescript
const recentSyntheticIdles = new Map<string, number>()
const recentRealIdles = new Map<string, number>()
const DEDUP_WINDOW_MS = 500
```

**Background**: `session.status(idle)` and `session.idle` may fire within 500ms of each other. The dedup mechanism ensures each idle event is processed only once.

### 6.4 Key Hooks

| Hook | Function |
|------|---------|
| `context-window-monitor` | Tracks token usage, distinguishes display limit (1M) from actual limit (200K or 1M) |
| `preemptive-compaction` | Proactively triggers compaction before context fills up |
| `tool-output-truncator` | Dynamically determines truncation threshold based on model context window size |
| `session-recovery` | Detects recoverable errors (e.g. API timeouts), auto-sends "continue" to recover |
| `anthropic-context-window-limit-recovery` | Handles Anthropic API context overflow, implements multiple recovery strategies |
| `session-notification` | Cross-platform notifications (macOS / Windows / Linux) |

### 6.5 Message Interception Chain

```
variant resolution
    ↓
stopContinuationGuard
    ↓
keywordDetector
    ↓
claudeCodeHooks
    ↓
autoSlashCommand
    ↓
noSisyphusGpt / noHephaestusNonGpt (model compatibility checks)
    ↓
startWork
    ↓
ralphLoop
    ↓
ultraworkModelOverride (executed at chain end)
```

### 6.6 Context Injector

**Priority system**:

```typescript
const PRIORITY_ORDER: Record<ContextPriority, number> = {
  critical: 0, high: 1, normal: 2, low: 3,
}
```

**Injection methods**:
- `chat.message` hook: inserts context before the text part of output
- `experimental.chat.messages.transform` hook: inserts a `synthetic: true` part into the last user message (hidden in UI)

The `synthetic: true` marker ensures agents see the necessary information without polluting the user's conversation view.

### 6.7 Claude Code Hooks Compatibility Layer

Provides compatibility with Claude Code native hooks, supporting 5 handler types:
- `experimental.session.compacting` — pre-compaction handling
- `chat.message` — message interception
- `tool.execute.before` — pre-tool-execution
- `tool.execute.after` — post-tool-execution
- `event` — event handling

---

## 7. Configuration System and Model Fallback

### 7.1 Multi-Layer Config Merge

**Config sources (highest to lowest priority)**:
1. Project-level: `.opencode/oh-my-opencode.json[c]`
2. User-level: `~/.config/opencode/oh-my-opencode.json[c]`
3. Built-in defaults

**Two-level fault tolerance**:
1. Full validation: `OhMyOpenCodeConfigSchema.safeParse()` attempts complete parse
2. Partial loading: on validation failure, tries field-by-field — valid fields kept, invalid ones skipped (one bad config field won't break everything)

**Merge rules**:
- `agents` / `categories` / `claude_code`: `deepMerge()` recursive merge (with prototype pollution protection)
- `disabled_*` arrays: set-deduplicated union merge
- Other fields: simple override

### 7.2 Dual-Layer Model Fallback

**Two independent mechanisms, intentionally separate**:
- **Install-time static fallback**: generates static config based on user-declared subscription state — simple and deterministic
- **Runtime dynamic fallback**: dynamically downgrades based on actually available models

**Runtime dynamic fallback — four-level priority pipeline**:

```
1. UI selection / user override  → return directly, no availability check (trust user intent)
2. Category default              → fuzzy match available model set; if empty, check provider connection
3. Fallback Chain                → traverse each provider in the chain, fuzzy match; supports cross-provider fuzzy match
4. System default                → fallback model
```

**Resolution results carry a `provenance` field**: marks model source (`override` / `category-default` / `provider-fallback` / `system-default`) for debugging.

### 7.3 Agent Model Fallback Chains

| Agent | Preferred Model | Fallback Path |
|-------|----------------|---------------|
| sisyphus | claude-opus-4-6 (max) | → kimi-k2p5 → glm-5 → big-pickle |
| hephaestus | gpt-5.3-codex (medium) | No fallback (requires specific provider) |
| oracle | gpt-5.2 (high) | → gemini-3-pro → claude-opus-4-6 |
| prometheus | claude-opus-4-6 (max) | → gpt-5.2 → kimi → gemini-3-pro |
| librarian | gemini-3-flash | → minimax-m2.5-free → big-pickle |
| explore | grok-code-fast-1 | → minimax-m2.5-free → claude-haiku → gpt-5-nano |

### 7.4 Category Model Preferences

| Category | Preferred Model | Reason |
|---------|----------------|--------|
| visual-engineering | gemini-3-pro | Visual capabilities |
| ultrabrain | gpt-5.3-codex xhigh | Strongest reasoning |
| deep | gpt-5.3-codex medium | Deep execution |
| artistry | gemini-3-pro | Creative generation |
| quick | claude-haiku-4-5 | Speed priority |
| unspecified-low | claude-sonnet-4-6 | General low-cost |
| unspecified-high | claude-opus-4-6 max | General high-quality |
| writing | kimi-k2p5 | Long-form writing |

### 7.5 fuzzyMatchModel Strategy

```
Priority: exact match > exact model ID match > shortest match

Features:
- Case-insensitive
- Version normalization (claude-opus-4-6 ↔ claude-opus-4.6)
- Substring matching
```

### 7.6 Config Migration System

Safety mechanisms:
- `_migrations` field records applied migrations to prevent repeats
- Creates timestamped `.bak` backups before migrating
- In-memory writeback ensures effect even if file write fails

Migration types:
- **Agent names**: `omo/OmO` → `sisyphus`, `OmO-Plan` → `prometheus`
- **Hook names**: `anthropic-auto-compact` → `anthropic-context-window-limit-recovery`
- **Model versions**: `claude-opus-4-5` → `claude-opus-4-6`
- **Field migration**: `omo_agent` → `sisyphus_agent`

### 7.7 Skill System: Six-Source Discovery

Deduplicated by priority:

```
opencode-project (.opencode/skills/)          ← highest priority
  > opencode-global (~/.config/opencode/skills/)
  > project-claude (.claude/skills/)
  > project-agents (.agents/skills/)
  > user-claude (~/.claude/skills/)
  > user-agents (~/.agents/skills/)            ← lowest priority
```

**5 built-in skills**: playwright / playwright-cli / agent-browser (one of three, selected by `browserProvider` config), frontend-ui-ux, git-master, dev-browser.

---

## 8. Claude Code Compatibility Layer

> 💡 **Design intent**: Zero-cost migration for users moving from Claude Code to OpenCode + OMO, with full reuse of existing config. OMO isn't just an OpenCode plugin — it can also load Claude Code ecosystem resources.

### 8.1 Four Loaders

| Loader | Responsibility |
|--------|---------------|
| `claude-code-plugin-loader` | Top-level entry point, scans node_modules for Claude Code plugin packages, 10-second timeout |
| `claude-code-command-loader` | Loads Markdown command files from 4 locations (OpenCode takes priority over Claude Code) |
| `claude-code-agent-loader` | Loads agent definitions from `~/.claude/agents/` and `.claude/agents/` |
| `claude-code-mcp-loader` | Loads `.mcp.json` config from 4 locations, converts format, handles env variable expansion |

### 8.2 Resource Path Reference

| Resource Type | Claude Code Path | OpenCode Path | Priority |
|--------------|-----------------|---------------|---------|
| Commands (user) | `~/.claude/commands/` | `~/.config/opencode/command/` | OpenCode > Claude Code |
| Commands (project) | `{cwd}/.claude/commands/` | `{cwd}/.opencode/command/` | OpenCode > Claude Code |
| Agents | `~/.claude/agents/` / `{cwd}/.claude/agents/` | — | Direct load |
| MCP | `~/.claude.json` / `~/.claude/.mcp.json` etc. | — | Direct load |
| Skills | `~/.claude/skills/` / `{cwd}/.claude/skills/` | `~/.config/opencode/skills/` etc. | OpenCode > Claude Code |

---

## Appendix: Eight Key Innovations

| Innovation | Problem Solved | Core Mechanism |
|-----------|---------------|----------------|
| **AgentPromptMetadata self-description** | Adding a new agent required manually editing the main agent prompt | Declarative metadata + auto-aggregation, true Open/Closed Principle |
| **Hashline Edit** | AI editing from stale content caused silent failures | Line-level CID hash; hash mismatch rejects the edit |
| **Ralph Loop completion_promise** | AI says "done" but task isn't actually complete | User-defined completion criteria + dual-channel detection |
| **Dual-track primary agents** | Different tasks need different behavior modes | Sisyphus (delegate-first) vs Hephaestus (autonomous-first) |
| **Three-level concurrency control** | Different models have different rate limits | Model-level → provider-level → global default |
| **QDEU architecture** | tmux internal cache diverges from actual state | tmux actual state as single source of truth, pure-function decision logic |
| **Prometheus three-stage quality chain** | AI-generated plans may not be executable | Metis (pre) → Prometheus → Momus (post) |
| **Context Injector synthetic part** | Hooks need to inject context without showing it in UI | `synthetic: true` marker: agent-visible, user-invisible |               

