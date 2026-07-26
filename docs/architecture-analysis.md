# Oh My Pi (OMP) — 软件架构分析

> 项目版本: 17.1.3  
> 分析日期: 2026-07-26  
> 主要入口: `packages/coding-agent/src/cli.ts` → `omp` CLI

---

## 1. 项目概览

**Oh My Pi (OMP)** 是一款 AI 编程代理 CLI 工具，提供 `read`、`bash`、`edit`、`write` 等工具并与 LLM 进行交互式会话。项目基于 **Bun** 运行时，采用 **Monorepo** 架构，包含 17 个 TypeScript 包和 10 个 Rust crate。

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
| `@oh-my-pi/pi-agent-core` | Agent 运行时 — 会话循环、工具调用、状态管理、压缩 | pi-ai |
| `@oh-my-pi/pi-ai` | 多供应商 LLM 客户端 — Anthropic、OpenAI、Google、Ollama 等 | pi-catalog |
| `@oh-my-pi/pi-catalog` | 模型目录 — 内置 models.json、供应商描述符、模型身份标识 | — |
| `@oh-my-pi/pi-tui` | 终端 UI 库 — 差分渲染、Markdown、编辑器、选择列表 | — |
| `@oh-my-pi/pi-utils` | 共享工具集 — 日志、路径、流、临时文件、进程管理 | — |
| `@oh-my-pi/pi-natives` | Rust N-API 原生绑定 — grep、剪贴板、语法高亮、PTY | crates/pi-natives |
| `@oh-my-pi/pi-wire` | 共享线协议类型 — 用于协作和跨进程通信 | — |
| `@oh-my-pi/omp-stats` | 本地可观测性仪表盘 (`omp stats` 命令) | — |
| `@oh-my-pi/hashline` | 散列行格式化 — 用于代码预览的行号/散列 | — |
| `@oh-my-pi/snapcompact` | 会话压缩引擎 | — |
| `@oh-my-pi/pi-mnemopi` | 记忆嵌入系统 | — |
| `@oh-my-pi/collab-web` | 协作 Web 客户端 (React) | pi-wire |
| `@oh-my-pi/pi-swarm-extension` | 多智能体扩展 | — |

### 2.3 Rust Crates

```text
crates/
├── pi-natives/      ← 主要 N-API 绑定 (grep, fd, 剪贴板, 高亮, sixel)
├── pi-ast/          ← AST 相关
├── pi-iso/          ← 文件系统操作
├── pi-shell/        ← Shell 解析 (uutils)
├── pi-uu-diff/      ← diff 实现
├── pi-uu-grep/      ← grep 实现
├── pi-uutils-ctx/   ← uutils 上下文
├── pi-walker/       ← 文件系统遍历
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
  ├─ runWorkerEntrypoint(workerArg)
  │   ├─ 工作线程调度 (__omp_worker_tiny_inference, stats_sync, tab, 
  │   │  computer, js_eval, stt, tts, mnemopi_embed, daemon_broker)
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
// 保留词防止泄露给 LLM: extensions, list, remove, marketplace
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
├─ sessionManager        (session/session-manager.ts)    持久化管理
├─ agent                 (@oh-my-pi/pi-agent-core)       核心 agent
├─ authStorage           (session/auth-storage.ts)       API密钥存储
├─ modelControls         (session/model-controls.ts)     模型切换控制
├─ sessionContext        (session/session-context.ts)    会话上下文
├─ sessionTools          (session/session-tools.ts)      工具注册
├─ sessionMemory         (session/session-memory.ts)     短期记忆
├─ sessionAdvisors       (session/session-advisors.ts)   顾问子系统
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
├─ mnemopiState          (mnemopi/)                      记忆嵌入
├─ artifactManager       (session/artifacts.ts)          产物管理
└─ eventBus              (utils/event-bus.ts)            事件总线
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
  parameters: Type<...>;     // arktype schema
  loadMode: ToolLoadMode;    // "essential" | "discoverable" | "disabled"
  render?: ToolRenderOptions;
  execute(ctx, args, update?): Promise<AgentToolResult>;
}
```

