# 09. Subagent 与任务系统：复杂工作是如何被拆开的

## 这篇文章要回答什么

Claude Code 之所以不像一个“会调几个工具的聊天壳”，很大一部分原因就在这里：

- 它会把复杂工作拆给别的 agent 去做
- 这些 agent 不是一次性函数调用，而是能持续运行、能被继续 steering、能被 kill、能恢复的后台任务

但如果只说“Claude Code 支持 subagent”，其实还是太模糊了。

真正的问题是：

**Claude Code 到底是怎么把一个子 agent 变成一个可管理的运行单元的？**

这一章要讲清楚的就是：

- `AgentTool` 到底负责什么
- fresh subagent、fork subagent、remote agent 各自是什么
- `runAgent()` 为什么才是真正的子 agent runtime
- 为什么子 agent 一旦进后台，就会变成一个 task
- `SendMessage`、`TaskStop`、output file、通知消息是怎么把它变成“可操作的后台 worker”的

## 先给结论

Claude Code 里的 `subagent`，不是一个单独类，也不是“模型又开了一个线程”。

它更像下面这条链：

```text
AgentTool
  -> 选择 spawn 路径
  -> runAgent() 建子 runtime
  -> LocalAgentTask / RemoteAgentTask 挂进任务系统
  -> SendMessage / TaskStop / output file / notification 形成控制面
```

也就是说，Claude Code 的 subagent 系统本质上是：

- 一个 delegation 入口
- 一个复用主 query loop 的子 runtime
- 一个后台任务壳
- 一套可继续操作的控制面

如果只盯着 `AgentTool.tsx`，你会误以为它只是“开一个 agent”。

但真正值得抄的，其实是后半段：

- 它怎么让这个 agent 在后台继续跑
- 怎么把进度、输出、恢复、kill、通知都串起来

## 先看地图

先不要陷进源码，先抓住大图：

```text
模型决定要委托子任务
  |
  v
AgentTool
  |
  +-> fresh subagent
  +-> fork subagent
  +-> remote agent
  +-> teammate spawn（另一套 team/swarm 路径）
  |
  v
runAgent() / remote session
  |
  v
Task system
  - LocalAgentTask
  - RemoteAgentTask
  - output file
  - transcript
  - progress
  - notification
  |
  v
控制面
  - SendMessage
  - TaskStop
  - Read output file
  - resume
```

真正关键的判断有三个：

- `AgentTool` 不是整个 subagent 系统，只是入口
- `runAgent()` 不是“另一套协议”，它还是同一个 Claude Code agent loop
- task system 不是可有可无的 UI 附件，而是后台 agent 能被产品化管理的关键

## 再看一条主流程

如果把一轮最常见的 subagent 调用压成流程，大致是这样：

```text
AgentTool.call()
  -> 选 agent definition / fork / remote
  -> 决定 sync 还是 async
  -> 构造 promptMessages / systemPrompt 策略 / tool pool
  -> runAgent(...)
  -> 如果是 async：
       registerAsyncAgent()
       runAsyncAgentLifecycle()
       返回 outputFile 和 agentId
  -> 子 agent 完成后：
       发送 <task-notification>
       父 agent 下一轮再消费结果
```

这条流程里最值得记住的一点是：

**子 agent 的结果，并不是立刻同步塞回父 agent 当前这次 tool call。**

如果它是后台运行，Claude Code 的设计是：

- 先把它变成一个 task
- 让它自己继续跑
- 完成后再用通知消息回到主线程

这和“函数调用拿返回值”完全不是同一个交互模型。

## 先记住五个容易混淆的边界

### 1. agent definition 不等于运行中的 subagent

`AgentDefinition` 只是配置：

- prompt
- tools
- model
- permissionMode
- skills
- MCP

真正运行中的 subagent 还需要：

- 自己的消息历史
- 自己的 `ToolUseContext`
- 自己的 transcript
- 自己的 task state

### 2. subagent 不等于 teammate

普通 subagent 是：

- 我委托一个 worker 做一段子任务

