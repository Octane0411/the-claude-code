# MCP + Plugins + Skills 研究笔记

## 这份笔记解决什么问题

Claude Code 的扩展系统，不是单一插件机制。

从源码看，它至少拆成三条不同的扩展面：

1. `MCP`：把外部 server 的 tools / commands / resources 接入当前 runtime
2. `plugin`：把一组本地扩展组件作为可安装、可启用、可刷新、可分发的包管理起来
3. `skills`：把 prompt 型能力单元注入 slash command 面，供模型或用户在需要时调用

如果后续正文只写“Claude Code 支持 MCP 和插件”，会把很多关键边界混掉。更准确的写法应该是：

- MCP 解决“远端能力如何进当前会话”
- plugin 解决“扩展包如何发现、安装、启用、刷新，并贡献多种组件”
- skill 解决“提示模板 / 工作流能力如何进入命令面”

## 核心结论

Claude Code 的扩展体系不是“一套插件 API”，而是三层组合：

- `MCP` 是运行时接入层：把 server 能力转成 Claude Code 内部的 `Tool` / `Command` / `Resource`
- `plugin` 是分发与组件容器层：一个 plugin 可以贡献 commands、skills、agents、hooks、MCP servers、output styles，甚至影响 settings 级联
- `skills` 是能力单元层：无论来自磁盘目录、bundled、plugin，还是某些 feature-gated 的 MCP 路径，最终都以 `Command` 形态进入 slash command 系统

最值得教程强调的一点是：

Claude Code 没有把“扩展”统一建模成某个抽象基类，而是让不同扩展面在不同层进入系统，再在 `commands.ts`、`AppState`、`REPL` 和 `query()` 周围汇合。

## 必读源码入口

### MCP 配置、连接与客户端能力转换

- `../claude-code-sourcemap/restored-src/src/services/mcp/config.ts`
- `../claude-code-sourcemap/restored-src/src/services/mcp/client.ts`
- `../claude-code-sourcemap/restored-src/src/services/mcp/useManageMCPConnections.ts`
- `../claude-code-sourcemap/restored-src/src/services/mcp/MCPConnectionManager.tsx`
- `../claude-code-sourcemap/restored-src/src/cli/handlers/mcp.tsx`

### Plugin 发现、加载、安装与刷新

- `../claude-code-sourcemap/restored-src/src/utils/plugins/pluginLoader.ts`
- `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginCommands.ts`
- `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginAgents.ts`
- `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginHooks.ts`
- `../claude-code-sourcemap/restored-src/src/utils/plugins/mcpPluginIntegration.ts`
- `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginOutputStyles.ts`
- `../claude-code-sourcemap/restored-src/src/utils/plugins/refresh.ts`
- `../claude-code-sourcemap/restored-src/src/services/plugins/PluginInstallationManager.ts`
- `../claude-code-sourcemap/restored-src/src/services/plugins/pluginOperations.ts`
- `../claude-code-sourcemap/restored-src/src/services/plugins/pluginCliCommands.ts`
- `../claude-code-sourcemap/restored-src/src/cli/handlers/plugins.ts`

### Skills 发现、注册与命令装配

- `../claude-code-sourcemap/restored-src/src/skills/loadSkillsDir.ts`
- `../claude-code-sourcemap/restored-src/src/skills/bundledSkills.ts`
- `../claude-code-sourcemap/restored-src/src/skills/mcpSkillBuilders.ts`
- `../claude-code-sourcemap/restored-src/src/commands.ts`
- `../claude-code-sourcemap/restored-src/src/main.tsx`

## 1. 先把三种扩展面分开

### 1.1 MCP 不是“插件格式”

MCP 这条链解决的是：

- 从哪些配置源收集 server 配置
- 如何连上这些 server
- 如何把 server 暴露的 tools / commands / resources 转成 Claude Code 内部 runtime 能消费的对象

也就是说，MCP 更像一个“远端能力适配层”。

### 1.2 plugin 不是“单个命令”

`pluginLoader.ts`、`loadPluginCommands.ts`、`loadPluginAgents.ts`、`loadPluginHooks.ts`、`mcpPluginIntegration.ts` 一起说明：

一个 plugin 可以贡献的东西远不止 slash command，还包括：

- commands
- skills
- agents
- hooks
- MCP servers
- output styles

所以 plugin 更像“扩展包容器”，不是单点能力。

### 1.3 skill 不是“插件的别名”

