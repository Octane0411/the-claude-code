# Tasks + Subagents 研究笔记

## 这份笔记解决什么问题

Claude Code 里的 `subagent` 不是一个单独类，也不是“模型再开一个线程”这么简单。

从源码看，它实际上是一整条链：

```text
AgentTool
  -> 选择 agent 定义 / fork 路径 / remote 路径
  -> runAgent() 建子 runtime
  -> LocalAgentTask / RemoteAgentTask 挂进任务系统
  -> SendMessage / TaskStop / TaskOutput / 通知消息 形成控制面
```

如果后续正文只写“Claude Code 支持 subagent”，会把几个非常关键的边界混掉：

- agent definition 不是运行中的 agent
- background task 不是 Todo task
- fork subagent 不是 fresh subagent
- teammate swarm 不是普通 subagent

这份笔记就是先把这些边界和控制流收清。

## 核心结论

Claude Code 的 subagent 系统，可以压成下面几个判断。

### 1. `AgentTool` 只是 spawn API，不是 subagent runtime 本体

`AgentTool` 负责：

- 决定这次到底要走哪条 spawn 路径
- 选哪个 agent definition
- 决定是同步、异步、worktree 还是 remote
- 最后把参数交给 `runAgent()`

真正跑子 agent query loop 的，是：

- `runAgent()`

真正把它挂进 UI / 通知 / output file / kill / resume 的，是：

- `LocalAgentTask`
- `RemoteAgentTask`
- `Task*` / `SendMessage`

### 2. Claude Code 里至少有 4 条“子工作流”路径

从 `AgentTool.tsx` 看，这次调用可能走成：

1. fresh subagent
   - 指定 `subagent_type`
   - 子 agent 用自己的 system prompt 和裁剪后的工具池
2. fork subagent
   - 在 fork gate 开启时省略 `subagent_type`
   - 子 agent 继承父 agent 的完整上下文和 system prompt
3. remote agent
   - `isolation: "remote"`
   - 通过 CCR / teleport 远端执行
4. teammate spawn
   - 有 `team_name` 且有 `name`
   - 走 swarm / mailbox 路径，不是普通 subagent

### 3. Claude Code 里有两个完全不同的 “Task” 体系

这个边界非常重要。

第一套是 runtime background tasks：

- `local_agent`
- `remote_agent`
- `local_bash`
- `in_process_teammate`
- `local_workflow`
- `monitor_mcp`

它们存在于：

- `AppState.tasks`

用途是：

- 展示运行中的后台工作
- 提供 output file
- 支持 kill / resume / 通知 / UI 面板

第二套是 Todo / task list：

- `TaskCreate`
- `TaskList`
- `TaskGet`
- `TaskUpdate`

它们存在于：

- `utils/tasks.ts` 背后的持久任务列表

用途是：

- 规划与协调工作
- 表达依赖关系、owner、blockedBy

这两套东西都叫 task，但不是一回事。

### 4. fork subagent 是一个专门为 prompt cache 设计的特殊路径

`forkSubagent.ts` 和 `runAgent.ts` 的实现说明，fork 的目标不是“另一种 agent 类型”，而是：

- 尽可能继承父 agent 的 cache-critical prefix
- 让子 agent 可以在共享 prompt cache 的前提下快速开工

为此它专门做了几件事：

- 复用父 agent 已渲染好的 system prompt bytes
- 复用父 agent 的 exact tool array
- 把父 assistant 的全部 `tool_use` 留在前缀里
- 用统一 placeholder 的 `tool_result` 保证不同 fork child 之间前缀尽量一致

### 5. “后台 subagent” 的产品形态，本质上是任务系统 + sidechain transcript

从 `LocalAgentTask` 和 `resumeAgentBackground()` 看，后台 subagent 不是只有一段结果文本。

它还有：

- sidechain transcript
- output file
- task state
- progress tracker
- summarization
- task notification
- 可恢复执行

