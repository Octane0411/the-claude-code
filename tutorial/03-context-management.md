# 03. 上下文管理：模型每次到底看到了什么

## 这篇文章要回答什么

上一章我们已经把 Claude Code 的核心抓成了：

- 一个持续运行的 `query()` agent loop

但只知道“它会进 loop”还不够。

真正决定模型这一轮怎么判断、怎么选工具、怎么继续工作的，是另一个更具体的问题：

**在调用模型之前，Claude Code 到底把哪些东西放进了上下文？**

很多人做 agent CLI 时，会把这个问题理解成下面三种之一：

- 把消息历史原样发给模型
- 把一大段仓库说明塞进 system prompt
- 做一个 repo summary，希望模型自己补全剩下的细节

Claude Code 不是这么做的。

它把“这一轮固定会带进去的内容”拆成了几层，而且这些层的职责很清楚：

- system prompt 负责“你应该怎么工作”
- user context 负责“这个用户 / 项目额外要求你怎么工作”
- system context 负责“当前会话开始时，外部环境大概是什么样”

这一章要讲清楚的就是这三层。

## 先给结论

Claude Code 并不会把“整个项目的一切信息”都塞进模型上下文。

它更像是在每轮 query 前组一个固定输入包：

```text
messages
+ systemPrompt
+ userContext
+ systemContext
+ toolUseContext
```

其中这一章真正关心的是前 3 个静态注入层：

1. `systemPrompt`
   - Claude Code 的基础行为规则、环境信息、工具使用规则、MCP 指令、scratchpad 规则等
2. `userContext`
   - `CLAUDE.md` 家族文件，加上当前日期
3. `systemContext`
   - 当前会话开始时的一份 git snapshot，外加少量系统级注入信息

更重要的是：

- 这一章只讲“每轮固定会被带进去的内容”
- 不讲动态相关记忆召回
- 不讲 session memory
- 也不讲上下文压缩

如果把这些全混在一起，你会很快把 Claude Code 误读成一个“把所有材料都拼进 prompt 的大拼接器”。

## 先看地图

如果你现在只想先抓住大图，可以先把 Claude Code 的上下文管理想成这样：

```text
启动后
  -> 后台预取一部分静态上下文

用户提交输入
  -> REPL 并行准备：
       - getSystemPrompt(...)
       - getUserContext()
       - getSystemContext()
  -> buildEffectiveSystemPrompt(...)
  -> 连同 messages / toolUseContext 一起传进 query()
```

再把它拆细一点，就是：

```text
                        +-----------------------------+
                        | getSystemPrompt(...)        |
                        | 基础行为规则 / env / MCP    |
                        +-----------------------------+
                                      |
                                      v
                        +-----------------------------+
                        | buildEffectiveSystemPrompt  |
                        | agent/custom/append 合成    |
                        +-----------------------------+
                                      |
                                      v
messages --------------------------> query(...)
                                      ^
                                      |
                 +-------------------------------------------+
                 |                                           |
                 v                                           v
   +-----------------------------+           +-----------------------------+
   | getUserContext()            |           | getSystemContext()          |
   | CLAUDE.md 家族 + 当前日期   |           | git snapshot + cacheBreaker |
   +-----------------------------+           +-----------------------------+
```

这张图里最重要的判断是：

- `systemPrompt` 不是项目知识
- `userContext` 不是消息历史
- `systemContext` 不是实时环境同步

它们都是“静态注入层”，但不是同一种东西。

## 再看一条主流程

在 `REPL.tsx` 里，真正调用 `query()` 之前，大致会经过下面这条路径：

```text
onQueryImpl()
  -> Promise.all(
       getSystemPrompt(...),
       getUserContext(),
       getSystemContext()
     )
  -> buildEffectiveSystemPrompt(...)
  -> query({
       messages,
       systemPrompt,
       userContext,
       systemContext,
       toolUseContext,
     })
```

另外还有一个容易被忽略的点：

- `getUserContext()` 是 memoized 的
- `getSystemContext()` 也是 memoized 的

也就是说，这两层不是每个 turn 都重新从头算一遍。

它们更像：

- “在当前会话里缓存的一份静态上下文快照”

这对首轮延迟和 prompt cache 都很重要。

## 先记住四个容易混淆的边界

### 1. `systemPrompt` 不等于 `userContext`

`systemPrompt` 主要回答的是：

- 你应该怎么工作

`userContext` 主要回答的是：

- 这个用户 / 这个项目额外要求你怎么工作

这两者都可能包含“规则”，但来源和生命周期不一样。

### 2. `Context` 不等于 `Memory`

这一章讲的是：

- 每轮固定静态注入

后面 `Memory` 章节才会讲：

- 动态相关记忆召回
- session memory
- durable extraction

### 3. `systemContext` 不等于“实时仓库状态”

Claude Code 注入的是：

- 会话开始时的一份 git snapshot

不是：

- 每一轮都重新抓一份最新 git 状态

### 4. 被注入进上下文，不等于“模型已经读过代码”

`CLAUDE.md`、git snapshot、system prompt 只是工作背景。

它们不能替代：

