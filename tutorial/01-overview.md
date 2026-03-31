# 01. 总览：Claude Code 到底是什么

## 这篇文章要回答什么

如果你第一次打开 Claude Code 的源码，很容易被目录规模吓到。

这份还原源码的 `src/` 目录当前有大约 1900 个文件，里面同时包含命令系统、工具系统、React/Ink UI、MCP、插件、skills、bridge、remote、tasks、memory、permissions 等多个子系统。直接按目录平铺阅读，通常会很快失去方向。

所以这篇文章不追求细节全覆盖。它只做一件事：

**先帮你建立一个足够准确的全局心智模型，知道 Claude Code 这个系统到底由哪些层组成，它为什么不像一个“终端里的聊天壳”，而像一个真正的 agent runtime。**

## 一句话结论

Claude Code 本质上不是一个“把用户输入发给模型，再把回答打印出来”的 CLI。

它更像一个运行在终端里的、长期持有状态的 agent 系统。这个系统至少同时包含下面几层：

- 一个 CLI 启动与配置层
- 一个基于 React/Ink 的交互壳层
- 一个面向会话的 agent loop 运行时
- 一套统一的工具执行与权限控制机制
- 一套上下文、记忆、任务、subagent 与外部集成体系

如果你要自己实现一个 Claude Code-like 产品，真正要复制的不是“UI 长什么样”，而是这套运行时组织方式。

## 先看全局地图

可以先把 Claude Code 粗略想成下面这张图：

```text
用户输入 / CLI 参数
        |
        v
main.tsx 启动与初始化
        |
        v
命令系统 / REPL 入口 / AppState 建立
        |
        v
QueryEngine + query loop
        |
        +--------------------+
        |                    |
        v                    v
   上下文与记忆           工具与权限系统
        |                    |
        +----------+---------+
                   |
                   v
        MCP / Plugins / Skills / Bridge / Remote
                   |
                   v
            UI 渲染、任务状态、会话持久化
```

这张图的重点不是“模块名称”，而是一个判断标准：

**Claude Code 的中心不是 UI，也不是命令列表，而是持续运转的 agent loop。**

后面的上下文、工具、权限、记忆、subagent，基本都可以理解为围绕这条主循环组织的配套系统。

## Claude Code 由哪些层组成

### 1. CLI 启动与系统引导层

最外层入口在 `src/main.tsx`。

这个文件并不只是做参数解析。它在很早阶段就开始做很多启动型工作，例如：

- 启动 profile 记录与预取逻辑
- 预取 MDM / keychain / auth / bootstrap 等依赖
- 初始化 telemetry、GrowthBook、managed settings、policy limits
- 解析 CLI 选项、模型、权限模式、session 信息
- 准备 commands、tools、agent definitions、MCP 配置
- 最终把系统送入 REPL、非交互流程或其他入口

也就是说，`main.tsx` 更像一个“产品启动编排器”，而不是一个薄薄的 `bin/cli.ts`。

从源码结构上看，Claude Code 很重视启动阶段的并行化和前置初始化，这也是它明显偏“成熟产品”而不是“脚本工具”的一个信号。

## 2. 命令层：用户显式调用什么

用户直接输入的 slash commands，主要由 `src/commands.ts` 统一注册和装配。

这里有两个值得先记住的点：

- `commands` 是面向用户显式触发的能力，比如 `/review`、`/config`、`/mcp`、`/memory`、`/tasks`
- 它不是静态死表，里面混入了 feature flag、skills、plugins、provider 可用性、用户环境等动态条件

`src/commands.ts` 里的 `COMMANDS()` 和 `getCommands(cwd)` 说明，Claude Code 的命令系统本质上是一个可扩展命令注册表，而不是一组手写 `if/else`。

这层很重要，但它不是系统的心脏。它更像用户进入运行时的“入口菜单”。

## 3. 工具层：模型可以调用什么

和 commands 并列但完全不同的一层，是 tools。

工具相关的主入口在：

- `src/Tool.ts`
- `src/tools.ts`
- `src/tools/*`

这里的核心区别必须尽早建立：

