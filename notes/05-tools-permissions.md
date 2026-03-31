# Tools + Permissions 研究笔记

## 这份笔记解决什么问题

Claude Code 的工具系统，不只是“给模型一组函数”。

从源码看，它至少同时处理四件事：

1. 哪些能力要暴露给模型
2. 这些能力如何以稳定 schema 进入 prompt
3. 一次 `tool_use` 如何被执行、回注成 `tool_result`
4. 在什么条件下这次调用应该被允许、阻止、询问或自动分类

后续正文如果只写“有 Bash / Read / Edit 这些工具”，会丢掉真正有价值的部分。真正值得解释的是：

- 工具暴露面如何被裁剪
- 运行时权限如何分层生效
- 为什么 Claude Code 的 permission system 不只是一个确认弹窗

## 核心结论

Claude Code 的 tools 和 permissions 是两个耦合但不同层的系统：

- `Tool` 是能力对象，定义 schema、描述、执行逻辑、UI 渲染、并发属性、自动分类输入、权限 matcher
- `tools.ts` 负责组装“模型这一轮看得到的工具池”
- `toolExecution.ts` / `toolOrchestration.ts` 负责把 `tool_use` 变成真正执行与 `tool_result`
- `permissions.ts` / `filesystem.ts` / `permissionSetup.ts` 负责在运行时决定“这次调用能不能做”

更精确地说：

- 有些工具在 prompt 生成前就被隐藏了
- 有些工具模型能看到，但调用时仍会被权限系统拦下
- 有些调用即使在 `bypassPermissions` 下也仍然必须 ask
- auto mode 并不是“全部自动允许”，而是“规则预处理 + 快速路径 + classifier + denial tracking”的一整套决策链

## 必读源码入口

### 工具抽象与工具池

- `../claude-code-sourcemap/restored-src/src/Tool.ts`
- `../claude-code-sourcemap/restored-src/src/tools.ts`
- `../claude-code-sourcemap/restored-src/src/constants/tools.ts`

### 工具执行编排

- `../claude-code-sourcemap/restored-src/src/services/tools/toolExecution.ts`
- `../claude-code-sourcemap/restored-src/src/services/tools/toolOrchestration.ts`
- `../claude-code-sourcemap/restored-src/src/services/tools/StreamingToolExecutor.ts`
- `../claude-code-sourcemap/restored-src/src/services/tools/toolHooks.ts`
- `../claude-code-sourcemap/restored-src/src/query.ts`

### 权限类型与权限总流程

- `../claude-code-sourcemap/restored-src/src/types/permissions.ts`
- `../claude-code-sourcemap/restored-src/src/hooks/useCanUseTool.tsx`
- `../claude-code-sourcemap/restored-src/src/utils/permissions/permissions.ts`
- `../claude-code-sourcemap/restored-src/src/utils/permissions/permissionSetup.ts`
- `../claude-code-sourcemap/restored-src/src/utils/permissions/filesystem.ts`

### Bash 作为最复杂特例

- `../claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx`
- `../claude-code-sourcemap/restored-src/src/tools/BashTool/bashPermissions.ts`
- `../claude-code-sourcemap/restored-src/src/tools/BashTool/shouldUseSandbox.ts`

## 1. `Tool` 在 Claude Code 里到底是什么

### 1.1 它不是“call 函数 + schema”这么简单

`Tool.ts` 里的 `Tool` 接口包含的内容远比普通 function calling 更丰富：

- `call(...)`
- `description(...)`
- `inputSchema` / `inputJSONSchema`
- `isConcurrencySafe(...)`
- `isReadOnly(...)`
- `isDestructive(...)`
- `validateInput(...)`
- `checkPermissions(...)`
- `preparePermissionMatcher(...)`
- `toAutoClassifierInput(...)`
- 一整套 transcript / REPL UI 渲染方法

这说明 Claude Code 的 tool 不是单纯给模型看的 schema，而是一个贯穿：

- prompt 暴露
- 本地执行
- 权限决策
- UI 展示
- telemetry

的统一能力对象。

### 1.2 `buildTool()` 的默认值很有意思

`buildTool()` 会填默认实现：

- `isEnabled` 默认 `true`
- `isConcurrencySafe` 默认 `false`
- `isReadOnly` 默认 `false`
- `isDestructive` 默认 `false`
- `checkPermissions` 默认直接 `allow`
- `toAutoClassifierInput` 默认空字符串