- Read 工具真正读取文件
- 搜索代码
- 检查当前实现

这也是为什么 Claude Code 仍然强依赖工具，而不是只靠超长 prompt。

## 先抓源码入口

如果你想顺着源码把这条链走一遍，最值得先看的就是这几个文件：

- `src/screens/REPL.tsx`
  - query 前怎么组装上下文
- `src/constants/prompts.ts`
  - 默认 system prompt 里到底放了哪些 section
- `src/utils/systemPrompt.ts`
  - 最终 system prompt 的合成优先级
- `src/context.ts`
  - `getUserContext()` 和 `getSystemContext()`
- `src/utils/claudemd.ts`
  - `CLAUDE.md` 家族文件的发现、排序和过滤规则
- `src/main.tsx`
  - 启动后如何做 deferred prefetch，以及为什么 `systemContext` 受 trust gate 约束

这几个文件合起来，基本就能把“模型这一轮固定看到什么”讲完整。

## 第 1 层：`getSystemPrompt()` 负责基础行为规则

从结构上说，Claude Code 最外层先准备的不是项目说明，而是 system prompt。

对应入口在：

- `src/constants/prompts.ts`

`getSystemPrompt(...)` 返回的不是一整块写死字符串，而是一组 section。

从当前源码看，里面主要包括：

- Claude Code 的基础身份和工作方式
- tool 使用规则
- 会话 guidance
- memory 使用规则
- 环境信息
- 语言与输出风格
- MCP instructions
- scratchpad instructions
- function result clearing
- summarize tool results

最值得注意的是：

**这里放的是“工作协议”，不是“项目事实”。**

这是一条很重要的设计边界。

如果你把项目说明也全部塞进 system prompt，会很快遇到两个问题：

- system prompt 膨胀得太快
- 行为规则和项目材料混在一起，后面很难按来源更新

## 第 2 层：`buildEffectiveSystemPrompt()` 负责最终组装

`getSystemPrompt(...)` 还不是最终发给模型的版本。

在：

- `src/utils/systemPrompt.ts`

里，Claude Code 还会再走一次显式组装：

```text
override
  -> coordinator
  -> agent
  -> custom
  -> default
  -> append
```

更准确地说：

- 如果有 `overrideSystemPrompt`，它直接替换全部
- 如果在 coordinator mode，就走 coordinator prompt
- 如果当前主线程 agent 自带 prompt，就用 agent prompt
- 否则才考虑 custom / default
- `appendSystemPrompt` 再统一挂到最后

这层非常值得抄。

因为它解决的是一个常见工程问题：

- prompt 不会永远只有一种来源

只要你后面想支持：

- custom agent
- coordinator mode
- `--system-prompt`
- runtime append prompt

你迟早都需要一个独立的“prompt 合成层”。

## 第 3 层：`getUserContext()` 负责注入 `CLAUDE.md` 家族

真正和“Claude Code 会读哪些 instruction files”直接相关的，是：

- `src/context.ts`
- `src/utils/claudemd.ts`

`getUserContext()` 的返回值其实很克制，当前只有两样：

- `claudeMd`
- `currentDate`

也就是说，Claude Code 的 user context 不是：

- 项目全量摘要
- 文件树索引
- 自动生成的 repo knowledge base

而是：

- 一组显式 instruction files
- 一个当前日期字符串

这一点非常重要。

它说明 Claude Code 默认更信任：

- 用户明确写下来的规则

而不是：

- 系统替用户“猜出来”的项目总结

### `getUserContext()` 什么时候会直接跳过

