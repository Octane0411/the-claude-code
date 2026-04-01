# 04. 上下文压缩：太长以后 Claude Code 怎么缩上下文

## 这篇文章要回答什么

上一章我们讲的是：

- 这一轮 query 开始前，Claude Code 会先带什么上下文进去

但那只是问题的一半。

真正到了长会话里，你马上会撞到另一个更现实的问题：

**如果消息历史、工具结果、附件和系统提示加起来太长了，Claude Code 怎么继续跑下去？**

很多 agent CLI 到这里就开始变得很脆弱：

- 要么上下文越聊越长，最后直接爆窗
- 要么粗暴截断历史，模型马上失忆
- 要么每次都做一次“大总结”，结果细节丢得很快

Claude Code 的处理方式要复杂得多，也成熟得多。

这一章要讲清楚的是：

- Claude Code 不是只有一种 compact，而是好几层压缩手段叠在一起
- 它会优先做哪些“便宜”的压缩，再做哪些“昂贵”的压缩
- prompt too long 之后它怎么恢复，而不是立刻报错退出
- 手动 `/compact`、自动 compact、partial compact 各自是什么
- 如果你自己做一个 Claude Code-like 系统，最小应该先实现哪一层

## 先给结论

Claude Code 的上下文压缩，不是一个单独命令，也不是一次“大总结”。

它更像一条多级管线：

```text
进入模型前
  -> 先缩 tool result
  -> 再做 snip
  -> 再做 microcompact
  -> 再做 context collapse
  -> 还不够再做 autocompact
  -> 真的报 prompt too long 时，再走恢复路径
```

也就是说，Claude Code 的思路不是：

- “超长了就总结一次”

而是：

- “先尽量保留细粒度历史”
- “只有在必要时才把更大块的历史压成摘要”
- “如果模型已经报溢出，再按从便宜到昂贵的顺序恢复”

这套策略的核心目标不是“省 token”这么简单，而是：

**在不丢掉继续工作的能力的前提下，把上下文压回可运行范围。**

## 先看地图

如果你现在只想先抓住大图，可以先把 Claude Code 的压缩系统想成下面这样：

```text
原始消息历史
    |
    v
+-----------------------------+
| 轻量压缩                    |
| - tool result budget        |
| - snip                      |
| - microcompact              |
+-----------------------------+
    |
    v
+-----------------------------+
| 结构化上下文管理            |
| - context collapse          |
| - 保留更细粒度历史          |
+-----------------------------+
    |
    v
+-----------------------------+
| 全量摘要式压缩              |
| - autocompact               |
| - compactConversation       |
+-----------------------------+
    |
    v
还能继续跑 query
```

这张图里最重要的判断是：

- 不是所有压缩都一样重
- 不是所有压缩都会直接改写 REPL 里看到的消息数组
- 也不是所有压缩都会立刻生成“总结消息”

## 再看一条主流程

在 `query.ts` 里，真正调用模型之前，大致会经过下面这条路径：

```text
messagesForQuery
  -> applyToolResultBudget()
  -> snipCompactIfNeeded()
  -> microcompactMessages()
  -> contextCollapse.applyCollapsesIfNeeded()
  -> autoCompactIfNeeded()
  -> 调模型
```

如果模型仍然返回 `prompt too long`，则会继续：

```text
prompt too long
  -> 先试 contextCollapse.recoverFromOverflow()
  -> 不行再试 reactive compact
  -> 再重试 query
  -> 还不行才真正报错
```

你可以把这理解成两阶段：

1. 预防性压缩
2. 溢出后恢复

这两个阶段都很重要。

## 先抓源码入口

如果你想先顺着源码把这条链走一遍，最值得先看的文件就是这几个：

- `src/query.ts`
  - 主入口。决定“进模型前先做哪些压缩”，以及 `prompt too long` 之后先怎么救
- `src/services/compact/autoCompact.ts`
  - 决定 autocompact 什么时候触发，什么时候直接跳过
