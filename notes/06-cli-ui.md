# CLI Entry + UI Shell 研究笔记

## 这份笔记解决什么问题

Claude Code 的 CLI / UI 层，不只是一个把 `query()` 包起来的终端壳。

从源码看，它至少拆成五层职责：

1. 进程启动与模式判定
2. 会话环境、副作用和后台服务初始化
3. 交互式 REPL 壳层与 provider 装配
4. 会话级共享状态模型
5. 真正把 UI、AppState、工具上下文和 query loop 接起来的 session controller

后续写教程时，如果只说“入口在 `main.tsx`，界面在 `REPL.tsx`”，会漏掉真正关键的设计点。更准确的写法应该是：

- `main.tsx`：启动编排器，决定这次会话到底走 headless 还是 interactive
- `setup.ts`：启动副作用层，负责把当前进程和会话环境准备好
- `App` / `AppState`：给 REPL 提供共享 runtime state，而不是只提供 UI 状态
- `REPL.tsx`：交互模式下的 session controller，不只是视图组件

## 核心结论

Claude Code 的 CLI / UI 栈，本质上是围绕 query loop 搭出来的一层 runtime shell：

- `setup()` 负责“把环境准备到能跑 agent”的状态
- `main.tsx` 负责“这次会话该怎么跑”
- `launchRepl()` 和 `App` 负责“把 provider 与 REPL 壳装起来”
- `AppState` 负责“把整个会话的共享状态集中存起来”
- `REPL.tsx` 负责“在交互模式下，把 prompt 输入、工具池、上下文加载、query() 调用、UI 更新和 turn 完成逻辑串起来”

最值得教程强调的一点是：

Claude Code 的 REPL 不是 query loop 外面的一层被动 UI，而是 interactive mode 下的运行时协调器。

## 必读源码入口

### 启动编排与模式分流

- `../claude-code-sourcemap/restored-src/src/main.tsx`
- `../claude-code-sourcemap/restored-src/src/setup.ts`

### REPL 壳层与 provider

- `../claude-code-sourcemap/restored-src/src/replLauncher.tsx`
- `../claude-code-sourcemap/restored-src/src/components/App.tsx`
- `../claude-code-sourcemap/restored-src/src/state/AppState.tsx`
- `../claude-code-sourcemap/restored-src/src/state/store.ts`
- `../claude-code-sourcemap/restored-src/src/state/AppStateStore.ts`

### 交互会话控制器

