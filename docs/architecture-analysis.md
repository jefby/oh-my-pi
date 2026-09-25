# Oh My Pi (OMP) — 软件架构分析

> 项目版本: 18.3.1  
> 分析日期: 2026-09-25  
> 主要入口: `packages/coding-agent/src/cli.ts` → `omp` CLI

---

## 1. 项目概览

**Oh My Pi (OMP)** 是一款 AI 编程代理 CLI 工具，提供 `read`、`bash`、`edit`、`write` 等工具并与 LLM 进行交互式会话。项目基于 **Bun** 运行时，采用 **Monorepo** 架构，包含 16 个 TypeScript 包和 11 个 Rust crate（另含 `vendor/` 第三方代码）。构建支持 `bun build` 与 Bazel（`BUILD.bazel` / `MODULE.bazel`）双流水线。

**与原始 Pi 的关系**: oh-my-pi (omp) 是 [pi-mono](https://github.com/badlogic/pi-mono)（Mario Zechner）的 fork，重写为 coding-first 表面。核心沿用 Pi 的架构与包名（agent loop、LLM 客户端、TUI、扩展模型、会话机制；`pi-ai`/`pi-agent-core`/`pi-tui`/`pi-utils`/`pi-natives`），但已深度分叉，不再同步 upstream。

- **重写**: 原生工具链全部 in-process（grep/glob/find/brush bash + 58 coreutils，零 fork/exec；Pi 原版 shell out 到 rg/find/bash）；新增 Rust crate `pi-edit`（编辑引擎）、`pi-vcs`、`pi-voice` 等
- **扩展**: subagents/task + wait、stats 仪表盘、mnemopi/hindsight 记忆、协作系统 + collab-web、browser/computer eval preludes + browser-relay (Chrome 扩展)、ACP/IRC、metaharness benchmark、Bazel 构建

### 1.1 核心数据流

```
User Input (CLI/TUI) 
  → cli.ts (entry) 
  → main.ts (parse args + create session)
  → sdk.ts (createAgentSession)
  → AgentSession (session/agent-session.ts)
    → @oh-my-pi/pi-agent-core Agent.run()
      → @oh-my-pi/pi-ai LLM Provider
        → External API (Anthropic, OpenAI, etc.)
  ← Tool Results ← tools/*.ts
  ← TUI Rendering ← packages/tui
```

---

## 2. 包依赖架构

### 2.1 依赖层级图

```
┌─────────────────────────────────────────────────────┐
│                  packages/coding-agent               │  ← 主 CLI 应用
│  (tools, session, modes, edit, prompts, config...)   │
├─────────────────┬─────────────────┬─────────────────┤
│ @oh-my-pi/      │ @oh-my-pi/      │ @oh-my-pi/      │
│ pi-agent-core   │ pi-ai           │ pi-tui           │
│ (Agent 运行时)   │ (LLM 客户端)    │ (终端 UI 库)     │
├─────────────────┼─────────────────┼─────────────────┤
│ @oh-my-pi/      │ @oh-my-pi/      │ @oh-my-pi/      │
│ pi-catalog      │ pi-utils        │ pi-natives       │
│ (模型目录)       │ (共享工具)       │ (Rust N-API)    │
├─────────────────┼─────────────────┼─────────────────┤
│ @oh-my-pi/      │ @oh-my-pi/      │ @oh-my-pi/      │
│ pi-wire         │ omp-stats       │ collab-web       │
│ (线协议类型)      │ (本地仪表盘)     │ (协作 Web 客户端) │
└─────────────────┴─────────────────┴─────────────────┘
```

### 2.2 包详细说明

| 包名 | 描述 | 关键依赖 |
|------|------|---------|
| `@oh-my-pi/pi-coding-agent` | 主 CLI 应用 — 工具、会话、模式、编辑系统、提示词管理 | 全部其余包 |
| `@oh-my-pi/pi-agent-core` | Agent 运行时 — 传输抽象、会话循环、工具调用、状态管理、附件支持 | pi-ai |
| `@oh-my-pi/pi-ai` | 多供应商 LLM 客户端 — Anthropic、OpenAI、Google、Bedrock、Ollama 等 | pi-catalog |
| `@oh-my-pi/pi-catalog` | 模型目录 — 内置 models.json、供应商描述符、模型身份/effort 策略 | — |
| `@oh-my-pi/pi-tui` | 终端 UI 库 — 差分渲染、Markdown、编辑器、键位绑定、工具渲染器 (`tools/*`) | — |
| `@oh-my-pi/pi-utils` | 共享工具集 — 日志、路径、流、临时文件、worker host、postmortem | — |
| `@oh-my-pi/pi-natives` | Rust N-API 原生绑定 — grep、glob、编辑引擎、VCS、桌面、SIXEL、token 计数等 | crates/pi-natives + 配套 crate |
| `@oh-my-pi/pi-wire` | 共享线协议类型 — 用于协作和跨进程通信 | — |
| `@oh-my-pi/omp-stats` | 本地可观测性仪表盘 (`omp stats`) + stats worker | — |
| `@oh-my-pi/snapcompact` | 会话压缩引擎（含原生渲染） | — |
| `@oh-my-pi/pi-mnemopi` | 记忆嵌入系统 | — |
| `@oh-my-pi/collab-web` | 协作 Web 客户端 (React) | pi-wire |
| `@oh-my-pi/browser-relay` | Chrome 扩展 + 本地 CDP 中继 (`omp browser-relay`)，让 browser prelude 驱动用户现有标签页 | — |
| `@oh-my-pi/pi-metaharness` | 统一 benchmark runner、Harbor run 存储、REST/SSE API、实时 Web 仪表盘 | — |
| `@oh-my-pi/omptype` | ArkType 兼容的运行时 schema 验证，懒 JIT 编译 | — |

### 2.3 Rust Crates

```text
crates/
├── pi-natives/      ← 主要 N-API 绑定 (grep, fd, glob, edit, vcs, desktop, sixel, …)
├── pi-ast/          ← AST 解析 / 语言元数据
├── pi-builtins/     ← 内嵌 brush shell 的内置命令与进程内 CLI 工具
├── pi-diff/         ← diff 引擎
├── pi-edit/         ← Rust 编辑引擎 (流式 EditSession、diff 预览)
├── pi-iso/          ← 隔离 / 文件系统操作
├── pi-shell/        ← Shell 解析 + 跨平台虚拟文件系统集成
├── pi-vcs/          ← 进程内 Git/Jujutsu 操作
├── pi-vfs/          ← 跨平台虚拟文件系统提供
├── pi-voice/        ← 语音 (STT/TTS) 支持
├── pi-walker/       ← 文件系统遍历 + 扫描缓存
└── vendor/          ← 第三方供应商代码
```

---

## 3. 主 CLI 启动流程

### 3.1 入口点

```
cli.ts (#!/usr/bin/env bun)
  │
  ├─ 清理 macOS malloc 环境变量
  ├─ 检查 Bun 版本 >= 1.3.14
  ├─ 设置进程标题
  │
  ├─ isProcessEntry?
  │   ├─ No → 模块导入模式 (SDK嵌入/测试)
  │   └─ Yes → runCli(process.argv.slice(2))
  │
  ├─ runWorkerEntrypoint(workerArg)   ← selector 常量见 cli/worker-selectors.ts + cli.ts
  │   ├─ __omp_worker_tiny_inference      → tiny/worker (子进程)
  │   ├─ __omp_worker_stats_sync          → omp-stats/sync-worker (worker_thread)
  │   ├─ __omp_worker_stats_activity      → stats/activity-worker (子进程, IPC)
  │   ├─ __omp_worker_terminal_output     → launch/terminal-output-worker (worker_thread)
  │   ├─ __omp_worker_tab                 → browser/tab-worker-entry (worker_thread)
  │   ├─ __omp_worker_computer            → computer/worker-entry (worker_thread)
  │   ├─ __omp_worker_js_eval             → eval/js/worker-entry (worker_thread)
  │   ├─ __omp_worker_js_eval_process     → eval/js/process-entry (子进程, IPC)
  │   ├─ __omp_worker_stt                 → stt/asr-worker (子进程, IPC)
  │   ├─ __omp_worker_tts                 → tts/tts-worker (子进程, IPC)
  │   ├─ __omp_worker_mnemopi_embed       → mnemopi/embed-worker (子进程, IPC)
  │   ├─ __omp_worker_daemon_broker       → launch/broker (daemon worker)
  │   ├─ __omp_worker_lsp_mux             → lsp/mux daemon
  │   ├─ __omp_worker_ida_host            → ida/host
  │   └─ __omp_worker_blob_broker         → blob-broker/server
  │   └─ 通过 argv selector 复用同一入口模块 (declareWorkerHostEntry)
  │
  ├─ --smoke-test → runSmokeTest()  
  │    覆盖: stats sync/activity、stats dashboard HTTP 检查、tiny title、stt、
  │    js_eval、computer、tts、mnemopi_embed、daemon broker、lsp mux、ida host、
  │    blob broker、terminal output
  │
  └─ runCli(argv)
      ├─ extractProfileFlags → 解析 --profile, --alias
      ├─ resolveProfileEnv → 设置活动 profile
      ├─ declareWorkerHostEntry() → 声明 worker 宿主
      ├─ resolveCliArgv → 解析子命令路由
      └─ run({ commands, argv }) → CLI 命令调度
```

### 3.2 命令注册表 (`cli-commands.ts`)

```typescript
// 所有顶层命令 (49)
commands: [
  launch, acp, auth-broker, auth-gateway, agents, bench, browser-relay,
  cleanse, clip, collab, commit, completions, __complete, compress,
  config, dry-balance, find, gc, grep, gallery, git, grievances,
  images, if-bench, install, join, login, models, plugin, ps, say,
  play, share, setup, shell, read, render, skill, ssh, stats,
  stream, tiny-models, token, toks, ttsr, update, usage, worktree,
  search
]
// 保留词防止泄露给 LLM: extensions, list, remove, uninstall,
// marketplace, discover, upgrade, enable, disable (提示 omp plugin …)
// reservedTopLevelWordMessage(): 单动词总提示; 多词仅当参数符合插件语法
```

### 3.3 主启动时序 (`main.ts`)

```
applyStartupCwd          设置工作目录
  ↓
discoverAuthStorage      OAuth/API密钥存储
  ↓
new ModelRegistry        模型注册表 (含认证信息)
  ↓
Settings.init            全局/项目配置加载
  ↓
initializeWithSettings   供应商发现持久化
  ↓
initTheme                主题初始化 (符号集/色盲模式/主题)
  ↓
resolveModelScope        作用域模型解析
  ↓
createSessionManager     创建/恢复/分叉 会话管理器
  ├─ --resume <id>       恢复指定会话
  ├─ --fork <id>         分叉已有会话
  ├─ --continue           继续最近会话
  ├─ autoResume           自动恢复 (设置项)
  └─ 默认                  新建会话
  ↓
buildSessionOptions      构建会话选项
  ├─ 自动发现 SYSTEM.md / APPEND_SYSTEM.md
  ├─ 模型解析 (provider/model pattern)
  ├─ prewalk/planYolo 配置
  ├─ Thinking level
  ├─ 工具/技能/规则 过滤
  └─ 扩展路径
  ↓
createAgentSession       (sdk.ts) 创建 AgentSession
  ↓
runInteractiveMode       进入交互模式 (TUI)
  ├─ setupWizard          首次引导
  ├─ versionCheck         版本更新检查
  ├─ initialMessages      初始消息
  └─ mode.getUserInput()  主循环
      └─ submitInteractiveInput → session.prompt() → Agent.run()
```

---

## 4. Agent 运行时架构

### 4.1 Agent 会话层 (`session/agent-session.ts`)

```
AgentSession
├─ agent                 (@oh-my-pi/pi-agent-core)       核心 agent
├─ sessionManager        (session/session-manager.ts)    持久化 + BlobStore + ArtifactManager
├─ settings              (config/settings.ts)            全局设置
├─ memoryEnabled         boolean                         记忆子系统开关
├─ yieldQueue            (yield-queue.ts)                yield/ask 队列
├─ rawSseDebugBuffer     RawSseDebugBuffer               SSE debug 缓冲
├─ tokenRate             TokenRateMeter                  token 速率
├─ #models               ModelControls                   模型切换控制
├─ #tools                SessionTools                    工具注册/门控
├─ #prewalk              PrewalkCoordinator              预读协调 (PrewalkMode)
├─ #providerBoundary     SessionProviderBoundary         供应商边界
├─ #advisors             SessionAdvisors                 顾问子系统
├─ #maintenance          SessionMaintenance              会话维护/清理
├─ #handoff              SessionHandoff                  会话交接 (fork/resume)
├─ #recovery             TurnRecovery                    轮次恢复
├─ #todo                 TodoTracker                     TODO 跟踪
├─ #modelMentions        ModelMentionRegistry            模型提及注册
├─ #bash                 BashRunner                      bash 执行器
├─ #eval                 EvalRunner + #evalToolSession     eval VM 运行时
├─ #asyncJobManager      AsyncJobManager                  后台任务 (wait 工具)
├─ #irc                  IrcBridge                       IRC 协作桥接
├─ #ttsr                 TtsrCoordinator                 语音协调
├─ #stats                SessionStatsTracker             会话统计
├─ #streamingEditGuard   StreamingEditGuard              流式编辑防护
├─ #loopGuards           LoopGuards                      循环守卫
└─ #memory               SessionMemory                   短期记忆
```

### 4.2 核心 Agent 循环

```
Agent.prompt(userInput)
  │
  ├─ 添加用户消息到上下文
  ├─ 构建系统提示 (系统提示 + 工具模式 + 技能 + 规则)
  ├─ 选择模型 & 供应商
  │
  ├─ Agent.run()
  │   ├─ 调用 LLM API (流式)
  │   ├─ 流式输出 → TUI 渲染
  │   ├─ 解析工具调用
  │   │   ├─ ToolChoiceQueue 决定执行哪个工具
  │   │   └─ 工具执行 (tools/*.ts)
  │   │       ├─ 工具结果渲染
  │   │       └─ 结果回填到上下文
  │   ├─ 循环直到:
  │   │   ├─ 最终回复
  │   │   ├─ 达到工具调用限制
  │   │   ├─ 超时
  │   │   └─ 错误/中断
  │   └─ 返回最终 AssistantMessage
  │
  ├─ 轮次后处理
  │   ├─ 会话持久化 (JSONL/sql)
  │   ├─ 压缩 (snapcompact)
  │   ├─ 标题刷新
  │   └─ 自动继续检查
  │
  └─ 等待下一轮输入
```

---

## 5. 工具系统

### 5.1 工具架构

每个工具实现 `AgentTool` 接口 (来自 `@oh-my-pi/pi-agent-core`)：

```typescript
interface AgentTool {
  name: string;
  description: string;       // 来自 prompts/tools/*.md
  parameters: Type<...>;     // ArkType-compatible schema (@oh-my-pi/omptype)
  loadMode: ToolLoadMode;    // "essential" | "discoverable" | "disabled"
  render?: ToolRenderOptions;
  execute(ctx, args, update?): Promise<AgentToolResult>;
}
```

### 5.2 核心工具分类

| 类别 | 工具 | 加载模式 |
|------|------|---------|
| **文件操作** | `read`, `write`, `edit`, `glob`, `find` | essential |
| **Shell** | `bash`; `computer` → eval prelude (tools.computer 门控) | essential / prelude |
| **代码分析** | `grep`, `ast-grep`, `ast-edit`, `ida` | discoverable |
| **子代理/任务** | `task` (子代理), `wait` (job 结果) | essential |
| **知识管理** | `learn`, `manage_skill`, `recall`, `edit`, `reflect`, `retain` (memory backend: hindsight/mnemopi/local) | essential / settings-gated |
| **Web** | `web_search`; `browser` → eval prelude; URL 抓取并入 `read` 管道 | discoverable / settings-gated |
| **协作** | `xdev` 设备挂载, `irc` 桥接 | — |
| **GitHub** | `gh` (PR/issue/搜索) | discoverable |
| **调试** | `debug` (DAP), `lsp` | discoverable |
| **其他** | `todo`, `checkpoint`, `rewind`, `context_notes`, `new_context`, `security_scan`, `ask`, `vibe*`, `image-gen*`; 隐藏: `think`, `yield`, `goal` | discoverable / settings-gated |

**关键变化 (vs v17):**
- **Eval prelude**: browser/computer 不再是模型工具，而是注入 JS/Python eval VM 的能力
  (`eval-preludes.ts`)，需 eval 工具激活；由 tools.browser/tools.computer 设置门控。
- **hub 已移除** — 后台任务改为 `wait` + AsyncJobManager。
- **sqlite-reader 不再是独立工具** — URL/data URI 读取并入 read/fetch/write 管道。
- **xdev**: tools.xdev 启用时，内置工具重新挂载到 xd:// 设备命名空间 (write 为执行传输)。

### 5.3 工具集注册

```
工具发现流程:
1. 内置注册表 (tools/index.ts) — BUILTIN_TOOLS 工厂映射 + HIDDEN_TOOLS;
   essential 名称由 ESSENTIAL_BUILTIN_TOOL_NAMES 钉住 (essential-tools.ts drift guard)
2. 设置门控 (isToolAllowed) — web_search, wait, task 递归深度,
   learn/manage_skill autolearn, memory backend 工具
3. 条件注册: vibe 工具对 (createVibeTools), image-gen (模型支持时);
   eval preludes (browser/computer — 非模型工具)
4. Extension registerTool (extensibility/extensions) — 扩展注册
5. MCP 工具 (mcp/tool-bridge.ts) — MCP 服务器提供
6. ACP 工具 (modes/acp) — Agent 客户端协议
7. RPC 主机工具 — 远程过程调用模式
8. xdev 挂载: tools.xdev 启用时，内置工具重挂到 xd:// 设备命名空间
```

每个工具使用 Handlebars 模板化的 `prompts/tools/*.md` 文件作为描述，渲染器使用 TUI 组件进行输出显示。

---

## 6. LLM 供应商架构

### 6.1 供应商系统 (`packages/ai/src/providers/`)

```
providers/
├── anthropic*           Anthropic API (Claude) ← 默认 (identity/slow-mode/user-profiles)
├── openai-responses/    OpenAI Responses API
├── openai-codex/        OpenAI Codex API (+ attestation/compaction)
├── openai-completions   OpenAI Completions API
├── azure-openai-responses/  Azure OpenAI
├── amazon-bedrock       Bedrock (mantle) + aws credentials/eventstream/sigv4
├── apple-foundation-models  Apple Foundation Models
├── google*              Google AI (Gemini) / Vertex AI / Gemini CLI
├── ollama/              本地 Ollama
├── cursor/              Cursor IDE 集成
├── devin/               Devin AI
├── kimi/                Moonshot Kimi
├── gitlab-duo*          GitLab Duo (+ workflow)
├── mock/                测试 Mock
├── synthetic/           合成供应商 (测试)
└── pi-native*           Pi Native server/client (本地推理桥接)
```

### 6.2 模型目录 (`packages/catalog/`)

```
catalog/
├── src/
│   ├── models.json        ← 自动生成 (禁止手动编辑)
│   ├── types.ts           模型/供应商/Usage 类型
│   ├── identity/          模型 ID 分类 (家族/版本解析)
│   ├── model-thinking.ts  思考元数据策略
│   ├── model-manager.ts   模型管理器/缓存
│   ├── model-cache.ts     模型缓存
│   ├── variant-collapse.ts 变体折叠
│   ├── provider-models/   供应商模型描述符/解析器
│   │   ├── descriptors.ts   供应商目录入口
│   │   └── openai-compat.ts OpenAI 兼容解析器
│   ├── discovery/         供应商发现逻辑
│   ├── compat/            OpenAI 兼容层
│   ├── effort/            努力级别 (Effort)
│   └── wire/              线协议格式化
```

### 6.3 认证 & 密钥管理

```
api-key-resolver.ts     ← API 密钥解析
auth-storage.ts         ← 认证存储 (加密)
auth-broker/            ← OAuth Broker
auth-gateway/           ← 认证网关
  providers/oauth/      ← OAuth 提供者注册
```

---

## 7. 编辑系统

### 7.1 编辑模块 (`packages/coding-agent/src/edit/`)

```
edit/
├── index.ts                编辑工具主入口
├── normalize.ts            路径规范化
├── schemas.ts              ArkType 参数 schema
├── settings.ts             编辑设置 (模式等)
├── store.ts                文件快照存储
├── auto-repair.ts          patch 自动修复 (+ auto-repair.md prompt)
└── blackbox.ts             黑盒验证

引擎实现位于 crates/pi-edit (Rust):
- EditSession — 流式编辑会话 (onPreview 回调 + host writer apply)
- EditStore   — 快照/剪贴板/no-op 状态
pi-natives/edit.rs 通过 N-API 暴露给 JS
```

### 7.2 编辑模式

- **默认模式**: 标准文本替换
- **散列行模式**: 使用带散列的行号进行精确编辑
- **AST 编辑**: 基于 AST 的结构化编辑

---

## 8. 配置系统

### 8.1 配置层级

```
settings.ts
├── 全局配置 (~/.omp/config.yml)
├── 项目配置 (.omp/config.yml)
├── CLI 参数覆盖 (--model, --system-prompt 等)
├── 运行时覆盖 (Settings.override())
└── 环境变量覆盖 (PI_SMOL_MODEL, PI_SLOW_MODEL 等)
```

### 8.2 关键配置模块

| 模块 | 功能 |
|------|------|
| `settings.ts` + `all-settings.ts`/`registry.ts` | 中心化设置注册表 (懒加载 cfg* accessor、默认值、继承链) |
| `config-file.ts` | 配置文件加载 (YAML > 目录层次) |
| `models-config-schema-bundle.ts` / `models-config-schema.ts` | Models 配置 schema (ArkType) |
| `model-registry.ts` | 模型注册表 + 认证 |
| `model-resolver.ts` | 模型字符串解析/角色解析 |
| `model-roles.ts` | 模型角色 (default/smol/slow/plan) |
| `models-config.ts` | Models 配置文件 |
| `prompt-templates.ts` | 提示模板 |
| `service-tier.ts` | 服务层级 (按模型家族) |
| `append-only-context-mode.ts` | 追加专用上下文模式 |
| `inline-tool-descriptors-mode.ts` | 内联工具描述模式 |

> 注: `settings-schema.ts` 已拆分为 `all-settings.ts` + `models-config-schema-bundle.ts`; `keybindings.ts` 移至 pi-tui (`app-keybindings.ts`); `provider-globals.ts` 已移除。

---

## 9. 提示系统

### 9.1 提示层次结构 (`prompts/`)

```
prompts/
├── system/
│   ├── system-prompt.md              ← 核心系统提示
│   ├── title-system.md               ← 标题生成提示
│   ├── tool-call-loop-redirect.md    ← 工具调用循环重定向
│   ├── thinking-loop-redirect.md     ← 思考循环重定向
│   └── ... (60+ 系统提示片段)
├── tools/
│   ├── bash.md                       ← Bash 工具描述
│   ├── read.md                       ← Read 工具描述
│   └── ... (工具描述 MD 文件)
├── agents/
│   ├── subagent-system-prompt.md     ← 子代理系统提示
│   └── subagent-user-prompt.md       ← 子代理用户提示
├── advisor/                          ← 顾问子系统提示
├── goals/                            ← 目标系统提示
├── memories/                         ← 记忆系统提示
├── skills/                           ← 技能提示
└── steering/                         ← 引导提示
```

### 9.2 提示加载约定

- 提示文件为静态 `.md` 文件
- 动态内容通过 **Handlebars** 模板化
- `import content from "./prompt.md" with { type: "text" }` 导入
- 禁止内联字符串构建

---

## 10. TUI 渲染系统

### 10.1 组件架构 (`packages/tui/src/`)

```typescript
tui/
├── index.ts                       ← 公共 API
├── components/
│   ├── box.ts                     框布局
│   ├── editor.ts                  TUI 编辑器
│   ├── input.ts                   输入框
│   ├── loader.ts                  加载动画
│   ├── markdown.ts                Markdown 渲染
│   ├── scroll-view.ts             滚动视图
│   ├── select-list.ts             选择列表
│   ├── text.ts                    文本显示
│   └── ...
├── tools/                         工具渲染器 (bash/read/edit/grep/task/todo/
│                                  think/resolve/vibe/streaming-output/xdev…) + apps/
├── keybindings.ts                 键位绑定系统
├── fuzzy.ts                       模糊匹配
├── canvas.ts                      画布
├── deccara.ts                     DECCARA 优化
└── ...
```

### 10.2 渲染流程 (编码代理)

```
InteractiveMode (modes/interactive-mode.ts)
├─ TUI (tui/index.ts)                   终端抽象
├─ processTerminal (ProcessTerminal)     终端渲染器
├─ 组件树
│   ├─ Container (根容器)
│   ├─ Header (状态行/标题)
│   ├─ ScrollView (消息历史)
│   │   ├─ Message
│   │   │   ├─ Markdown (用户/助手消息)
│   │   │   └─ ToolExecution (工具调用)
│   │   │       ├─ ToolBashRenderer
│   │   │       ├─ ToolReadRenderer
│   │   │       ├─ ToolEditRenderer
│   │   │       └─ ToolWriteRenderer
│   └─ Footer (输入行)
│       └─ Input (文本输入)
```

---

## 11. 扩展系统

### 11.1 扩展类型

```
extensibility/
├── extensions/                  扩展运行时
│   ├── runner.ts                扩展执行器
│   ├── types.ts                 扩展类型定义
│   └── load-errors.ts           加载错误处理
├── custom-tools/                自定义工具
├── custom-commands/             自定义命令
├── hooks/                       钩子系统
└── plugins/
    ├── marketplace/             插件市场
    └── marketplace-auto-update.ts 自动更新
```

扩展可通过以下发现:
1. `~/.omp/extensions/` — 用户全局扩展
2. `.omp/extensions/` — 项目扩展
3. `--extension <path>` — CLI 参数扩展
4. 插件市场安装

---

## 12. MCP (Model Context Protocol)

### 12.1 MCP 子系统 (`mcp/`)

```
mcp/
├── manager.ts          MCP 服务器管理器 (生命周期)
├── config.ts           配置文件解析
├── config-writer.ts    配置写入
├── loader.ts           服务器加载
├── tool-bridge.ts      MCP 工具桥接到 Agent 工具系统
├── tool-cache.ts       工具缓存
├── oauth-discovery.ts  OAuth + MCP 发现
├── oauth-flow.ts       OAuth 认证流
├── oauth-credentials.ts OAuth 凭证
├── smithery-connect.ts Smithery 注册表连接
├── smithery-registry.ts Smithery 注册表
├── json-rpc.ts         JSON-RPC 传输
├── request-id.ts       请求 ID 生成
├── settings.ts         MCP 设置解析
├── startup-events.ts   启动事件
├── smithery-auth.ts    Smithery 认证
├── timeout.ts          MCP 超时
├── types.ts            MCP 类型定义
├── client.ts           MCP 客户端
├── errors.ts           MCP 错误类型
└── transports/         MCP 传输层 (stdio/SSE)
```

---

## 13. ACP (Agent Client Protocol)

### 13.1 ACP 模式 (`modes/acp/`)

ACP 模式允许 OMP 作为 Agent 客户端协议的宿主，接收来自 IDE 或其他 ACP 兼容客户端的连接。

```
acp/
├── index.ts                    ACP 模式入口
├── acp-agent.ts                ACP Agent 适配
├── acp-client-bridge.ts        客户端桥接 (权限/会话)
├── acp-event-mapper.ts         ACP 事件映射
├── acp-mode.ts                 ACP 模式运行时
└── terminal-auth.ts            终端认证
(权限门移至 session/acp-permission-gate.ts)
```

---

## 14. 协作系统

### 14.1 实时协作 (`collab/`)

```
collab/
├── host.ts                 协作主机 (广播会话)
├── guest.ts                协作访客 (远程查看/交互)
├── relay-client.ts         中继客户端
├── controller.ts           协作控制器
├── protocol.ts             协作协议
├── crypto.ts               加密
├── registry.ts             会话注册表
├── settings.ts             协作设置
├── display-name.ts         显示名称
└── replication-shrink.ts   复制压缩
```

### 14.2 Web 协作客户端

`packages/collab-web/` — 基于 React 的 Web 客户端，通过 WebSocket 中继连接以进行远程会话查看。

---

## 15. 子代理 & 任务系统

### 15.1 层次化任务

```
AgentSession
  ├─ Agent (主要) 
  │   └─ 任务工具调用 → TaskTool (task/)
  │       ├─ persisted-revive.ts    持久化子代理恢复
  │       ├─ output-manager.ts      子代理输出管理
  │       ├─ types.ts               子代理/任务类型
  │       └─ ...
  └─ wait 工具 + AsyncJobManager    后台任务/结果等待 (hub 工具已移除)
      ├─ async/async-job-manager.ts  后台 job 生命周期
      └─ task/wait.ts               job 结果查询
```

### 15.2 Agent 注册表 (`registry/`)

```
registry/
├── agent-lifecycle.ts    Agent 生命周期管理
├── agent-registry.ts     Agent 注册表
└── ...
```

---

## 16. 核心数据流图

```
┌──────────────────────────────────────────────────────────────────┐
│                         用户 (TUI/CLI)                            │
└──────────┬───────────────────────────────────────────────────────┘
           │ prompt()
           ▼
┌─────────────────────┐    ┌─────────────────────┐
│    InteractiveMode   │    │  SessionManager      │
│  (TUI 渲染/输入)      │◄──►│  (会话持久化)        │
└──────┬──────────────┘    └─────────────────────┘
       │ session.prompt()
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       AgentSession                                  │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Agent (@oh-my-pi/pi-agent-core)                               │ │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │ │
│  │  │  System   │──►│ Messages │──►│  LLM     │──►│  Tool    │   │ │
│  │  │  Prompts  │   │  (Context)│  │  API Call│   │  Calls   │   │ │
│  │  └──────────┘   └──────────┘   └────┬─────┘   └────┬─────┘   │ │
│  │                                     │              │          │ │
│  │                                     ▼              ▼          │ │
│  │                              pi-ai (LLM)     tools/*.ts       │ │
│  │                              providers/      (工具执行)        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  工具: read, write, bash, edit, glob, grep, browser, eval, ...      │
│  子系统: MCP, Memory, Advisor, Goals, Collab, Checkpoint            │
│  存储: SQLite, JSONL, Redis (可选)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 17. 进程模型

### 17.1 Worker 架构

CLI 入口 (`cli.ts`) 作为中央 dispatcher (selector 常量见 `cli/worker-selectors.ts`)。Worker 进程/线程通过 **隐藏 argv selector** 复用同一入口：

```
主进程 (cli.ts)
├── runWorkerEntrypoint(argv[0])
│   ├── __omp_worker_tiny_inference       → tiny/worker.ts (子进程)
│   ├── __omp_worker_stats_sync           → omp-stats/sync-worker (worker_thread)
│   ├── __omp_worker_tab                  → browser/tab-worker-entry (worker_thread)
│   ├── __omp_worker_computer             → computer/worker-entry (worker_thread)
│   ├── __omp_worker_js_eval              → eval/js/worker-entry (worker_thread)
│   ├── __omp_worker_js_eval_process      → eval/js/process-entry (子进程)
│   ├── __omp_worker_stt                  → stt/asr-worker (子进程)
│   ├── __omp_worker_tts                  → tts/tts-worker (子进程)
│   ├── __omp_worker_mnemopi_embed        → mnemopi/embed-worker (子进程)
│   ├── __omp_worker_daemon_broker        → launch/broker (daemon worker)
│   ├── __omp_worker_stats_activity       → stats/activity-worker (子进程, IPC)
│   ├── __omp_worker_terminal_output      → launch/terminal-output-worker (worker_thread)
│   ├── __omp_worker_lsp_mux              → lsp/mux daemon
│   ├── __omp_worker_ida_host             → ida/host
└── __omp_worker_blob_broker          → blob-broker/server
    └── 主交互循环 (InteractiveMode)
```

### 17.2 子进程隔离

音频/ML Workers (tiny, STT, TTS) 运行在独立子进程中，使用 `process.send()` IPC 通信，父进程在退出时 `SIGKILL` 子进程以规避 NAPI 终结器崩溃。

---

## 18. 会话持久化

```
Session Storage 层次:
├── SQLite  (默认, sql-session-storage.ts)
│   ├── sessions 表
│   ├── entries 表 (消息/工具调用)
│   └── blobs 表 (二进制数据)
├── JSONL   (旧格式, jsonl-session-storage.ts)
├── Redis   (可选, redis-session-storage.ts)
└── 索引  (indexed-session-storage.ts)

持久化内容:
├── 用户消息 & 助手回复
├── 工具调用 & 结果
├── 会话配置 / 模型
├── 检查点状态
├── 二进制文件 (图片/产物)
└── 会话统计信息
```

---

## 19. 关键配置项

| 设置路径 | 描述 | 默认值 |
|---------|------|--------|
| `enabledModels` | 启用的模型列表 | [] |
| `defaultThinkingLevel` | 默认思考级别 | "normal" |
| `prewalk.enabled` | 首次编辑后切换快速模型 | false |
| `tools.approvalMode` | 工具审批模式 | "interactive" |
| `autoResume` | 自动恢复上次会话 | false |
| `advisor.enabled` | 顾问子系统开关 | false |
| `memory.backend` | 记忆后端 | null |
| `memories.enabled` | 记忆系统启用 | false |
| `bash.autoBackground.enabled` | 自动后台 bash | false |
| `startup.checkUpdate` | 启动时检查更新 | true |
| `startup.setupWizard` | 启动引导向导 | true |
| `symbolPreset` | 符号预设 | "unicode" |
| `colorBlindMode` | 色盲模式 | false |

---

## 20. 编译与发布

### 20.1 构建流水线

```bash
bun build      # 编译二进制 (scripts/build-binary.ts)
  ├─ bundle-dist.ts           # npm 包打包
  ├─ compile-binary.ts        # 编译为独立二进制
  ├─ embed-mupdf-wasm.ts      # 嵌入 PDF 渲染 wasm
  └─ embed-native.ts          # 嵌入原生 N-API 绑定
bazel build    # Bazel 流水线 (BUILD.bazel / MODULE.bazel, CI 双轨)

bun check      # 类型检查 + Lint
bun test       # 测试 (含 --smoke-test worker 冒烟验证)
bun run release   # 发布流程 (版本更新 + CHANGELOG + publish)
```

### 20.2 Docker 支持

构建选项: `Dockerfile` (标准), `Dockerfile.robomp` (机器人/无人值守)

---

## 21. 关键设计决策

1. **单一入口 Worker 复用**: 所有 worker 复用 `cli.ts` 入口，通过 hidden argv selector 分发，避免多个编译入口点
2. **无 `any` 策略**: 代码库严格遵守 TypeScript 类型安全性
3. **提示即文件**: 所有 prompt 在 `.md` 文件中，Handlebars 用于动态部分，禁止内联字符串
4. **Bun 原生优先**: 偏好 Bun API (`Bun.file`, `` $`cmd` ``, `Bun.spawn`) 而非 Node.js 对应项
5. **TUI 安全渲染**: 所有工具输出经过清理 (tab→空格, 截断, 路径缩短)
6. **Catalog 不可手动编辑**: `models.json` 自动从上游源生成
7. **Git 委托工具**: git/jj 操作通过中心化工具 (`src/utils/git.ts`, `src/utils/jj.ts`) 进行，不手写 spawn
8. **Session 恢复**: 支持从各种状态恢复 — 同一目录、不同目录、fork、move
