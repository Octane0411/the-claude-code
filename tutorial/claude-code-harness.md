# Claude Code 是怎么构建 harness 的

## 问题不是“怎么调用模型”，而是“怎么让它持续工作”

讲 Claude Code 时，最容易把注意力放在三样东西上：

- system prompt
- tool schema
- agent loop

这些当然重要，但如果只把系统做成“模型 + 工具 + 一个循环”，很快就会撞上几个很现实的问题。

第一，工具输出会失控。  
一条 shell 命令、一次搜索、一次代码生成，很容易产生远超当前上下文窗口承受能力的结果。直接把这些内容塞回下一轮消息里，不但贵，而且会污染后续推理。

第二，模型需要一个稳定的工作面。  
如果 agent 在多轮交互里会读自己的计划、检查上一次生成的摘要、回看大型工具结果，那它需要的就不只是用户工作区，而是一组由 runtime 自己维护的内部状态文件。

第三，权限和宿主控制不能散在工具内部。  
本地模式、远端模式、UI 审批、SDK host 审批，本质上都在回答同一个问题：这次工具调用到底谁说了算。如果这个问题没有单独的一层来处理，工具系统很快会变得不可维护。

第四，有些信号根本不应该进模型上下文。  
例如插件安装提示、宿主级通知、控制事件。这些信息对产品有用，但如果直接暴露给模型，只会增加噪声，甚至改变行为。

我现在对 Claude Code 的理解是：  
**harness 就是为了解决这些问题而长出来的一层 runtime。**

在当前可见源码里，它不是一个单独的 `Harness.ts`，而是一组围绕 session、文件、工具输出、权限和控制协议搭起来的工程约束。

## naive 实现为什么会失效

先把对照组说清楚。

一个最朴素的 coding agent 通常长这样：

1. 收到用户输入
2. 调模型
3. 解析 `tool_use`
4. 执行工具
5. 把工具结果原样喂回模型
6. 循环直到模型停止

这个版本在短任务上能工作，但一旦任务变长，失败模式会很稳定。

### 失败模式一：工具结果没有归宿

如果工具输出很长，系统通常只有两个选择：

- 截断
- 强行塞回上下文

截断会丢信息。强行塞回上下文会让后续每一轮都背着历史包袱。

这类系统表面上看“支持工具”，实际上并没有给工具结果一个持久、可寻址、可延迟读取的落点。

### 失败模式二：模型没有自己的内部工作面

长任务里的 agent 不只需要读项目文件，还会不断生成中间产物：

- 本轮摘要
- 会话记忆
- 计划文件
- 测试输出
- 大型工具结果

如果这些东西只是临时字符串，而不是 runtime 管理的文件面，agent 每过几轮就得重新“猜”系统当前处于什么状态。

### 失败模式三：权限和审批逻辑会渗到每个工具

如果没有统一控制层，工具会开始各自处理：

- 能不能执行
- 谁批准
- UI 怎么提示
- 远端怎么回传

最后结果通常是本地模式能跑，远端模式另写一套，SDK host 又是第三套。

### 失败模式四：模型和宿主之间没有 side channel

有些信息应该给用户，不该给模型。

如果系统没有 side channel，工程上就只剩两条路：

- 信息彻底丢弃
- 信息进入模型上下文

这两种都不好。前者损失产品能力，后者污染推理环境。

## Claude Code 的做法：先搭一个 session-local runtime

Claude Code 当前源码最有意思的一点，是它并没有把这些问题分散留给各个工具自己解决，而是先搭了一层 session runtime，再让 query loop 在上面工作。

这个思路在 `src/setup.ts` 很明显。

`setup()` 做的不只是启动参数处理。它会在首轮 query 之前注册或预热一批 runtime 设施，包括：

- `initSessionMemory()`
- plugin hooks 的预加载
- commands 的预取
- 某些 background feature 的初始化

这意味着 Claude Code 不是“等模型第一次需要某个能力时再临时创建环境”，而是先把运行时壳搭好，再进入主循环。

这类设计有个很实际的好处：后面的工具执行、权限协商、状态落盘，都能建立在一个已经存在的 session world 上，而不是每轮现拼。

## 关键设计一：给 agent 一组内部路径，而不只是一个工作区

在 Claude Code 里，模型不是只面对用户项目目录。

源码里可以看到几类明确的内部路径：

- `session-memory`
- `plans`
- `tool-results`
- `auto memory`
- `agent memory`
- `scratchpad`
- project temp