- `src/services/compact/compact.ts`
  - 真正做“把长历史压成可继续工作的交接包”
- `src/services/compact/microCompact.ts`
  - 负责清旧 `tool_result`，不是做整段会话总结
- `src/services/compact/prompt.ts`
  - 定义 compact 时给模型的交接提示词
- `src/setup.ts`
  - 启动期初始化 `context collapse`
- `src/screens/REPL.tsx`
  - 交互式路径里的 `/compact` 和 partial compact 入口

这几个文件合起来，基本就能把 Claude Code 的压缩系统看完整。

## 先记住四个容易混淆的边界

### 1. `snip` 不是 `compact`

`snip` 更像是：

- 先剪掉最不值得继续带着走的历史片段

而 `compact` 更像是：

- 把一段较长历史改写成摘要

### 2. `microcompact` 不是会话总结

`microcompact` 的目标主要是清理旧的工具结果内容，而不是总结整段对话。

### 3. `context collapse` 不等于 `autocompact`

源码注释已经明确暗示了这点：

- `context collapse` 的目标是尽量保留更细粒度的上下文结构
- `autocompact` 则更像“真的不够了，做一次摘要式压缩”

### 4. `/compact` 不是唯一 compact 入口

Claude Code 里至少有：

- 自动 compact
- 手动 `/compact`
- partial compact
- reactive compact

所以不要把“上下文压缩”误读成一个单命令功能。

## 第 1 层：先缩太大的 tool result

在真正进入 compact 逻辑之前，`query.ts` 先会调用：

- `applyToolResultBudget(...)`

它的目的很直接：

- 某些工具返回结果特别大
- 与其把整份结果一直带在上下文里，不如先把它们做预算控制和内容替换

这件事虽然不在 `services/compact/` 目录里，但它实际上就是上下文压缩链的第一层。

这里非常值得注意，因为它体现了 Claude Code 的一个核心思路：

**先压那些最“机械性膨胀”的内容。**

工具结果往往是最容易暴涨、也最不需要原样永久保留的东西，所以先从它们下手最划算。

## 第 2 层：`snip` 是最便宜的历史裁剪

在：

- `query.ts`

里，`snip` 发生在 microcompact 之前。

源码注释已经说明了它的定位：

- 先删一部分历史
- 并把删掉的大概 token 数传给 autocompact 阈值判断

这说明 `snip` 的角色不是“聪明总结”，而是：

- 快速减重
- 尽量不引入新的 summarization 成本

你可以把它理解成：

**在不做大规模改写的前提下，先把最不值钱的旧内容从活动上下文里移掉。**

正文里现在还没有专门一节把 snip 的启发式规则拆开，但从 `query.ts` 的位置就能看出来，它属于“先上便宜手段”的那一层。

## 第 3 层：`microcompact` 清旧工具结果，不总结整段对话

`microCompact.ts` 很值得单独看，因为它解决的是另一个问题：

- 很多旧 `tool_result` 继续留在上下文里，其实只会占空间

它当前有两条主要思路。

### 时间触发型 microcompact

如果距离上次 assistant 回复已经过了较长时间，源码会认为：

- 服务端 cache 可能已经冷了
- 这时继续带着一堆旧工具结果重写整段前缀，不划算

于是它会：

- 清掉较老的 compactable tool results
- 只保留最近的少数几个

这不是摘要，而是把旧结果内容直接替换成类似：

- `[Old tool result content cleared]`

的占位消息。

### Cached microcompact

另一条更高级的路径是 cached microcompact。

它的目标不是直接改写本地消息内容，而是：

- 利用 cache editing API
- 把可以删除的旧工具结果从缓存前缀里剔掉
- 尽量保持前缀缓存还能命中

这一层非常 Claude Code。

因为它优化的不是“模型理解效果”，而是：

- prompt cache 成本
- 重写前缀的代价

换句话说，microcompact 关心的是：

**怎么把上下文变轻，但尽量不做高成本总结。**

## 第 4 层：`context collapse` 想保留更细粒度历史