这里最关键的不是默认 `allow`，而是它依赖“通用权限系统在外层统一兜底”。也就是说：

- 工具自己的 `checkPermissions()` 只负责工具特有规则
- 通用 rule / mode / classifier / async-agent 限制，在外层统一做

### 1.3 一个好工具对象在 Claude Code 里要回答哪些问题

从接口设计倒推，一个工具至少要回答：

- 模型怎么调用我
- 我在什么情况下能并行
- 我是否会改状态
- 我该如何进入 auto classifier transcript
- 我该如何和 permission rules 做 pattern 匹配
- 我的结果如何序列化成 API 可接受的 `tool_result`
- 我的调用和结果在终端里怎么显示

这比很多 agent 框架常见的“函数列表”成熟得多。

## 2. 工具池是怎么组出来的

### 2.1 `getAllBaseTools()` 是所有内建工具的源头

`tools.ts` 的 `getAllBaseTools()` 是 built-in tools 的单一真相来源。

它会受这些因素影响：

- feature flags
- `process.env`
- `USER_TYPE`
- REPL / plan / worktree / LSP 等模式

也就是说，“Claude Code 有哪些工具”不是静态答案，而是运行时装配结果。

### 2.2 `getTools()` 会先做能力裁剪

`getTools(permissionContext)` 的关键步骤：

1. 根据 `--bare` / simple mode 切换到极简工具集
2. 在 REPL mode 下用 `REPLTool` 包裹底层原语，隐藏 primitive tools
3. 过滤掉被 deny rule blanket 拒绝的工具
4. 再过一遍 `isEnabled()`

这说明权限系统的第一层其实不是“点了工具以后弹窗”，而是“某些工具模型根本看不到”。

### 2.3 deny rule 会在 prompt 前就裁掉工具

`filterToolsByDenyRules()` 复用了运行时同一套 deny matcher。

这有个很重要的后果：

- 比如某个 MCP server 被规则整体 deny 掉，模型不仅不能调用它，连 schema 都看不到

这是 Claude Code 很工程化的一点。它不把“可见”与“可用”完全分开，而是尽量把确定不能用的能力前移隐藏。

### 2.4 内建工具和 MCP 工具如何合并

`assembleToolPool(permissionContext, mcpTools)` 的逻辑是：

1. built-in tools 先按当前 context 算出来
2. MCP tools 也按 deny rule 过滤
3. built-in 和 MCP 分区排序
4. 用 `uniqBy(name)` 去重，built-in 优先

源码里特别强调排序是为了 prompt cache 稳定性。也就是说，工具池顺序本身就是系统设计的一部分，不只是数组实现细节。

### 2.5 deferred tools 也是工具池策略的一部分

`Tool` 上有：

- `shouldDefer`
- `alwaysLoad`
- `searchHint`

再结合 `ToolSearchTool`，可以看出 Claude Code 并不是总把所有工具 schema 都完整发给主模型。它会把一部分工具延迟加载，用 tool search 把“能力发现”从“首轮 prompt 体积”里拆出去。

## 3. 一次 `tool_use` 是怎么被执行的

### 3.1 大路径

主路径可以压成：

1. query 收到 assistant 的 `tool_use`
2. 找到工具对象
3. 做 schema 校验和输入校验
4. 跑 PreToolUse hooks
5. 走 permission 决策
6. 真正执行 `tool.call(...)`
7. 把结果序列化成 `tool_result`
8. 跑 PostToolUse / PostToolUseFailure hooks
9. 把 `tool_result` 回注回 query loop

这条链的主入口在：

- `toolExecution.ts`
- `toolOrchestration.ts`
- `query.ts`

### 3.2 `runToolUse()` 是单次执行的主入口

`runToolUse()` 做的第一件事不是执行，而是解析工具身份：

- 先在当前可见工具集里找
- 如果没找到，再检查是不是 deprecated alias
- 如果仍找不到，直接产出带错误的 `tool_result`

这说明“工具名解析”和“工具执行”是分开的，而且兼容了 transcript 老名字。

### 3.3 输入校验分两层

`checkPermissionsAndCallTool()` 先过两层校验：

1. `inputSchema.safeParse(...)`
2. `tool.validateInput(...)`

这两层分别解决不同问题：

