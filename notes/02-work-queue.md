# 当前工作队列

## 用法

这份文档是长程任务的可执行 backlog。

当 agent 被要求“继续做”时，默认流程是：

1. 先读 `AGENTS.md`
2. 再读 `notes/01-parallel-work-plan.md`
3. 然后按本文件的优先级选择一个 `ready` 或 `in_progress` 任务
4. 完成一轮工作后更新本文件，并在 `notes/03-work-log.md` 追加记录

## 状态定义

- `in_progress`：当前主任务，默认优先继续
- `ready`：可立即开工
- `blocked`：存在明确阻塞条件
- `parked`：暂时不做，但保留
- `done`：当前定义下已完成

## 选择规则

- 如果存在 `in_progress` 的主任务，优先继续它，除非日志明确说明应该切换
- 若没有 `in_progress`，选择优先级最高的 `ready`
- 不要同时推进两篇 `tutorial/*.md` 正文
- 并行性优先体现在 `notes/*.md`、验证脚本和结构化素材，而不是多篇正文同时写

## 当前队列

### T1. 收稳 `02-agent-loop` 章节

- 优先级：P0
- 状态：done
- 类型：正文主骨架
- 目标：把 `tutorial/02-agent-loop.md` 从草稿推进到可作为后续章节支点的状态
- 主要文件：
  - `tutorial/02-agent-loop.md`
  - `../claude-code-sourcemap/restored-src/src/query.ts`
  - `../claude-code-sourcemap/restored-src/src/QueryEngine.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/handlePromptSubmit.ts`
  - `../claude-code-sourcemap/restored-src/src/utils/processUserInput/processUserInput.ts`
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
- Definition of Done：
  - 覆盖从输入提交到 loop 终止的完整主线
  - 明确 REPL 路径与 SDK 路径的共享核心与差异
  - 标出恢复路径、继续条件、工具执行回注和状态持有点
  - 至少列出关键源码入口，避免只有概念描述
- 备注：
  - 2026-03-31：已补齐 shared core 与 wrapper 差异、Terminal 返回语义、recoverable error 延迟显露、fallback model、tool result 回注、`refreshTools()` 与 `maxTurns` 等关键控制流
  - 后续仅在其他章节倒逼术语调整时回补

### T2. Context + Memory 研究笔记

- 优先级：P1
- 状态：done
- 类型：并行素材
- 目标：形成后续“上下文管理”和“记忆系统”章节的研究底稿
- 建议产物：
  - `notes/04-context-memory.md`
- 主要文件：
  - `../claude-code-sourcemap/restored-src/src/context.ts`
  - `../claude-code-sourcemap/restored-src/src/memdir/*`
  - `../claude-code-sourcemap/restored-src/src/services/SessionMemory/*`
  - `../claude-code-sourcemap/restored-src/src/utils/memory/*`
- 依赖：
  - 可以先做研究
  - 正文成文前最好等待 T1 术语稳定
- 备注：
  - 2026-03-31：已完成 `notes/04-context-memory.md`，明确拆出静态上下文注入、动态相关记忆召回、session memory、durable auto-memory extraction 四条链路，并补入与 `agent loop` 的连接点、最小复刻顺序和待验证问题

### T3. Tools + Permissions 研究笔记

- 优先级：P1
- 状态：done
- 类型：并行素材
- 目标：形成工具契约、工具可见性裁剪、执行编排和权限边界的研究底稿
- 建议产物：
  - `notes/05-tools-permissions.md`
- 主要文件：
  - `../claude-code-sourcemap/restored-src/src/Tool.ts`
  - `../claude-code-sourcemap/restored-src/src/tools.ts`
  - `../claude-code-sourcemap/restored-src/src/services/tools/*`
  - `../claude-code-sourcemap/restored-src/src/tools/BashTool/*`
  - `../claude-code-sourcemap/restored-src/src/types/permissions.ts`
- 依赖：
  - 可以先做研究
  - 与 T1 紧密相关，注意术语一致
- 备注：
  - 2026-03-31：已完成 `notes/05-tools-permissions.md`，明确工具抽象、工具池装配、`tool_use -> tool_result` 执行链、通用 permission pipeline、文件系统边界、Bash 特例与 auto mode 下危险规则剥离逻辑

### T4. CLI Entry + UI Shell 研究笔记

- 优先级：P1
- 状态：done
- 类型：并行素材
- 目标：解释启动编排、REPL 壳层、AppState 与主循环的边界关系
- 建议产物：
  - `notes/06-cli-ui.md`
