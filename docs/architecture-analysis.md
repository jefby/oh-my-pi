# Oh My Pi (OMP) — 软件架构分析

> 项目版本: 17.1.8  
> 分析日期: 2025-03-27  
> 主要入口: `packages/coding-agent/src/cli.ts` → `omp` CLI

---

## 1. 项目概览

**Oh My Pi (OMP)** 是一款 AI 编程代理 CLI 工具，提供 `read`、`bash`、`edit`、`write` 等工具并与 LLM 进行交互式会话。项目基于 **Bun** 运行时，采用 **Monorepo** 架构，包含 16 个 TypeScript 包和 9 个 Rust crate。

核心亮点（来自当前 README）：
- **40+** 供应商 · **32** 内置工具 · **14** LSP 操作 · **28** DAP 操作 · **~55k** 行 Rust 核心
- 单一入口 Worker 复用、Bun 原生优先、提示即文件、TUI 安全渲染

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
| `@oh-my-pi/pi-coding-agent` | 主 CLI 应用 — 工具、会话、模式、编辑系统、提示词管理、SDK | 全部其余包 |
| `@oh-my-pi/pi-agent-core` | Agent 运行时 — 会话循环、工具调用、状态管理、压缩 | pi-ai |
| `@oh-my-pi/pi-ai` | 多供应商 LLM 客户端 — Anthropic、OpenAI、Google、Ollama 等 | pi-catalog |
| `@oh-my-pi/pi-catalog` | 模型目录 — 内置 models.json、供应商描述符、模型身份标识 | — |
| `@oh-my-pi/pi-tui` | 终端 UI 库 — 差分渲染、Markdown、编辑器、选择列表 | — |
| `@oh-my-pi/pi-utils` | 共享工具集 — 日志、路径、流、临时文件、进程管理、CLI 框架 | — |
| `@oh-my-pi/pi-natives` | Rust N-API 原生绑定 — grep、剪贴板、语法高亮、PTY、shell、AST、sixel、媒体 | crates/pi-natives |
| `@oh-my-pi/pi-wire` | 共享线协议类型 — 用于协作和跨进程通信 | — |
| `@oh-my-pi/omp-stats` | 本地可观测性仪表盘 (`omp stats` 命令) | — |
| `@oh-my-pi/hashline` | 散列行格式化 — 用于代码预览的行号/散列 | — |
| `@oh-my-pi/snapcompact` | 会话压缩引擎 | — |
| `@oh-my-pi/pi-mnemopi` | 记忆嵌入系统（本地 ONNX/Transformers.js 嵌入引擎） | pi-utils |
| `@oh-my-pi/collab-web` | 协作 Web 客户端 (React) | pi-wire |
| `@oh-my-pi/swarm-extension` | 多智能体扩展（Swarm 编排） | — |
| `@oh-my-pi/pi-metaharness` | 元评测/基准框架 | — |
| `@oh-my-pi/typescript-edit-benchmark` | TypeScript 编辑基准 | — |

### 2.3 Rust Crates

