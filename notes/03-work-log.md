# 工作日志

## 用法

这份文档用于记录长程任务中的实际执行历史，避免每一轮都重新判断上下文。

每次较完整的工作结束后，至少记录：

- 日期
- 完成了什么
- 依据了哪些源码或文件
- 更新了哪些仓库内文件
- 当前阻塞或风险
- 建议的下一步

## 记录模板

```md
## YYYY-MM-DD - 简短标题

- 完成：
- 依据：
- 更新文件：
- 阻塞 / 风险：
- 下一步：
```

## 2026-03-31 - Harness 初始化

- 完成：
  - 确认 `the-claude-code/` 当前目标是产出一套中文优先的 Claude Code 架构与实现教程
  - 识别当前主骨架是 `01-overview` 与进行中的 `02-agent-loop`
  - 新增并行产出计划、工作队列和工作日志机制
  - 在 `AGENTS.md` 中补入 planning rules 与 autonomous execution rules
- 依据：
  - `README.md`
  - `tutorial/how-to-build-a-claude-code.md`
  - `tutorial/01-overview.md`
  - `tutorial/02-agent-loop.md`
  - `../HANDOFF.md`
  - `../claude-code-sourcemap/restored-src/src` 目录级扫描
- 更新文件：
  - `AGENTS.md`
  - `notes/01-parallel-work-plan.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - `tutorial/02-agent-loop.md` 仍处在演进阶段，后续模块正文不宜过早并发写
  - 本次检查时 `../extracts/` 目录未返回可见提取文件，运行时验证链路暂时不完整
- 下一步：
  - 优先继续 `tutorial/02-agent-loop.md`
  - 然后开始 `Context + Memory` 或 `Tools + Permissions` 的结构化研究笔记

## 2026-03-31 - 收稳 02-agent-loop 第一轮

- 完成：
  - 核对 `query.ts`、`QueryEngine.ts`、`handlePromptSubmit.ts`、`processUserInput.ts`、`REPL.tsx` 的关键控制流
  - 收紧 `tutorial/02-agent-loop.md` 中 REPL / SDK 路径差异的表述
  - 补入 `QueuedCommand` 统一入口、`UserPromptSubmit` hooks、`Terminal` 结束语义、`transition` 状态、recoverable error 延迟显露、fallback model、`refreshTools()`、`maxTurns`、`QueryEngine` 包装职责等内容
  - 修正 `notes/01-parallel-work-plan.md` 中建议产物文件名与 work queue 不一致的问题
- 依据：
  - `../claude-code-sourcemap/restored-src/src/query.ts`
  - `../claude-code-sourcemap/restored-src/src/QueryEngine.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/handlePromptSubmit.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/processUserInput/processUserInput.ts`
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
- 更新文件：
  - `tutorial/02-agent-loop.md`
  - `notes/01-parallel-work-plan.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - `02-agent-loop` 当前定义下已可作为后续章节支点，但后面写 Context / Tools 章节时，仍可能发现少量术语需要回调
  - 运行时验证链路仍未恢复，`../extracts/` 现状与 `../HANDOFF.md` 记载不一致
- 下一步：
  - 开始 `Context + Memory` 研究笔记
  - 如中途发现工具契约更影响主线，可并行切到 `Tools + Permissions` 笔记

## 2026-03-31 - 完成 Context + Memory 研究笔记

- 完成：
  - 新增 `notes/04-context-memory.md`
  - 把 Claude Code 的 context / memory 机制拆成四条链：静态上下文注入、动态相关记忆召回、session memory、durable auto-memory extraction
  - 明确 `Context` 与 `Memory` 后续正文应拆章，而不是把 `CLAUDE.md`、`MEMORY.md`、session notes 和后台提取混写
  - 补入与 `agent loop` 的连接点、最小可复刻顺序以及仍需运行时验证的问题
- 依据：
  - `../claude-code-sourcemap/restored-src/src/context.ts`
  - `../claude-code-sourcemap/restored-src/src/main.tsx`
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
  - `../claude-code-sourcemap/restored-src/src/utils/claudemd.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/memdir.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/memoryTypes.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/memoryScan.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/findRelevantMemories.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/paths.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/attachments.ts`
  - `../claude-code-sourcemap/restored-src/src/query.ts`
  - `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemory.ts`
  - `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemoryUtils.ts`
  - `../claude-code-sourcemap/restored-src/src/services/SessionMemory/prompts.ts`
  - `../claude-code-sourcemap/restored-src/src/services/extractMemories/extractMemories.ts`
