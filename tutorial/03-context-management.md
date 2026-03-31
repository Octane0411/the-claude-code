# 03. 上下文管理：模型每次到底看到了什么

## 这篇文章要回答什么

在上一章里，我们把 Claude Code 的中心定位成了 `query()` 驱动的 agent loop。

但只知道“它会进 loop”还不够。真正决定系统表现的另一个关键问题是：

**每次进 loop 之前，模型到底看到了什么？**

很多人实现 agent CLI 时，会把“上下文管理”理解成三件事之一：

- 把消息历史直接发给模型
- 在 system prompt 里塞一大段项目说明
- 做一个 repo summary，然后希望模型自己想明白剩下的事

Claude Code 的做法要克制得多，也更工程化。

这篇文章要拆清楚的是：

- `system prompt`、`user context`、`system context` 在源码里分别是什么
- `CLAUDE.md` 家族文件是怎么发现、排序和注入的
- git 状态为什么只取会话开始时的快照
- 哪些内容属于“静态上下文注入”，哪些内容已经属于 memory 系统
- 如果你自己实现一个 Claude Code-like 系统，最小可用的上下文层应该怎么搭

## 先给结论

Claude Code 并不是把“当前仓库的一切信息”都塞进模型上下文。

它真正做的是把上下文拆成三层：

1. `defaultSystemPrompt`
   - Claude Code 的基础行为规则、工具使用规则、环境信息、MCP 提示、scratchpad 规则等
2. `userContext`
   - `CLAUDE.md` 家族文件拼出来的用户 / 项目指令，以及当前日期
3. `systemContext`
   - 当前会话开始时的 git snapshot，以及少量系统级注入信息

在 REPL 里，这三层会在 query 前并行准备，然后一起传进 `query()`。

更重要的是：

- `Context` 这一章只处理“这一轮固定会被带进去的内容”
- 动态相关记忆召回、session memory、durable extraction 不属于这一章，它们属于后面的 `Memory` 章节

如果把这两类东西混在一起，你很快就会把 Claude Code 的上下文系统误读成“一个大而全的 prompt 拼接器”。

## 先建立边界：Context 不等于 Memory

这套教程后面会专门写 `Memory`，所以这一章必须先把边界卡住。

从当前源码看，可以先做一个简单判断：

- 每轮 query 前固定准备的，属于 `Context`
- 查询进行中异步召回的、跨会话沉淀的、长会话压缩的，属于 `Memory`

所以：

- `getSystemPrompt(...)`
- `getUserContext()`
- `getSystemContext()`

是这章的主角。

而下面这些，先只点到为止：

- `startRelevantMemoryPrefetch(...)`
- `SessionMemory`
- `extractMemories`

它们会影响模型最终看到的材料，但不属于“每轮固定静态注入”的主路径。

## 一次 query 之前，Claude Code 实际会组哪些输入

在 `src/screens/REPL.tsx` 里，`onQueryImpl()` 会在真正调用 `query()` 之前并行准备：

- `getSystemPrompt(...)`
- `getUserContext()`
- `getSystemContext()`

然后再通过 `buildEffectiveSystemPrompt(...)` 把默认 system prompt 和 agent / custom prompt 合成最终的 `systemPrompt`。

所以进入 `query()` 的关键上下文，不只是消息历史，而更像这样：

```text
messages
+ systemPrompt
+ userContext
+ systemContext
+ toolUseContext
```

这里最值得注意的不是字段名称，而是 Claude Code 的组织方式：

- 把“行为规则”与“项目 / 用户说明”分开
- 把“用户说明”与“系统快照”分开
- 把“固定注入”与“动态召回”分开

这比“所有东西都拼进一个 prompt 字符串”更容易维护，也更容易缓存。

## 第 1 层：`getSystemPrompt()` 负责通用行为规则，不负责项目知识

很多人第一次看到 `Context` 会先去找 `CLAUDE.md`。但从 Claude Code 的结构看，真正最外层还是 system prompt。