`skills/loadSkillsDir.ts` 和 `commands.ts` 很清楚地表明：

- skill 会被加载成 `Command`
- 它和 built-in command、plugin command 一起进入 slash command 面
- 它本质上是 prompt 型能力单元，而不是安装机制本身

也就是说，skill 是能力形态，plugin 是分发容器，MCP 是运行时接入协议。

## 2. MCP 是运行时外部能力接入层

### 2.1 `getClaudeCodeMcpConfigs()` 负责把多来源配置汇总

`services/mcp/config.ts` 的 `getClaudeCodeMcpConfigs()` 会把这些来源汇总起来：

- enterprise scope
- user / project / local scope
- plugin 提供的 MCP servers
- 动态传入的 MCP config

这里有两个关键边界：

- enterprise MCP config 可以独占控制权，其他来源都被压掉
- managed policy 还可以把 MCP 锁成 plugin-only

这说明 Claude Code 的 MCP 不是“本地随便配一个 `.mcp.json` 就完了”，而是纳入了完整的 settings / policy 级联。

### 2.2 plugin 也可以贡献 MCP servers

`mcpPluginIntegration.ts` 表明 plugin 的 MCP 入口可以来自：

- plugin 根目录的 `.mcp.json`
- `plugin.manifest.mcpServers`
- manifest 中指向的 JSON 文件
- `.mcpb` 包或 URL

而且会处理：

- user config 变量替换
- 环境变量展开
- 未配置状态
- 错误收集与回传

这说明 plugin 和 MCP 不是平行互斥关系。更准确地说：

- plugin 是“把 MCP server 带进来”的一种上游来源
- MCP 是“运行时真正接上能力”的下游执行层

### 2.3 `useManageMCPConnections()` 才是 interactive 模式的 MCP 主控

`useManageMCPConnections.ts` 负责：

- 读取汇总后的 MCP config
- 建立连接
- 抓取 tools / commands / resources
- 监听列表变化通知
- 处理 reconnect、toggle enabled、channel permission、elicitation
- 把状态批量回写进 `AppState.mcp`

所以在交互模式下，MCP 不是启动时一次性解析完毕的静态配置，而是一个会持续更新的连接管理子系统。

### 2.4 `fetchToolsForClient()` 把 MCP tool 转成 Claude Code `Tool`

`services/mcp/client.ts` 最关键的设计点在这里：

- 从 server 侧 `tools/list`
- 清洗返回数据
- 给工具名加 `mcp__server__tool` 风格前缀
- 保留权限、只读、破坏性、searchHint、alwaysLoad 等能力元数据
- 最终映射成 Claude Code 内部统一的 `Tool`

这点很重要，因为它说明 MCP tool 在进入 Claude Code 之后，不再是“另一套调用系统”，而是直接进入现有 tool / permission / query runtime。

### 2.5 MCP resources 和 commands 也是并行接入的

MCP 这一层不只抓 tools：

- `fetchCommandsForClient()` 把服务端命令接进 slash command 面
- `fetchResourcesForClient()` 把资源目录接进 `AppState.mcp.resources`
- 当 server 支持 resources 时，还会自动补上 `ListMcpResourcesTool` / `ReadMcpResourceTool`

所以从产品层看，MCP 在 Claude Code 里不只是 function calling 扩展，而是完整的外部能力表面。

### 2.6 interactive 和 headless 共用底层 MCP runtime

`main.tsx` 在 headless 路径里也会用 `getClaudeCodeMcpConfigs(...)`；interactive 则通过 `MCPConnectionManager` 和 `useManageMCPConnections()` 持续维护连接状态。

这说明：

- 配置解析层是共享的
- 连接管理层在 interactive 模式下更重、更动态

## 3. plugin 是分发、发现、启用与刷新层

### 3.1 `pluginLoader.ts` 解决的是“这个扩展包到底是什么”

从注释和实现看，plugin loader 负责：

- 从 marketplace、session-only `--plugin-dir`、built-in plugin 等来源发现 plugin
- 读取 manifest
- 解析 commands / skills / agents / hooks / output styles / MCP component 路径
- 处理启用 / 禁用状态
- 做去重、依赖校验、错误收集
- 缓存 plugin settings，供后续 settings cascade 同步读取

这说明 plugin loader 是扩展系统的材料装配层。

### 3.2 启动时优先走 `cache-only`，避免阻塞

`loadAllPluginsCacheOnly()` 和 `loadAllPlugins()` 的区分很重要：