- 更新文件：
  - `notes/04-context-memory.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 这轮仍然是 sourcemap 级确认，没有用本机客户端提取物验证 `relevant_memories` attachment、session memory 落盘和 `extractMemories` gate 的真实运行时表现
  - `Context` 与 `Memory` 的正文拆章顺序现在已有明确建议，但正式成文前仍要和后续 `Tools + Permissions` 的术语保持一致
- 下一步：
  - 开始 `Tools + Permissions` 研究笔记
  - 重点梳理工具注册、schema、可见性裁剪、tool-use 回注和权限边界

## 2026-03-31 - 完成 Tools + Permissions 研究笔记

- 完成：
  - 新增 `notes/05-tools-permissions.md`
  - 明确 `Tool` 不是单纯 schema，而是同时承载 prompt 暴露、本地执行、权限匹配、UI 渲染和 telemetry 的统一能力对象
  - 梳理 built-in / MCP 工具池的装配逻辑，以及 deny rule 如何在 prompt 前直接隐藏工具
  - 梳理 `tool_use -> tool_result` 的主执行链，包括 schema 校验、`validateInput`、Pre/Post hooks、permission 决策、结果包装与失败回注
  - 梳理通用 permission pipeline、文件系统边界、Bash 特例，以及 auto mode 下危险 allow rule 的剥离机制
- 依据：
  - `../claude-code-sourcemap/restored-src/src/Tool.ts`
  - `../claude-code-sourcemap/restored-src/src/tools.ts`
  - `../claude-code-sourcemap/restored-src/src/query.ts`
  - `../claude-code-sourcemap/restored-src/src/services/tools/toolExecution.ts`
  - `../claude-code-sourcemap/restored-src/src/services/tools/toolOrchestration.ts`
  - `../claude-code-sourcemap/restored-src/src/services/tools/StreamingToolExecutor.ts`
  - `../claude-code-sourcemap/restored-src/src/services/tools/toolHooks.ts`
  - `../claude-code-sourcemap/restored-src/src/types/permissions.ts`
  - `../claude-code-sourcemap/restored-src/src/hooks/useCanUseTool.tsx`
  - `../claude-code-sourcemap/restored-src/src/utils/permissions/permissions.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/permissions/permissionSetup.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/permissions/filesystem.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx`
  - `../claude-code-sourcemap/restored-src/src/tools/BashTool/bashPermissions.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/BashTool/shouldUseSandbox.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/FileReadTool/FileReadTool.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/FileEditTool/FileEditTool.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/FileWriteTool/FileWriteTool.ts`
- 更新文件：
  - `notes/05-tools-permissions.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 这轮仍然主要是 sourcemap 级确认，尚未用本机客户端验证 deferred tools、`streamingToolExecution`、REPL 隐藏 primitive tools 的实际 prompt 暴露形态
  - 后续写 CLI / UI 章节时，可能需要回补少量关于 REPL wrapper 与 raw tools 的措辞
- 下一步：
  - 开始 `CLI Entry + UI Shell` 研究笔记
  - 重点梳理 `main.tsx`、`setup.ts`、`replLauncher.tsx`、`REPL.tsx` 与 `AppStateStore` 的边界

## 2026-03-31 - 完成 CLI Entry + UI Shell 研究笔记

- 完成：
  - 新增 `notes/06-cli-ui.md`
  - 明确 `main.tsx` 与 `setup.ts` 的职责边界：前者负责启动编排与模式分流，后者负责进程环境、副作用和后台初始化
  - 梳理 headless 与 interactive 两条外壳路径，明确它们共享底层 runtime，但不共享同一个 UI 壳
  - 梳理 `launchRepl()`、`App.tsx`、`AppState.tsx`、`store.ts` 与 `AppStateStore.ts` 的关系，明确 `AppState` 是会话级 runtime model，不只是 UI state
  - 梳理 `REPL.tsx` 中 `getToolUseContext(...)` 与 `onQueryImpl(...)` 的桥接职责，明确 interactive 模式下 REPL 本身就是 session controller
