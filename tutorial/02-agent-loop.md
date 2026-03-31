# 02. Agent Loop：一次请求是怎么跑起来的

## 这篇文章要回答什么

在上一章里，我们已经建立了一个全局判断：

**Claude Code 的中心不是命令列表，也不是终端 UI，而是 agent loop。**

这篇文章要做的，就是把这句话展开成一条真正的运行链路：

- 用户输入之后，系统先做了什么
- 什么阶段会决定“本地处理”还是“继续问模型”
- `query()` 为什么不是一次简单的 API 调用
- tool use、tool result、memory、attachments、queued commands 是怎么重新回到下一轮上下文里的
- REPL 和 SDK 为什么看起来入口不同，但本质上又共用同一个核心

如果你要自己实现一个 Claude Code-like 系统，这一章应该是整套教程里最重要的一章之一。

## 先给结论

Claude Code 的 agent loop 不是：

1. 读用户输入
2. 调一次模型
3. 输出结果

它真正做的是：

1. 把用户输入规范化成消息、附件和命令结果
2. 构造一份带上下文、权限和工具池的执行环境
3. 进入 `query()` 驱动的循环
4. 在循环里反复执行：
   - 调模型
   - 流式接收输出
   - 识别 `tool_use`
   - 执行工具
   - 把 `tool_result`、memory、notification、attachment 回注进上下文
   - 判断是否继续下一轮
5. 直到满足某个终止条件为止

换句话说，Claude Code 的一个用户请求，本质上是一个**多阶段、可恢复、带状态的执行过程**。

## 两条入口，一颗心脏

理解 agent loop，第一步不是直接看 `query.ts`，而是先知道系统有两条主要入口。

### 交互式 REPL 路径

交互式主线程的核心路径在这些文件里：

- `src/screens/REPL.tsx`
- `src/utils/handlePromptSubmit.ts`
- `src/utils/processUserInput/processUserInput.ts`
- `src/query.ts`

可以粗略理解成：

```text
PromptInput / onSubmit
  -> handlePromptSubmit
  -> executeUserInput
  -> processUserInput
  -> onQuery
  -> onQueryImpl
  -> query()
```

这里最重要的结论是：

**REPL 主线程并不通过 `QueryEngine` 进入主循环，它是直接调 `query()` 的。**

### SDK / Headless 路径

另一条入口在：

- `src/QueryEngine.ts`

这里的路径更像：

```text
QueryEngine.submitMessage
  -> processUserInput
  -> query()
  -> 归一化为 SDKMessage / result
```

`QueryEngine` 的价值，不是替代 `query()`，而是把下面这些能力包装起来：

- conversation 级消息状态
- transcript 持久化
- SDK message 归一化
- permission denial 跟踪
- structured output / budget / session 级结果包装

所以你可以把二者的关系理解为：

- `query()` 是共享的执行引擎
- `REPL.tsx` 是交互式 UI 外壳
- `QueryEngine.ts` 是 headless / SDK 外壳

## 一个“turn”不是一次 API 请求

这是理解 Claude Code 的第一个关键概念。

在 `src/query.ts` 里，主循环是一个 `while (true)` 结构。也就是说，一个用户提交的请求，并不对应一次固定的模型调用。

一次用户 turn 里，可能发生：

- 第一次模型采样
- 发现 `tool_use`
- 执行工具
- 再次调用模型
- 因为 `max_output_tokens` 继续补一轮
- 因为 stop hook 阻塞再补一轮
- 因为 token budget 再补一轮
- 因为 prompt too long / reactive compact / fallback model 触发恢复路径后重试

这意味着 Claude Code 里的“turn”更接近：

**围绕一次用户意图展开的完整代理执行事务。**

而不是“一次 HTTP 请求”。

## 第 1 步：用户输入先被送进提交层

REPL 路径里，入口在 `src/screens/REPL.tsx` 的 `onSubmit`。

这层先处理很多“还没进入主循环之前”的事情，例如：

- 即时命令是否应该绕过队列直接执行
- 当前是否已经有 query 在跑
- 是否应该中断正在运行的 turn
- remote mode 是否要把消息发给远端
- 输入框、历史记录、pasted contents、UI loading 状态如何更新