源码里有两个显式 gate：

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`
- `--bare` 且没有 `--add-dir`

这说明 Claude Code 对 instruction files 的态度是：

- 默认自动发现
- 但用户可以明确关掉
- bare mode 下也不会偷偷替你扫项目

### `CLAUDE.md` 不是单文件，而是一套加载规则

`claudemd.ts` 顶部注释已经把发现顺序写得很清楚：

1. `Managed`
2. `User`
3. 从 root 到当前工作目录逐层发现的 `Project`
4. 同路径上的 `Local`
5. 可选的 `--add-dir`
6. memdir 的 `MEMORY.md`
7. team memory 的 `MEMORY.md`

而且 Project 层不只是 `CLAUDE.md` 一个文件，还包括：

- `.claude/CLAUDE.md`
- `.claude/rules/*.md`

其中离当前工作目录更近的规则，优先级更高。

你可以把这套设计理解成：

- 全局规则
- 用户私有规则
- 仓库共享规则
- 当前项目本地私有规则

四类来源被显式拆开了。

这比“一份无限膨胀的全局提示词”更可追溯，也更适合团队协作。

### `filterInjectedMemoryFiles()` 暗示了 Context 和 Memory 正在继续拆开

还有一个很值得注意的细节。

在 `getUserContext()` 里，`getMemoryFiles()` 的结果不会直接全部注入，而是先经过：

- `filterInjectedMemoryFiles(...)`

这个过滤背后的关键开关是：

- `tengu_moth_copse`

当它打开时：

- `AutoMem`
- `TeamMem`

这些记忆入口就不再作为静态注入文件直接进 prompt。

它们会改走后面的动态召回链。

这件事透露出的信号很明确：

**Claude Code 自己也在把“静态指令”与“动态记忆”继续拆开。**

所以这章只讲 `Context`，不把 `Memory` 硬塞进来，是符合源码演进方向的。

## 第 4 层：`getSystemContext()` 负责会话级系统快照

`src/context.ts` 里的另一条链是：

- `getSystemContext()`

它和 `getUserContext()` 不是一回事。

`getSystemContext()` 处理的不是 instruction files，而是系统级快照。

当前最核心的内容是：

- current branch
- default branch
- `git status --short`
- recent commits
- 可选的 `cacheBreaker`

其中 `git status` 还有一个很重要的限制：

- 超过 2000 个字符会被截断
- 并明确提示模型，如果还想看更多，请用 BashTool 自己跑 `git status`

这也是很成熟的做法。

它说明 Claude Code 不会因为“想多给一点上下文”就把 prompt 无限撑大。

### 最关键的语义：这是一份 snapshot

源码返回的文案已经直接写明了：

- 这是会话开始时的 git status snapshot
- 它在对话过程中不会自动更新

这点必须记住。

Claude Code 不是想维护一份“永远最新的 repo 镜像 prompt”。

它只是给模型一个：

- 当前会话开始时的大致定位

后面如果状态变了，模型应该靠工具重新确认。

### 并不是所有模式都会去抓 git snapshot

源码里还有两个显式跳过条件：

- `CLAUDE_CODE_REMOTE`
- `shouldIncludeGitInstructions()` 关闭

也就是说，`systemContext` 不是无条件注入的。

它依赖运行模式和配置。

## 第 5 层：启动后会做 deferred prefetch，但 `systemContext` 受 trust gate 保护

这一层很多人会忽略，但其实很能体现 Claude Code 的工程味。

在：

- `src/main.tsx`

里，`startDeferredPrefetches()` 会在首屏渲染后做后台预取。

这里至少有两条和上下文直接相关：

- 总是预取 `getUserContext()`
- 只在安全条件满足时预取 `getSystemContext()`

为什么 `systemContext` 要更谨慎？

源码注释直接说了原因：

- git 命令可能通过 hooks 和 config 执行任意代码

所以 Claude Code 不会在 interactive 模式下一启动就盲目预取 git snapshot。

只有两种情况下它才会这么做：

1. non-interactive 模式
2. 用户已经通过 trust gate

这点非常值得抄。

因为它体现的是：

- 首轮响应速度优化
- 和安全边界

被同时考虑了，而不是只顾着“能不能更快”。

## 把这一章放回整体教程里看

到这里，可以把第 2、3、4 章连起来看：

```text
第 2 章：主循环怎么跑
第 3 章：每轮固定带什么进去
第 4 章：如果太长了，怎么缩回可运行范围
```

这样就比较完整了。

如果只看第 2 章，你知道 Claude Code 会循环调用模型。

如果只看第 3 章，你知道每轮 query 前会塞哪些固定上下文。

如果再加上第 4 章，你才知道这些上下文一旦太长，会如何被管理和压缩。

而真正的 `Memory`，还要等后面单独展开。

## 如果你自己实现，最小应该先做哪几层

不要一上来就照抄 Claude Code 的整套上下文系统。

更现实的顺序是：

1. 先做一份稳定的 default system prompt
2. 再做 `CLAUDE.md` 风格的 instruction file 注入
3. 再加一份会话级 git snapshot
4. 然后补上 prompt 合成层
5. 最后再做 memoization 和 deferred prefetch

这样你会更快得到一个：

- 能工作
- 好解释
- 不太容易失控

的最小版本。

## 最值得抄的设计

如果只能抄几件事，我会优先抄下面这些。

### 1. 把“行为规则”和“项目规则”分层

system prompt 和 `CLAUDE.md` 家族不要混写。

### 2. 把 `systemContext` 做成 snapshot，而不是每轮实时同步

这能显著减少开销，也更符合 prompt 的用途。

### 3. 把 prompt 合成逻辑抽成单独一层

只要系统支持多个 agent 或多种运行模式，这层迟早都需要。

### 4. 让 instruction source 可追溯

读了哪些文件、按什么优先级进 prompt，最好都能解释清楚。

## 这一层最容易犯的错误

### 1. 把所有信息都塞进 system prompt

结果是 prompt 又长又乱，后面也很难做差异更新。

### 2. 每轮都重新抓 git 状态

这样既慢，也会让上下文变得不稳定。

### 3. 把 `Context` 和 `Memory` 混成一个概念

最后往往会得到一个“什么都想管、什么都讲不清”的章节。

### 4. 以为注入了规则文件，模型就等于理解了当前代码

规则文件只是工作背景，不是代码阅读本身。

## 下一章应该看什么

到这里，关于“这一轮先带什么进去”这件事已经够清楚了。

接下来最自然的问题就是：

- 这些东西如果太长了怎么办？

所以下一章就该接：

- [04. 上下文压缩：太长以后 Claude Code 怎么缩上下文](./04-context-compaction.md)
