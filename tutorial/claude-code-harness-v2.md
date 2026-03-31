# Harness：把模型装进一个经过工程化处理的世界

大多数人第一次写 coding agent 时，思路差不多都是这样的：拿一个模型，给它几个工具，写一个循环，让它不断调用工具直到任务完成。这个方案能跑通短任务，但一旦任务时间拉长——超过十几轮工具调用，或者涉及大量文件读写——你会发现系统开始以一些不太起眼但很顽固的方式坏掉。

读 Claude Code 的源码之前，我以为它的核心竞争力在 system prompt 和工具设计上。读完之后我意识到，真正区分它和那些玩具 agent 的，是一层不太起眼的东西：它在模型和真实世界之间插了一个精心设计的 runtime，决定模型能看到什么、能写到哪里、哪些工具结果会被改写后再回流、哪些信号压根不该进入上下文。

这层 runtime 就是 harness。

## 问题出在哪

先不看 Claude Code，先想想一个 naive 的 agent 在长任务上会怎么死。

**工具输出会把上下文撑爆。** 一条 `grep` 返回几万行，一次 `cat` 吐出整个文件——如果你把这些原样塞回下一轮消息，上下文窗口几轮就满了。截断？你丢了信息。不截断？后续每一轮都背着历史垃圾推理。

**模型没有自己的"桌面"。** 它生成了一份计划、跑了一组测试、看了一段很长的输出——这些中间产物如果只是聊天历史里的一段文本，那几轮之后它就会忘记系统当前处于什么状态。人类工程师有文件系统、笔记、TODO 列表；模型什么都没有。

**权限逻辑会渗透到每个工具里。** 这个 shell 命令能不能跑？那个文件能不能写？用户批准了还是拒绝了？如果每个工具各自处理这些问题，你很快就会发现本地模式能跑的东西，换到远端或 SDK 模式就炸了。

**有些信号根本不应该让模型看到。** 比如一个 CLI 工具在 stderr 里输出了一个"建议安装某个插件"的提示。这条信息对用户有用，但如果模型看到了，它可能会试图"帮忙安装"——或者更糟，把这条提示当成工具执行失败的信号，改变整个推理方向。

这四个问题不是边缘情况，它们是任何认真做长任务 agent 都绕不开的工程现实。Claude Code 的源码里，harness 本质上就是对这四个问题的系统性回应。

## 先搭壳，再进循环

Claude Code 不是在第一次调用模型时才开始拼环境。

`setup.ts` 在首轮 query 之前做了大量工作：注册 session memory 的 post-sampling hook、初始化文件变更监听、创建 worktree（如果需要）、启动进程间通信的 Unix Domain Socket、锁定当前二进制版本防止被 GC、预取插件 hooks 和 API key——甚至还会检查 `--dangerously-skip-permissions` 是否在非沙箱环境下被误用（如果是 Anthropic 员工，还会并行检查 Docker 和网络隔离）。

这跟"先读用户输入再决定需要什么"完全是两种思路。Claude Code 选择先把 runtime 壳搭好，让后续的 query loop 运行在一个已经准备就绪的环境上。这意味着工具执行、权限协商、状态落盘，都不需要每轮现拼。

从工程上看，这个选择的好处很朴素：它把"环境准备"和"任务执行"解耦了。你不会看到某个工具在第一次被调用时才慌忙创建临时目录，也不会看到权限系统在第五轮才发现自己还没初始化。

## 给模型一组 runtime 管理的内部路径

这是 harness 里最关键的一个设计决定。

Claude Code 的模型不是只面对用户的项目目录。在 `utils/permissions/filesystem.ts` 里，你能看到权限系统对一批路径做了特殊处理——它们不需要用户批准就可以读写：

读取自动放行的路径包括 `session-memory/`、`tool-results/`、`plans/`、`scratchpad/`，以及整个项目级 `.claude/` 目录。写入自动放行的路径包括 plan 文件、scratchpad、agent memory、auto memory。

这不是偷懒省掉了权限弹窗。这是一个有意识的架构决定：**harness 为模型维护了一组和用户工作区平行的内部路径，并且让权限系统显式地认识它们。**

`memdir/memdir.ts` 里有个很能说明问题的细节：`ensureMemoryDirExists` 会在 agent 开始工作前就把 memory 目录创建好。系统 prompt 里甚至会写"这个目录已经存在，你可以直接写"。这本质上是把文件系统的前置条件变成了 runtime 契约——模型不需要先 `mkdir`，因为 harness 已经保证了这个目录存在。

这层路径划分解决了两个实际问题。第一，模型写中间状态（计划、摘要、大工具结果引用）时不会触发权限提示，长任务不会被频繁打断。第二，权限系统对"用户文件"和"runtime 自己的产物"有了明确边界，安全模型更清晰。