所以它更像一个被托管的后台 job，而不是一次普通 tool call。

## 先看地图

```text
模型调用 AgentTool
  |
  +-> fresh subagent ------------------------------+
  |                                                |
  +-> fork subagent ---------------------------+   |
  |                                            |   |
  +-> remote agent -------------------------+  |   |
  |                                         |  |   |
  +-> teammate spawn (另一套 team/swarm)    |  |   |
                                            v  v   v
                                         runAgent() / remote session
                                              |
                                              v
                           +---------------------------------------+
                           | Task system                           |
                           | - LocalAgentTask                      |
                           | - RemoteAgentTask                     |
                           | - output file / transcript            |
                           | - progress / notification / kill      |
                           +---------------------------------------+
                                              |
                                              v
                           SendMessage / TaskStop / TaskOutput / UI
```

## 必读源码入口

### Spawn 入口与 agent 定义

- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/forkSubagent.ts`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/loadAgentsDir.ts`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/builtInAgents.ts`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/prompt.ts`

### 子 agent runtime

- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/runAgent.ts`
- `../claude-code-sourcemap/restored-src/src/utils/forkedAgent.ts`
- `../claude-code-sourcemap/restored-src/src/utils/agentContext.ts`
- `../claude-code-sourcemap/restored-src/src/services/AgentSummary/agentSummary.ts`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/agentToolUtils.ts`

### Runtime task 系统

- `../claude-code-sourcemap/restored-src/src/Task.ts`
- `../claude-code-sourcemap/restored-src/src/tasks.ts`
- `../claude-code-sourcemap/restored-src/src/tasks/LocalAgentTask/LocalAgentTask.tsx`
- `../claude-code-sourcemap/restored-src/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`
- `../claude-code-sourcemap/restored-src/src/tasks/LocalMainSessionTask.ts`
- `../claude-code-sourcemap/restored-src/src/tasks/stopTask.ts`
- `../claude-code-sourcemap/restored-src/src/utils/task/framework.ts`

### 控制面

- `../claude-code-sourcemap/restored-src/src/tools/SendMessageTool/SendMessageTool.ts`
- `../claude-code-sourcemap/restored-src/src/tools/AgentTool/resumeAgent.ts`
- `../claude-code-sourcemap/restored-src/src/tools/TaskOutputTool/TaskOutputTool.tsx`
- `../claude-code-sourcemap/restored-src/src/tools/TaskStopTool/TaskStopTool.ts`

### Todo / task list

- `../claude-code-sourcemap/restored-src/src/tools/TaskCreateTool/TaskCreateTool.ts`
- `../claude-code-sourcemap/restored-src/src/tools/TaskListTool/TaskListTool.ts`
- `../claude-code-sourcemap/restored-src/src/tools/TaskGetTool/TaskGetTool.ts`
- `../claude-code-sourcemap/restored-src/src/tools/TaskUpdateTool/TaskUpdateTool.ts`
- `../claude-code-sourcemap/restored-src/src/utils/tasks.ts`

## 1. 先分清 4 个最容易混淆的边界

### 1.1 agent definition 不等于运行中的 subagent

`loadAgentsDir.ts` 里的 `AgentDefinition` 只是一份静态定义，里面描述：

- 这个 agent 叫什么
- 什么时候用
- 它有哪些工具
- 它的 prompt、model、permissionMode、skills、MCP、hooks、memory、isolation

真正运行中的子 agent，要到 `runAgent()` 里才会变成：

- 一份独立的 `ToolUseContext`
- 一组消息
- 一个 sidechain transcript
- 一个 runtime task

### 1.2 subagent 不等于 teammate

普通 subagent 走的是：

- `AgentTool -> runAgent -> LocalAgentTask / RemoteAgentTask`

teammate 走的是：

- `spawnTeammate()`
- `InProcessTeammateTask`
- mailbox / team roster / plan approval