相关入口主要在：

- `src/constants/prompts.ts`
- `src/utils/systemPrompt.ts`

`getSystemPrompt(...)` 负责生成默认 system prompt section。它里面装的主要不是“当前项目的事实”，而是：

- 会话指导规则
- memory 使用规则
- 环境信息
- 语言与输出风格
- MCP instructions
- scratchpad instructions
- function result clearing / summarize tool results 之类的运行规则

这里有一个很重要的判断：

**Claude Code 把“怎么工作”放进 system prompt，把“当前项目是什么”放进 user/system context。**

这是个非常好的分层。

如果你把项目说明也全塞进 system prompt，会遇到两个问题：

- prompt 太长，且难以缓存
- 行为规则和项目事实混在一起，后续难以做差异更新

## 第 2 层：`buildEffectiveSystemPrompt()` 决定最终 system prompt 长什么样

`getSystemPrompt(...)` 还不是最终传给模型的版本。

在：

- `src/utils/systemPrompt.ts`

里，`buildEffectiveSystemPrompt(...)` 还会继续处理优先级：

1. override system prompt
2. coordinator prompt
3. agent prompt
4. custom system prompt
5. default system prompt
6. append system prompt

这说明 Claude Code 从一开始就承认：

- system prompt 不是一个固定常量
- 不同运行模式可能会替换或追加 prompt
- 但这个替换逻辑应该发生在专门的 prompt 组合层，而不是散落在 REPL 或 query loop 里

如果你自己实现，最好也把这层显式抽出来。否则后面一旦引入：

- 自定义 agent
- 协调器模式
- 用户自定义 `--system-prompt`

你的 prompt 拼装逻辑很快就会失控。

## 第 3 层：`getUserContext()` 负责项目 / 用户指令注入

真正和“Claude Code 会读哪些 instruction files”直接相关的，是：

- `src/context.ts`
- `src/utils/claudemd.ts`

`getUserContext()` 本身非常克制。它最终只返回两样东西：

- `claudeMd`
- `currentDate`

也就是说，Claude Code 的“用户上下文”不是：

- 整个项目文件树
- 全量代码索引
- 大模型生成的 repo summary

而是：

- `CLAUDE.md` 家族文件
- 一个当前日期字符串

这很重要。Claude Code 并不默认替用户做一个“大上下文蒸馏器”，而是先把最稳定、最明确、最可控的 instruction files 注入进去。

## `CLAUDE.md` 不是单文件，而是一整套发现规则

`src/utils/claudemd.ts` 顶部的注释已经把发现规则写得很清楚。

当前顺序可以概括成：

1. `Managed`
2. `User`
3. 从 root 到 CWD 逐层发现的 `Project`
4. 同路径上的 `Local`
5. 可选的 `--add-dir`
6. memdir 的 `MEMORY.md`
7. team memory 的 `MEMORY.md`

其中最值得你实现时学习的，不是“文件很多”，而是这两个原则：

- checked-in instructions 和 private instructions 被明确分开
- 离当前工作目录更近的项目级规则，优先级更高

这让 Claude Code 能同时支持：

- 全局个人偏好
- 仓库级规范
- 私有项目本地规则

而不用靠一份无限膨胀的 `~/.config/agent/prompt.md` 硬扛所有事情。

## `CLAUDE.md` 家族的真正价值，是把“指令来源”显式化

很多系统喜欢在 UI 里做“自动学习你的偏好”，但最后很难解释模型为什么会这样表现。

Claude Code 在这件事上相对保守：

- 用户知道哪些文件会被读
- 仓库维护者知道哪些规则是 checked-in 的
- 私有规则和共享规则是分开的

这种显式化非常重要，因为它让行为可追溯。

如果你想做一个“真的能让团队放心用”的 agent CLI，这种可追溯性通常比“再多一点神秘智能”更值钱。

## `filterInjectedMemoryFiles()` 暗示了 Context 和 Memory 正在被继续拆开