这不是实现细节，而是 harness 的核心。

例如，`src/utils/toolResultStorage.ts` 会把 session 目录组织成 `projectDir/sessionId/tool-results`。  
`src/services/SessionMemory/sessionMemory.ts` 会创建 `session-memory/summary.md`。  
`src/memdir/memdir.ts` 甚至直接写了注释，说明 harness 会保证 memory 目录已存在，模型可以直接写。

这几个点放在一起看，信号很强：

**Claude Code 不只是让模型“能读写文件”，而是先定义了一组 runtime 自己维护的内部文件面。**

这组文件面解决了两个问题。

第一，它给长任务一个稳定的中间状态承载层。  
第二，它让权限系统有机会区分“用户项目内容”和“runtime 自己的产物”。

## 关键设计二：把工具输出变成受控 artifact，而不是一段大字符串

这一点在 `src/tools/BashTool/BashTool.tsx` 里最直观。

当 shell 输出过大时，Claude Code 并不尝试把完整内容继续塞进消息里。它会：

1. 把输出持久化到 `tool-results/`
2. 给当前回合保留一个较小的预览
3. 把完整结果留成后续可读取文件

这其实是在把“工具结果”从一次性文本，改造成 session 内的 artifact。

这一步很关键，因为它改变了模型和工具结果之间的关系：

- 在 naive 版本里，工具结果只能“现在就看”
- 在 Claude Code 里，工具结果可以“先落盘，后续再按需读”

这不只是 token 优化。它直接影响 agent 能不能在长任务里维持工作连续性。

同样的思路也出现在 `src/utils/mcpOutputStorage.ts` 里。MCP 返回过大的文本或二进制内容时，系统也倾向于先落成文件，再告诉模型如何顺序读取。

也就是说，Claude Code 的 harness 在这里承担的是 artifact management，不是简单 I/O 包装。

## 关键设计三：有些输出只给宿主，不给模型

`src/utils/claudeCodeHints.ts` 很能说明 Claude Code 的工程取向。

这里定义了一种 `<claude-code-hint />` 协议：CLI 或 SDK 可以在 stderr 发一个自闭合标签，shell 工具会扫描输出，把这个标签剥离掉，然后只把提示信息交给 UI 侧使用。

注释里说得很直白：  
这是 a harness-only side channel。

这个设计解决的是前面提到的第四种失败模式。

系统终于有了第三条路：

- 不是把提示丢掉
- 也不是让模型看到提示
- 而是通过 harness 拦截并转交给宿主

这种设计看起来不起眼，但它说明 Claude Code 已经明确把“模型上下文”和“产品级宿主信号”分成了两条通道。

这是成熟 agent runtime 和 demo agent 的一个明显分水岭。

## 关键设计四：权限系统认识这些内部路径

只有内部路径还不够。系统还得知道这些路径和普通用户文件不一样。

`src/utils/permissions/filesystem.ts` 的处理方式很有代表性。权限判断里专门有一段 internal path 逻辑，会对下列内容直接放行读取：

- `session-memory`
- `plans`
- `tool-results`
- `scratchpad`
- project temp
- `agent memory`
- `auto memory`
- `tasks`
- `teams`

这说明 Claude Code 的权限模型不是简单的“工作区内允许，工作区外询问”。

它实际上至少区分了三层东西：

1. 用户工作区
2. harness 自己维护的内部状态路径
3. 更外部的系统路径

这层区分非常值钱。

如果没有它，模型想读上轮的大工具结果时，要么被迫重新运行工具，要么被权限系统错误拦截。两种都会把长任务体验拉坏。

Claude Code 在这里做的事很朴素，但很有效：

**先定义一批内部 artifact 路径，再让权限系统显式认识它们。**

## 关键设计五：harness 还是一个 control plane

如果只看到文件和工具结果，你会把 harness 理解成 I/O 层。Claude Code 比这更进一步。

`src/cli/structuredIO.ts` 里可以看到，权限协商是通过 `control_request` / `control_response` 跟宿主交互的。也就是说，工具是否可用，不是工具自己最终拍板，而是 runtime 把请求送到外部控制层，再等答复。

远端模式下，这一点更明显。

`src/utils/teleport.tsx` 在创建远端 session 时，会把 `set_permission_mode` 当作初始事件预先写进去。这样远端 CLI 在第一轮用户消息到达之前，就已经处在正确的权限模式里。