- 主要文件：
  - `../claude-code-sourcemap/restored-src/src/main.tsx`
  - `../claude-code-sourcemap/restored-src/src/setup.ts`
  - `../claude-code-sourcemap/restored-src/src/replLauncher.tsx`
  - `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
  - `../claude-code-sourcemap/restored-src/src/state/AppStateStore.ts`
- 备注：
  - 2026-03-31：已完成 `notes/06-cli-ui.md`，明确 `main.tsx` / `setup.ts` 的启动分层、headless 与 interactive 分流、`launchRepl()` 与 `App` 的薄壳职责、`AppState` 作为会话级 runtime model 的边界，以及 `REPL.tsx` 作为 interactive session controller 的关键桥接职责

### T5. MCP + Plugins + Skills 研究笔记

- 优先级：P2
- 状态：done
- 类型：并行素材
- 目标：解释外部扩展面如何接入 Claude Code runtime
- 建议产物：
  - `notes/07-mcp-plugins-skills.md`
- 主要文件：
  - `../claude-code-sourcemap/restored-src/src/services/mcp/*`
  - `../claude-code-sourcemap/restored-src/src/cli/handlers/mcp.tsx`
  - `../claude-code-sourcemap/restored-src/src/cli/handlers/plugins.ts`
  - `../claude-code-sourcemap/restored-src/src/services/plugins/*`
  - `../claude-code-sourcemap/restored-src/src/skills/*`
- 依赖：
  - 研究可先做
  - 正文最好在 T1、T3、T4 之后
- 备注：
  - 2026-03-31：已完成 `notes/07-mcp-plugins-skills.md`，明确区分 MCP 运行时接入层、plugin 分发与组件容器层、skill 能力单元层；补入 `loadAllPluginsCacheOnly()` / `refreshActivePlugins()` 的三层模型、`commands.ts` 的命令面汇合，以及 `pluginReconnectKey` 如何把 plugin refresh 与 MCP reconnection 接起来

### T6. Tasks + Subagents 研究笔记

- 优先级：P2
- 状态：ready
- 类型：并行素材
- 目标：解释任务拆分、多 agent 和远端执行路径
- 建议产物：
  - `notes/08-tasks-subagents.md`
- 主要文件：
  - `../claude-code-sourcemap/restored-src/src/tasks/*`
  - `../claude-code-sourcemap/restored-src/src/tasks.ts`
  - `../claude-code-sourcemap/restored-src/src/tools/AgentTool/*`
  - `../claude-code-sourcemap/restored-src/src/tools/Task*/*`
  - `../claude-code-sourcemap/restored-src/src/tools/SendMessageTool/*`
- 依赖：
  - 最好在 T3 稳定后再提升为正文

### T7. Runtime 验证与脚本化

- 优先级：P2
- 状态：blocked
- 类型：验证素材
- 目标：把高风险结论与本机客户端行为对应起来，并尽量脚本化
- 建议产物：
  - `notes/09-runtime-validation.md`
  - 必要时新增 `../scripts/` 下的验证脚本
- 当前阻塞：
  - 本次检查时 `../extracts/` 目录下没有可直接读取的提取文件
  - `../HANDOFF.md` 记录了提取物路径，但这些文件当前不在工作区可见结果中
- 解阻方式：
  - 先确认提取物是否在未纳入版本控制的位置
  - 若缺失，则按 `../HANDOFF.md` 记载的方法重新生成

### T8. Claude Code Harness 独立文章

- 优先级：P1
- 状态：done
- 类型：独立专题 / 正文草稿
- 目标：解释 Claude Code 的 harness 不是单独类，而是一层分散实现的 runtime shell
- 产物：
  - `notes/10-harness.md`
  - `tutorial/claude-code-harness.md`
- 主要文件：
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
- 备注：
  - 2026-03-31：先在 `notes/` 里固定 “harness = runtime shell” 的研究提纲，再产出一篇独立教程文章
  - 暂不调整主教程章节编号，把这篇作为 standalone topic 处理
  - 2026-03-31：按用户反馈重写正文风格，从“分层概念讲解”改成更接近 Anthropic Engineering 的问题驱动、失败模式与设计取舍叙事

## 下一个默认动作

如果用户只说“继续”或“开始这个长程任务”，默认先执行：

1. 执行 T6，开始 `Tasks + Subagents` 研究笔记
2. 若 T6 暂时无法推进，再执行 T7