`getUserContext()` 在读取 memory files 后，不是直接全部拼进去，而是会先经过：

- `filterInjectedMemoryFiles(...)`

这个函数背后有一个关键 feature flag：

- 当 `tengu_moth_copse` 关闭时，`AutoMem` / `TeamMem` 也会像普通注入文件那样进入 prompt
- 当它开启时，这些 entrypoint 会被跳过，因为相关记忆会改走动态召回链

这里透露出一个很重要的架构方向：

**Claude Code 正在把“固定静态指令”与“动态记忆材料”进一步分离。**

这也是为什么这一章一定不要把 Memory 混进来。连 Claude Code 自己都在把两者拆开。

## 第 4 层：`getSystemContext()` 负责会话级系统快照

`src/context.ts` 里的 `getSystemContext()` 是另一条完全不同的链。

它处理的不是用户指令，而是系统级快照信息，当前最核心的是：

- git status
- current branch
- default branch
- recent commits
- 可选的 cache breaker 注入

这里最值得注意的是它的语义：

源码明确说明，这是一份：

**conversation start snapshot**

也就是说，它不是一个会自动刷新、始终反映最新工作区状态的 live context。

Claude Code 这里做了一个很清醒的取舍：

- 只在会话开始时给模型一个 repo 状态定位
- 后面如果状态变了，更依赖工具调用去拿最新信息

这是对的。

如果你强行想让 prompt 永远携带“最新 git 状态”，会有几个明显问题：

- 代价高
- prompt cache 命中率差
- 状态变化会造成上下文不停抖动
- 最终反而让模型更依赖一份可能过时的描述，而不是实际去读文件或跑命令

## 为什么 git status 会被截断

`getGitStatus()` 里有一个 `MAX_STATUS_CHARS = 2000` 的硬限制。

超过长度时，它不会继续扩展 prompt，而是给出一个明确提示：如果你需要更多，请调用 Bash 工具去看真正的 `git status`。

这个设计非常成熟，因为它体现了一个 Claude Code 反复在做的选择：

- prompt 里只放“够定位问题”的摘要
- 真正的大量信息，交给工具按需获取

这和很多简单 agent 的思路正好相反。后者往往喜欢把能塞的都塞进去，结果 prompt 又长又不稳定。

## 第 5 层：预取策略本身也是上下文系统的一部分

在：

- `src/main.tsx`

里，`startDeferredPrefetches()` 会在首屏之后启动几类预取，其中和上下文直接相关的是：

- 总是预取 `getUserContext()`
- 只在 non-interactive 或 trust 已建立时预取 `getSystemContext()`

这个策略很值得注意。

它说明 Claude Code 不是只关心“上下文内容对不对”，还关心：

- 哪些内容便宜，适合预热
- 哪些内容有额外副作用或信任要求，不该太早做

也就是说，上下文系统在 Claude Code 里不只是 prompt 内容问题，也是启动性能和安全边界问题。

## `Context` 这一层为什么看起来“信息不多”

第一次读到这里，你可能会觉得 Claude Code 的静态上下文非常少：

- 没有全量代码库摘要
- 没有自动生成的 architecture overview
- 没有预先塞进去的函数索引

这其实正是它做得比较成熟的地方。

Claude Code 依赖的是一个组合：

- system prompt 给行为规则
- `CLAUDE.md` 给明确指令
- system context 给 repo 起点定位
- tool system 在需要时获取真实细节

这种设计有两个好处：

1. prompt 更稳定
2. 模型被鼓励去“看真实世界”，而不是过度相信一份提前蒸馏好的摘要

这也是为什么 Claude Code 能比较自然地把工具系统和上下文系统扣在一起。

## 这一章不该展开的内容

为了后面的章节清楚，这里要明确三件暂时不展开的事。

### 1. 动态相关记忆召回

`query.ts` 里的 `startRelevantMemoryPrefetch(...)` 会在 turn 内异步找“相关记忆文件”，然后以 attachment 的方式补进上下文。