真正重要的是：`onSubmit` 并不会直接调用模型，而是把后续工作交给 `handlePromptSubmit()`。

这说明 Claude Code 把“提交输入”和“执行 agent loop”明确拆成了两个阶段。

## 第 2 步：handlePromptSubmit 负责把输入送入统一执行通道

`src/utils/handlePromptSubmit.ts` 是第二个关键点。

它做了几件非常像“runtime gateway”的事情：

- 处理 pasted text / image 引用展开
- 识别本地即时命令
- 在系统忙碌时把新输入排队，而不是直接并发跑第二个 loop
- 为这次执行创建新的 `AbortController`
- 调用 `executeUserInput()`，让 direct input 和 queued input 走同一条管道

这里最值得注意的是 `queryGuard`。

它的意义不是单纯防抖，而是：

- 避免多个 query loop 并发跑在同一个主会话上
- 确保新的输入要么排队，要么触发中断，而不是破坏当前 turn
- 把“会话级串行执行”做成运行时约束

如果你自己实现类似系统，这一层非常值得抄。很多 toy agent 最大的问题，就是根本没有这层串行调度约束。

## 第 3 步：processUserInput 决定“这条输入到底是什么”

真正把输入转成运行时对象的逻辑，在：

- `src/utils/processUserInput/processUserInput.ts`

这里不是简单地把字符串变成 `user message`。它实际上会判断：

- 这是普通文本 prompt 吗
- 这是 slash command 吗
- 这是 bash 模式输入吗
- 有没有 pasted image / pasted text / IDE selection / attachment
- 这个输入是否应该触发真正的 query
- 是否顺带改写 allowed tools / model / effort

也就是说，`processUserInput()` 的结果不是一个字符串，而是一组结构化结果：

- `messages`
- `shouldQuery`
- `allowedTools`
- `model`
- `effort`
- `nextInput`

这一点非常关键，因为 Claude Code 不是“先分类，再走完全不同的分支树”，而是尽量把不同类型输入都规整成同一种执行协议。

这也是后面整个 agent loop 能稳定工作的原因之一。

## 第 4 步：系统构造 query 的执行上下文

当输入被处理完、并且 `shouldQuery === true` 时，REPL 路径会进入 `onQueryImpl()`，位置在：

- `src/screens/REPL.tsx`

这一步会准备进入 `query()` 所需的关键上下文。

### 4.1 ToolUseContext

这是 Claude Code 的核心运行时上下文之一。它里面包含的不是单一参数，而是一整套执行环境，例如：

- commands
- tools
- mcpClients
- mainLoopModel
- thinkingConfig
- getAppState / setAppState
- readFileState
- abortController
- notification / prompt / file history / attribution 等各种回调

你可以把 `ToolUseContext` 理解为：

**工具执行、会话状态和运行时 side effects 的总线对象。**

### 4.2 System Prompt / User Context / System Context

在进入 `query()` 之前，REPL 还会显式拉取：

- `getSystemPrompt(...)`
- `getUserContext()`
- `getSystemContext()`

这一步通常在 `src/screens/REPL.tsx` 的 `onQueryImpl()` 里完成；而 SDK/headless 路径则由 `src/QueryEngine.ts` 里的 `fetchSystemPromptParts()` 完成。

这一步再次说明：

Claude Code 的主循环输入，不是“聊天消息数组”，而是：

- 消息历史
- system prompt
- user context
- system context
- tool use context

这也是为什么我们前面一直强调，Claude Code 不是简单的聊天壳。

## 第 5 步：正式进入 `query()`

真正的共享核心，在：

- `src/query.ts`

REPL 会直接：

```ts
for await (const event of query(...)) {
  onQueryEvent(event)
}
```

SDK/headless 则是 `QueryEngine.submitMessage()` 调 `query()`，再把里面的消息流重新包装成 `SDKMessage`。

所以本章的主角其实不是 `QueryEngine`，而是 `query()`。

## 第 6 步：`query()` 先做的不是调模型，而是准备 loop state

`query()` 的第一层只是一个包装器，真正的主体在内部的 `queryLoop()`。