`memdir/paths.ts` 里还有一个值得注意的安全处理：auto memory 的路径配置刻意排除了 `.claude/settings.json`（仓库级配置）。原因很实际——如果一个恶意仓库在自己的 settings 里把 `autoMemoryDirectory` 指向 `~/.ssh`，模型就能通过 auto-allow 的权限通道直接写到 SSH 密钥目录。排除仓库级配置堵住了这个攻击面。

## 把工具输出变成可管理的 artifact

这一步很关键，因为它改变了模型和工具结果之间的基本关系。

在 naive 实现里，工具结果只能"现在就看"——它作为一段文本存在于消息历史中，要么全量保留（撑爆上下文），要么截断丢弃（丢失信息）。Claude Code 加了第三条路。

`utils/toolResultStorage.ts` 做的事情很直接：当工具输出超过一个阈值（比如 `BashTool` 的阈值是 30,000 字符），系统把完整输出写到 `{projectDir}/{sessionId}/tool-results/{toolUseId}.txt`，然后在消息里只留一个 2KB 的预览加上文件路径，用 `<persisted-output>` 标签包裹。模型后续想看完整内容，可以用 Read 工具去读那个文件。

写入用了 `flag: 'wx'`（O_CREAT|O_EXCL），保证幂等——如果在 compact 重放时文件已存在，不会重写。这是个小细节，但说明这套机制是考虑过 session 生命周期的。

更进一步，`applyToolResultBudget` 会在每轮 API 调用前扫描整个消息历史，把老旧的工具结果替换成 `[Old tool result content cleared]`。被替换的内容会记录到 transcript 里，保证 resume 时能恢复。这是一个主动的 token 预算管理机制，不是被动等上下文满了再 compact。

对于特别大的 shell 输出（超过 64MB），`BashTool` 甚至不做文件拷贝，而是用 hard link 加 truncate，避免在磁盘上复制巨量数据。

这套设计的效果是：工具结果从一次性的文本变成了 session 内的持久化 artifact。它们有路径、有生命周期、有预算管理，agent 可以在后续任何一轮按需回读。这不只是省 token，它直接决定了 agent 能不能在长任务里维持工作连续性。

## 有些输出只给宿主，不给模型

`utils/claudeCodeHints.ts` 实现了一个很小但很能说明设计意图的机制。

运行在 Claude Code 下的 CLI 工具可以在 stderr 里输出一个自闭合 XML 标签：`<claude-code-hint v="1" type="plugin" value="some-package@marketplace" />`。BashTool 在处理 shell 输出时会用正则扫描这些标签，把它们从模型可见的输出中剥离，然后把提取出的 hint 传给 UI 层显示为插件安装提示。

实现上很克制：先检查 `output.includes('<claude-code-hint')`，不包含就跳过所有正则处理。每个 session 最多显示一次提示。用 `useSyncExternalStore` 兼容的订阅模式暴露给 React 组件。

这个机制解决的问题很具体：系统需要一种方式把信息传达给用户而不传达给模型。没有 side channel 的情况下，你只有两个选择——要么丢掉这条信息，要么让模型看到。前者损失产品能力，后者污染推理环境。Claude Code 在这里选了第三条路：harness 拦截、转交、剥离。

这个模式看起来不大，但它标志着一个很重要的架构分水岭：系统开始把"模型上下文"和"产品级宿主信号"当成两条独立的通道来处理。

## Harness 同时也是 control plane

权限协商是 harness 里最不显眼但最有架构价值的一部分。

`cli/structuredIO.ts` 实现了一个 JSON-over-stdin/stdout 的控制协议。当工具需要权限时，系统不是在工具内部弹窗，而是往 stdout 写一个 `{ type: 'control_request', request_id, request }` 消息，然后等 stdin 上对应 `request_id` 的 `control_response` 回来。

这意味着"谁批准工具调用"这个问题被完全解耦出了工具本身。本地 CLI 模式下，是终端 UI 处理这个请求；SDK 模式下，是宿主程序处理；claude.ai 的 Web 模式下，`injectControlResponse` 允许 bridge 直接注入应答。同一套工具代码，不需要知道自己运行在哪种环境里。

`utils/teleport.tsx` 里有一个具体的例子。创建远端（CCR）session 时，系统把 `set_permission_mode` 作为 `control_request` 注入到 session 的初始事件里。这些事件会在容器的 CLI 连接之前就写入 threadstore，所以远端 CLI 在处理第一条用户消息之前就已经处在正确的权限模式里。注释里写得很明确：这是为了避免 readiness race。

把控制逻辑收到一层统一的协议里，换来的是本地、远端、SDK 宿主可以共享同一套运行时契约。没有这层抽象，三种模式的权限逻辑迟早会分叉成三套独立实现。

## Session memory：后台运行的状态提取器

`services/SessionMemory/sessionMemory.ts` 实现了一个有意思的机制：它在 `setup.ts` 里注册为 post-sampling hook，在模型每次采样完成后检查是否满足触发条件（token 阈值 + 工具调用次数阈值都满足，或者最近一轮 assistant 回复没有工具调用）。条件满足时，它 fork 一个子 agent，给这个子 agent 一个极度受限的工具集——只允许 `FileEditTool` 操作一个特定文件：`{sessionId}/session-memory/summary.md`。