teammate 可以再开 subagent，但它自己不是普通 subagent。

### 1.3 runtime task 不等于 Todo task

`Task.ts` 里的 `TaskType` 是执行期后台 job。

`TaskCreate/List/Get/Update` 操作的是任务列表。

一个是：

- “谁正在跑”

另一个是：

- “工作应该怎么拆”

### 1.4 fork 不等于 general-purpose fallback

`AgentTool.tsx` 里这段分支很关键：

- gate 关：省略 `subagent_type` => `general-purpose`
- gate 开：省略 `subagent_type` => fork path

也就是说，“不填 `subagent_type`” 在不同 feature 状态下语义不一样。

## 2. `AgentTool` 是 spawn 路由器，不是运行器

`AgentTool.call()` 里做的第一件事，不是直接跑 query，而是先判断：

- 这是 teammate spawn 吗
- 这是 fork path 吗
- 指定的 agent 是否存在
- MCP prerequisite 是否满足
- 这次是否要走 remote
- 这次是否必须 async
- 是否要建 worktree

大致流程可以压成：

```text
AgentTool.call()
  -> resolve selectedAgent / fork path
  -> check deny rules / MCP requirements
  -> decide isolation / async / background
  -> build promptMessages / systemPrompt strategy
  -> build worker tool pool
  -> optional worktree / remote registration
  -> runAgent(...)
```

从这个结构看，`AgentTool` 更像：

- 一个 delegation orchestrator

而不是：

- 子 agent 自己的 runtime

## 3. `runAgent()` 才是真正的子 agent runtime 入口

`runAgent.ts` 是这条链最关键的文件。

它做的事情可以分成 8 步。

### 3.1 先准备消息前缀

如果有 `forkContextMessages`，它会：

- 过滤不完整 tool calls
- 把父上下文消息作为前缀

然后和新的 `promptMessages` 拼成：

- `initialMessages`

### 3.2 再准备 user/system context

它会拿：

- `getUserContext()`
- `getSystemContext()`

但并不是所有 agent 都原样继承。

源码里专门做了两种 slim 化：

- `omitClaudeMd`
  - 某些 read-only agent 不带 `claudeMd`
- Explore / Plan
  - 会去掉 `gitStatus`

这说明 Claude Code 已经开始按 agent 类型裁剪“静态上下文税”。

### 3.3 再决定 agent 自己的 permission / effort / tools

`agentGetAppState()` 会重写一部分父 state：

- permissionMode
- shouldAvoidPermissionPrompts
- awaitAutomatedChecksBeforeDialog
- allowedTools
- effortValue

然后 `resolveAgentTools()` 再结合：

- built-in / custom 限制
- async agent 白名单
- disallowedTools
- agent frontmatter 的 allowlist

生成真正给子 agent 的工具池。

### 3.4 再建立 agent 自己的 system prompt

普通 subagent 路径会：

- 读取 agent 的 `getSystemPrompt()`
- 再做 env details 增强

fork path 则相反：

- 不重新渲染子 prompt
- 直接复用父 agent 已经渲染好的 `renderedSystemPrompt`

这是为了 prompt cache。

### 3.5 再注册 hooks / skills / agent-specific MCP

在真正进 query 之前，`runAgent()` 还会做：

- `executeSubagentStartHooks()`
- frontmatter hooks 注册
- frontmatter skills preload
- agent 专属 MCP server 初始化

这一步说明 agent definition 不只是一个 prompt 模板，它还是：

- runtime capability bundle

### 3.6 再通过 `createSubagentContext()` 建独立执行上下文

`createSubagentContext()` 是整套设计里最值得看的点之一。

默认情况下它会把所有“容易互相污染的可变状态”隔离出去：

- `readFileState`
- `contentReplacementState`
- `abortController`
- `nestedMemoryAttachmentTriggers`
- `toolDecisions`
- 多数 mutation callback