- `command` 是用户主动触发的
- `tool` 是模型在 agent loop 里被允许调用的

例如：

- Bash、FileRead、FileEdit、FileWrite 是执行型工具
- Agent、Task、SendMessage 是多 agent / 任务型工具
- MCP、ReadMcpResource、ListMcpResources 是外部能力接入工具
- Skill、WebSearch、LSP、NotebookEdit 则分别代表更高层的能力封装

`src/Tool.ts` 定义了统一的工具执行上下文 `ToolUseContext` 和权限上下文 `ToolPermissionContext`。这说明 Claude Code 并不是把工具当成一批散落函数，而是把它们放进了一个统一的运行契约里。

而 `src/tools.ts` 则负责几件更系统化的事情：

- 组装 built-in tools
- 根据 feature flag / mode / env 过滤工具
- 根据 deny rules 和权限上下文裁剪工具可见性
- 把内建工具和 MCP 工具组装成同一个 tool pool

这一步很关键，因为它意味着 Claude Code 给模型看到的并不是一个固定工具表，而是一个**按会话上下文、权限与运行模式实时组装出的能力池**。

## 4. Agent Runtime：真正的核心在 QueryEngine 和 query loop

Claude Code 最值得优先研究的不是 UI，而是运行时主循环。

核心文件至少包括：

- `src/QueryEngine.ts`
- `src/query.ts`

`src/QueryEngine.ts` 里对自身的定位非常明确：

- 一个 `QueryEngine` 对应一个 conversation
- 每次 `submitMessage()` 是同一会话里的一个新 turn
- 消息、文件缓存、usage、permission denial 等状态会跨 turn 持续存在

这段设计已经说明很多事情：

- Claude Code 不是“无状态 request/response”
- 它是带持久会话状态的 agent runtime
- 它需要自己管理消息历史、预算、工具调用、文件状态和恢复能力

`src/query.ts` 则更像低层的 agent loop 执行器。它接收：

- `messages`
- `systemPrompt`
- `userContext`
- `systemContext`
- `toolUseContext`
- `canUseTool`
- `taskBudget`

然后在一个 async generator 风格的循环里处理：

- 模型请求
- 流式输出
- tool use
- tool result 回注
- compact / token budget / fallback
- stop hooks / post-sampling hooks

如果只能选一条主线来理解 Claude Code，应该优先选这里。

## 5. 上下文层：模型看到的不是只有“聊天记录”

Claude Code 的上下文组织也不是把历史消息简单拼起来。

`src/context.ts` 至少展示了两类重要上下文：

- `getSystemContext()`
- `getUserContext()`

从实现上看，它会把下面这些信息以缓存化、结构化的方式注入上下文：

- git 状态
- 当前 branch / main branch / recent commits
- `CLAUDE.md` 和 memory 文件
- 当前日期
- 其他系统级注入信息

这件事非常重要，因为很多人会把 agent CLI 理解为“模型 + 工具”。但 Claude Code 实际上是：

**模型 + 工具 + 精心构造的上下文层 + 权限系统 + 状态管理。**

缺少上下文层，你很难得到稳定、可持续的工程行为。

## 6. UI 层：终端界面不是附属品，而是运行时外壳

Claude Code 的交互壳并不是传统 readline，而是 React/Ink。

从 `src/replLauncher.tsx` 可以直接看到，它会动态加载：

- `src/components/App.js`
- `src/screens/REPL.js`

再把二者组合成：

```tsx
<App {...appProps}>
  <REPL {...replProps} />
</App>
```

这意味着终端 UI 在架构里不是“最后打印一下文本”，而是完整的应用外壳。

再看 `src/state/AppStateStore.ts`，你会发现它管理的状态范围非常广，包括：

- settings、model、status line
- tool permission context
- remote bridge 状态
- task / foreground task / viewing agent
- MCP clients / MCP tools / MCP resources
- plugins
- agent definitions
- file history、attribution、todos

这说明 Claude Code 的 UI 并不是一个薄展示层。它承接了大量会话态、任务态、远端态和产品态。

## 7. 平台扩展层：它不是闭门自转的单体程序