子 agent 的任务是把当前会话的关键状态提取成一份结构化摘要。这份摘要写在 session-memory 目录下，模型在后续轮次可以读取它来恢复对当前工作状态的理解。

触发阈值通过 GrowthBook feature flag 远程控制，只在主线程（`repl_main_thread`）运行，不在子 agent 或 teammate 里运行。整个过程对用户不可见。

这不是一个花哨的功能。它解决的是一个很实际的问题：在长任务中，上下文压缩（compact）会丢失细节，但如果不压缩，上下文又会爆。session memory 提供了一个折中——让一个独立的 agent 把关键信息萃取到文件里，主 agent 可以在需要时回读，不占用主上下文窗口。

## 这套设计换来了什么

把上面几层放在一起看，Claude Code 的 harness 解决的其实是一个统一的问题：**当 agent 需要跨多轮、多工具、多宿主持续工作时，模型和真实世界之间需要一层代码专门管理这个边界。**

具体来说：

工具结果有了生命周期。它们不再是消息历史里的一段文本，而是可以落盘、回读、按预算淘汰的 session artifact。

长任务有了稳定的中间状态面。session-memory、plans、scratchpad 这些内部路径让 agent 不必每次都从聊天历史里重建全部状态。

产品信号和模型上下文解耦了。hint 协议让"给用户看"和"给模型看"成为两件独立的事。

控制逻辑有了单一的协商层。不管你在本地跑、远端跑还是通过 SDK 跑，权限请求走的是同一套 `control_request` / `control_response` 协议。

## 代价

这套设计的代价也很直接。

复杂度上升。一旦你有了内部路径、artifact 管理、side channel 和控制协议，调试一个"工具结果为什么没正确回流"的问题就不再是看一行代码能解决的。

恢复和迁移变难。session 可以 resume、可以 fork、可以切到远端——那 `tool-results/` 里的文件怎么办？plan 文件呢？memory 文件呢？这些都是真实的工程问题，源码里能看到不少处理这类边界情况的代码。

权限模型更绕。"用户项目文件 / harness 内部文件 / 系统外部路径"三层区分虽然必要，但规则体系确实更复杂了。`memdir/paths.ts` 里排除仓库级配置来防止路径投毒，就是这种复杂度的一个缩影。

但从工程角度看，这些代价和收益是一体的。Harness 解决的不是"怎么多加几个功能"，而是"怎么让 agent 在长任务上不散架"。如果你的 agent 只需要跑三五轮工具调用，你可能不需要这些。但一旦任务时间拉长，这层 runtime 的缺席会以各种不起眼的方式让系统行为退化。

## 如果自己实现

Claude Code 当前源码里的 harness 是渐进长出来的——职责分散在 `setup.ts`、`toolResultStorage.ts`、`filesystem.ts`、`structuredIO.ts` 等多个文件里，没有一个统一的 `Harness` 类。这很正常，产品代码通常都是这样演进的。

但如果从头做一个干净实现，我会把 harness 显式收束成一个 runtime 对象，至少包含四层：

1. **Session 目录与内部路径注册。** 启动时创建 session 级别的目录结构，注册哪些路径是"内部的"，告知权限系统。
2. **工具输出持久化与预算管理。** 大输出落盘、生成预览引用、按 token 预算淘汰历史结果。
3. **权限网关。** 统一的 `canUseTool` 判断层，区分内部路径和用户路径，对接不同宿主的审批方式。
4. **宿主控制协议。** 一套和具体宿主无关的 `request` / `response` 消息协议，让本地 CLI、远端容器和 SDK 宿主共享同一套权限和控制语义。

Side channel（hint 剥离）、session memory（后台状态提取）、context compaction 这些可以作为后续扩展点加上去。但上面四层是基础——没有它们，agent loop 在长任务上的稳定性很难保证。

## 关键源码入口

- `src/setup.ts` — runtime 启动编排
- `src/utils/toolResultStorage.ts` — 工具输出持久化与 token 预算
- `src/tools/BashTool/BashTool.tsx` — 大输出处理的具体实现
- `src/utils/claudeCodeHints.ts` — stderr side channel
- `src/utils/permissions/filesystem.ts` — 内部路径的权限自动放行
- `src/services/SessionMemory/sessionMemory.ts` — 后台记忆提取
- `src/memdir/memdir.ts` + `paths.ts` — auto memory 路径管理与安全校验
- `src/cli/structuredIO.ts` — SDK 控制协议
- `src/utils/teleport.tsx` — 远端 session 权限模式注入

## 版本说明

本文基于 `claude-code-sourcemap/restored-src/src`（从 npm 包 2.1.88 的 sourcemap 还原）在 2026-03-31 可见的代码。涉及运行时行为的判断，建议结合实际客户端观察交叉验证。