只在明确 opt-in 时共享：

- `setAppState`
- `setResponseLength`
- `abortController`

这就是为什么 Claude Code 能在同一进程里挂多个 agent，而不至于互相踩状态。

### 3.7 再把 transcript / metadata 落盘

在真正跑 query 前，它会 fire-and-forget 地写：

- `recordSidechainTranscript(initialMessages, agentId)`
- `writeAgentMetadata(agentId, ...)`

这为后面的：

- output file
- resume
- viewed transcript
- notification

都提供了基础。

### 3.8 最后才进入 `query()`

真正的子 agent query 调用，和主线程一样仍然是：

- `query({ messages, systemPrompt, userContext, systemContext, ... })`

也就是说，subagent 不是一套单独推理协议，它只是：

- 用另一份 context/options/messages 再跑一次同一个 agent loop

## 4. fork subagent 是专门为 cache sharing 做的特殊路径

`forkSubagent.ts` 基本把设计目的写在注释里了。

### 4.1 fork 的目标不是“专业化”

fresh subagent 的目标是：

- 换 agent prompt
- 换工具范围
- 换权限边界

fork 的目标是：

- 继承父 agent 的完整上下文
- 尽量复用 prompt cache

### 4.2 它为什么要保留完整 assistant message + placeholder tool_result

`buildForkedMessages()` 会：

- 保留父 assistant 的整条消息
- 收集里面全部 `tool_use`
- 给每个 `tool_use` 生成完全一致的 placeholder `tool_result`
- 最后再加当前 child 的 directive

这样做的目的不是可读性，而是：

- 让不同 fork child 之间的请求前缀尽量 byte-identical

### 4.3 它为什么强制 exact tools / inherited thinking config

`runAgent.ts` 在 fork path 下会：

- `availableTools: parent tools`
- `useExactTools: true`
- 继承父 `thinkingConfig`

原因很直接：

- tools
- system prompt
- messages prefix
- thinking config

都属于 cache-sensitive 参数。

fork 想共享缓存，就得尽量别改这些字节。

### 4.4 它为什么禁止递归 fork

fork child 会保留 `AgentTool`，但 `AgentTool.call()` 专门做了递归保护：

- 优先看 `querySource`
- 再回退到消息里是否存在 fork boilerplate

因为 Claude Code 想保留：

- cache-identical tool definitions

但不想真的让 fork worker 再无限分叉。

## 5. background subagent 会被挂成 `LocalAgentTask`

一旦决定异步执行，`AgentTool.tsx` 会：

- `registerAsyncAgent(...)`
- 然后把真正执行交给 `runAsyncAgentLifecycle(...)`

`LocalAgentTaskState` 里最关键的字段有：

- `agentId`
- `prompt`
- `agentType`
- `progress`
- `messages`
- `pendingMessages`
- `retain`
- `diskLoaded`
- `outputOffset`
- `notified`

这些字段说明它不是“任务已启动”这么简单，而是一个完整的后台运行对象。

### 5.1 output file 和 transcript 是任务系统的一部分

`LocalAgentTask` 会给每个 agent 绑定：

- output file path
- sidechain transcript path

所以后续不管是：

- 读输出
- 查看 transcript
- resume
- 发 completion notification

都不是临时拼出来的，而是基于这个 sidechain 存储层。

### 5.2 `runAsyncAgentLifecycle()` 才是后台 agent 的真正生命周期壳

它负责：

- 读 `runAgent()` 的 stream
- 累积 `agentMessages`
- 更新 progress tracker
- 触发 `AgentSummary`
- 完成后 `finalizeAgentTool()`
- 做 handoff classifier
- 发 task notification
- killed 时保留 partial result

也就是说，后台 agent 的“壳”并不在 `query()` 里，而在这个 lifecycle wrapper 里。

## 6. remote agent 不是另一种本地 task，而是一层远端 session 代理

