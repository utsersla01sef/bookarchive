# Claude Code 源码架构深度分析

> **LEAKED SOURCE · 2026-03-31**
>
> 基于 2026 年 3 月 31 日通过 npm registry 的 `.map` 文件泄露的完整 TypeScript 源码,对 Anthropic 官方 CLI 工具 Claude Code 的架构进行系统性解析。

| 属性 | 值 |
|------|-----|
| 语言 | TypeScript |
| 运行时 | Bun |
| 规模 | ~1,900 文件 / 512K+ 行 |
| UI | React + Ink |

---

## 目录

1. [项目总览与技术栈](#1-项目总览与技术栈)
2. [整体架构分层](#2-整体架构分层)
3. [QueryEngine 核心引擎](#3-queryengine-核心引擎)
4. [工具调用循环](#4-工具调用循环)
5. [Tool 工具系统](#5-tool-工具系统)
6. [Command 命令系统](#6-command-命令系统)
7. [权限系统](#7-权限系统)
8. [Bridge 远程桥接](#8-bridge-远程桥接)
9. [服务层与外部集成](#9-服务层与外部集成)
10. [状态管理](#10-状态管理)
11. [启动与初始化流程](#11-启动与初始化流程)
12. [特性标志与死代码消除](#12-特性标志与死代码消除)
13. [关键设计模式总结](#13-关键设计模式总结)

---

## 1. 项目总览与技术栈

Claude Code 是 Anthropic 的官方 CLI 工具,允许用户在终端直接与 Claude 交互完成软件工程任务。

### 代码规模

| 指标 | 数值 |
|------|------|
| 文件总数 | 1,903 |
| TypeScript (.ts) | 1,332 |
| React TSX (.tsx) | 552 |
| 代码行数 | 512K+ |

### 技术栈

| 类别 | 技术 | 用途 |
|------|------|------|
| 运行时 | Bun | JavaScript/TypeScript 运行时 + 打包器,提供 `bun:bundle` 编译时特性 |
| 语言 | TypeScript (strict) | 类型安全的开发体验 |
| 终端 UI | React + Ink | 用 React 组件模型构建终端界面 |
| CLI 解析 | Commander.js (extra-typings) | 命令行参数解析 |
| Schema 校验 | Zod v4 | 工具输入参数验证 |
| 代码搜索 | ripgrep | GrepTool 底层搜索引擎 |
| 协议 | MCP SDK / LSP | Model Context Protocol 和语言服务器协议集成 |
| API | Anthropic SDK | 调用 Claude 模型 API |
| 遥测 | OpenTelemetry + gRPC | 性能监控和事件追踪 |
| 特性标志 | GrowthBook | 运行时 A/B 测试和功能开关 |
| 认证 | OAuth 2.0 / JWT / Keychain | 多层级身份认证 |

> ⚠️ **注意 · 泄露背景**
>
> npm 包中发布的 source map 文件(`.map`)引用了完整的、未混淆的 TypeScript 源码,该文件可从 Anthropic 的 R2 存储桶直接下载。本文档基于该泄露源码的 `src/` 目录进行分析。

---

## 2. 整体架构分层

Claude Code 采用分层架构,从 CLI 入口到 LLM API 调用,每一层职责清晰。

```
┌─────────────────────────────────────────────────────────────┐
│  CLI 入口层                                                  │
│  cli.tsx → init.ts → replLauncher.tsx                       │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  UI 渲染层                                                   │
│  App.tsx → REPL.tsx → Components/ (140+ Ink 组件)           │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  查询引擎层                                                  │
│  QueryEngine.ts → query.ts → claude.ts (Anthropic API)      │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  工具与命令层                                                │
│  tools.ts (40+ 工具) | commands.ts (70+ 命令)               │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  基础设施层                                                  │
│  services/ | permissions/ | state/ | bridge/                │
└─────────────────────────────────────────────────────────────┘
```

### 目录结构

```
src/
├── main.tsx                 # CLI 入口 (Commander.js)
├── QueryEngine.ts           # LLM 查询引擎 (46K 行)
├── query.ts                 # 工具调用循环 (68K 行)
├── Tool.ts                  # 工具类型定义 (29K 行)
├── commands.ts              # 命令注册表 (25K 行)
├── tools.ts                 # 工具注册表
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 成本追踪
│
├── entrypoints/             # 入口逻辑 (cli, init, mcp)
├── tools/                   # 40+ 工具实现
├── commands/                # 70+ 斜杠命令
├── components/              # 140+ Ink UI 组件
├── hooks/                   # React hooks
├── services/                # 外部服务集成
├── bridge/                  # IDE 远程桥接
├── state/                   # 状态管理
├── permissions/             # 权限系统
├── skills/                  # 技能系统
├── plugins/                 # 插件系统
├── ink/                     # Ink 渲染器封装
├── keybindings/             # 快捷键配置
├── migrations/              # 配置迁移
└── utils/                   # 工具函数 (200+ 文件)
```

---

## 3. QueryEngine 核心引擎

`QueryEngine.ts` 是每个会话的拥有者,管理跨轮次的可变状态,核心入口是 `submitMessage()` 异步生成器。

### 核心职责

| # | 职责 | 说明 |
|---|------|------|
| 01 | 会话生命周期 | 管理消息列表、文件缓存、usage 累计等跨轮次可变状态 |
| 02 | 系统提示构建 | 组装默认提示、用户上下文、系统上下文、memory mechanics |
| 03 | SDK 消息转换 | 将内部消息流转换为 SDK 消费方所需的标准格式 |
| 04 | 预算与权限控制 | USD 预算检查、权限拒绝追踪、结构化输出重试限制 |

### 完整处理链路

```
用户输入
   │
   ▼
初始化阶段 (解析模型/Thinking配置)
   │
   ▼
系统提示构建 (fetchSystemPromptParts)
   │
   ▼
用户输入处理 (processUserInput: 解析斜杠命令/附件)
   │
   ▼
会话持久化 (写入 transcript)
   │
   ▼
┌─► 查询循环 query()
│     │
│     ▼
│   消息类型判断
│     ├─ stream_event → 更新 usage (reset/accumulate)
│     ├─ assistant → push 到 messages, yield 给 SDK
│     ├─ api_error → 转换为 api_retry
│     └─ attachment → yield error 并 return
│     │
└─────┘
   │
   ▼
结果返回 (success / error)
   │
   ▼
成本计算 (total_cost_usd)
```

### 关键实现细节

#### 1. 权限拒绝追踪

在 `QueryEngine.ts` 第 244 行,`wrappedCanUseTool` 包装原始 `canUseTool`,记录所有非 allow 的决策到 `permissionDenials[]`,最终在 result 消息中返回给 SDK 调用方。

#### 2. 模型解析优先级

```typescript
// QueryEngine.ts 第 274-276 行
const model = userSpecifiedModel
  ? parseUserSpecifiedModel(userSpecifiedModel)
  : getMainLoopModel()
```

#### 3. Thinking 配置决策

```typescript
// QueryEngine.ts 第 278-282 行
const thinkingConfig = passedThinkingConfig
  ?? (shouldEnableThinkingByDefault()
      ? { type: 'adaptive' }
      : { type: 'disabled' })
```

#### 4. USD 预算检查

在查询循环每次迭代中都检查预算(第 972 行),超预算立即终止:

```typescript
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield { type: 'result', subtype: 'error_max_budget_usd', ... }
  return
}
```

> 💡 **设计亮点**
>
> `ask()` 便利函数(第 1186 行)是 `QueryEngine` 的一次性包装器:创建 engine → `submitMessage()` → finally 中回写 readFileCache,简化了单次查询的调用方式。

---

## 4. 工具调用循环

`query.ts` 中的 `queryLoop()` 是一个 `while(true)` 循环,每次迭代代表一个"turn",实现了完整的 tool-call loop。

### 循环结构

```
queryLoop 开始
   │
   ▼
┌─────────────────────────────────────────┐
│  上下文压缩管道 (5级)                    │
│  1. applyToolResultBudget (限制大小)     │
│  2. snipCompactIfNeeded (snip 压缩)      │
│  3. microcompactMessages (微压缩)        │
│  4. applyCollapsesIfNeeded (上下文折叠)  │
│  5. autoCompactIfNeeded (自动摘要压缩)   │
└──────────────────┬──────────────────────┘
                   ▼
         调用 LLM API (deps.callModel)
                   │
                   ▼
           流式消费消息
                   │
                   ▼
            有 tool_use 块?
           ┌───────┴───────┐
           │否             │是
           ▼               ▼
     执行 stop hooks    执行工具
           │           ┌───┴───┐
           ▼           │流式   │批量
     return:         │Streaming│runTools
     completed       │ToolExec │
                     └───┬───┘
                         ▼
                   收集工具结果
                         │
                         ▼
                   注入附件 (文件变更/memory/skills)
                         │
                         ▼
                   组装下一轮 State
                         │
                         └─────────► (回到上下文压缩管道)
```

### 流式工具执行

当启用 `streamingToolExecutor` 时,工具在 API 仍在流式输出时就开始执行,大幅降低延迟:

```typescript
// query.ts 第 838-844 行
if (streamingToolExecutor && !aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
// 在流循环中持续检查已完成的结果
for (const result of streamingToolExecutor.getCompletedResults()) { ... }
```

### State 转换模式

每次 `continue` 时创建新的 `State` 对象,`transition` 字段记录"为什么继续",用于恢复路径去重:

| transition 值 | 含义 |
|---------------|------|
| `next_turn` | 正常的下一轮工具调用 |
| `stop_hook_blocking` | 停止钩子阻止继续 |
| `max_output_tokens_recovery` | 输出 token 超限恢复 |
| `max_output_tokens_escalate` | 输出 token 超限升级 |
| `reactive_compact_retry` | 反应式压缩重试 |
| `collapse_drain_retry` | 折叠排空重试 |
| `token_budget_continuation` | Token 预算续传 |

### 循环终止条件

- `needsFollowUp === false` — 无工具调用,执行 stop hooks 后返回 `completed`
- 用户中断 — 返回 `aborted_streaming`
- `maxTurns` 超限 — yield `max_turns_reached` 后返回
- Hook 阻止 — 返回 `hook_stopped`
- Prompt too long 无法恢复 — 返回 `prompt_too_long`

---

## 5. Tool 工具系统

每个工具都是自包含模块,定义输入 schema、权限模型和执行逻辑。`tools.ts` 是注册表,使用条件 import 实现死代码消除。

### 工具注册机制

`tools.ts` 通过两种方式注册工具:静态 import 和条件 require:

```typescript
// 静态导入 - 所有构建都包含
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'

// 条件 require - 通过 feature gate 控制死代码消除
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [CronCreateTool, CronDeleteTool, CronListTool]
  : []
```

### 完整工具清单

| 工具 | 描述 | 特性门控 |
|------|------|----------|
| BashTool | Shell 命令执行 | — |
| PowerShellTool | PowerShell 命令执行 | — |
| FileReadTool | 文件读取(图片/PDF/notebook) | — |
| FileWriteTool | 文件创建/覆盖 | — |
| FileEditTool | 文件部分修改(字符串替换) | — |
| GlobTool | 文件模式匹配搜索 | — |
| GrepTool | 基于 ripgrep 的内容搜索 | — |
| WebFetchTool | 获取 URL 内容 | — |
| WebSearchTool | Web 搜索 | — |
| AgentTool | 子代理生成 | — |
| SkillTool | 技能执行 | — |
| MCPTool | MCP 服务器工具调用 | — |
| LSPTool | 语言服务器协议集成 | — |
| NotebookEditTool | Jupyter notebook 编辑 | — |
| TodoWriteTool | 任务列表管理 | — |
| TaskCreateTool | 任务创建 | — |
| TaskUpdateTool | 任务更新 | — |
| TaskListTool | 任务列表 | — |
| TaskGetTool | 任务获取 | — |
| TaskStopTool | 任务停止 | — |
| TaskOutputTool | 任务输出 | — |
| SendMessageTool | 代理间消息传递 | — |
| TeamCreateTool | 团队代理创建 | — |
| TeamDeleteTool | 团队代理删除 | — |
| EnterPlanModeTool | 进入计划模式 | — |
| ExitPlanModeTool | 退出计划模式 | — |
| EnterWorktreeTool | 进入 Git worktree 隔离 | — |
| ExitWorktreeTool | 退出 Git worktree | — |
| ToolSearchTool | 延迟工具发现 | — |
| ConfigTool | 配置管理 | — |
| BriefTool | 简报生成 | — |
| AskUserQuestionTool | 向用户提问 | — |
| ListMcpResourcesTool | 列出 MCP 资源 | — |
| ReadMcpResourceTool | 读取 MCP 资源 | — |
| TungstenTool | 内部工具 | — |
| SyntheticOutputTool | 结构化输出生成 | — |
| SleepTool | 主动模式等待 | PROACTIVE / KAIROS |
| CronCreateTool | 定时任务创建 | AGENT_TRIGGERS |
| CronDeleteTool | 定时任务删除 | AGENT_TRIGGERS |
| CronListTool | 定时任务列表 | AGENT_TRIGGERS |
| RemoteTriggerTool | 远程触发 | AGENT_TRIGGERS_REMOTE |
| REPLTool | REPL 工具 | USER_TYPE=ant |

### AgentTool:子代理与 Swarm 架构

`AgentTool` 是多代理系统的核心,支持生成子代理执行独立任务:

- **内置代理**:`generalPurposeAgent`(通用)、`exploreAgent`(探索)、`planAgent`(规划)、`verificationAgent`(验证)、`claudeCodeGuideAgent`(向导)
- **异步代理生命周期**:通过 `LocalAgentTask` 管理前台/后台执行,`RemoteAgentTask` 支持远程执行
- **Swarm 模式**:`TeamCreateTool` 创建团队代理,`spawnTeammate` 生成队友,`isAgentSwarmsEnabled` 控制启用
- **Worktree 隔离**:`createAgentWorktree` 为代理创建独立 git worktree,避免文件冲突
- **自动后台化**:超过 120 秒(`getAutoBackgroundMs`)自动转入后台

### Skill 系统与 Plugin 系统

| 系统 | 说明 |
|------|------|
| **BundledSkillDefinition** | 内置技能编译进 CLI,通过 `registerBundledSkill` 注册。支持 `files` 字段提取参考文件到磁盘,支持 `context: 'inline' \| 'fork'` 两种执行模式 |
| **Plugin 系统** | 通过 `plugins/builtinPlugins.ts` 加载内置插件,支持第三方插件市场(`BrowseMarketplace`)、插件信任(`PluginTrustWarning`) |
| **MCP Skills** | 通过 `feature('MCP_SKILLS')` 门控,从 MCP 服务器加载技能(`fetchMcpSkillsForClient`) |

---

## 6. Command 命令系统

用户面向的斜杠命令(`/` 前缀),`commands/` 目录下有 70+ 个命令实现。

### 命令分类

| 分类 | 命令 |
|------|------|
| Git 工作流 | `/commit`, `/review`, `/pr_comments`, `/branch`, `/install-github-app` |
| 会话管理 | `/resume`, `/rename`, `/rewind`, `/session`, `/share`, `/export`, `/clear` |
| 配置管理 | `/config`, `/permissions`, `/hooks`, `/model`, `/theme`, `/vim`, `/keybindings` |
| 上下文控制 | `/compact`, `/context`, `/add-dir`, `/memory`, `/files`, `/diff` |
| 成本与使用 | `/cost`, `/usage`, `/stats`, `/status`, `/extra-usage` |
| 认证 | `/login`, `/logout`, `/passes`, `/upgrade` |
| IDE 集成 | `/ide`, `/desktop`, `/mobile`, `/chrome`, `/terminalSetup` |
| 诊断 | `/doctor`, `/heapdump`, `/debug-tool-call`, `/env` |
| 技能与插件 | `/skills`, `/plugin`, `/reload-plugins` |
| 任务管理 | `/tasks`, `/plan`, `/tag` |
| 远程与桥接 | `/bridge`, `/remote-env`, `/remote-setup` |
| 其他 | `/help`, `/exit`, `/feedback`, `/voice`, `/output-style`, `/effort`, `/fast` |

---

## 7. 权限系统

每次工具调用都经过多层权限检查,安全检查对 bypass 模式免疫。

### 权限检查流程

```
工具调用请求
   │
   ▼
useCanUseTool Hook
   │
   ▼
createPermissionContext
   │
   ▼
有 forceDecision?
   ├─是─► 直接使用
   │
   └─否
       │
       ▼
   hasPermissionsToUseTool
       │
       ├─ 1a. getDenyRuleForTool (整工具拒绝规则)
       │      └─ 匹配 deny? → DENY
       │
       ├─ 1b. getAskRuleForTool (整工具询问规则)
       │      └─ 匹配 ask? → ASK
       │
       ├─ 1c. tool.checkPermissions (工具自身权限检查)
       │      └─ 返回 deny? → DENY
       │
       ├─ 1e. requiresUserInteraction?
       │
       ├─ 1g. 安全路径检查 (.git/.claude/.vscode)
       │
       ├─ 2a. bypassPermissions 模式?
       │      └─ 是? → ALLOW
       │
       ├─ 2b. 整工具允许规则
       │      └─ 匹配? → ALLOW
       │
       └─ 3. passthrough → ASK
```

### 模式转换

| 模式 | ask 决策转换为 | 说明 |
|------|----------------|------|
| `dontAsk` | `deny` | 不询问,直接拒绝 |
| `auto` | AI 分类器决策 | 使用 `TRANSCRIPT_CLASSIFIER` 分类 |
| `asyncAgent` | headless hook | 运行 `PermissionRequest` hooks |
| `bypassPermissions` | `allow` | 全部允许(安全检查免疫) |

> ⚠️ **安全检查免疫**
>
> `.git/`、`.claude/`、`.vscode/`、shell 配置等敏感路径的安全检查即使在 `bypassPermissions` 模式下也会执行,防止模型修改关键配置文件。

---

## 8. Bridge 远程桥接

Bridge 系统将本地 CLI 转变为可被 claude.ai 远程控制的"环境",实现 IDE/网页端与本地 CLI 的双向通信。

### 架构组件

```
┌─────────────────┐         ┌─────────────────────────────┐         ┌─────────────────┐
│  远程 (claude.ai)│         │       Bridge 层              │         │    本地 CLI      │
│                 │         │                             │         │                 │
│  WebSocket      │◄──────►│  bridgeMain.ts (轮询主循环)  │         │  CLI 子进程      │
│  SessionsWebSocket│       │  replBridge.ts (REPL 桥接)   │────────►│  child_process   │
│                 │         │  bridgeMessaging.ts (消息协议)│         │  .spawn          │
│                 │         │  jwtUtils.ts (JWT 认证)      │         │                 │
│                 │         │  sessionRunner.ts (会话执行) │         │  OAuth+Keychain │
└─────────────────┘         └─────────────────────────────┘         └─────────────────┘
```

### 关键设计

- **环境注册**:`registerBridgeEnvironment` 获取 `environmentId` + `environment_secret`
- **JWT 心跳认证**:`sessionIngressTokens` 独立于 OAuth token,主动刷新调度器在过期前 5 分钟刷新
- **容量唤醒机制**:`capacityWake` 在会话完成时提前唤醒"已满"休眠,立即接受新工作
- **消息去重**:`recentPostedUUIDs` 检测回声,`recentInboundUUIDs` 防御性去重
- **控制请求超时**:服务器控制请求必须在 10-14 秒内响应,否则断开 WebSocket
- **依赖注入**:`BridgeCoreParams` 注入所有外部依赖,避免 bun 打包时拖入整个 REPL 树

### IDE 检测

- **VSCode**:通过 `supportedIdeConfigs[ide].ideKind === 'vscode'` 判断
- **JetBrains**:异步执行 `initJetBrainsDetection`,通过 `execFileNoThrow` 检查父进程命令行,匹配 IDEA/PyCharm 等列表

---

## 9. 服务层与外部集成

`services/` 目录封装所有外部服务交互,从 API 调用到 MCP 服务器管理。

### Anthropic API 客户端

`services/api/client.ts` 的 `getAnthropicClient` 是核心工厂函数,支持多供应商:

- **默认**:标准 `Anthropic` 客户端
- **Bedrock**(`CLAUDE_CODE_USE_BEDROCK`):使用 `AnthropicBedrock`,支持 AWS Bearer Token、区域覆盖
- **Vertex AI**(`CLAUDE_CODE_USE_VERTEX`):类似 Bedrock 分支

关键参数:超时 600 秒、`dangerouslyAllowBrowser: true`、代理支持。

### 流式响应与重试

`services/api/claude.ts` 使用 Anthropic SDK 的 raw stream(非 `BetaMessageStream`,避免 O(n²) 的 partial JSON 解析),逐块解析 `message_start`、`content_block_delta`、`message_stop` 等事件。

> 📋 **流式看门狗**
>
> `STREAM_IDLE_TIMEOUT_MS`(默认 90 秒):无 chunk 到达则 abort。`STREAM_IDLE_WARNING_MS`(45 秒):先发出警告。每个 chunk 到达时 `resetStreamIdleTimer()`。

### 重试策略 (`withRetry.ts`)

| 场景 | 策略 |
|------|------|
| 默认重试 | 最多 10 次,指数退避(500ms × 2^n,上限 32s,25% jitter) |
| 529 过载 | 连续 3 次后触发 `FallbackTriggeredError` 切换 fallback model |
| Fast mode 降级 | 429/529 且 retry-after < 20s 保持 fast mode,否则进入 10 分钟 cooldown |
| 401 认证错误 | 强制 OAuth token refresh 后重试 |
| Context overflow | 解析错误,调整 `maxTokensOverride` 后重试 |
| 持久重试模式 | `CLAUDE_CODE_UNATTENDED_RETRY`:无限重试,最大退避 5 分钟,上限 6 小时 |

### 其他服务

| 服务 | 说明 |
|------|------|
| **MCP 服务器管理** | 支持 Stdio/SSE/StreamableHTTP/WebSocket 多种传输,通过 React Context 暴露 `reconnectMcpServer` 和 `toggleMcpServer` |
| **OAuth 2.0 认证** | Authorization Code + PKCE 流程,本地 HTTP 服务器监听回调,支持 Claude.ai 和 Console 两种登录 |
| **GrowthBook 特性标志** | `remoteEval: true` 服务器端预评估,`getFeatureValue_CACHED_MAY_BE_STALE` 非阻塞读取 |
| **组织策略限制** | 失败开放(fail open)策略,ETag 缓存 + 后台轮询(1 小时间隔),30 秒超时 |
| **上下文压缩** | 压缩前移除图像块,压缩后恢复最多 5 个文件(每文件 5K token),流式压缩最多重试 2 次 |
| **Keychain 安全存储** | macOS 使用回退存储模式:Keychain 为主,纯文本文件为备,主存储失败自动回退 |

---

## 10. 状态管理

自建极简 Store,通过 `useSyncExternalStore` 与 React 集成。

### Store 实现

`state/store.ts` 实现了类似 Zustand 的极简 store:

```typescript
export function createStore<T>(initialState, onChange?): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### React 集成

`useAppState(selector)` 使用 `useSyncExternalStore` 订阅,`Object.is` 比较避免不必要重渲染。**禁止返回新对象**(会被视为变化)。

### AppState 内容

`AppState` 是 `DeepImmutable` 类型,包含:

- 设置、模型、verbose
- `toolPermissionContext`(权限上下文)
- 多个 bridge 状态字段(`replBridgeEnabled`、`replBridgeConnected` 等)
- 远程会话状态(`remoteConnectionStatus`、`remoteBackgroundTaskCount`)
- 任务状态(`tasks`)、speculation 状态
- `denialTracking`(权限拒绝追踪)

---

## 11. 启动与初始化流程

Claude Code 的启动经过精心优化,通过并行预取和快速路径减少冷启动时间。

### 快速路径

`entrypoints/cli.tsx` 为不同命令提供零/最少模块加载的快速路径:

- `--version`:零导入,直接输出版本号
- `--dump-system-prompt`:`feature('DUMP_SYSTEM_PROMPT')` 门控
- `remote-control`:`feature('BRIDGE_MODE')` 门控,进入 bridge 主循环
- `--daemon-worker`:`feature('DAEMON')` 门控,精简 worker

### 初始化流程 (`init.ts`)

```
enableConfigs() (验证配置)
        │
        ▼
applySafeConfigEnvironmentVariables (仅安全环境变量)
        │
        ▼
applyExtraCACertsFromConfig (CA 证书)
        │
        ▼
setupGracefulShutdown (优雅关闭)
        │
        ▼
异步: 1P 事件日志 + GrowthBook
        │
        ▼
populateOAuthAccountInfoIfNeeded
        │
        ▼
initJetBrainsDetection (异步 IDE 检测)
        │
        ▼
detectCurrentRepository (GitHub 仓库检测)
        │
        ▼
远程管理设置 + 策略限制
        │
        ▼
configureGlobalMTLS + Agents (网络配置)
        │
        ▼
preconnectAnthropicApi (预连接 TCP+TLS)
        │
        ▼
setShellIfWindows (Windows shell)
        │
        ▼
注册清理: LSP/swarm teams
```

### 并行预取优化

`main.tsx` 顶部在所有其他 import 之前启动三个并行副作用:

```typescript
// main.tsx — 在其他 import 之前执行
profileCheckpoint('main_tsx_entry')
startMdmRawRead()           // MDM 子进程 (plutil/reg query)
startKeychainPrefetch()      // macOS keychain 读取
```

这样这些耗时操作(~135ms)可以与后续模块加载并行进行。

---

## 12. 特性标志与死代码消除

Claude Code 使用编译时和运行时双重特性标志系统,实现灵活的功能控制。

### 编译时:`bun:bundle` 的 `feature()`

`feature()` 是 Bun bundler 的编译时函数,在打包时求值为 `true`/`false` 常量。当为 `false` 时,对应的 `if` 块被死代码消除(DCE)整块移除:

```typescript
// 编译时门控
if (feature('BRIDGE_MODE') && args[0] === 'remote-control') { ... }
if (feature('DAEMON') && args[0] === '--daemon-worker') { ... }

// 条件 require 模式
const classifierModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./classifierDecision.js')
  : null
```

### 主要特性标志

| 标志 | 功能 |
|------|------|
| `PROACTIVE` | 主动模式(SleepTool 等) |
| `KAIROS` | 助手模式(session transcript、SendUserFileTool 等) |
| `BRIDGE_MODE` | 远程桥接模式 |
| `DAEMON` | 守护进程 worker |
| `VOICE_MODE` | 语音输入 |
| `AGENT_TRIGGERS` | 定时任务(Cron 工具) |
| `AGENT_TRIGGERS_REMOTE` | 远程触发 |
| `MONITOR_TOOL` | 监控工具 |
| `HISTORY_SNIP` | 历史 snip 压缩 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `TOKEN_BUDGET` | Token 预算续传 |
| `TRANSCRIPT_CLASSIFIER` | 转录分类器(auto 模式) |
| `BASH_CLASSIFIER` | Bash 命令分类器 |
| `MCP_SKILLS` | MCP 技能加载 |
| `COORDINATOR_MODE` | 多代理协调器 |
| `ABLATION_BASELINE` | 实验基线 |

### 运行时:GrowthBook

GrowthBook 提供运行时动态特性控制,与编译时 `feature()` 协同工作:

- `getFeatureValue_CACHED_MAY_BE_STALE(flag, default)` — 非阻塞读取,内存优先
- `remoteEval: true` — 服务器端预评估,客户端不持有规则引擎
- `loggedExposures` Set — 去重曝光事件,防止热路径重复触发

---

## 13. 关键设计模式总结

Claude Code 的架构展现了多个精巧的设计模式,值得深入理解。

### 01. AsyncGenerator 贯穿全链路

从 `QueryEngine.submitMessage()` → `query()` → `queryModelWithStreaming()` → `withRetry()` 全部使用 `async function*`,实现真正的流式背压传导。

### 02. 依赖注入模式

`QueryDeps` 类型注入 `callModel`、`microcompact`、`autocompact` 等依赖,`productionDeps()` 提供生产实现,测试时可注入 fake。

### 03. State 状态机模式

`State` 对象 + `transition` 字段实现显式状态机,每个 continue 点创建新 State,便于调试和测试断言。

### 04. Withhold-Recovery 模式

可恢复错误在流式循环中被扣留,不 yield 给 SDK,直到确认恢复是否成功,避免中间错误泄漏。

### 05. 多级上下文压缩

snip → microcompact → contextCollapse → autocompact → reactiveCompact,五级管道各有不同粒度和触发条件。

### 06. 编译时 + 运行时双重门控

`bun:bundle` 的 `feature()` 实现 DCE,GrowthBook 提供运行时动态控制,两者协同实现灵活的功能管理。

### 07. 并行预取优化

启动时并行执行 MDM 读取、Keychain 预取、API 预连接,与模块加载重叠,显著减少冷启动时间。

### 08. 极简 Store + React 集成

自建轻量 store(类似 Zustand),`useSyncExternalStore` 高效订阅,避免 Redux 的复杂性。

---

> 💡 **核心洞察**
>
> Claude Code 的架构核心是 **流式优先**(AsyncGenerator 贯穿全链路)和 **安全优先**(多层权限检查 + bypass 免疫的安全路径检查)。整个系统通过编译时 DCE 和运行时特性标志实现灵活的功能控制,既支持 Anthropic 内部的实验性功能,又能生成精简的外部发布包。

---

*本文档基于 2026-03-31 泄露的 Claude Code TypeScript 源码分析生成。所有源码版权归原作者 Anthropic 所有。分析仅用于技术研究和学习目的。*