- `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
- `../claude-code-sourcemap/restored-src/src/query.ts`

## 1. 启动不是单文件完成的，而是 `main.tsx` + `setup.ts` 分层

### 1.1 `main.tsx` 的职责是 orchestration，不是环境副作用

`main.tsx` 真正关心的是“这次 Claude Code 会话怎么跑”：

- 解析启动参数和模式
- 预注册 bundled plugins / skills
- 并行启动 `setup(...)`、`getCommands(...)`、`getAgentDefinitionsWithOverrides(...)`
- 判断是 `--print` headless、继续历史会话、remote connect / ssh，还是普通 interactive REPL
- 组装 `initialState`、`sessionConfig` 和 agent definitions
- 最终把这些东西交给 `runHeadless(...)` 或 `launchRepl(...)`

也就是说，`main.tsx` 解决的是“控制流分发”和“启动期数据装配”。

### 1.2 `setup.ts` 的职责是把进程环境准备好

`setup()` 做的大多不是 UI 逻辑，而是启动副作用：

- Node 版本检查
- 切换 session id
- 启动 UDS messaging
- 捕获 teammate mode snapshot
- 恢复 iTerm2 / Terminal backup
- 尽早 `setCwd(cwd)`
- 捕获 hooks config snapshot
- 初始化 file changed watcher
- 处理 worktree / tmux 路径切换
- 初始化 session memory
- 初始化 context collapse
- 预取 slash commands
- 后台加载 plugin hooks、recent activity、release notes、API key helper
- 初始化 sinks 与 startup telemetry

这说明 Claude Code 并没有把启动期副作用塞进 React tree 或 REPL component，而是在进入 UI 之前先完成进程级准备。

### 1.3 这两个文件的边界很值得复刻

如果自己实现一个 Claude Code 风格 CLI，最容易犯的错是把：

- 参数解析
- 副作用初始化
- 首次状态装配
- UI 启动

全塞进一个 `main()`。

Claude Code 的拆法更干净：

- `setup.ts` 负责“环境 ready”
- `main.tsx` 负责“模式选择 + 会话装配”

这样后续要支持 headless、resume、remote 或 worktree，复杂度不会全部挤进一棵 if/else 树里。

## 2. headless 和 interactive 不是同一个壳，只是共享底层 runtime

### 2.1 headless 路径直接跑 `runHeadless(...)`

在 `--print` 之类的 non-interactive 模式里，`main.tsx` 会：

- 启动 `startDeferredPrefetches()`
- 做 background housekeeping
- 准备 headless store / commands / tools / MCP config
- 直接调用 `runHeadless(...)`

这里没有 `launchRepl()`，也没有 Ink UI tree。

这说明 Claude Code 不是“把 REPL 隐藏掉就得到 headless”。更准确地说：

- headless 和 interactive 共享 query / tools / context / permission runtime
- 但它们的外层控制壳是分开的

### 2.2 interactive 路径先构造一个很重的 `initialState`

普通 REPL 模式下，`main.tsx` 会先把大量启动状态同步装进 `initialState`：

- `toolPermissionContext`
- `agent` / `agentDefinitions`
- MCP / plugin 列表占位
- remote / bridge 状态
- notification / elicitation 队列
- file history / attribution
- speculation、prompt suggestion、worker sandbox request
- 初始消息、模型、effort、fast mode、team context

也就是说，REPL 并不是在挂载后慢慢自己“推导出状态”，而是由入口一次性注入一个会话级 runtime snapshot。

### 2.3 deferred prefetch 是首屏性能策略的一部分

`startDeferredPrefetches()` 明确只做“首屏之后再跑也不影响第一次渲染”的工作：

- `initUser()`
- `getUserContext()`
- 在信任已建立或 non-interactive 下预取 `getSystemContext()`
- tips、云厂商 auth、文件数估计
- analytics gates、official MCP URLs、model capabilities
- settings / skill change detector
- event loop stall detector

这点很值得教程强调。Claude Code 的启动优化不是“少做事”，而是：

- 先把必须项放进 critical path
- 把缓存预热和探测类工作尽量挪到 first render 之后

## 3. `launchRepl()` 和 `App` 只是薄壳，不承载主业务

### 3.1 `launchRepl()` 的职责非常克制

`replLauncher.tsx` 的 `launchRepl(...)` 只做两件事：

- lazy import `App` 和 `REPL`
- 渲染 `<App {...appProps}><REPL {...replProps} /></App>`

这说明它不是另一个业务层，只是一个轻量装配器，顺便把重量较大的模块延迟到真正需要进入 interactive 模式时再加载。

### 3.2 `App.tsx` 是 provider shell

`App` 本身也很薄，主要就是包三层 provider：

- `FpsMetricsProvider`
- `StatsProvider`
- `AppStateProvider`

这条链的意思很明确：

- metrics / stats / app state 属于整个交互会话的基础设施
- 它们先于 REPL 存在
- 但真正的业务控制流不放在这里

### 3.3 这层壳的价值在于把 UI 框架和 runtime state 解耦

如果以后想替换终端 UI 技术栈，或加一个不同外壳，只要还能提供同样的 store / provider 接口，底层很多逻辑都不必重写。

## 4. `AppState` 不是普通 UI state，而是会话级 runtime model

### 4.1 `createStore()` 是非常小的自定义 store

`state/store.ts` 没有引入 Redux / Zustand 之类的外部状态库，而是自己提供了最小 store：

- `getState()`
- `setState()`
- `subscribe()`

同时用 `Object.is(next, prev)` 避免无效通知。

这说明 Claude Code 想要的是：

- 一个足够轻的共享状态容器
- 方便 React 和非 React 代码共同读写
- 不把整个 runtime 强耦合到某个状态管理框架

### 4.2 `AppStateProvider` 不是单纯 context 包装

`AppState.tsx` 里的 `AppStateProvider` 做了三件重要的额外工作：

- 创建并持有 store
- 订阅 settings change，把外部配置变化回写到 store
- 在 provider 内再包一层 mailbox / voice provider

这说明它不是一个“把对象塞进 React context”的薄封装，而是会话级 runtime 宿主。

### 4.3 `AppStateStore.ts` 才是真正的状态边界定义

`AppStateStore.ts` 的 `AppState` 包含的不只是 UI 字段，还包括：

- `tasks`
- `toolPermissionContext`
- `agent` / `agentDefinitions`
- `mcp` / `plugins`
- `notifications` / `elicitation`
- `fileHistory`
- `attribution`
- `sessionHooks`
- `inbox`
- `promptSuggestion`
- `speculation`
- `workerSandboxPermissions`
- remote / bridge / team context

这说明 `AppState` 更接近“交互会话 runtime state”，而不是组件树的 view model。

### 4.4 默认权限模式直接进默认状态

`getDefaultAppState()` 会在创建默认状态时就初始化 `toolPermissionContext`，而且 teammate + `plan_mode_required` 会直接把初始 mode 设成 `plan`。

这很关键，因为它说明 permission mode 不是 query 前临时算出来的参数，而是整个会话状态的一部分。

## 5. `REPL.tsx` 其实是 interactive runtime controller

### 5.1 它不是只负责 render prompt input

`REPL.tsx` 内部除了 UI 渲染，还承担了大量控制职责：

- 从 AppState 读取会话状态
- 维护本地 UI 状态
- 计算当前可见工具与命令
- 构造 query 需要的 `ProcessUserInputContext`
- 处理 prompt 提交
- 在 turn 前刷新 IDE / MCP 状态
- 加载 system prompt / user context / system context
- 调用 `query(...)`
- 消费 streaming 输出并更新状态
- 在 turn 完成时做收尾

所以把 `REPL.tsx` 理解成“一个终端页面组件”会严重低估它的角色。

### 5.2 `getToolUseContext()` 是 REPL 的关键桥

`getToolUseContext(...)` 做的事情非常能说明问题：

- 从 store 读取最新状态，而不是依赖渲染时闭包
- 动态组装 built-in + MCP 工具池
- 按当前 permission mode 裁剪工具
- 提供 `getAppState()` / `setAppState()`
- 提供 `messages` / `setMessages`
- 提供 `refreshTools()`、notification hook、spinner/progress callback
- 提供 file history / attribution / memory trigger / content replacement 等 runtime 能力

这说明 REPL 并没有把 query loop 当作一个黑盒 API 调用，而是主动构造一个非常丰富的 runtime bridge。

### 5.3 `onQueryImpl(...)` 才是交互模式下的每轮主入口

在 interactive 模式里，真正把一轮用户输入串起来的是 `onQueryImpl(...)`：

1. 刷新 IDE / MCP 状态
2. 把 slash command 限定的 `allowedTools` 写进 `toolPermissionContext`
3. 构造 `toolUseContext`
4. 并行加载 `getSystemPrompt(...)`、`getUserContext()`、`getSystemContext()`
5. 生成 `buildEffectiveSystemPrompt(...)`
6. 调用 `query(...)`
7. 在 streaming 过程中更新消息和 UI
8. turn 完成后做清理与回调

这条路径说明：

- `query()` 不是直接从 UI 组件里裸调
- 交互式一轮 turn 之前，Claude Code 还要完成大量上下文和 runtime 装配

### 5.4 PromptInput 也不是直接触发 query

REPL render 部分把 `getToolUseContext`、`toolPermissionContext`、`onSubmit` 等能力传给 `PromptInput`。

这意味着：

- 输入组件负责收集用户动作
- 但真正的 runtime context 构建仍然回到 REPL

这是比较健康的边界。输入框不直接持有 agent runtime。

## 6. CLI / UI 层和 agent loop 的真实连接点

### 6.1 `main.tsx` 决定“用哪个壳跑 loop”

- headless：`runHeadless(...)`
- interactive：`launchRepl(...)`
- continue / remote / ssh：也是先恢复或装配，再落到对应壳

### 6.2 `REPL.tsx` 决定“这一轮 loop 带着什么上下文进去”

它会在 query 前确定：

- 当前消息历史
- 当前工具池和 permission mode
- system prompt
- user / system context
- MCP / IDE / hooks / memory 相关 runtime 接口

### 6.3 `query.ts` 决定“loop 内部怎么跑”

也就是说，Claude Code 在架构上明确分了三层：

- 外层：CLI mode orchestration
- 中层：interactive session controller
- 内层：query / tool / permission runtime

这比“一个 REPL 组件直接自己调用模型和工具”更容易扩展。

## 7. 如果自己复刻，最小实现顺序应该是什么

### 7.1 不要先做复杂 TUI

先做这几个最小部件更合理：

1. 一个 `setup()`，只负责 cwd、session、日志、配置和后台初始化
2. 一个 `main()`，只负责模式分发
3. 一个最小 store，允许 React 和非 React 共用
4. 一个 `REPL controller`，专门负责拼 `toolUseContext` 和调用 `query()`
5. 最后再把终端 UI 组件拆细

### 7.2 AppState 应该优先建成 runtime model，不是页面状态

如果一开始就按“聊天窗口有哪些局部状态”设计 state，后面接入：

- tools
- permissions
- tasks
- MCP
- subagents
- remote bridge

会越来越混乱。

Claude Code 这里更像是在维护“整个 agent session 的状态机”。

### 7.3 交互控制器要显式保留非 React 接口

`getAppState()` / `setAppState()`、`refreshTools()`、消息更新器这些接口都很值得复刻，因为 query loop、hooks、tools、background services 经常不在 React 渲染语境里。

## 8. 当前仍需运行时验证的问题

- `setup()` 里的 plugin hooks、recent activity、release notes、API key helper 在本机客户端中哪些是 feature-gated，哪些在当前版本已裁掉
- interactive 首屏之后的 deferred prefetch 对实际首 token 延迟影响有多大
- REPL bridge / remote control / CCR mirror 这些状态字段，在当前本机客户端里哪些真的可达
- `initialState` 里一些 ant-only 或 feature-flag 字段，在 external build 中是否仍保留占位

## 9. 后续正文建议怎么写

后续正式教程可以拆成两章：

- `CLI 入口与启动编排`
- `REPL 壳层、AppState 与交互会话控制`

这样能把“程序怎么启动”与“交互模式下怎么驱动 agent loop”分开写，避免一章里同时混：

- 参数解析
- provider 树
- query 前装配
- AppState 模型
- Ink UI 渲染

这些层次最好不要压成同一章。