这不是本章的静态注入链。

### 2. Session memory

`SessionMemory` 的目标是给长会话维护 working notes，它本质上是压缩与持久化机制，不是 query 前固定注入的上下文。

### 3. Durable memory extraction

`extractMemories` 这种会后沉淀机制，解决的是跨会话复用，不是本轮 query 的静态构造。

换句话说：

- 本章回答“每轮开始时先带了什么”
- 后面的 `Memory` 章节回答“中途还会补什么，以后还会留下什么”

## 把 Context 放回 Agent Loop 里看

如果把上一章的 loop 路径和这一章拼起来，可以把 REPL 主线程的 query 前半段简化成：

```text
用户输入
  -> processUserInput
  -> getToolUseContext
  -> Promise.all(
       getSystemPrompt(...),
       getUserContext(),
       getSystemContext(),
     )
  -> buildEffectiveSystemPrompt(...)
  -> query(...)
```

这条路径的关键点不在“调用了三个函数”，而在它体现出的架构边界：

- `getSystemPrompt(...)` 负责默认规则
- `buildEffectiveSystemPrompt(...)` 负责最终覆盖与组合
- `getUserContext()` 负责 instruction files
- `getSystemContext()` 负责 repo snapshot

这四件事被故意拆开了。

如果你自己实现，最好也保留这种拆法。

## 如果你要自己实现，一个干净的最小版本应该怎么做

很多人会在第一版就试图做：

- repo indexing
- embedding retrieval
- 自动 architecture summary
- 多层长期记忆

其实没必要。

一个足够像 Claude Code 的最小上下文层，可以只做这四步：

1. 写一份稳定的默认 system prompt
2. 扫描 `CLAUDE.md` / `.claude/CLAUDE.md` / `CLAUDE.local.md`
3. 在会话开始时抓一次 git status snapshot
4. 把这些内容作为结构化字段传进 query runtime

然后先停下来。

这时候你已经拥有了一个很像 Claude Code 的基础上下文层，而且复杂度还可控。

## 最小实现时，最值得保留的设计

如果只能保留几件事，我会优先保留下面这些。

### 1. 把 `system prompt` 和 `user/system context` 分开

这能显著减少后面维护 prompt 组合逻辑时的混乱。

### 2. 指令文件做显式优先级，而不是拼接一堆隐式规则

显式优先级意味着行为更可解释，也更适合团队协作。

### 3. git status 只做 snapshot，不做实时刷新

实时刷新听起来更智能，实际上通常更不稳定。

### 4. 不要急着把 memory 混进静态上下文

先把静态上下文做稳，再考虑动态召回和持久化。

这条顺序非常重要。

## 这一层最容易犯的错误

如果你自己实现，很容易犯下面几种错误。

### 1. 试图把整个仓库压缩成一个 summary

这样做往往很贵，而且一旦过时，误导比帮助更大。

### 2. 把行为规则、项目事实、用户偏好全塞进同一个 prompt 字符串

短期看省事，长期看几乎一定会让 prompt 维护变成灾难。

### 3. 把动态 memory 召回和静态 instruction 注入混为一谈

这样后面你会很难解释：

- 这条信息为什么每轮都在
- 那条信息为什么只在某些 turn 出现

### 4. 让上下文层承担过多“聪明推断”

Claude Code 在这里的成熟之处恰恰是克制。它没有强迫上下文层去替模型和工具系统做所有推理。

## 下一章应该看什么

到这里，我们已经知道：

- query 前会准备哪些上下文
- 它们分别从哪里来
- Claude Code 为什么不把“上下文”做成一个超大 prompt

下一步最自然的展开，其实有两条：

- 如果你想继续理解“模型在 loop 里能做什么”，应该去看工具系统
- 如果你想继续理解“上下文窗口之外的信息怎么流回来”，应该去看记忆系统

按主线顺序，下一章我们先看工具系统：Claude Code 是怎么把工具注册、裁剪、执行并重新回注进 loop 的。