teammate 是：

- 我加入一个 team runtime 做长期协作

它们会共享一部分底层机制，但产品语义不一样。

### 3. runtime task 不等于 Todo task

Claude Code 里有两套 task：

- `LocalAgentTask` / `RemoteAgentTask`
  - 谁正在后台跑
- `TaskCreate/List/Get/Update`
  - 工作计划和依赖关系

这两套都叫 task，但不要写混。

### 4. fork 不是“没写 subagent_type 时的普通默认值”

在 fork gate 关闭时：

- 不写 `subagent_type` = `general-purpose`

在 fork gate 开启时：

- 不写 `subagent_type` = fork path

这个区别会直接影响：

- system prompt
- tools
- messages prefix
- prompt cache

### 5. 后台 agent 的结果不是“立刻可见”

一旦 agent 被放进后台，你当前这轮只知道：

- 它启动了
- 它的 `agentId`
- 它的 `outputFile`

真正的结果要等：

- task notification
- 或你后面主动去看 output file / transcript

## 先抓源码入口

如果你只想先顺着源码把主线走一遍，最该先看的就是这几组文件：

- `src/tools/AgentTool/AgentTool.tsx`
  - spawn 路由器
- `src/tools/AgentTool/runAgent.ts`
  - 子 agent runtime
- `src/utils/forkedAgent.ts`
  - fork / isolated context / cache sharing 基础设施
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx`
  - 本地后台 agent 的任务壳
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`
  - 远端 agent 的任务壳
- `src/tools/SendMessageTool/SendMessageTool.ts`
  - 后台 agent 的继续控制
- `src/tools/AgentTool/resumeAgent.ts`
  - 恢复后台 agent
- `src/Task.ts`、`src/utils/task/framework.ts`
  - 通用 task 框架

## 第 1 层：`AgentTool` 是 delegation API，不是执行器

`AgentTool.call()` 的工作，核心不是“跑模型”，而是“选路”。

它至少要先决定下面这些事：

- 这是 teammate spawn 吗
- 这是 fork path 吗
- 要用哪个 agent definition
- 这个 agent 的 MCP prerequisite 满足了吗
- 这次要不要 remote
- 要不要 worktree
- 是同步返回，还是后台运行

所以更准确地说，`AgentTool` 是：

- 子任务委托的 API 层

而不是：

- 子 agent 自己的 runtime loop

这点很重要，因为很多人做第一版 agent CLI 时，会把“spawn worker”和“worker runtime”揉成一个函数，后面很快就失控。

## 第 2 层：一条 `AgentTool` 调用，可能走成 4 种完全不同的路径

### 1. fresh subagent

这是最直观的一种。

你指定：

- `subagent_type`

Claude Code 就会：

- 读取这个 agent 的 prompt
- 给它裁一套自己的工具池
- 用它自己的 permission mode / model / skills / MCP 设定

这更像：

- “开一个有明确角色设定的新 worker”

### 2. fork subagent

fork path 则完全不是这个思路。

它的目标不是：

- 让子 agent 换角色

而是：

- 让子 agent 继承父 agent 的完整上下文
- 尽量共享 prompt cache

所以 fork 子 agent 会：

- 复用父 agent 已渲染好的 system prompt
- 复用父工具数组
- 复用父 thinking config
- 复用父消息前缀

这是一个非常 Claude Code 的设计，因为它明显不是单纯从产品语义出发，而是把 runtime 成本也一起考虑进去了。

### 3. remote agent

如果 `isolation === "remote"`，则本地不会直接跑 `runAgent()`。

它会：

- 检查远端环境 eligibility
- teleport 到远端 session
- 本地只保留一个 remote task 代理

所以 remote agent 本质上是：

- 本地控制面 + 远端执行面

### 4. teammate spawn

这条路径经常会和普通 subagent 混掉。

但从 `AgentTool.tsx` 看，只要有：

- `team_name`
- `name`

就可能走到：

- `spawnTeammate()`

这条线后面接的是：

- mailbox
- team roster
- in-process teammate task