这是当前 restored source 里最“只见入口、不见完整实现”的一层。

我们在本地看到的证据主要来自：

- `setup.ts` 会调用 `initContextCollapse()`
- `query.ts` 会在 autocompact 前调用 `applyCollapsesIfNeeded(...)`
- overflow 时会调用 `recoverFromOverflow(...)`

而且 `query.ts` 里的注释已经把它的定位说得很清楚：

- collapse 先于 autocompact
- 如果 collapse 已经把上下文压回阈值以内，就不需要 autocompact
- 它的目标是“保留 granular context”，而不是直接把一大段历史合成一个 summary

你可以把它理解成：

**Claude Code 在尝试一种比“整段总结”更保真的压缩方式。**

这也是为什么它会被放在 autocompact 之前。

不过这里要明确一个证据边界：

- 当前 restored source tree 没有 `services/contextCollapse/index.js` 的完整实现文件
- 所以我们能确认它在主链中的位置和职责
- 但还不能像 `autoCompact.ts` 那样把内部算法完全讲细

这类内容后面如果要写得更实，需要结合本机提取物再验证。

## 第 5 层：`autocompact` 才是“真的做总结”的主路径

到了：

- `services/compact/autoCompact.ts`
- `services/compact/compact.ts`

这一层，Claude Code 才真正进入“摘要式压缩”。

### 什么时候会触发 autocompact

`autoCompact.ts` 先会算：

- 模型的 context window
- 要为 compact summary 预留多少输出 token
- 当前 token 使用量是否超过 autocompact threshold

也就是说，autocompact 不是“快到上限才碰碰运气”，而是会提前预留一部分空间，保证 compact 自己还有生成摘要的预算。

### autocompact 之前还会先试 session memory compaction

这点特别容易被忽略。

在真正调用 `compactConversation(...)` 之前，`autoCompactIfNeeded()` 还会先试：

- `trySessionMemoryCompaction(...)`

这说明 Claude Code 的思路是：

- 如果能通过 session memory 这类更有针对性的方式减少上下文，就先别做全量 compact

这再次体现了它“先保细节、后做总摘要”的倾向。

### `compactConversation()` 真正在做什么

`compactConversation()` 的核心不是“删旧消息”，而是：

1. 执行 PreCompact hooks
2. 构造 compact prompt
3. 用 forked agent 生成一段结构化摘要
4. 清理旧的 readFileState / nested memory path 等状态
5. 重新注入一部分必须保留的附件
   - 文件附件
   - agent 附件
   - plan mode 附件
   - skill 附件
   - deferred tool delta
   - agent listing delta
   - MCP instructions delta
6. 执行 SessionStart / PostCompact hooks
7. 产出 compact boundary + summary messages + messagesToKeep + attachments

所以 compact 后留下来的，不只是“一段摘要”。

更准确地说，它会生成一个新的 post-compact 上下文包。

## 为什么 compact 不是“写个小摘要”

`services/compact/prompt.ts` 很能说明 Claude Code 为什么 compact 不是随手做的事。

compact prompt 明确要求模型输出一个结构化 summary，里面要覆盖：

- 用户的主要请求
- 技术概念
- 文件和代码片段
- 错误与修复
- 问题解决过程
- 全部用户消息
- 待办事项
- 当前工作
- 下一步

这说明 compact 在 Claude Code 里的角色，不是“顺手写两句总结”，而是：

**给后续继续工作的 agent 重建一份足够可用的工作交接。**

这也解释了为什么它比 microcompact 贵得多，但也更可靠。

## 第 6 层：如果还是溢出，Claude Code 会先恢复，再报错

`query.ts` 的另一个非常成熟的地方，是它不会在第一次 `prompt too long` 时立刻把错误抛给用户。

它会先走恢复路径。

### 第一步：先试 `context collapse drain`

如果启用了 context collapse，而且上一次 transition 还不是 `collapse_drain_retry`，它会先调用：

- `contextCollapse.recoverFromOverflow(...)`