- `cache-only`：启动消费方使用，不主动触发网络拉取或 clone
- `full load`：显式刷新路径使用，允许做 materialization

源码注释明确说，interactive startup 不应该因为 ref-tracked plugin 的 clone 而卡住。

这点非常值得复刻，因为它把：

- “尽快开始会话”
- “最终把扩展装到最新”

拆成了两条路径，而不是要求启动时一次做完。

### 3.3 plugin 不是单层“安装”，而是三层模型

`refresh.ts` 直接给出了一个很清楚的三层模型：

- Layer 1：intent，也就是 settings 里的启用意图
- Layer 2：materialization，也就是 marketplace / cache / 本地目录里的实际插件文件
- Layer 3：active components，也就是当前运行会话里真正挂上的 commands / agents / hooks / MCP / LSP

这是整套系统最值得写进教程的一点。很多系统把“安装完成”误当成“当前会话已经启用”，Claude Code 则明确把它们分开。

### 3.4 plugin 组件的刷新不是一个 `clear cache` 就完了

`refreshActivePlugins()` 做的是 Layer 3 的完整刷新：

- 清所有 plugin 相关 cache
- 重新 full load plugins
- 重新取 plugin commands 和 agent definitions
- 预热 plugin MCP / LSP server 缓存
- 更新 `AppState.plugins`
- bump `mcp.pluginReconnectKey`
- 重载 plugin hooks

也就是说，`/reload-plugins` 的语义不是“重新读一下 manifest”，而是“把当前会话的扩展组件整体换新”。

### 3.5 背景 marketplace 安装和手动刷新是两条路径

`PluginInstallationManager.ts` 说明 startup 期间可能会后台 reconcile marketplace：

- 新 marketplace 安装成功时，可以自动触发 `refreshActivePlugins()`
- 仅有更新时，则只把 `needsRefresh` 设为 `true`，提示用户手动 `/reload-plugins`

这再次体现了 Claude Code 的分层：

- 安装是分发层动作
- 当前会话是否立刻切换到新组件，是单独决策

### 3.6 plugin 的 CLI 和 runtime API 是分开的

`pluginOperations.ts` 是纯操作层，不写 stdout / 不 `process.exit()`；`pluginCliCommands.ts` 则是 CLI wrapper。

这是个很好的工程边界，因为：

- interactive UI 可以直接复用纯操作层
- CLI handler 只负责 console output、telemetry、exit code

## 4. skill 是进入命令面的 prompt 型能力单元

### 4.1 `commands.ts` 才能看清 skill 的真实地位

`commands.ts` 的 `loadAllCommands()` 会把这些来源合并到同一个命令面：

- bundled skills
- built-in plugin skills
- 磁盘上的 skill dir commands
- workflow commands
- plugin commands
- plugin skills
- 内建 commands

也就是说，skill 最终不是“特殊菜单项”，而是 slash command 系统里的一等公民。

### 4.2 `loadSkillsDir.ts` 同时支持多种来源

从 `loadSkillsDir.ts` 看，skills 可以来自：

- managed
- user
- project
- additional directories
- legacy commands 目录

而且会：

- 解析 frontmatter
- 做 realpath 级去重
- 支持 `allowed-tools`
- 支持 `context: fork`
- 支持路径条件激活

这说明 skills 本身就是一个不小的系统，不只是“读几个 markdown 文件”。

### 4.3 bundled skills 是启动时同步注册的

`main.tsx` 会在并行启动 `getCommands()` 之前先执行：

- `initBuiltinPlugins()`
- `initBundledSkills()`

`bundledSkills.ts` 则把这些技能注册成 `Command`，必要时还会在首次调用时把附带文件提取到磁盘，供模型 Read / Grep。

这点很关键，因为它说明 bundled skill 不是通过扫描磁盘发现的，而是编译进 CLI 的内置能力。

### 4.4 plugin skill 和 plugin command 是两套并行装载链

`loadPluginCommands.ts` 明确拆了：

- `getPluginCommands()`
- `getPluginSkills()`

两者都来自 enabled plugins，但 skill 会按 `isSkillMode` 走专门的加载路径。

这说明在 Claude Code 内部：

- “plugin 贡献命令”
- “plugin 贡献 skill”

不是同一回事，只是最终都汇入 slash command 面。

### 4.5 MCP 也可能提供 skills，但这条链在当前 restored source 里仍有缺口