- 依据：
  - `../claude-code-sourcemap/restored-src/src/main.tsx`
  - `../claude-code-sourcemap/restored-src/src/setup.ts`
  - `../claude-code-sourcemap/restored-src/src/replLauncher.tsx`
  - `../claude-code-sourcemap/restored-src/src/components/App.tsx`
  - `../claude-code-sourcemap/restored-src/src/state/AppState.tsx`
  - `../claude-code-sourcemap/restored-src/src/state/store.ts`
  - `../claude-code-sourcemap/restored-src/src/state/AppStateStore.ts`
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
  - `../claude-code-sourcemap/restored-src/src/query.ts`
- 更新文件：
  - `notes/06-cli-ui.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 这轮仍然是 sourcemap 级确认，尚未结合本机提取物验证 remote bridge、CCR mirror、deferred prefetch 与 startup bookkeeping 在当前安装版本里的真实可达性
  - `main.tsx` 中 ant-only / feature-flag 相关路径较多，后续写正式教程时需要避免把条件分支误写成“所有版本默认都有”
- 下一步：
  - 开始 `MCP + Plugins + Skills` 研究笔记
  - 重点梳理扩展系统的加载入口、运行时接入点、工具与命令暴露面，以及它们与 `REPL` / `query` / `AppState` 的连接方式

## 2026-03-31 - 完成 MCP + Plugins + Skills 研究笔记

- 完成：
  - 新增 `notes/07-mcp-plugins-skills.md`
  - 明确区分 MCP、plugin、skill 三种扩展面：MCP 是运行时接入层，plugin 是分发与组件容器层，skill 是命令面上的 prompt 型能力单元
  - 梳理 `getClaudeCodeMcpConfigs()`、`useManageMCPConnections()`、`fetchToolsForClient()` 的主路径，明确 MCP tools / commands / resources 如何进入当前会话
  - 梳理 `pluginLoader.ts`、`refresh.ts`、`PluginInstallationManager.ts` 的三层模型，明确 plugin 的 intent / materialization / active components 刷新边界
  - 梳理 `commands.ts`、`loadSkillsDir.ts`、`bundledSkills.ts`、`loadPluginCommands.ts` 的命令面汇合关系，明确 skills 最终如何以 `Command` 形态进入 slash command 系统
  - 补入 `pluginReconnectKey` 作为 plugin refresh 与 MCP reconnection 的关键连接点
- 依据：
  - `../claude-code-sourcemap/restored-src/src/main.tsx`
  - `../claude-code-sourcemap/restored-src/src/commands.ts`
  - `../claude-code-sourcemap/restored-src/src/services/mcp/config.ts`
  - `../claude-code-sourcemap/restored-src/src/services/mcp/client.ts`
  - `../claude-code-sourcemap/restored-src/src/services/mcp/useManageMCPConnections.ts`
  - `../claude-code-sourcemap/restored-src/src/services/mcp/MCPConnectionManager.tsx`
  - `../claude-code-sourcemap/restored-src/src/cli/handlers/mcp.tsx`
  - `../claude-code-sourcemap/restored-src/src/cli/handlers/plugins.ts`
  - `../claude-code-sourcemap/restored-src/src/services/plugins/PluginInstallationManager.ts`
  - `../claude-code-sourcemap/restored-src/src/services/plugins/pluginOperations.ts`
  - `../claude-code-sourcemap/restored-src/src/services/plugins/pluginCliCommands.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/pluginLoader.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginCommands.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginAgents.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginHooks.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/mcpPluginIntegration.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/loadPluginOutputStyles.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/plugins/refresh.ts`
  - `../claude-code-sourcemap/restored-src/src/skills/loadSkillsDir.ts`
  - `../claude-code-sourcemap/restored-src/src/skills/bundledSkills.ts`
  - `../claude-code-sourcemap/restored-src/src/skills/mcpSkillBuilders.ts`
- 更新文件：
  - `notes/07-mcp-plugins-skills.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 当前 restored source tree 中缺少被 `useManageMCPConnections.ts` / `client.ts` 引用的 `mcpSkills.ts` 实现，因此 MCP-sourced skills 的完整运行路径仍有一段未能源码级闭环
  - plugin / MCP 相关路径 feature flag 较多，正式教程里需要明确哪些是确认存在，哪些只是在当前 sourcemap 中保留了 feature-gated 分支
- 下一步：
  - 开始 `Tasks + Subagents` 研究笔记
  - 重点梳理任务对象、subagent 工具入口、远端执行路径，以及它们与 `agent loop` / `AppState` / `tool runtime` 的连接方式