源码注释已经把设计意图写得很清楚：

- 先试这个，是因为它更便宜
- 而且能尽量保留更细粒度的历史

### 第二步：再试 reactive compact

如果 collapse 没救回来，接下来才会走：

- `reactiveCompact.tryReactiveCompact(...)`

它会生成新的 post-compact messages，然后直接继续当前 query call，而不是要求用户重新提交输入。

这件事非常重要，因为它意味着：

- “上下文溢出恢复”被内建进了主循环
- 它不是让用户手动擦屁股的异常处理

### 只有两层恢复都失败，错误才真正浮出来

这就是 Claude Code 比很多简单 agent 更成熟的地方：

- 先尝试恢复
- 恢复失败再暴露错误

而不是把内部资源管理问题直接甩给用户。

## 手动 `/compact` 和 partial compact 是另一类能力

到目前为止我们讲的，主要是 query loop 里的自动路径。

但 REPL 里还有另外两类交互式压缩能力：

- `/compact`
- partial compact

从 `REPL.tsx` 看，用户可以在 message selector 里选中某条消息，然后触发：

- `partialCompactConversation(...)`

这说明 Claude Code 不是只支持“系统觉得该 compact 时自动 compact”，还支持：

- 用户主动指定从哪里开始压
- 或者把某一段压成 summary，保留别的部分

所以从产品角度看，Claude Code 的 compact 已经不是单一后台机制，而是用户可操作的会话编辑能力。

## 把这一章放回整体教程里看

现在可以把第 3 章和这一章合起来看：

```text
第 3 章：这一轮先带什么进去
第 4 章：如果太长了，怎么把它缩回可运行范围
```

这两章合起来，才构成比较完整的“上下文管理”。

如果只讲注入，不讲压缩，读者会误以为：

- 上下文只是在 query 前静态拼一下

但 Claude Code 实际上做的是：

- 先构造上下文
- 再持续管理上下文大小
- 并在溢出时做恢复

## 如果你自己实现，最小应该先做哪一层

不要一上来就复刻 Claude Code 全套 compact 系统。

更现实的顺序是：

1. 先做最基础的 message window 裁剪
2. 再做手动 `/compact`
3. 再做自动 compact
4. 最后再考虑 microcompact / cached microcompact / context collapse

因为真正最难的不是“做一个摘要”，而是：

- 让 compact 后的会话还能继续稳定工作
- 让附件、文件状态、工具状态、MCP 指令重新对齐

这部分复杂度远比表面上高。

## 最值得抄的设计

如果只能抄几件事，我会优先抄下面这些。

### 1. 分层压缩，而不是只有一个 compact 按钮

先做便宜压缩，再做昂贵压缩，这会让系统稳定很多。

### 2. compact 产物不是只有摘要，还要有“继续工作所需的附件”

这是很多简单实现最容易漏掉的点。

### 3. 把 overflow recovery 放进 query loop

不要让用户一看到 `prompt too long` 就只能自己想办法。

### 4. 把“保细节”和“保成本”当成两种不同目标

microcompact、collapse、autocompact 本质上就是在不同目标之间做平衡。

## 这一层最容易犯的错误

### 1. 只有一种“全量总结”

这样很容易每次 compact 都丢太多细节。

### 2. compact 后只留摘要，不恢复关键附件

结果下一轮模型虽然有 summary，但缺关键工作材料。

### 3. prompt too long 直接报错

这会让长会话体验非常差。

### 4. 把所有压缩都做成对消息数组的硬改写

Claude Code 明显在尝试更细粒度的投影和 collapse，而不是一切都落成新的 summary message。

## 下一章应该看什么

到这里，关于上下文这条线，至少已经有两个核心问题被补齐了：

- 这一轮带什么进去
- 太长以后怎么缩回可运行范围

下一步最自然的展开，就是看工具系统。

因为上下文再好，真正让 Claude Code 变成 agent 的，还是：

- 模型能调用哪些工具
- 工具怎么执行
- 工具结果怎么重新回到 loop 里