它首先建立一份可跨 iteration 持续存在的状态对象，包括：

- `messages`
- `toolUseContext`
- `autoCompactTracking`
- `maxOutputTokensRecoveryCount`
- `hasAttemptedReactiveCompact`
- `pendingToolUseSummary`
- `turnCount`
- `transition`

这一步很值得注意，因为它说明 Claude Code 从一开始就把这些恢复与控制变量当成 loop 的一等公民，而不是在异常分支里零散塞一些 flag。

也正因为如此，Claude Code 才能把下面这些能力自然塞进主循环：

- compaction
- prompt-too-long recovery
- max-output-tokens recovery
- fallback model
- stop hooks
- token budget continuation

## 第 7 步：进入每一轮 iteration 前，先整理上下文

每一轮循环开始，`query()` 并不会立刻调用模型，而是会先做一批上下文整理工作，例如：

- 从 compact boundary 之后切出真正应该发给模型的消息窗口
- 应用 tool result budget
- 执行 snip / microcompact / context collapse / autocompact
- 更新 `toolUseContext.messages`
- 构造本轮真正要发给模型的 `messagesForQuery`

这一层非常重要，因为它说明：

**模型看到的上下文，并不等于内存里的完整消息数组。**

Claude Code 会在进入 API 调用前，先对上下文做一轮 runtime 投影和裁剪。

这也是“message history”和“query input”必须区分开的原因。

## 第 8 步：模型流式输出开始后，系统一边接收，一边记录，一边准备工具

进入采样阶段后，`query()` 会调用底层 `deps.callModel(...)`，并开始流式消费输出。

这时系统会同时维护这些对象：

- `assistantMessages`
- `toolResults`
- `toolUseBlocks`
- `needsFollowUp`

这里有一个非常关键的点：

### `tool_use` 是 loop 是否继续的真正信号

Claude Code 并不只看“模型是否说完了”，还会看本轮 assistant 消息里是否包含 `tool_use`。

如果有：

- 说明这一轮还没完成
- 还需要去执行工具
- 后面还要把结果回注给模型

如果没有：

- 才进入 stop hooks、token budget、最终结束判断

这正是 agent loop 和普通 chat loop 的分水岭。

### StreamingToolExecutor 让工具执行可以和流式输出并行

`query.ts` 里还有一个非常成熟的设计：

- `StreamingToolExecutor`

当这个能力开启时，Claude Code 不一定等模型整段回复完全结束，才开始考虑工具，而是可以在流式阶段就开始登记并逐步处理 tool updates。

这类设计说明 Claude Code 已经不是“功能正确即可”，而是在明显优化交互时延和执行效率。

## 第 9 步：如果本轮没有 tool use，就走结束分支或恢复分支

当 `needsFollowUp === false` 时，Claude Code 不会直接结束，而是先检查一系列“是否应该继续”的条件。

这包括：

- prompt-too-long 恢复
- reactive compact
- `max_output_tokens` 恢复
- stop hooks
- token budget continuation

这一步非常值得你在自己的实现里重点借鉴。

因为这说明 Claude Code 的 loop 不是：

- 成功就结束
- 出错就失败

而是：

- 先判断这个问题是不是可恢复的
- 如果可恢复，就把恢复动作显式地塞回 loop state，然后 `continue`

这是一种非常典型的“runtime-driven recovery”思路。

## 第 10 步：如果本轮有 tool use，就进入工具执行阶段

当存在 `toolUseBlocks` 时，`query()` 会进入工具执行阶段。

这里有两种来源：

- `StreamingToolExecutor.getRemainingResults()`
- `runTools(...)`

无论走哪条路，本质都一样：

- 根据 assistant 里发出的 `tool_use` 执行对应工具
- 产出新的消息或 attachment
- 必要时更新 `ToolUseContext`
- 把这些 tool result 收集起来，准备进入下一轮

这里有一个架构上的关键点：

**工具执行不是一个“外部 side effect”，而是 agent loop 的正式组成部分。**

工具跑完之后，系统不是简单地把结果打印给用户，而是会把它们转成后续模型可消费的消息材料。

## 第 11 步：工具结果之外，还会注入 memory、queued commands 和其他附件

