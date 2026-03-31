# 并行产出计划

## 这份文档解决什么问题

把“研究 Claude Code”变成“稳定产出教程”的过程，需要明确哪些工作必须串行推进，哪些工作可以并行做素材沉淀。

这份文档的目标不是替代教程总纲，而是给当前阶段提供一个高效率的执行拆分。

相关文档：

- `notes/00-roadmap.md`：教程主题路线图
- `tutorial/how-to-build-a-claude-code.md`：读者视角的章节总纲
- `notes/02-work-queue.md`：当前可执行任务队列
- `notes/03-work-log.md`：持续执行日志

## 文档放置规则

- 稳定规则放在 `AGENTS.md`
- 阶段性执行规划、并行拆分、章节排期放在 `notes/`
- 当前可执行 backlog 放在 `notes/02-work-queue.md`
- 每轮执行后的状态回写放在 `notes/03-work-log.md`
- 教程承诺和对外导航放在 `README.md` 或 `tutorial/`

## 当前判断

当前最重要的工作，不是同时写很多教程正文，而是先把主骨架立稳。

这个骨架就是：

1. `01-overview.md`
2. `02-agent-loop.md`
3. 围绕 `agent loop` 展开的模块章节

原因：

- `agent loop` 是整套教程的中心解释轴
- 很多章节的叙事都要回勾 `query()`、`QueryEngine`、`processUserInput()` 和 `ToolUseContext`
- 如果主线没收稳，后续并行正文很容易重复、漂移或术语不一致

## 执行原则

### 1. 主骨架串行推进

以下内容默认只由一个人负责，避免叙事冲突：

- `tutorial/02-agent-loop.md`
- 教程总纲的章节顺序调整
- 全局术语约定
- “源码确认 / 运行时确认 / 推断” 的表达规范

### 2. 模块素材并行沉淀

其他主题可以先不直接写正文，而是先在 `notes/` 产出结构化笔记。

每条并行轨道都应优先交付：

- 值得先读的源码文件
- 关键控制流
- 章节核心结论
- 最小可复刻实现建议
- 需要运行时验证的问题

### 3. 正文晚于笔记

只有在以下条件满足时，才把某个模块从 `notes/` 提升为 `tutorial/`：

- 主线依赖已经足够明确
- 术语和结论没有明显漂移
- 关键源码路径已经确认
- 运行时验证需求已经标注清楚

## 推荐并行轨道

### A. Context + Memory

目标：

- 搞清模型每轮真正看到了什么
- 区分系统上下文、用户上下文、memory 文件、session 级状态

建议入口：

- `../claude-code-sourcemap/restored-src/src/context.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/*`
- `../claude-code-sourcemap/restored-src/src/services/SessionMemory/*`
- `../claude-code-sourcemap/restored-src/src/utils/memory/*`

建议产物：

- `notes/04-context-memory.md`

### B. Tools + Permissions

目标：

- 解释工具注册、可见性裁剪、执行编排和权限边界
- 明确 command 与 tool 的运行时区别

建议入口：

- `../claude-code-sourcemap/restored-src/src/Tool.ts`
- `../claude-code-sourcemap/restored-src/src/tools.ts`
- `../claude-code-sourcemap/restored-src/src/services/tools/*`
- `../claude-code-sourcemap/restored-src/src/tools/BashTool/*`
- `../claude-code-sourcemap/restored-src/src/types/permissions.ts`

建议产物：

- `notes/05-tools-permissions.md`

### C. CLI Entry + UI Shell

目标：

- 搞清 `main.tsx`、`setup.ts`、`replLauncher.tsx`、`REPL.tsx` 和 `AppState` 如何接起来
- 区分启动编排、会话状态、终端壳层和主循环边界

建议入口：

- `../claude-code-sourcemap/restored-src/src/main.tsx`
- `../claude-code-sourcemap/restored-src/src/setup.ts`
- `../claude-code-sourcemap/restored-src/src/replLauncher.tsx`
- `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`
- `../claude-code-sourcemap/restored-src/src/state/AppStateStore.ts`

建议产物：

- `notes/06-cli-ui.md`

### D. MCP + Plugins + Skills

目标：

- 解释 Claude Code 如何接入外部能力，并把它们并入统一 runtime
- 区分 MCP、插件、skills 三类扩展面的角色

建议入口：

- `../claude-code-sourcemap/restored-src/src/services/mcp/*`
- `../claude-code-sourcemap/restored-src/src/cli/handlers/mcp.tsx`
- `../claude-code-sourcemap/restored-src/src/cli/handlers/plugins.ts`
- `../claude-code-sourcemap/restored-src/src/services/plugins/*`
- `../claude-code-sourcemap/restored-src/src/skills/*`

建议产物：

- `notes/07-mcp-plugins-skills.md`

### E. Tasks + Subagents

目标：

- 解释复杂工作如何被拆分成任务、多 agent 和远端执行路径
- 搞清它和主 query loop 的交界面

建议入口：

- `../claude-code-sourcemap/restored-src/src/tasks/*`
- `../claude-code-sourcemap/restored-src/src/tasks.ts`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/*`
- `../claude-code-sourcemap/restored-src/src/tools/Task*/*`
- `../claude-code-sourcemap/restored-src/src/tools/SendMessageTool/*`

建议产物：

- `notes/08-tasks-subagents.md`

### F. Runtime 验证 + 脚本

目标：

- 用本机安装客户端和提取物验证高风险结论
- 优先把重复验证步骤脚本化

建议入口：

- `../extracts/`
- `../scripts/`
- `../HANDOFF.md`

建议产物：

- `notes/09-runtime-validation.md`
- 必要时新增 `../scripts/` 下的验证脚本

## 推荐执行顺序

### 第 1 阶段

- 串行完成或收稳 `tutorial/02-agent-loop.md`
- 并行沉淀 A、B、C 三条笔记线

### 第 2 阶段

- 把 A、B、C 收敛成教程正文
- 并行沉淀 D、E、F 三条笔记线

### 第 3 阶段

- 完成扩展系统、任务系统和运行时验证章节
- 开始“最小可用克隆”与“产品化”两章

## 正文转化顺序

建议按这个顺序把 `notes/` 转成 `tutorial/`：

1. Context
2. Tools
3. Permissions
4. CLI Entry
5. UI Shell
6. Memory
7. MCP / Plugins / Skills
8. Tasks / Subagents

其中：

- Context、Tools、Permissions 最贴近 `agent loop`，应该最早成文
- Tasks / Subagents 依赖更多，不建议过早写成正文

## 每条并行轨道的最小交付标准

一份合格的 `notes/*.md` 至少应包含：

1. 这个模块解决什么问题
2. 必读源码文件列表
3. 一条最关键的运行时控制流
4. 与 `agent loop` 的连接点
5. 自己实现时可以先简化什么
6. 哪些结论仍需运行时验证

## 协作约定

- 不要多人同时修改同一篇 `tutorial/*.md`
- 并行协作优先落在 `notes/*.md`
- 如果某条笔记开始影响全局写法，先更新 `02-agent-loop.md` 或总纲，再继续分支写作
- 如果某条规划只对当前阶段有效，留在 `notes/`；只有稳定规则才进入 `AGENTS.md`