如果 `isolation === "remote"`，`AgentTool.tsx` 会先走：

- `checkRemoteAgentEligibility()`
- `teleportToRemote(...)`
- `registerRemoteAgentTask(...)`

这里的本质变化是：

- 执行不再发生在本地 `runAgent()`
- 本地只保留一个 remote task 代理、轮询器和 output/notification 控制面

所以 `RemoteAgentTask` 的重点不在“跑 agent”，而在：

- 跟远端 session 对齐状态
- 轮询事件
- 把远端结果变成本地 task notification

## 7. `SendMessage` / `TaskStop` / `TaskOutput` 形成了后台 agent 的控制面

### 7.1 `SendMessage` 不是只给 swarm 用

`SendMessageTool.ts` 除了写 teammate mailbox，还能：

- 找到 `LocalAgentTask`
- 触发 `resumeAgentBackground()`
- 把新消息排进后台 agent 的 `pendingMessages`

这意味着 Claude Code 的后台 subagent 不是 fire-and-forget。

它是可以被继续 steering 的。

### 7.2 `TaskStop` 是统一 kill 接口

`TaskStopTool -> stopTask()` 会：

- 按 task id 查当前运行任务
- 找对应 `Task` 实现
- 调对应 kill

这层抽象让：

- local bash
- local agent
- remote agent

可以共用一套 stop 入口。

### 7.3 `TaskOutput` 已经是兼容层，不是推荐主路径

`TaskOutputTool.tsx` 顶部已经写得很清楚：

- Deprecated
- 更推荐直接 `Read` output file

这说明 Claude Code 当前更偏好的控制面是：

- 任务完成后给你一个 output path
- 需要时再用普通 Read 工具读取

而不是长期维护一个专门的“读任务输出 API”。

## 8. Todo 任务列表是另一套协调层

`TaskCreate/List/Get/Update` 背后操作的是：

- `utils/tasks.ts`

它表达的是：

- subject
- description
- status
- owner
- blockedBy
- blocks
- metadata

这套东西跟 background task 的关系是：

- 它不负责执行 runtime
- 它负责表达“工作怎么拆”和“谁在做什么”

这在 swarm / teammate 路径里尤其重要，因为 owner 和 blockedBy 可以被多个 agent 共享。

所以后面写正文时一定要单独点明：

- runtime task system = execution control plane
- Todo task system = planning / coordination layer

## 9. teammate swarm 是邻近系统，但不要和 subagent 混写

`InProcessTeammateTask` 说明还有另一条多 agent 路径：

- 同进程运行
- 有 team identity
- 有 mailbox
- 有 idle / active 状态
- 有 plan approval flow

这条线和普通 subagent 共享了很多底层能力：

- `runAgent()`
- task state
- progress

但它的产品语义完全不同：

- subagent 是“我委托一个 worker 做子任务”
- teammate 是“我加入一个 team runtime 做协作”

后续正文可以把 teammate 当成 subagent 的近邻系统，但最好不要写成同一件事。

## 后续正文最该强调什么

如果把这套源码转成 reader-facing tutorial，我认为最该强调的是：

1. `subagent` 不是单点功能，而是 “spawn API + child runtime + task control plane”
2. `Task` 这个词在 Claude Code 里有两层含义，必须先拆开
3. fork path 的真正动机是 prompt cache，而不是 agent specialization
4. background subagent 的真正产品形态是 sidechain transcript + notification
5. teammate 是相邻系统，不应拿来替代普通 subagent 解释

## 仍待验证的点

这轮主要还是基于 restored source。

还没做本机提取物或运行时级确认的点包括：

- remote agent 路径在当前已安装版本里的真实可达性
- fork subagent gate 在当前版本里的默认开启状态
- `SendMessage -> resumeAgentBackground()` 在复杂多轮恢复里的实际 transcript 表现
- task list 与 swarm owner/blocking 更新在真实会话里的 UI 联动