这是 Claude Code 很容易被低估的一层。

工具执行之后，系统还会把很多额外信息装配进下一轮输入，包括：

- attachment messages
- queued commands snapshot
- memory prefetch 结果
- skill discovery 结果
- file change attachments

这些逻辑主要都发生在 `query.ts` 的工具执行之后、下一轮递归之前。

也就是说，下一轮模型看到的并不只是“tool_result”，还包括很多运行时补充信息。

这一点特别重要，因为它决定了 Claude Code 为什么能在多轮中持续保持“知道自己刚刚做了什么、外部世界发生了什么、有哪些补充上下文应该被重新喂给模型”。

## 第 12 步：组装下一轮 state，然后 `continue`

到这里，Claude Code 会把下一轮状态重新装配成：

- `messages: [...messagesForQuery, ...assistantMessages, ...toolResults]`
- 更新后的 `toolUseContext`
- 新的 `turnCount`
- `pendingToolUseSummary`
- `transition: { reason: 'next_turn' }`

然后再次回到 `while (true)` 顶部。

这正是整套 agent loop 的本质：

**不是递归调用模型，而是持续更新一份运行时状态，然后让状态驱动下一轮决策。**

## 为什么这一套设计比“工具聊天机器人”强很多

### 1. 它把用户输入、工具执行、恢复机制放进同一个循环

很多实现会把这些东西拆成互相不一致的多段逻辑：

- 输入一套逻辑
- 工具另一套逻辑
- 异常恢复再单独补几条 if

Claude Code 则更接近一个统一 runtime。

### 2. 它明确区分了消息历史、query input 和 UI transcript

这几个在简单系统里常常被混成一份数组。

Claude Code 里它们虽然互相关联，但并不完全相同：

- 会话历史会长期保存
- 发给模型前会重新投影和裁剪
- UI 还可能保留更多 scrollback 或中间态消息

这使得系统可以同时满足：

- 模型可用
- 用户可读
- 恢复可做
- 性能可控

### 3. 它把“继续下一轮”视为正常路径，而不是异常路径

一旦进入 agent 系统，继续下一轮应该是常态，不应该是 hack。

Claude Code 的 `query()` 设计已经非常清楚地表达了这一点。

## 如果你自己要实现一个最小版 agent loop，最先应该保留什么

如果你不打算一开始就实现 Claude Code 那么复杂的版本，我建议最少保留这几个结构：

1. 一个统一的 `LoopState`
2. 一个统一的 `ToolUseContext`
3. 输入规范化层，负责产出 `messages + shouldQuery`
4. 一个显式的“模型输出里出现 `tool_use` 就继续”的循环
5. 一个把 tool result 回注进下一轮消息数组的机制

你可以先不做：

- MCP
- plugins
- skills
- context collapse
- reactive compact
- token budget

但上面那五个边界最好一开始就立住。否则你的系统很容易长成一堆彼此耦合的 if/else。

## 建议的源码阅读顺序

如果你准备顺着这一章继续下钻，推荐按下面顺序读：

1. `src/screens/REPL.tsx`
   重点看 `onSubmit`、`onQuery`、`onQueryImpl`
2. `src/utils/handlePromptSubmit.ts`
   重点看 `handlePromptSubmit()`、`executeUserInput()`
3. `src/utils/processUserInput/processUserInput.ts`
   重点看输入如何变成 `messages` 和 `shouldQuery`
4. `src/query.ts`
   这是整条主循环真正的核心
5. `src/QueryEngine.ts`
   再看它如何把同一个核心包成 SDK / headless 入口

## 本篇结论

这篇文章最重要的结论是三条：

1. Claude Code 的 agent loop 是一个持续更新状态的执行循环，而不是一次模型调用。
2. 交互式 REPL 和 SDK 路径虽然入口不同，但都围绕 `query()` 这个共享核心组织。
3. tool use、memory、attachments、恢复逻辑、stop hooks、budget continuation 都不是外围 patch，而是这个 loop 的正式组成部分。

下一篇我们继续沿这条主线往下拆，但会把焦点收窄到“模型每轮到底看到了什么”，也就是上下文管理系统。