`useManageMCPConnections.ts` 和 `client.ts` 都 feature-gated 地引用了 `fetchMcpSkillsForClient(...)`；`mcpSkillBuilders.ts` 与 `loadSkillsDir.ts` 也专门为 MCP skill discovery 预留了 builder registry。

但当前 restored source tree 里看不到对应的 `mcpSkills.ts` 实现文件。

因此可以确认的只有：

- Claude Code 设计上为 MCP-sourced skills 留了入口
- 这些 skill 也会走 `Command` 形态接入

至于当前版本完整实现细节，还需要结合本机提取物或缺失源码继续验证。

## 5. 三条扩展链最终怎么接回 runtime

### 5.1 命令面在 `commands.ts` 汇合

- built-in commands
- disk skills
- bundled skills
- plugin commands
- plugin skills
- 某些 workflow commands

这些最终都会出现在 `getCommands(cwd)` 的结果里，供 REPL 和 slash command 系统消费。

### 5.2 工具面在 MCP client 转换后汇合

MCP tools 进入 Claude Code 后直接变成内部 `Tool`，然后在 `REPL.tsx` 的 `getToolUseContext(...)` 中与 built-in tools 一起装进工具池。

### 5.3 会话状态在 `AppState` 汇合

`AppState` 至少持有这些扩展相关状态：

- `mcp.clients`
- `mcp.tools`
- `mcp.commands`
- `mcp.resources`
- `mcp.pluginReconnectKey`
- `plugins.enabled`
- `plugins.disabled`
- `plugins.commands`
- `plugins.errors`
- `plugins.installationStatus`
- `plugins.needsRefresh`

这说明扩展系统不是外围附属物，而是会话主状态的一部分。

### 5.4 刷新协议通过 `pluginReconnectKey` 把 plugin 和 MCP 接起来

`refreshActivePlugins()` 会 bump `mcp.pluginReconnectKey`，而 `useManageMCPConnections()` 会把它作为 effect 依赖之一。

所以：

- plugin 层刷新后
- MCP connection manager 会重新跑
- 新 plugin 带来的 MCP servers 才会真正进当前会话

这是一个非常值得写进教程的连接点。

## 6. 如果自己复刻，最小实现顺序应该是什么

### 6.1 先把三层拆开，不要做一个“大而全插件系统”

更合理的顺序是：

1. 先做 skills / commands 这类最轻量扩展
2. 再做 plugin 作为组件容器和分发层
3. 最后做 MCP 这种运行时远端能力接入

如果反过来先做一个“所有扩展统一一套抽象”，很容易把分发、命令、远端连接三件事搅在一起。

### 6.2 plugin refresh 一定要和 install 分开

Claude Code 的三层模型很值得直接借：

- settings 表达 intent
- materialization 处理 clone / cache / 安装
- refresh 负责当前 session 的 active components 切换

这会让系统更稳定，也更容易解释。

### 6.3 skill 最好直接接在命令系统上

不要把 skill 设计成“插件里的另一种奇怪对象”。Claude Code 的设计更自然：

- skill 本质上就是 prompt command
- 只是来源比普通 built-in command 更多

### 6.4 MCP tool 最好尽早转成内部统一抽象

`fetchToolsForClient()` 这种“进门就转成内部 `Tool`”的策略非常好，因为后面权限、tool orchestration、UI、query 都不必再 special-case MCP。

## 7. 当前仍需运行时验证的问题

- `fetchMcpSkillsForClient(...)` 在当前安装版本里到底暴露了多少 MCP skills，当前 restored source 缺失的 `mcpSkills.ts` 应如何补证
- plugin marketplace reconcile、background install、`needsRefresh` 与 `/reload-plugins` 在本机客户端中的真实交互节奏
- plugin 提供的 MCP / LSP / output styles 在 external build 中哪些默认可见，哪些仍受 feature flag 或 user type 限制
- enterprise / managed policy 对 MCP 与 plugin-only restriction 的最终优先级，是否和当前 sourcemap 版本完全一致

## 8. 后续正文建议怎么写

后续正式教程最好拆成三章，而不是压成一个“扩展系统”大章：

- `MCP：外部能力如何接入 Claude Code runtime`
- `Plugins：扩展包的发现、安装、刷新与组件注入`
- `Skills：命令面上的提示型能力单元`

这样才能把：

- 协议接入
- 扩展包管理
- prompt command 体系

这三件本来就不同的事讲清楚。