- schema parse：模型传参类型对不对
- validateInput：这次调用在语义上是否合理，比如文件是否存在、是否已读过、是否 stale

Claude Code 明显不信任模型能稳定给出合法输入。

### 3.4 hook 在权限前就能改输入或阻断继续

`runPreToolUseHooks()` 的结果可以：

- 发消息
- 给出 `hookPermissionResult`
- 更新输入
- `preventContinuation`
- 直接 stop

也就是说 hook 不是简单日志回调，而是真正能参与 tool-use control flow 的一级机制。

### 3.5 permission 不是工具内部自己决定

`runToolUse()` 最终通过：

- `resolveHookPermissionDecision(...)`
- `canUseTool(...)`

拿到 permission decision。

这意味着真正的权限决策是三部分汇合：

- tool 自己的 `checkPermissions()`
- hook 决策
- 通用 permission pipeline

### 3.6 工具结果不是直接丢回去

工具执行成功后，会经历：

- `mapToolResultToToolResultBlockParam(...)`
- `processPreMappedToolResultBlock(...)` 或 `processToolResultBlock(...)`
- 再包装成 user message 里的 `tool_result`

这里还有几个重要点：

- 结果太大时会触发持久化与 preview
- MCP tools 的后处理路径与 built-in tools 略有不同
- permission allow 时附带的反馈、图片等 content blocks 也会拼进结果消息

所以 Claude Code 的 `tool_result` 不是简单字符串，而是一个带包装、预算和 UI 含义的产物。

### 3.7 失败和拒绝也走标准回注

无论是：

- schema 错
- validateInput 失败
- permission deny / ask
- tool.call 抛错

最终都会变成一个标准 `tool_result` 回注，而不是只在本地 UI 层报错。

这就是为什么 agent loop 还能继续推理下一步。

## 4. 工具执行编排不是单线程串行

### 4.1 默认批处理执行

`toolOrchestration.ts` 的 `runTools()` 会把工具调用分成批次：

- 并发安全的工具可以成组并发
- 非并发安全工具单独串行

分组依据不是 `isReadOnly()`，而是 `isConcurrencySafe()`。这是个更精确的抽象，因为：

- 有些 read-only 工具也可能因为内部副作用不适合并发
- 有些工具虽然会写状态，但可能通过 context modifier 控制一致性

### 4.2 context modifier 是并发执行的关键补丁

并发工具执行时，`contextModifier` 不会立刻乱序改 context，而是：

- 先按 `toolUseID` 排队
- 等并发批次结束后再按原顺序回放

这说明 Claude Code 在并发工具执行上并不是“只管开线程”，而是专门处理了 context 一致性问题。

### 4.3 还有一条 streaming execution 路径

`query.ts` 在开启 `streamingToolExecution` gate 时会使用 `StreamingToolExecutor`。

它的职责是：

- 工具边流出边启动
- 并发安全工具可并行
- 非并发工具独占执行
- 结果仍按接收顺序产出
- 流式回退时丢弃孤儿结果

这条链说明 Claude Code 已经不是传统的“整条 assistant message 收完再统一跑工具”模型，而是在尝试进一步降低 tool latency。

## 5. Permission system 的真实结构

### 5.1 permission mode 只是外层开关

`types/permissions.ts` 里定义了模式：

- `default`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `plan`
- 内部模式里还有 `auto` / `bubble`

但 mode 只是 permission pipeline 的一部分，不是全部。

### 5.2 `ToolPermissionContext` 真正承载的状态

`ToolPermissionContext` 里不止 mode，还有：