## 2026-03-31 - 调整 Tools + Permissions 笔记的可读性写法

- 完成：
  - 按用户反馈重写 `notes/05-tools-permissions.md` 的开头结构
  - 新增“先看地图”总览区块，先把工具系统拆成工具定义、工具池裁剪、`tool_use` 执行回注三层
  - 新增两张 ASCII 框图：一张总图，一张单次 `tool_use` 执行流程图
  - 补入三组最容易混淆的边界：`command` vs `tool`、可见 vs 可执行、`Tool.checkPermissions()` vs 完整权限系统
- 依据：
  - `notes/05-tools-permissions.md`
  - 用户对现有写法“还不够清晰易懂，需要先给 overview / 框图”的直接反馈
- 更新文件：
  - `notes/05-tools-permissions.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 目前只对 `notes/05` 做了试改，其他章节仍然保留更偏源码拆解的旧写法
  - 新写法是否足够清楚，还要看用户是否认可这种“先给总图、再讲边界、最后讲细节”的结构
- 下一步：
  - 如果这种写法有效，就把后续 tutorial 和 notes 默认改成“overview 先行”的表达方式
  - 若继续正文链，回头重写 `tutorial/03-context-management.md` 的开头结构再推进下一章

## 2026-03-31 - 产出 Harness 独立专题

- 完成：
  - 围绕 “Claude Code 是怎么构建 harness 的” 梳理了一份结构化提纲
  - 明确当前源码里的 `harness` 更像一层分散实现的 runtime shell，而不是单独的 `Harness` 类
  - 产出一篇独立教程文章，重点解释启动期建壳、内部状态通道、输出改写、权限放行与 control plane
  - 在教程索引中补入该独立专题链接，避免正文产出后不可发现
- 依据：
  - `../claude-code-sourcemap/restored-src/src/setup.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/toolResultStorage.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx`
  - `../claude-code-sourcemap/restored-src/src/utils/claudeCodeHints.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/permissions/filesystem.ts`
  - `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemory.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/memdir.ts`
  - `../claude-code-sourcemap/restored-src/src/cli/structuredIO.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/teleport.tsx`
  - `../claude-code-sourcemap/restored-src/src/context.ts`
- 更新文件：
  - `notes/10-harness.md`
  - `tutorial/claude-code-harness.md`
  - `tutorial/how-to-build-a-claude-code.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 文中关于远端 CCR 模式的判断主要基于 restored source，尚未用 `../extracts/` 或运行时行为交叉验证
  - `harness` 目前作为独立专题存在，是否升格为主教程章节，后续仍要看整体目录是否需要调整
- 下一步：
  - 回到默认主线，继续 `Tasks + Subagents` 研究笔记
  - 若后续 Tools / Permissions 章节成文，可把 harness 文章里的部分判断回勾进主教程正文

## 2026-03-31 - 重写 Harness 文章风格

- 完成：
  - 按用户反馈重写 `tutorial/claude-code-harness.md`
  - 把原先偏“分层定义 + 总结归纳”的写法，改成更接近 Anthropic Engineering 的问题驱动叙事
  - 新版本以 naive harness 的失败模式开场，再展开 Claude Code 在 session runtime、内部路径、artifact 管理、side channel、权限放行和 control plane 上的具体做法
  - 显式补入这套设计的代价与待验证点，减少泛化结论和教学腔
- 依据：
  - `tutorial/claude-code-harness.md` 旧稿
  - `https://www.anthropic.com/engineering/harness-design-long-running-apps`
  - `https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents`
  - `../claude-code-sourcemap/restored-src/src/setup.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/toolResultStorage.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx`
  - `../claude-code-sourcemap/restored-src/src/utils/mcpOutputStorage.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/claudeCodeHints.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/permissions/filesystem.ts`
  - `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemory.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/memdir.ts`
  - `../claude-code-sourcemap/restored-src/src/cli/structuredIO.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/teleport.tsx`