```text
crates/
├── pi-natives/      ← 主要 N-API 绑定 (grep, fd, 剪贴板, 高亮, sixel, shell, AST)
├── pi-ast/          ← AST / tree-sitter 代码摘要与结构化查询
├── pi-iso/          ← 文件系统隔离/操作 (APFS clone, reflink, overlayfs, projfs)
├── pi-shell/        ← 嵌入式 shell / PTY / 进程管理 (wraps brush-*)
├── pi-uu-diff/      ← diff 实现
├── pi-uu-grep/      ← grep 实现
├── pi-uutils-ctx/   ← uutils 上下文
├── pi-walker/       ← 文件系统遍历
└── vendor/          ← 第三方供应商代码 (brush-core, brush-builtins, ...)
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
  ├─ runWorkerEntrypoint(workerArg)
  │   ├─ 工作线程调度 (__omp_worker_tiny_inference, stats_sync, tab, 
  │   │  computer, js_eval, js_eval_process, stt, tts, mnemopi_embed, daemon_broker)
  │   └─ 通过 argv selector 复用同一入口模块
  │
  ├─ --smoke-test → runSmokeTest()  (所有 worker 的启动验证)
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
// 所有顶层命令
commands: [
  launch, acp, auth-broker, auth-gateway, agents, bench, commit,
  completions, __complete, config, dry-balance, gc, grep, gallery,
  grievances, install, join, models, plugin, say, setup, shell,
  read, ssh, stats, update, usage, tiny-models, token, ttsr, 
  worktree, search
]
// 保留词防止泄露给 LLM: extensions, list, remove, uninstall, marketplace,
// discover, upgrade, enable, disable
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
initializeWithSettings   供应商发现持久化 / 插件根目录注入
  ↓
preloadPluginRoots       预加载插件
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
  ├─ 扩展路径
  └─ MCP 配置
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
├─ sessionManager        (session/session-manager.ts)    持久化管理
├─ agent                 (@oh-my-pi/pi-agent-core)       核心 agent
├─ authStorage           (session/auth-storage.ts)       API密钥存储
├─ modelControls         (session/model-controls.ts)     模型切换控制
├─ sessionContext        (session/session-context.ts)    会话上下文
├─ sessionTools          (session/session-tools.ts)      工具注册
├─ sessionMemory         (session/session-memory.ts)     短期记忆
├─ sessionAdvisors       (session/session-advisors.ts)   顾问子系统接入
├─ toolChoiceQueue       (session/tool-choice-queue.ts)  工具选择队列
├─ streamingOutput       (session/streaming-output.ts)   流式输出
├─ turnPersistence       (session/turn-persistence.ts)   轮次持久化
├─ blobStore             (session/blob-store.ts)         二进制存储
├─ clientBridge          (session/client-bridge.ts)      IDE桥接
├─ asyncJobManager       (async/)                        后台任务
├─ checkpointState       (tools/checkpoint.ts)           检查点
├─ hindsightState        (hindsight/)                    后视镜状态
├─ planModeState         (plan-mode/)                    计划模式
├─ goalRuntime           (goals/)                        目标运行时
├─ mnemopiState          (mnemopi/)                      记忆嵌入状态
├─ xdevRegistry          (tools/xdev.ts)                 xd:// 虚拟设备注册表
├─ artifactManager       (session/artifacts.ts)          产物管理
└─ eventBus              (utils/event-bus.ts)            事件总线
```

### 4.2 核心 Agent 循环

```
Agent.prompt(userInput)
  │
  ├─ 添加用户消息到上下文
  ├─ 构建系统提示 (系统提示 + 工具模式 + 技能 + 规则 + capability 项目)
  ├─ 选择模型 & 供应商 (含 fallback chain、角色解析)
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
  │   ├─ 标题刷新 (tiny title worker)
  │   ├─ 记忆保留 (retain → mnemopi/hindsight)
  │   ├─ 顾问观察 (advisor)
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
  parameters: Type<...>;     // arktype schema
  loadMode: ToolLoadMode;    // "essential" | "discoverable" | "disabled"
  render?: ToolRenderOptions;
  execute(ctx, args, update?): Promise<AgentToolResult>;
}
```

### 5.2 核心工具分类

当前内置工具按 settings 开关动态激活，均定义在 `BUILTIN_TOOLS` (公开) + `HIDDEN_TOOLS` (隐藏) 注册表中。