### 5.2 核心工具分类

| 类别 | 工具 | 加载模式 |
|------|------|---------|
| **文件操作** | `read`, `write`, `edit`, `glob` | essential |
| **Shell** | `bash`, `computer` | essential |
| **代码分析** | `grep`, `ast-grep`, `ast-edit`, `sqlite-reader` | essential |
| **工具调用** | `task` (子任务), `hub` (hub), `eval` (沙箱) | essential |
| **知识管理** | `learn`, `manage_skill`, `memory-recall`, `memory-edit`, `memory-reflect`, `memory-retain` | essential |
| **Web** | `browser`, `fetch`, `web-search` | essential |
| **协作** | `xdev` | essential |
| **GitHub** | `gh` (PR/issue/搜索) | discoverable |
| **调试** | `debug` (DAP) | discoverable |
| **审核** | `review` | discoverable |
| **其他** | `todo`, `checkpoint`, `rewind`, `context`, `image-gen`, `tts`, `vibe` | discoverable |

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
```

每个工具使用 Handlebars 模板化的 `prompts/tools/*.md` 文件作为描述，渲染器使用 TUI 组件进行输出显示。

---

## 6. LLM 供应商架构

### 6.1 供应商系统 (`packages/ai/src/providers/`)

```
providers/
├── anthropic/           Anthropic API (Claude) ← 默认
├── anthropic-client/    Anthropic 客户端 SDK 封装
├── openai-responses/    OpenAI Responses API
├── openai-codex-responses/  OpenAI Codex API
├── openai-completions/  OpenAI Completions API
├── azure-openai-responses/  Azure OpenAI
├── google/              Google AI (Gemini)
├── google-vertex/       Google Vertex AI
├── google-gemini-cli/   Google Gemini CLI 模式
├── ollama/              本地 Ollama
├── cursor/              Cursor IDE 集成
├── devin/               Devin AI
├── kimi/                Moonshot Kimi
├── gitlab-duo/          GitLab Duo
├── gitlab-duo-workflow/ GitLab Duo Workflow
├── mock/                测试 Mock
└── synthetic/           合成供应商 (测试)
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
| `settings.ts` | 全局设置系统 (懒加载、缓存、继承链) |
| `settings-schema.ts` | 设置 schema (ArkType 类型定义) |
| `config-file.ts` | 配置文件加载 (YAML > 目录层次) |
| `model-registry.ts` | 模型注册表 + 认证 |
| `model-resolver.ts` | 模型字符串解析/角色解析 |
| `model-roles.ts` | 模型角色 (default/smol/slow/plan) |
| `models-config.ts` | Models 配置文件 |
| `keybindings.ts` | TUI 键位绑定 |
| `prompt-templates.ts` | 提示模板 |
| `provider-globals.ts` | 供应商全局配置 |
| `service-tier.ts` | 服务层级 (按模型家族) |
| `append-only-context-mode.ts` | 追加专用上下文模式 |
| `inline-tool-descriptors-mode.ts` | 内联工具描述模式 |

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
├── render.ts           MCP 工具渲染
├── timeout.ts          MCP 超时
├── types.ts            MCP 类型定义
├── client.ts           MCP 客户端
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
├── acp-events.ts               ACP 事件映射
├── acp-permission-gate.ts      ACP 权限门
├── acp-session.ts              ACP 会话管理
└── ...
```

---

## 14. 协作系统

### 14.1 实时协作 (`collab/`)

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
  └─ Hub 工具调用 → HubTool (tools/hub/)
      ├─ jobs.ts                    Hub 任务管理
      ├─ launch.ts                  Hub 子代理启动
      └─ messaging.ts               Hub 消息传递
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

CLI 入口 (`cli.ts`) 作为中央 dispatcher。Worker 进程/线程通过 **隐藏 argv selector** 复用同一入口：

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
│   └── __omp_worker_daemon_broker        → launch/broker (worker_thread)
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

bun check      # 类型检查 + Lint
bun test       # 测试
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