它和普通 subagent 的产品模型已经不同了。

## 第 3 层：`runAgent()` 才是真正的子 agent runtime

### 它做的第一件事，不是换模型，而是建立子上下文

`runAgent()` 会先准备：

- `initialMessages`
- `userContext`
- `systemContext`
- agent 自己的 `systemPrompt`
- agent 自己的 `ToolUseContext`

你可以把它理解成：

- 再开一份缩小版 Claude Code runtime

而不是：

- 调一次特殊的“子模型接口”

### 它会按 agent 类型裁剪上下文税

这里有个很值得注意的细节。

Claude Code 并不是所有子 agent 都无脑继承同样的静态上下文。

例如：

- 某些只读 agent 可以不带 `claudeMd`
- `Explore` / `Plan` 会去掉 `gitStatus`

这说明 Claude Code 在这里已经不是“能跑就行”，而是在认真优化：

- 子 agent 每次启动的上下文成本

### 它会在真正进 query 前注册很多 agent 专属运行时能力

在 `query()` 之前，`runAgent()` 还会做：

- `SubagentStart` hooks
- frontmatter hooks
- frontmatter skills preload
- agent-specific MCP server 初始化

这意味着 agent definition 不是只有 prompt，它实际上是：

- 一份会在 runtime 上展开的能力包

## 第 4 层：`createSubagentContext()` 解释了 Claude Code 为什么能并发跑多个 agent

这是整套设计里最值得借鉴的一块。

默认情况下，它会把容易互相污染的可变状态隔离掉：

- `readFileState`
- `contentReplacementState`
- `abortController`
- `toolDecisions`
- 多数 mutation callback

只有在明确 opt-in 时，才共享：

- `setAppState`
- `setResponseLength`
- `abortController`

这套设计的价值在于：

- 你不需要为每个子 agent 开一个完全独立进程
- 但也不会因为共享太多状态而相互踩踏

换句话说：

**Claude Code 的 subagent 并发能力，靠的不是“全进程隔离”，而是“有选择地隔离可变状态”。**

## 第 5 层：fork path 真正特别的地方，是它围绕 prompt cache 设计

如果你只看产品行为，fork 似乎只是：

- “继承父上下文的 subagent”

但源码说明它远不止这个。

### 它为什么要保留父 assistant message 里的全部 `tool_use`

因为 fork 想让请求前缀尽量和父 agent 保持一致。

### 它为什么要给每个 `tool_use` 配统一 placeholder `tool_result`

因为不同 fork child 之间真正会变的，应该尽量只剩：

- 最后一段 directive

这样 prompt cache 才更容易命中。

### 它为什么不能递归 fork

因为 fork child 仍然保留 `AgentTool`，这是为了：

- 保持工具定义字节稳定

但产品上它并不想真的让 fork worker 再无限继续分叉。

所以运行时又专门加了递归保护。

## 第 6 层：一旦进后台，subagent 就变成了一个 task

这一步是很多简单 agent runtime 没做好的地方。

Claude Code 没有把后台 subagent 简化成：

- “留一个 Promise 等它结束”

它会把它注册成一个完整的 task。

### `LocalAgentTask` 里到底多了什么

至少多了这些：

- `agentId`
- `prompt`
- `progress`
- `messages`
- `pendingMessages`
- `retain`
- `diskLoaded`
- `outputOffset`
- `notified`

这些字段合起来，意味着后台 agent 有了：

- 自己的身份
- 自己的进度
- 自己的 transcript
- 自己的输出路径
- 自己的交互状态

这已经不是一次普通 tool call 的抽象了，而是：

- 一个被 runtime 托管的后台 job

### 为什么这件事重要

因为只有这样，Claude Code 才能自然支持：

- completion notification
- `SendMessage`
- `TaskStop`
- viewed transcript
- resume
- output file

## 第 7 层：`runAsyncAgentLifecycle()` 才是后台 agent 的真正产品壳

后台 agent 最成熟的地方，不在 spawn，而在 lifecycle。