| 类别 | 工具 | 激活条件 |
|------|------|---------|
| **文件操作** | `read`, `write`, `edit`, `glob` | 始终 essential (read/write 额外支持 xd:// 传输) |
| **Shell / 执行** | `bash`, `computer`, `eval` | bash.enabled / computer.enabled / 按后端可用性 |
| **任务/协作** | `task`, `hub` (IRC/ad-hoc agents) | taskDepth 不超过 maxRecursionDepth / IRC 启用 |
| **代码分析** | `grep`, `ast_grep`, `ast_edit`, `lsp`, `debug` | 按 settings 开关 (默认大部分启用) |
| **知识/技能** | `learn`, `manage_skill` | 需 autolearn.enabled && taskDepth===0 |
| **记忆** | `memory_edit`, `retain`, `recall`, `reflect` | 按 memory.backend (hindsight/mnemopi 开启后者) |
| **Web** | `browser`, `web_search`, `fetch` | 按 settings 开关 (默认仅 web_search 启用) |
| **GitHub** | `github` | 按 github.enabled |
| **图像/语音** | `inspect_image`, `generate_image`, `tts` | 按 settings 开关 / 图像模式 |
| **其他** | `checkpoint`, `rewind`, `resolve`, `review`, `vibe`, `todo`, `ask`, `yield` (hidden), `goal` (hidden) | 按 settings 开关 / Goal 模式激活 |

### 5.3 工具集注册

```
工具发现流程:
1. 核心工具集 (essential-tools.ts) — 始终注册
2. Extension registerTool (extensibility/custom-tools) — 扩展注册
3. MCP 工具 (mcp/tool-bridge.ts) — MCP 服务器提供
4. ACP 工具 (modes/acp) — Agent 客户端协议
5. RPC 主机工具 — 远程过程调用模式
6. 自定义工具 (extensibility/custom-tools) — 用户自定义
7. 技能绑定工具 (extensibility/skills) — 技能触发的工具
8. xdev 挂载 (tools/xdev.ts) — discoverable 工具转为 xd:// 虚拟设备
```

每个工具使用 Handlebars 模板化的 `prompts/tools/*.md` 文件作为描述，渲染器使用 TUI 组件进行输出显示。

### 5.4 `xd://` 虚拟设备

当 `tools.xdev` 启用时，discoverable 工具从顶层 schema 卸载，挂载为 `xd://<tool>` 虚拟设备：

- `read xd://` — 列出已挂载设备
- `read xd://<tool>` — 获取工具文档与参数 schema
- `write xd://<tool>` — 执行工具（`content` 为 JSON 参数对象）

pinned 始终顶层的工具（by `ESSENTIAL_BUILTIN_TOOL_NAMES`）：`read`、`write`、`bash`、`edit`、`glob`、`computer`、`eval`、`task`、`hub`、`learn`、`manage_skill`（11 个）。

---

## 6. LLM 供应商架构

### 6.1 供应商系统 (`packages/ai/src/providers/`)

当前供应商实现以独立 TypeScript 模块形式存在：

```
providers/
├── anthropic.ts                   Anthropic Messages API (Claude) ← 默认
├── anthropic-client.ts            Anthropic 客户端 SDK 封装
├── anthropic-messages-server.ts   Anthropic Messages 服务端
├── anthropic-messages-server-schema.ts 服务端 schema
├── anthropic-wire.ts              线协议格式化
├── openai-responses.ts            OpenAI Responses API
├── openai-responses-server.ts     OpenAI Responses 服务端
├── openai-responses-server-schema.ts 服务端 schema
├── openai-responses-wire.ts       线协议格式化
├── openai-chat-server.ts          Chat 服务端
├── openai-chat-server-schema.ts   服务端 schema
├── openai-chat-wire.ts            线协议格式化
├── openai-codex-responses.ts      OpenAI Codex API
├── openai-anthropic-shim.ts       Anthropic ↔ OpenAI 适配
├── openai-reasoning-fallback.ts   推理回退
├── azure-openai-responses.ts      Azure OpenAI
├── google.ts                      Google Generative AI (Gemini)
├── google-vertex.ts               Google Vertex AI
├── google-gemini-cli.ts           Google Gemini CLI 模式
├── google-auth.ts                 Google 认证
├── google-shared.ts               Google 共享类型
├── google-types.ts                Google 类型
├── amazon-bedrock.ts              Amazon Bedrock
├── ollama.ts                      本地 Ollama
├── cursor.ts / cursor/            Cursor IDE 集成
├── devin.ts / devin/              Devin AI
├── kimi.ts                        Moonshot Kimi
├── gitlab-duo.ts                  GitLab Duo
├── gitlab-duo-workflow.ts         GitLab Duo Workflow
├── claude-code-fingerprint.ts     Claude Code 指纹
├── github-copilot-headers.ts      GitHub Copilot 头
├── pi-native-client.ts            Native Pi 客户端
├── pi-native-server.ts            Native Pi 服务端
├── mock.ts                        测试 Mock
├── synthetic.ts                   合成供应商 (测试)
├── register-builtins.ts           内置注册
├── transform-messages.ts          消息转换
├── vision-guard.ts                视觉防护
└── grammar.ts                     LLM 语法
```

供应商通过 `packages/catalog/src/provider-models/` 中的描述符和解析器注册到模型目录。

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
│   │   ├── openai-compat.ts OpenAI 兼容解析器
│   │   ├── google.ts        Google 特定解析
│   │   ├── ollama.ts        Ollama 发现
│   │   └── ...
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

## 7. Capability 系统

`capability/` 是 OMP 的中央能力注册表，用于解耦“需要什么”与“到哪里找”。

```
capability/
├── index.ts           注册表 API: defineCapability / registerProvider / loadCapability
├── types.ts           Capability / Provider / SourceMeta 类型
├── fs.ts              基于文件系统的能力加载与缓存
└── rule.ts            Capability rule 抽象
```

能力注册流程：
1. `defineCapability<T>({ id, name, ... })` 定义能力
2. `registerProvider<T>(capabilityId, provider)` 注册提供者
3. `loadCapability(id, context)` 按优先级跨提供者加载、去重、过滤
4. 设置层可持久化启用/禁用特定 provider

典型能力：`project-context`、`rules`、`skills`、`system-prompt`。

---

## 8. 内部 URL 路由 (`internal-urls/`)

OMP 将多种外部/内部资源统一为路径形态，让 `read`、`write`、`search` 等文件型工具无需学习新接口即可访问：

```
internal-urls/
├── router.ts                InternalUrlRouter
├── parse.ts                 URL 解析
├── types.ts                 类型定义
├── json-query.ts            JSON 查询工具
├── registry-helpers.ts      注册表辅助
├── docs-index.ts            docs:// 文档索引
├── filesystem-resource.ts   文件系统资源
├── agent-protocol.ts        agent:// 子代理产物字段
├── artifact-protocol.ts     artifact:// 产物
├── history-protocol.ts      history:// 会话历史
├── issue-pr-protocol.ts     issue:// / pr:// PR/issue 统一处理
├── local-protocol.ts        local:// 本地资源
├── memory-protocol.ts       memory:// 记忆
├── mcp-protocol.ts          mcp:// MCP 资源
├── omp-protocol.ts          omp:// OMP 内部
├── rule-protocol.ts         rule:// 规则
├── skill-protocol.ts        skill:// 技能
├── ssh-protocol.ts          ssh:// 远程文件
├── vault-protocol.ts        vault:// 密钥库
├── xd-protocol.ts           xd:// 虚拟设备
```

示例：`read pr://can1357/oh-my-pi/1063`、`search skill://typescript`、`read agent://<id>/findings.0.path`。

---

## 9. 编辑系统

### 9.1 编辑模块 (`packages/coding-agent/src/edit/`)

```
edit/
├── index.ts                编辑工具主入口
├── diff.ts                 Diff 引擎
├── modes/                  编辑模式
│   ├── default/            标准文本编辑
│   ├── hashline/           散列行编辑
│   └── ...
├── apply-patch/            补丁应用
├── hashline/               散列行格式化
├── file-snapshot-store.ts  文件快照存储
├── normalize.ts            路径规范化
├── notebook.ts             Jupyter notebook 编辑
├── read-file.ts            文件读取
├── renderer.ts             编辑渲染器
├── snapshot-details.ts     快照细节
└── streaming.ts            流式编辑
```

### 9.2 编辑模式

- **默认模式**: 标准文本替换
- **散列行模式**: 使用带散列的行号进行精确编辑（Hashline）
- **AST 编辑**: 基于 AST 的结构化编辑，先预览再 `resolve` 应用

---

## 10. LSP & DAP 代码智能

### 10.1 LSP (`lsp/`)

```
lsp/
├── index.ts           LspTool 及主入口
├── client.ts          LSP 客户端生命周期
├── clients/           各语言服务器适配
├── types.ts           LSP 类型
└── utils.ts           辅助函数
```

`lsp` 工具暴露 14 种操作：diagnostics、references、definition、rename、symbols、codeAction、workspaceSymbol、raw 等。写入文件前通过 `workspace/willRenameFiles` 联动重命名。

### 10.2 DAP (`dap/`)

```
dap/
├── index.ts           DAP 主入口
├── client.ts          DAP 客户端/传输
├── config.ts          适配器配置
├── session.ts         DAP 会话管理
└── types.ts           类型
```

`debug` 工具驱动 DAP 会话：launch/attach、断点、单步、线程、调用栈、变量、evaluate。支持 lldb-dap、dlv、debugpy 等适配器。

---

## 11. 配置系统

### 11.1 配置层级

```
settings.ts
├── 全局配置 (~/.omp/config.yml)
├── 项目配置 (.omp/config.yml)
├── CLI 参数覆盖 (--model, --system-prompt 等)
├── 运行时覆盖 (Settings.override())
└── 环境变量覆盖 (PI_SMOL_MODEL, PI_SLOW_MODEL 等)
```

### 11.2 关键配置模块

| 模块 | 功能 |
|------|------|
| `settings.ts` | 全局设置系统 (懒加载、缓存、继承链) |
| `settings-schema.ts` | 设置 schema (ArkType 类型定义) — 设定面板 tab/group 元数据 |
| `config-file.ts` | 配置文件加载 (YAML > 目录层次) |
| `model-registry.ts` | 模型注册表 + 认证 |
| `model-resolver.ts` | 模型字符串解析/角色解析 |
| `model-roles.ts` | 模型角色 (default/smol/slow/plan/commit) |
| `models-config.ts` | Models 配置文件 |
| `models-config-schema.ts` | Models 配置 schema |
| `models-config-schema-bundle.ts` | Models 配置 schema 打包 |
| `model-discovery.ts` | 模型供应商发现 |
| `keybindings.ts` | TUI 键位绑定 |
| `prompt-templates.ts` | 提示模板 |
| `provider-globals.ts` | 供应商全局配置 |
| `service-tier.ts` | 服务层级 (按模型家族) |
| `append-only-context-mode.ts` | 追加专用上下文模式 |
| `inline-tool-descriptors-mode.ts` | 内联工具描述模式 |
| `resolve-config-value.ts` | 配置值解析 (环境变量覆盖) |
| `file-lock.ts` | 配置文件锁 (防并发写入) |
| `api-key-resolver.ts` | API 密钥解析 |

---

## 12. 提示系统

### 12.1 提示层次结构 (`prompts/`)

```
prompts/
├── system/
│   ├── system-prompt.md              ← 核心系统提示
│   ├── title-system.md               ← 标题生成提示
│   ├── tool-call-loop-redirect.md    ← 工具调用循环重定向
│   ├── thinking-loop-redirect.md     ← 思考循环重定向
│   ├── auto-continue.md              ← 自动继续
│   ├── plan-mode-active.md           ← 计划模式激活
│   ├── orchestrate.md                ← orchestrate 关键词
│   ├── ultrathink.md                 ← ultrathink 关键词
│   ├── workflow.md                   ← workflowz 关键词
│   └── ... (60+ 系统提示片段)
├── tools/
│   ├── bash.md                       ← Bash 工具描述
│   ├── read.md                       ← Read 工具描述
│   └── ... (工具描述 MD 文件)
├── agents/
│   ├── subagent-system-prompt.md     ← 子代理系统提示
│   └── subagent-user-prompt.md       ← 子代理用户提示
├── advisor/                          ← 顾问子系统提示
├── autoresearch/                     ← 自动研究提示
├── goals/                            ← 目标系统提示
├── memories/                         ← 记忆系统提示
├── skills/                           ← 技能提示
└── steering/                         ← 引导提示
```

### 12.2 提示加载约定

- 提示文件为静态 `.md` 文件
- 动态内容通过 **Handlebars** 模板化
- `import content from "./prompt.md" with { type: "text" }` 导入
- 禁止内联字符串构建
- 总计 152 个 `.md` 文件：70 系统提示 + 45 工具描述 + 7 子代理 + 其余各子系统

### 12.3 系统提示构建 (`system-prompt.ts`)

`buildSystemPrompt()` 聚合：
1. 项目上下文文件（Capability 系统）
2. 自定义 `SYSTEM.md` / `APPEND_SYSTEM.md`
3. 技能（skills）
4. 规则（rules）
5. 工具元数据（含 xd:// 设备文档）
6. 工作站信息、GPU/CPU 探测
7. 计划模式 / 目标模式 / 追加上下文模式 等状态片段

---

## 13. TUI 渲染系统

### 13.1 组件架构 (`packages/tui/src/`)

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
├── keybindings.ts                 键位绑定系统
├── fuzzy.ts                       模糊匹配
├── canvas.ts                      画布
├── deccara.ts                     DECCARA 优化
└── ...
```

### 13.2 渲染流程 (编码代理)

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

## 14. 扩展系统

### 14.1 扩展类型

```
extensibility/
├── extensions/                  扩展运行时
│   ├── runner.ts                扩展执行器
│   ├── types.ts                 扩展类型定义
│   └── load-errors.ts           加载错误处理
├── custom-tools/                自定义工具
├── custom-commands/             自定义命令
├── hooks/                       钩子系统
├── skills/                      技能系统
└── plugins/
    ├── marketplace/             插件市场
    └── marketplace-auto-update.ts 自动更新
```

扩展可通过以下发现:
1. `~/.omp/extensions/` — 用户全局扩展
2. `.omp/extensions/` — 项目扩展
3. `--extension <path>` — CLI 参数扩展
4. 插件市场安装
5. 首次运行时自动继承 `.claude`、`.cursor`、`.windsurf`、`.gemini`、`.codex`、`.cline`、`.github/copilot`、`.vscode` 中的规则/技能/MCP 配置

---

## 15. MCP (Model Context Protocol)

### 15.1 MCP 子系统 (`mcp/`)

```
mcp/
├── manager.ts            MCP 服务器管理器 (生命周期)
├── config.ts             配置文件解析
├── config-writer.ts      配置写入
├── loader.ts             服务器加载
├── tool-bridge.ts        MCP 工具桥接到 Agent 工具系统
├── tool-cache.ts         工具缓存
├── oauth-discovery.ts    OAuth + MCP 发现
├── oauth-flow.ts         OAuth 认证流
├── oauth-credentials.ts  OAuth 凭证
├── smithery-auth.ts      Smithery 认证
├── smithery-connect.ts   Smithery 注册表连接
├── smithery-registry.ts  Smithery 注册表
├── json-rpc.ts           JSON-RPC 传输
├── render.ts             MCP 工具渲染
├── timeout.ts            MCP 超时
├── types.ts              MCP 类型定义
├── client.ts             MCP 客户端
├── startup-events.ts     MCP 启动事件
├── index.ts              公共 API
└── transports/           MCP 传输层 (stdio/SSE/HTTP)
```

---

## 16. ACP (Agent Client Protocol)

### 16.1 ACP 模式 (`modes/acp/`)

ACP 模式允许 OMP 作为 Agent 客户端协议的宿主，接收来自 IDE 或其他 ACP 兼容客户端的连接。

```
acp/
├── index.ts                    ACP 模式入口
├── acp-agent.ts                ACP Agent 适配
├── acp-events.ts               ACP 事件映射
├── acp-permission-gate.ts      ACP 权限门
├── acp-session.ts              ACP 会话管理
└── ...
```

| omp tool                      | ACP route                           |
| ----------------------------- | ----------------------------------- |
| `bash`                        | `terminal/create + terminal/output` |
| `read`                        | `fs/read_text_file`                 |
| `write`                       | `fs/write_text_file`                |
| `edit, bash`                  | `session/request_permission`        |

---

## 17. 协作系统

### 17.1 实时协作 (`collab/`)

```
collab/
├── host.ts                 协作主机 (广播会话)
├── guest.ts                协作访客 (远程查看/交互)
├── relay-client.ts         中继客户端
├── protocol.ts             协作协议
├── crypto.ts               加密
├── display-name.ts         显示名称
└── replication-shrink.ts   复制压缩
```

### 17.2 Web 协作客户端

`packages/collab-web/` — 基于 React 的 Web 客户端，通过 WebSocket 中继连接以进行远程会话查看。

---

## 18. 子代理 & 任务系统

### 18.1 层次化任务

```
AgentSession
  ├─ Agent (主要) 
  │   └─ 任务工具调用 → TaskTool (task/)
  │       ├─ persisted-revive.ts    持久化子代理恢复
  │       ├─ output-manager.ts      子代理输出管理
  │       ├─ types.ts               子代理/任务类型
  │       ├─ isolation.ts           工作区隔离
  │       └─ ...
  └─ Hub 工具调用 → HubTool (tools/hub/)
      ├─ jobs.ts                    Hub 任务管理
      ├─ launch.ts                  Hub 子代理启动
      └─ messaging.ts               Hub 消息传递
```

### 18.2 Agent 注册表 (`registry/`)

```
registry/
├── agent-lifecycle.ts    Agent 生命周期管理
├── agent-registry.ts     Agent 注册表
└── ...
```

### 18.3 IRC 总线 (`irc/`)

进程级 agent-to-agent 邮箱总线，支持向已注册 agent 发送消息、唤醒 idle/parked 接收者、阻塞 `wait` 等待回复。

---

## 19. 记忆与后视镜 (Memory & Hindsight)

### 19.1 记忆后端 (`memory-backend/`)

```
memory-backend/
├── local-backend.ts     本地文件型记忆后端
├── off-backend.ts       禁用态后端
├── resolve.ts           根据 settings.memory.backend 路由
├── runtime.ts           运行时统一接口
└── types.ts             类型
```

`memory.backend` 可配置为 `local`、`mnemopi`、`hindsight` 或关闭。

### 19.2 Mnemopi (`mnemopi/`)

```
mnemopi/
├── config.ts            Mnemopi 后端配置
├── state.ts             会话状态
├── embed-client.ts      嵌入 worker 客户端
├── embed-worker.ts      嵌入 worker
└── runtime.ts           运行时集成
```

Mnemopi 是长期记忆后端，负责 recall/retain/consolidation，通过 tiny model worker 在独立子进程中运行嵌入模型，避免主进程崩溃。

### 19.3 Hindsight (`hindsight/`)

会话级后视镜系统：将多轮对话压缩为项目范围的持久记忆，`retain` 写入事实，`recall`/`reflect` 检索/综合。

---

## 20. 顾问、自动学习与自动研究

### 20.1 Advisor (`advisor/`)

只读“观察者”顾问子系统。发现 `WATCHDOG.yml`，运行二级 advisor agent 观察主代理 transcript，通过 `advise` 工具以内联 note 形式提出建议，支持去重、隔离和投递通道控制。

### 20.2 AutoLearn (`autolearn/`)

自动学习控制器：监听会话事件，可选地自动运行一轮 capture，并将自动生成的 `SKILL.md` 限制写入 `~/.omp/agent/managed-skills`。

### 20.3 AutoThinking (`auto-thinking/`)

为 `auto` 思考级别做逐 prompt 难度分类，根据 prompt 内容选择具体 `Effort`（在线 smol 模型或本地 tiny memory 模型）。

### 20.4 Autoresearch (`autoresearch/`)

实验性自动研究扩展，注册 4 个实验工具：

- `init_experiment` — 打开/重新配置实验会话
- `run_experiment` — 执行 harness、捕获输出、解析 METRIC/ASI
- `log_experiment` — keep/discard/crash/checks_failed 记录
- `update_notes` — 更新实验笔记

---

## 21. Launch Daemon / 子进程工作器

### 21.1 Launch Daemon (`launch/`)

每项目守护代理 (daemon broker)，管理长期运行进程的生命周期：

```
launch/
├── broker.ts            daemon broker 启动
├── client.ts            客户端
├── protocol.ts          协议
├── spawn-options.ts     生成选项
├── presence.ts          项目存在注册
└── terminal-output.ts   终端输出渲染
```

支持 Unix socket / Windows named pipe、PTY、pipe、detached 模式、就绪探测和日志轮转。

### 21.2 Tiny Model Worker (`tiny/`)

本地 tiny 模型工作器，在独立子进程中运行 ONNX/Transformers.js：

- 会话标题生成
- 记忆提取
- auto-thinking 分类

通过 `subprocess/worker-client.ts` 统一封装 spawn/IPC/smoke。

### 21.3 子进程基础设施 (`subprocess/`)

ONNX/Transformers.js 推理子进程的共享脚手架，处理 worker spawn 命令、IPC fan-out、stderr 捕获、smoke 探测、子进程端运行时安装。

---

## 22. Vibe 与 Live 模式

### 22.1 Vibe Mode (`vibe/`)

Vibe 模式工作器会话运行时，维护持久的 `fast`/`good` 两个 CLI 工作子代理，由 vibe director 跨轮次驱动，生命周期通过 `AgentRegistry` / `AgentLifecycleManager` 持久化。

### 22.2 Live Mode (`live/`)

实时对话/语音模式，协调 Codex WebRTC 音频传输与正常 `AgentSession` 轮次循环，处理麦克风输入、助手输出、barge-in 和中断委托。

---

## 23. 安全与隐私

### 23.1 Secrets (`secrets/`)

密钥占位/脱敏系统：

- 从 YAML 加载 secrets、收集环境 secrets
- 管理每安装占位密钥
- `SecretObfuscator` 在向供应商发送前脱敏上下文
- 支持去混淆以恢复工具参数

### 23.2 Markit (`markit/`)

内部文档转 Markdown 引擎，将 PDF、DOCX、PPTX、XLSX、EPUB 转为 Markdown，供 `read` 工具消费。

---

## 24. 遥测导出 (`telemetry-export.ts`)

OpenTelemetry OTLP/proto 导出：

- trace：GenAI span
- log：agent 运行日志
- metric：运行指标

仅支持 `http/protobuf` 传输，通过标准 `OTEL_*` 环境变量配置，30 秒周期性 flush。

---

## 25. 核心数据流图

```
┌──────────────────────────────────────────────────────────────────┐
│                         用户 (TUI/CLI/ACP/RPC)                    │
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
│  子系统: MCP, Memory, Advisor, Goals, Collab, Checkpoint, LSP, DAP  │
│  存储: SQLite, JSONL, Redis (可选)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 26. 进程模型

### 26.1 Worker 架构

CLI 入口 (`cli.ts`) 作为中央 dispatcher。Worker 进程/线程通过 **隐藏 argv selector** 复用同一入口：

```
主进程 (cli.ts)
├── runWorkerEntrypoint(argv[0])
│   ├── __omp_worker_tiny_inference       → tiny/worker.ts (子进程)
│   ├── __omp_worker_stats_sync           → omp-stats/sync-worker (worker_thread)
│   ├── __omp_worker_tab                  → tools/browser/tab-worker-entry (worker_thread)
│   ├── __omp_worker_computer             → tools/computer/worker-entry (worker_thread)
│   ├── __omp_worker_js_eval              → eval/js/worker-entry (worker_thread)
│   ├── __omp_worker_js_eval_process      → eval/js/process-entry (子进程)
│   ├── __omp_worker_stt                  → stt/asr-worker (子进程)
│   ├── __omp_worker_tts                  → tts/tts-worker (子进程)
│   ├── __omp_worker_mnemopi_embed        → mnemopi/embed-worker (子进程)
│   └── __omp_worker_daemon_broker        → launch/broker (worker_thread)
└── 主交互循环 (InteractiveMode)
```

### 26.2 子进程隔离

音频/ML Workers（tiny、STT、TTS、mnemopi_embed）运行在独立子进程中，使用 `process.send()` IPC 通信。父进程在退出时 `SIGKILL` 子进程以规避 NAPI 终结器崩溃。

---

## 27. 会话持久化

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

## 28. 关键配置项

| 设置路径 | 描述 | 默认值 |
|---------|------|--------|
| `enabledModels` | 启用的模型列表 | [] |
| `disabledProviders` | 禁用的供应商 | [] |
| `modelRoles` | 模型角色映射 | {} |
| `modelProviderOrder` | 供应商优先级排序 | [] |
| `defaultThinkingLevel` | 默认思考级别 | "normal" |
| `prewalk.enabled` | 首次编辑后切换快速模型 | false |
| `tools.approvalMode` | 工具审批模式 | "off" |
| `tools.xdev` | 启用 xd:// 虚拟设备挂载 | false |
| `autoResume` | 自动恢复上次会话 | false |
| `advisor.enabled` | 顾问子系统开关 | false |
| `memory.backend` | 记忆后端 | null |
| `memories.enabled` | 记忆系统启用 | false |
| `bash.autoBackground.enabled` | 自动后台 bash | false |
| `startup.checkUpdate` | 启动时检查更新 | true |
| `startup.setupWizard` | 启动引导向导 | true |
| `theme.preset` | 主题预设 | "titanium" |
| `theme.mode` | 主题模式 (light/dark) | "light" |
| `symbolPreset` | 符号预设 | "unicode" |
| `colorBlindMode` | 色盲模式 | false |
| `telemetry.otlp.enabled` | OTLP 遥测导出 | false |
| `task.maxRecursionDepth` | 子代理最大递归深度 | 2 |
| `task.isolation.mode` | 子代理隔离模式 | 依设置 |
| `autoresearch.enabled` | 自动研究 | false |
| `lsp.enabled` | LSP 集成 | true |
| `debug.enabled` | DAP 调试器 | false |
| `goal.enabled` | Goal 模式 | false |
| `bash.enabled` | Bash Shell | true |
| `glob.enabled` | Glob 搜索 | true |
| `grep.enabled` | Grep 搜索 | true |
| `web_search.enabled` | Web 搜索 | true |
| `browser.enabled` | 浏览器 | false |
| `computer.enabled` | 计算机使用 (CUA) | false |
| `astGrep.enabled` | AST 搜索 | true |
| `astEdit.enabled` | AST 编辑 | true |
| `checkpoint.enabled` | 检查点/回退 | true |
| `autolearn.enabled` | 自动学习 | true |
| `tiny.device` | Tiny 模型设备 | (自动探测) |
| `tiny.dtype` | Tiny 模型精度 | (自动探测) |

---

## 29. 编译与发布

### 29.1 构建流水线

```bash
bun build      # 编译二进制 (scripts/build-binary.ts)
  ├─ bundle-dist.ts           # npm 包打包
  ├─ compile-binary.ts        # 编译为独立二进制
  ├─ embed-mupdf-wasm.ts      # 嵌入 PDF 渲染 wasm
  └─ embed-native.ts          # 嵌入原生 N-API 绑定

bun check      # 类型检查 + Lint (biome check + tsgo)
bun test       # 测试
bun run release   # 发布流程 (版本更新 + CHANGELOG + publish)
```

### 29.2 Docker 支持

构建选项: `Dockerfile` (标准), `Dockerfile.robomp` (机器人/无人值守)

---

## 30. 关键设计决策

1. **单一入口 Worker 复用**: 所有 worker 复用 `cli.ts` 入口，通过 hidden argv selector 分发，避免多个编译入口点
2. **无 `any` 策略**: 代码库严格遵守 TypeScript 类型安全性
3. **提示即文件**: 所有 prompt 在 `.md` 文件中，Handlebars 用于动态部分，禁止内联字符串
4. **Bun 原生优先**: 偏好 Bun API (`Bun.file`, `` $`cmd` ``, `Bun.spawn`) 而非 Node.js 对应项
5. **TUI 安全渲染**: 所有工具输出经过清理 (tab→空格, 截断, 路径缩短)
6. **Catalog 不可手动编辑**: `models.json` 自动从上游源生成
7. **Git 委托工具**: git/jj 操作通过中心化工具 (`src/utils/git.ts`, `src/utils/jj.ts`) 进行，不手写 spawn
8. **Session 恢复**: 支持从各种状态恢复 — 同一目录、不同目录、fork、move
9. **Capability 解耦**: 需求与来源分离，统一能力注册表管理项目上下文/规则/技能
10. **内部 URL 统一协议**: agent://、pr://、issue://、skill://、memory://、xd://、mcp://、docs://、omp://、local:// 等让文件型工具无感访问异构资源
11. **服务端/线协议分离**: Anthropic Messages、OpenAI Responses/Chat 均有独立 server/wire 层，支持 ACP/RPC/server 部署形态
12. **单一入口 Worker 复用**: 所有 worker 复用 cli.ts 入口，隐藏 argv selector 分发；Tiny/STT/TTS/Mnemopi 推理运行在 IPC 子进程中，父进程 SIGKILL 子进程以规避 NAPI 终结器崩溃 (issue #1606)
