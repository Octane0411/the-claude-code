# Claude Code Harness 研究提纲

## 这份笔记解决什么问题

回答一个容易被误解的问题：

- Claude Code 里的 `harness` 到底是什么
- 它为什么不是一个单独的 `Harness.ts`
- 如果自己实现一个 Claude Code-like 系统，应该先实现 harness 的哪几层

这份笔记对应一篇独立教程文章：

- `tutorial/claude-code-harness.md`

## 核心判断

当前源码更像把 `harness` 实现成一组分散但协同的 runtime shell，而不是一个单独类。

这里的 `harness` 可以先定义为：

**包在模型外面的一层运行时外壳。它决定模型能看到什么、能调用什么、哪些结果会被改写后再回流，以及哪些状态会跨回合持久化。**

换句话说：

- 模型负责生成决策
- harness 负责把这些决策约束在一个可执行、可恢复、可审计的环境里

## 拟定文章大纲

### 1. 先纠正一个误区：源码里没有单一 Harness 类

- 不要把文章写成“找到了几个 harness 注释”
- 要把它写成一层横切 runtime

### 2. Harness 先在启动期把运行时壳搭起来

- `setup()` 里提前注册 session memory、commands、plugin hooks、context collapse
- 说明 Claude Code 不是在第一轮采样时才临时拼环境

### 3. Harness 会先准备一组内部状态通道

- `session-memory`
- `plans`
- `tool-results`
- `auto memory`
- `agent memory`

这些路径不是普通用户工作区的一部分，而是模型与 runtime 之间的受控回流面。

### 4. Harness 改写工具执行结果，而不是原样回注

- 大输出落盘到 `tool-results/`
- 再把“文件路径 + 预览”回给模型
- `<claude-code-hint />` 这类标签被扫描并剥离，只给 UI 侧消费

### 5. Harness 不只有 I/O，还包含 control plane

- 权限请求通过 `control_request` 往宿主协商
- 远端模式下可以在 session 启动前注入 `set_permission_mode`
- 这说明 harness 还负责“谁说了算”而不仅是“谁能读写”

### 6. 如果自己实现一个更干净的版本

先做最小四层：

1. session runtime 与内部目录
2. tool execution + result persistence
3. permission gate
4. host control protocol

后续再加 UI、远端模式、自动记忆、插件提示侧通道。

## 必读源码文件

- `../claude-code-sourcemap/restored-src/src/setup.ts`
- `../claude-code-sourcemap/restored-src/src/utils/toolResultStorage.ts`
- `../claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx`
- `../claude-code-sourcemap/restored-src/src/utils/claudeCodeHints.ts`
- `../claude-code-sourcemap/restored-src/src/utils/permissions/filesystem.ts`
- `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemory.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/memdir.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/paths.ts`
- `../claude-code-sourcemap/restored-src/src/cli/structuredIO.ts`
- `../claude-code-sourcemap/restored-src/src/utils/teleport.tsx`
- `../claude-code-sourcemap/restored-src/src/context.ts`

## 关键源码确认

### A. 启动时就建立 background runtime，而不是等第一轮采样

- `setup.ts` 在进入首轮 query 前注册 `initSessionMemory()`，并预取 commands / plugin hooks
- 这更像“搭壳”而不是“按需补丁”

### B. Session 级内部目录是 harness 的基础设施

- `toolResultStorage.ts` 把大输出持久化到 `projectDir/sessionId/tool-results`
- `filesystem.ts` 允许读取这些内部路径
- `SessionMemory/sessionMemory.ts` 会创建 `session-memory/summary.md`

### C. 模型看到的是 harness 改写后的结果

- `BashTool.tsx` 会把大输出拷贝到 `tool-results/`
- 同时扫描 `<claude-code-hint />` 并在模型可见输出前剥离
- `claudeCodeHints.ts` 明确把 hints 定义成 harness-only side channel

### D. 远端模式下 harness 还负责 control plane

- `structuredIO.ts` 用 `control_request` / `control_response` 跟宿主对接
- `teleport.tsx` 在远端 session 初始事件里预注入 `set_permission_mode`

### E. “目录已存在，可直接写” 也是 harness 承诺的一部分

- `memdir/memdir.ts` 多处注释写明 harness 会保证 memory dir 存在
- 这类承诺本质上是在把文件系统条件前移成 runtime 契约

## 一条最关键的控制流

```text
setup()
  -> 注册 session memory / commands / plugin hooks
  -> query loop 中调用工具
  -> tool output 被 harness 过滤 / 落盘 / 生成受控引用
  -> 内部路径在权限层被放行
  -> 模型再通过 Read/FileRead 读取这些 runtime 产物
```

这条链路说明，Claude Code 不是“模型直接操作世界”，而是“模型操作 harness 暴露出来的世界”。

## 与 agent loop 的连接点

- `query()` 负责循环
- harness 负责给 `query()` 准备可运行的环境
- `ToolUseContext`、内部目录、权限路径、控制协议都属于 agent loop 外围但决定 loop 能否稳定运转的 runtime 设施

## 自己实现时可以先简化什么

- 不做 React/Ink，先做 headless runtime
- 不做复杂的 `AppStateStore`，先做 session object
- 不做完整插件 hint 协议，先实现“大输出持久化 + 内部路径放行”
- 不做完整 remote bridge，先做本地 host 的 permission callback

## 设计意见

下面这些是教程里可以明确说出的设计意见，不应伪装成源码事实：

- 从工程角度看，Claude Code 真正的“产品壁垒”不只是 prompt 或 loop，而是 harness 这一层
- 很多 toy agent 做不稳，不是因为模型不够强，而是没有把 harness 单独作为 runtime 设计对象
- 如果要做一个干净实现，建议显式引入 `HarnessRuntime` 或 `AgentRuntime` 概念，而不是把职责散在工具代码里

## 仍需运行时验证的问题

- 远端 CCR 模式下内部目录的真实挂载布局，与 restored source 注释是否完全一致
- `claude-code-hint` 协议在已安装客户端里是否还有额外 UI 限流或 telemetry 行为
- `tool-results` 在 resume / fork / remote reconnect 下的实际复制或迁移边界

## 版本假设

- 当前判断基于 `../claude-code-sourcemap/restored-src/src` 在 2026-03-31 可见的还原源码
- 结论以“当前源码显示如此”为准，涉及已安装客户端行为的断言仍应与 `../extracts/` 或运行时观察交叉验证