- `additionalWorkingDirectories`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `isBypassPermissionsModeAvailable`
- `isAutoModeAvailable`
- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`
- `prePlanMode`

也就是说 permission 不是单个枚举值，而是一整份运行时 policy context。

### 5.3 context 是怎么建出来的

`permissionSetup.ts` 会从这些来源装配 `ToolPermissionContext`：

- settings on disk
- CLI 允许 / 禁止规则
- base tools preset
- `--add-dir`
- `PWD` 与真实 cwd 的 symlink 差异
- feature flags

并且它会在进入 auto mode 时做一件很关键的事：

- `stripDangerousPermissionsForAutoMode(...)`

也就是把那些会绕过 classifier 的危险 allow 规则先剥掉。

### 5.4 危险 allow rule 会在 auto mode 被移除

源码明确把这些视为危险：

- `Bash(*)`
- `Bash(python:*)`
- `PowerShell(iex:*)`
- 过宽的 Agent allow 规则

原因很直接：

- 这些规则会让高风险动作在 classifier 之前就被自动放行

所以 auto mode 不是“在现有 allow rules 上再套一个 classifier”，而是会先重写 permission context。

## 6. 通用 permission pipeline 的顺序

### 6.1 `hasPermissionsToUseToolInner()` 的主顺序

从 `permissions.ts` 看，主顺序大致是：

1. tool-level deny
2. tool-level ask
3. 调用工具自己的 `checkPermissions()`
4. 如果工具要求用户交互，保留 ask
5. 如果是 content-specific ask rule，保留 ask
6. 如果是 safety check，保留 ask
7. 若当前 mode 是 `bypassPermissions` 或 plan-bypass，直接 allow
8. 若命中 always allow rule，allow
9. 其余 `passthrough` 转成 `ask`

这意味着：

- tool 自己的权限逻辑在通用 mode 之前执行
- 但 tool 自己不能跳过更高优先级的 deny / safety constraints

### 6.2 外层 wrapper 还会再做二次转换

`hasPermissionsToUseTool(...)` 在 inner decision 之后还会继续处理：

- `dontAsk`：把 ask 变 deny
- `auto` / `plan+auto`：进入 classifier 路径
- headless / async agent：先跑 PermissionRequest hooks，再 auto-deny

所以 permission pipeline 的真实结构是：

- inner：规则与工具特定逻辑
- outer：mode、classifier、headless 策略

### 6.3 `useCanUseTool()` 才是 interactive UI 入口

REPL 里的真正交互在 `useCanUseTool.tsx`：

- 若已 allow，直接返回
- 若 deny，直接构造成拒绝
- 若 ask，再进入 interactive / coordinator / swarm worker 各自 handler

因此 permission dialog 只是 `ask` 分支的一种表现形式，不是权限系统本身。

## 7. 文件系统权限边界是单独一套系统

### 7.1 File 工具不是自己手写权限判断

`FileReadTool` / `FileEditTool` / `FileWriteTool` 都把核心权限委托给：

- `checkReadPermissionForTool(...)`
- `checkWritePermissionForTool(...)`

这让 Claude Code 能把文件类工具的规则统一起来。

### 7.2 working directory 是默认允许边界

`filesystem.ts` 里一个核心概念是：

- `pathInAllowedWorkingPath(...)`

默认策略可以概括成：

- working directory 内，某些模式下直接 allow
- working directory 外，默认 ask，除非有明确 allow rule

再加上 `additionalWorkingDirectories`，Claude Code 得到的是“可操作工作区集合”，不是单一路径。

### 7.3 safety check 在 allow 之前

`checkWritePermissionForTool()` 的顺序很重要：

- deny rule
- 内部可写路径 carve-out
- `.claude/**` session allow 的特殊旁路
- `checkPathSafetyForAutoEdit(...)`
- ask rule
- `acceptEdits` working dir allow
- allow rule

也就是说危险路径的 safety check 比普通 allow rule 更早执行。

这解释了为什么某些路径即使用户配了 broad allow，也不会静默放行。

### 7.4 Claude 自己管理的内部路径有 carve-out

源码明确对这些内部路径做了特殊 allow：

- session memory
- project transcript / project dir
- current session plan files
- tool results dir
- scratchpad
- auto memory
- agent memory

这反过来也说明：Claude Code 的很多“内部产品能力”，本质上也是靠工具 + 文件系统权限 carve-out 组合起来的。

### 7.5 `.claude` 不是简单禁区

`.claude` 目录既是危险目录，又允许通过更细粒度方式临时放行。

尤其是：

- `getClaudeSkillScope(...)` 会把 `.claude/skills/{name}` 识别成更窄 scope
- session 级 allow rule 可以只放开单个 skill，而不是整个 `.claude`

这说明 Claude Code 在权限体验上追求的是“最小授权面”，而不是只有全开 / 全关两种选择。

## 8. Bash 是最复杂的特例

### 8.1 Bash 的复杂度远超其他工具

`BashTool` 不只是一个 shell wrapper。它还额外处理：

- search/read/list 命令识别
- sandbox 选择
- compound command 解析
- sed edit 预览
- classifier prompt
- permission rule prefix 匹配
- background task

如果要复刻 Claude Code，Bash 一定要单独成章。

### 8.2 Bash 权限不是只看工具名

`bashPermissions.ts` 里有大量内容相关规则：

- exact command
- prefix rule
- wildcard rule
- subcommand 级拆分
- compound command 权限聚合
- sandbox auto-allow

这意味着 Bash 的权限决策本质上是“命令语言级别”的，不是普通工具名级别。

### 8.3 sandbox 只是 permission system 的一个分支

`shouldUseSandbox()` 的判断会受这些影响：

- sandbox 是否开启
- `dangerouslyDisableSandbox`
- unsandboxed 是否被 policy 允许
- excluded commands

更重要的是源码明确写了：

- `excludedCommands` 是用户体验功能，不是安全边界

真正的安全边界仍然在 permission / sandbox policy 本身。

### 8.4 auto mode 会给 Bash 走多条快速路径

在 auto mode 下，Bash 不一定总进 classifier。可能先走：

- acceptEdits fast-path
- safe-tool allowlist
- sandbox auto-allow

只有这些都没命中时，才会进入 classifier。

这说明 Claude Code 的 auto mode 目标不是“凡事都调 classifier”，而是“尽量把昂贵判定留给真正需要的高风险动作”。

## 9. 与 `agent loop` 的连接点

最关键的连接点有这些：

1. `tools.ts` 组出来的工具池会进入 system prompt / API tool schema
2. query 里 `tool_use` block 的出现，才是真正的继续条件
3. `runTools()` / `StreamingToolExecutor` 把 `tool_use` 执行成 user-side `tool_result`
4. 这些 `tool_result` 再喂回下一轮 loop
5. permissions 决策本身也会生成 transcript 可见结果，因此会直接影响后续推理

因此教程里应该明确写：

- tool system 是 `agent loop` 的内部子系统
- permission system 不是 loop 外的 UI 选项，而是 tool execution 的一部分

## 10. 如果自己实现，可以先简化什么

最小可用克隆建议按这个顺序实现：

### 第一阶段

- 一个简化版 `Tool` 接口：`name`、`schema`、`call`
- 一个工具池装配器
- 一个串行 `runToolUse()`

### 第二阶段

- 为工具加 `isConcurrencySafe`
- 加 schema 校验与 `validateInput`
- 把执行结果统一封成 `tool_result`

### 第三阶段

- 加 working directory 边界
- 加 allow / deny / ask 规则
- 文件工具统一走共享文件系统权限逻辑

### 第四阶段

- 再补 hooks、streaming execution、tool search、auto mode classifier

如果一开始就试图复刻 Claude Code 全量权限模式，很容易把系统复杂度拉爆。

## 11. 写正文时最值得强调的 design takeaway

- Claude Code 把“工具可见性”和“工具可执行性”拆成两层，但会尽量前移隐藏确定不可用的能力
- `Tool` 是统一能力对象，不只是 schema
- permission system 的本体是规则、mode、classifier、filesystem boundary 和 hooks，不是弹窗
- `bypassPermissions` 也不是绝对豁免，safety check 与某些 ask rule 仍然优先
- 文件工具的核心不是 Edit/Write 本身，而是路径边界与 stale-state 检查
- Bash 必须被视为一个语言级子系统，而不是普通工具

## 12. 仍需运行时验证的问题

- 当前安装版客户端里 deferred tools / `ToolSearchTool` 的默认开启条件
- REPL mode 下 primitive tools 被隐藏后，模型看到的实际工具描述文本
- `streamingToolExecution` 在本机版本里的真实启用情况
- auto mode 下 dangerous allow rules 是否会在 transcript / UI 中有明确提示
- MCP tool 在本机客户端里的命名规范是否与 sourcemap 恢复源码完全一致

## 13. 对后续正文的建议拆法

后续教程最好不要把 Tools 和 Permissions 完全揉成一章，建议拆成两章但彼此强引用：

### `04-tools.md`

- `Tool` 抽象
- 工具池装配
- `tool_use -> tool_result` 执行链
- 并发与 streaming tool execution

### `05-permissions.md`

- `ToolPermissionContext`
- rule / mode / classifier / hooks
- working directory 与文件系统边界
- Bash / File 工具的权限特例

这样更贴近源码真实结构，也更方便读者按实现顺序复刻。