从目录结构就能看出，Claude Code 很强调系统边界以外的接入能力：

- `src/services/mcp/`
- `src/plugins/`
- `src/skills/`
- `src/bridge/`
- `src/remote/`
- `src/tasks/`

这几块加在一起，意味着 Claude Code 不是“一个只能本地跑工具的 agent”，而是一个：

- 能接外部能力
- 能扩展行为
- 能在多会话、多 agent、多环境之间流动
- 能把本地 CLI 接到 IDE、remote session、MCP server 的系统

这也是为什么它更接近“agent platform 的终端前端”，而不只是一个 CLI。

## 为什么 Claude Code 看起来像“真正的软件系统”

读到这里，可以总结出几个关键特征。

### 它有清晰的运行时中心

很多 AI CLI 工具的问题在于：功能很多，但没有中心。

Claude Code 不一样。它的中心很明确，就是由 `QueryEngine` 和 `query.ts` 支撑的主循环。其他模块大多可以理解为：

- 给主循环提供上下文
- 给主循环提供能力
- 给主循环施加约束
- 给主循环保存状态
- 给主循环提供 UI 和系统接入面

### 它把“能力暴露”做成了一套制度，而不是临时函数调用

`Tool.ts`、`tools.ts`、permission context、MCP tool pool 这些设计一起说明：

- 能力不是随便暴露给模型的
- 工具可见性和权限是运行时的一部分
- 外部工具和内建工具尽量被拉进同一张抽象表

这比“让模型直接生成 shell 命令”要成熟得多。

### 它把状态当成一级公民

从 `AppStateStore.ts`、session storage、memory、task、plugin state 来看，Claude Code 明显是按“长生命周期会话系统”在设计，而不是按“每次输入独立处理”在设计。

一旦你接受这个前提，很多看似复杂的设计就会自然起来：

- 为什么要有 task 和 subagent
- 为什么要有 session resume
- 为什么要有 memory / history / compact
- 为什么权限系统不能只是弹窗确认

### 它把产品化问题放进了核心架构

从 `main.tsx` 里大量启动初始化逻辑、feature flag、managed settings、policy limits、analytics、bridge、remote 相关代码可以看出，Claude Code 不是“先写功能，再补产品化”。

很多产品化问题从一开始就在主架构里。

## 如果你要自己实现一个 Claude Code-like 系统，应该先抄什么

不是先抄 UI，也不是先抄 prompt。

优先应该抄的是这几个抽象边界：

1. 一个长期持有状态的 conversation runtime
2. 一个统一的 tool contract 和 tool pool 组装机制
3. 一个明确分层的 context system
4. 一个内建权限与安全边界的执行系统
5. 一个能承接 task / subagent / session 的状态容器

如果这几个边界没有建立好，做出来的通常不会像 Claude Code，而更像一个“会调用 shell 的聊天机器人”。

## 建议的源码阅读顺序

如果你准备从源码进入，推荐先读这几组文件：

### 第一组：先建立系统轮廓

- `src/main.tsx`
- `src/commands.ts`
- `src/tools.ts`
- `src/Tool.ts`

### 第二组：进入真正的运行时核心

- `src/QueryEngine.ts`
- `src/query.ts`

### 第三组：理解状态和上下文是怎么支撑主循环的

- `src/context.ts`
- `src/state/AppStateStore.ts`
- `src/memdir/`
- `src/tasks/`

### 第四组：最后再看外围扩展

- `src/services/mcp/`
- `src/plugins/`
- `src/skills/`
- `src/bridge/`
- `src/remote/`

## 本篇结论

这篇文章最想让你记住三件事：

1. Claude Code 的核心不是命令列表，也不是 UI，而是 agent loop。
2. Claude Code 的价值不在“能调工具”，而在“如何把上下文、工具、权限、状态和扩展能力组织成一个长期运行系统”。
3. 后面的所有章节，都应该被理解为对这条主循环的展开，而不是彼此孤立的功能模块。

下一篇我们就进入主线，专门拆 `agent loop`：一次用户请求进入 Claude Code 之后，到底是怎么一步一步跑起来的。