这个 wrapper 负责：

- 从 `runAgent()` 流式接消息
- 更新 progress tracker
- 周期性生成 agent summary
- 完成后做 final result 整理
- 必要时跑 handoff classifier
- 发 task notification
- 被 kill 时保留 partial result

这层的意义在于：

- `runAgent()` 只负责“跑子 loop”
- lifecycle wrapper 负责“把这个 loop 变成可管理产品对象”

这个分层非常对。

如果你把两者揉在一起，后面加：

- summary
- notification
- kill semantics
- worktree cleanup

都会非常难收拾。

## 第 8 层：控制面让后台 subagent 变成“可继续操作的 worker”

### `SendMessage` 让后台 agent 不是 fire-and-forget

`SendMessageTool` 不只是 swarm 邮箱工具。

它还能：

- 找到本地 agent task
- 往它的 `pendingMessages` 里塞新消息
- 需要时走 `resumeAgentBackground()`

这意味着 Claude Code 的后台 agent 可以被继续 steering。

### `TaskStop` 让 kill 走统一入口

`TaskStopTool -> stopTask()` 这一层把：

- local shell
- local agent
- remote agent

统一成了一套 stop API。

这是 task framework 真正有价值的地方之一。

### `TaskOutput` 已经退到兼容层

源码已经把 `TaskOutputTool` 标成 deprecated。

Claude Code 更推荐：

- 完成后给一个 output file path
- 真要看内容，再用普通 `Read`

这反而是更干净的设计，因为它让“读后台输出”重新回到统一文件工具面，而不是长期维护一个特殊 API。

## 第 9 层：不要把 runtime task system 和 Todo task system 混成一章

Claude Code 里还有另一套 `TaskCreate/List/Get/Update`。

它们处理的不是：

- 运行中的后台 agent

而是：

- 工作怎么拆
- 谁负责什么
- 哪些任务互相阻塞

所以后面如果单独写教程，最好明确拆成：

- runtime task system
- todo / planning task system

否则读者很容易误以为：

- `TaskCreate` 创建的就是后台 agent task

实际上根本不是。

## 如果你自己实现，最小应该先做哪一层

不要一上来就复刻 Claude Code 的全部 multi-agent 产品面。

更现实的顺序是：

1. 先做最基础的 `AgentTool -> runAgent`
2. 再做 background task registration
3. 再加 output file 和 completion notification
4. 再加 `SendMessage` / resume
5. 最后再考虑 fork path、worktree、remote、summary

因为真正最难的，不是“让子 agent 跑起来”，而是：

- 让它跑完之后还能被管理、被恢复、被继续 steering

## 最值得抄的设计

### 1. 把 spawn API 和子 runtime 分层

`AgentTool` 管路由，`runAgent()` 管执行，这样结构才稳。

### 2. 用 task system 托管后台 agent

没有这层，后台 subagent 很快就会沦为不可管理的 Promise。

### 3. 用 sidechain transcript + output file 做持久化接口

这为 resume、Read、通知、UI 都提供了统一基础。

### 4. fork 专门围绕 cache sharing 设计

这不是“锦上添花”，而是 Claude Code 真正拉开工程质量差距的地方。

## 这一层最容易犯的错误

### 1. 把 subagent 写成一次同步函数调用

这样你几乎拿不到可继续操作的后台工作流。

### 2. 把 agent definition 当成运行时对象

结果后面 transcript、task、permission state 都会没地方放。

### 3. 不区分 runtime task 和 Todo task

最后 task 这个词会在教程里越讲越乱。

### 4. 把 fork 理解成“换个默认 agent”

它真正特殊的地方是 cache sharing，不是角色语义。

## 下一章应该看什么

这章其实已经把 Claude Code 的“复杂工作怎么拆”讲到了 runtime 层。

下一步最自然的回勾有两条：

- 回到工具系统，解释 `AgentTool` 本身是怎么进入整个 tool runtime 的
- 回到记忆系统，解释长任务里哪些信息应该留在主线程，哪些应该沉到 session memory