- 更新文件：
  - `tutorial/claude-code-harness.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 目前只是叙事风格重构，没有同步把 `notes/10-harness.md` 改写成同样风格；提纲仍偏研究笔记格式
  - 远端与 runtime 行为的判断仍主要来自 restored source 与官网工程文章，不等于已安装客户端逐项验证
- 下一步：
  - 如用户认可这版叙事风格，可继续用同样方法改写后续正文章节
  - 回到默认主线，继续 `Tasks + Subagents` 研究笔记

## 2026-04-01 - 新增上下文压缩章节

- 完成：
  - 新增 `tutorial/04-context-compaction.md`
  - 把“上下文太长以后怎么缩回可运行范围”从 `Context` 章节中拆出来，单独写成 tutorial 正文
  - 采用“先结论、先地图、先主流程，再逐层解释”的结构，补齐 `tool result budget`、`snip`、`microcompact`、`context collapse`、`autocompact`、overflow recovery、`/compact` 与 partial compact 的相对关系
  - 在教程索引中加入第 4 章，并顺延后续暂定章节编号
- 依据：
  - `../claude-code-sourcemap/restored-src/src/query.ts`
  - `../claude-code-sourcemap/restored-src/src/setup.ts`
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
  - `../claude-code-sourcemap/restored-src/src/services/compact/autoCompact.ts`
  - `../claude-code-sourcemap/restored-src/src/services/compact/compact.ts`
  - `../claude-code-sourcemap/restored-src/src/services/compact/microCompact.ts`
  - `../claude-code-sourcemap/restored-src/src/services/compact/prompt.ts`
- 更新文件：
  - `tutorial/04-context-compaction.md`
  - `tutorial/how-to-build-a-claude-code.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 当前 restored source tree 没有 `services/contextCollapse/index.js` 的完整实现，所以这一章能清楚讲主链位置和职责，但还不能把 collapse 内部算法写得很实
  - `notes/diagrams/permission-context-and-tool-pool.drawio` 当前是未跟踪文件，这轮没有动它
- 下一步：
  - 若继续正文链，可以回头把 `tutorial/03-context-management.md` 再按同样风格压一轮
  - 若继续默认 backlog，则按队列进入 `T6 Tasks + Subagents`

## 2026-04-01 - 重写上下文管理章节

- 完成：
  - 重写 `tutorial/03-context-management.md`
  - 把原先偏“按源码分段解释”的写法，改成和第 4 章一致的结构：先结论、先总图、先主流程，再讲源码入口和四层静态注入
  - 单独讲清 `systemPrompt`、`userContext`、`systemContext` 的职责边界，以及 `Context` 与 `Memory`、git snapshot 与实时状态、静态注入与代码读取之间的区别
  - 补入 `main.tsx` 里 deferred prefetch 和 trust gate 对 `systemContext` 的影响，避免把上下文预取写成无条件行为
- 依据：
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
  - `../claude-code-sourcemap/restored-src/src/constants/prompts.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/systemPrompt.ts`
  - `../claude-code-sourcemap/restored-src/src/context.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/claudemd.ts`
  - `../claude-code-sourcemap/restored-src/src/main.tsx`
- 更新文件：
  - `tutorial/03-context-management.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - 当前章节仍然以 sourcemap 为主，没有结合本机提取物验证某些 feature-flag 分支在当前安装版本里的实际开启状态
  - 文中保留了少量源码名词如 `toolUseContext`，后续如果用户继续要求降术语密度，还可以再压一轮
- 下一步：
  - 按用户要求提交并推送当前文档改动
  - 推送后默认仍回到 `T6 Tasks + Subagents`

## 2026-04-01 - 入库并推送 Prompt Cache 文章

- 完成：
  - 确认 `tutorial/05-prompt-cache.md` 已写完且已经被主教程总纲引用
  - 更新 `README.md` 的阅读入口，把 `05-prompt-cache` 加到直接阅读列表里
  - 在工作队列中补入 Prompt Cache 正文章节条目
  - 按用户要求移除未跟踪的 `notes/diagrams/permission-context-and-tool-pool.drawio`
- 依据：
  - `tutorial/05-prompt-cache.md`
  - `README.md`
  - `tutorial/how-to-build-a-claude-code.md`
- 更新文件：
  - `README.md`
  - `tutorial/05-prompt-cache.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- 阻塞 / 风险：
  - `tutorial/05-prompt-cache.md` 本轮只做入库与导航补齐，没有再做一轮术语压缩；如果用户继续要求“更像 03/04 那样先图后流程”，后面还可以再收一轮
  - 这轮没有动其余未提交文件，避免把不相关草稿一并推上去
- 下一步：
  - 提交并推送 Prompt Cache 相关文章
  - 推送后默认仍回到 `T6 Tasks + Subagents`