这背后的设计很清楚：

- query loop 负责跑任务
- harness 负责跟宿主同步控制状态

如果没有这一层，本地 CLI、远端容器、Web UI、SDK host 很快就会各有一套审批逻辑。

而一旦把这些行为都抽到 control plane 里，系统才有机会在不同宿主之间共享一套运行时契约。

## 这套设计带来了什么

把前面几层放在一起看，Claude Code 的 harness 至少换来了四件事。

### 1. 工具结果终于有了生命周期

它们不再只是模型看到的一段文本，而是可以落盘、回读、跨轮引用的 session artifact。

### 2. 长任务有了更稳定的中间状态面

`session-memory`、`plans`、`tool-results` 这类内部路径，让 agent 不必每次都从聊天历史里重新恢复全部状态。

### 3. 宿主信号不再污染模型上下文

像 `claude-code-hint` 这样的协议，让产品能力和模型上下文可以解耦。

### 4. 本地与远端至少有机会共享同一套控制语义

权限模式、审批请求、控制消息都有了一个明确的 runtime 承载层。

## 代价也很明显

这套设计不是免费的。

第一，系统复杂度会上升。  
一旦你引入内部路径、artifact 管理、控制协议和 side channel，调试成本会明显增加。

第二，恢复和迁移会变难。  
只要 session 可以 resume、fork、切远端，`tool-results`、plan 文件、memory 文件到底怎么复制和引用，就会变成真实工程问题。

第三，权限模型会更绕。  
“什么是工作区内文件，什么是 harness 内部文件，什么是完全外部路径” 这种区分虽然必要，但会让规则体系更复杂。

第四，local / remote 差异更难完全藏住。  
源码里已经能看到不少 `CLAUDE_CODE_REMOTE` 相关逻辑。只要运行时跨宿主，这层差异迟早会冒出来。

所以从工程角度看，harness 的价值和代价是一体两面：

**它解决的是长任务 runtime 的稳定性问题，而不是便宜地加几个功能。**

## 如果让我自己实现，我会把这层显式收束

Claude Code 当前源码里的做法是“功能上已经形成 harness，但代码结构上仍然分散”。

这很正常，因为产品通常是渐进长出来的。但如果要做一个更干净的实现，我不会让 harness 只以隐含概念存在。

我会把它显式收束成一个 runtime 对象，至少包含：

- session 目录与内部路径注册
- tool artifact 持久化
- output rewriting
- permission gate
- host control channel

原因很简单。Claude Code 现在这篇源码最有价值的启发，不是“某个函数怎么写”，而是：

**当 agent 开始跨多轮、多工具、多宿主持续工作时，必须有一层代码专门负责管理模型和外部世界之间的边界。**

那层代码，就是 harness。

## 我现在对 Claude Code harness 的定义

看完这些实现后，我会把 Claude Code 的 harness 定义成下面这句话：

**它是一层 session-scoped runtime，负责给模型提供受控的工作面，把工具结果转成可管理的 artifact，维护模型不可见的宿主侧信号，并在本地与远端宿主之间同步控制语义。**

如果只说“Claude Code 会调用工具”，这个判断远远不够。

更准确的说法应该是：

**Claude Code 让模型运行在一个经过工程化处理的 harness 里。模型并不是直接碰世界，而是在操作 harness 先整理过的世界。**

## 关键源码入口

- `src/setup.ts`
- `src/utils/toolResultStorage.ts`
- `src/tools/BashTool/BashTool.tsx`
- `src/utils/mcpOutputStorage.ts`
- `src/utils/claudeCodeHints.ts`
- `src/utils/permissions/filesystem.ts`
- `src/services/SessionMemory/sessionMemory.ts`
- `src/memdir/memdir.ts`
- `src/cli/structuredIO.ts`
- `src/utils/teleport.tsx`

## 还需要验证的问题

这篇文章里的判断主要来自 restored source。还有几件事最好后续拿真实运行时验证：

- 远端 CCR 模式下内部目录的真实挂载布局
- `tool-results` 在 resume / fork 场景下的复制边界
- `claude-code-hint` 在已安装客户端中的完整 UI 行为

## 版本假设

- 本文基于 `../claude-code-sourcemap/restored-src/src` 在 2026-03-31 可见的还原源码
- 文中的“看起来如何工作”以当前源码为准
- 涉及已安装客户端的运行时差异，仍建议结合 `../extracts/` 或真实运行观察交叉验证
