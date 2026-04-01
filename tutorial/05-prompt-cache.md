# 05. Prompt Cache：Claude Code 到底缓存了什么

## 这篇文章要回答什么

前面几章我们讲了：

- Claude Code 每轮会带什么上下文进去
- 上下文太长时它怎么做 compact

但只讲“带什么进去”还不够。真正决定 Claude Code 为什么能在长会话里保持还不错的响应速度、为什么 fork subagent 没有想象中那么贵、为什么有些 mode 切换会让下一轮突然变慢的，是另一条更底层的机制：

**prompt cache。**

这篇文章要把下面几个问题一次讲清：

- `cache key` 到底是什么
- Claude Code 每轮请求里，哪些部分是真正的 cache-critical prefix
- `tools[]` 到底会不会进入 prompt cache
- 权限 mode、MCP、ToolSearch、subagent 为什么有时会打碎缓存，有时不会
- 能不能把 Claude Code 理解成“后台同时存在很多 prompt cache”

如果把这件事讲歪了，后面几乎所有关于工具系统、权限系统、subagent、context compaction 的理解都会变形。

## 先给结论

先把结论摆在前面。

### 1. `cache key` 不是 transcript

Claude Code 并不是把“完整 transcript 文本”扔给服务端，然后服务端按整段聊天记录做缓存。

从当前源码里能看到，真正的 cache-critical 前缀至少包含这些维度：

- `system prompt`
- `tools`
- `model`
- `messages(prefix)`
- `thinking config`

最硬的证据在 `src/utils/forkedAgent.ts` 顶部注释里，源码直接写明：

```text
The Anthropic API cache key is composed of:
system prompt, tools, model, messages (prefix), and thinking config.
```

也就是说，`tools[]` 不是“附加元数据”，而是 cache key 的一部分。

### 2. 不是“一个 session 只有一个 prompt cache”

更准确的理解应该是：

- 每一个不同的 cacheable prefix，都可能对应一个不同的服务端 cache entry
- 同一个 session 随着消息增长、工具变化、system prompt 变化，会不断生成新的 prefix
- 不同 query source，例如主线程、SDK、subagent，也会有各自独立演化的一串 prefix

所以从工程心智模型上，你完全可以把 Claude Code 理解成：

**它不是在“维护一个 prompt cache 对象”，而是在不断生成和复用一系列前缀缓存。**

### 3. `tools[]` 变化确实可能打碎缓存

而且打碎得很早。

`src/utils/toolSchemaCache.ts` 里有一句很关键的注释：

```text
Tool schemas render at server position 2 (before system prompt), so any byte-level
change busts the entire tool block AND everything downstream.
```

这句话非常重要。它告诉你：

- tool schema 在请求里位置很靠前
- 一旦 tool bytes 改变，后面的 system prompt 和 messages prefix 也没法沿用同一条前缀命中

### 4. 但不是每次切 mode 都会打碎工具缓存

很多人会误以为：

`default -> acceptEdits -> plan -> auto`

每切一次 mode，`tools[]` 就一定重写一遍并造成 cache miss。

这不准确。

很多 mode 切换只改变：

- 运行时权限语义
- tool execution 时的 allow / ask / deny 行为

而不是工具集合本身。

这时即使权限行为变了，`tools[]` 也可能保持字节稳定，缓存仍然能继续命中。

## 先把三层东西分开

理解 prompt cache 之前，必须先把下面三层分开：

```text
1. transcript / session log
   - 本地保存的消息历史

2. API request payload
   - 每一轮临时构造的 tools + system + messages

3. server-side prompt cache
   - 服务端按请求前缀复用的缓存
```

很多误解都来自把 1 和 2 混成一件事。

例如：

- transcript 里未必长期保存一份完整 `tools[]`
- 但每轮 API 请求仍然会重新带上本轮的 `tools[]`
- prompt cache 命中的，是请求前缀，不是本地 transcript 文件

## `cache key` 到底是什么

如果用一句工程上最实用的话来定义：

**`cache key` 就是服务端用来回答“这段前缀我之前算过没有”的身份指纹。**

当前源码虽然没有把 Anthropic 服务端的内部实现完整暴露出来，但 Claude Code 客户端已经把 cache-critical 维度暴露得相当清楚了。

最核心的来源有三处：

- `src/utils/forkedAgent.ts`
- `src/utils/toolSchemaCache.ts`
- `src/services/api/promptCacheBreakDetection.ts`

尤其是 `promptCacheBreakDetection.ts`，它会按这些维度做变化检测：

- `systemHash`
- `toolsHash`
- `cacheControlHash`
- `model`
- `fastMode`
- `globalCacheStrategy`
- `betas`
- `effortValue`
- `extraBodyHash`

这不等于“这些字段一定逐字参与 Anthropic 内部 cache key 计算”，但它已经足够说明：

- 这些字段里相当一部分属于 cache-sensitive 维度
- Claude Code 自己就在严密监控哪些变化会导致 cache break

## 一次请求里，哪些内容会被当成 cache-critical prefix

从 Claude Code 这一侧看，一次请求的大致结构可以画成这样：

```text
+----------------------------------------------------------------------------------+
|                               API Request Payload                                |
+----------------------------------------------------------------------------------+
| tools[]                                                                          |
| system[]                                                                         |
| messages[]                                                                       |
| model                                                                            |
| thinking / effort / output config / extra body params                            |
+----------------------------------------------------------------------------------+
```

再细化一点：

```text
+----------------------------------------------------------------------------------+
| 1. tools[]                                                                       |
|    - built-in tools                                                              |
|    - MCP tools                                                                   |
|    - defer_loading overlays                                                      |
|                                                                                  |
| 2. system blocks                                                                 |
|    - system prompt prefix                                                        |
|    - static system blocks                                                        |
|    - dynamic system blocks                                                       |
|                                                                                  |
| 3. messages prefix                                                               |
|    - 历史 user / assistant messages                                              |
|    - tool_result                                                                 |
|    - attachments / meta messages                                                 |
|    - 其中某个位置打 cache_control marker                                         |
|                                                                                  |
| 4. model / thinking / effort                                                     |
+----------------------------------------------------------------------------------+
```

这里面最该记住的是：

- `tools[]` 不是旁路信息，它就在 cache-critical prefix 里
- `system` 不是一个单块字符串，而是会被拆成多个 block
- `messages` 也不是“全部一起缓存”，而是通过 marker 定义出一个可复用前缀

## 全流程：Claude Code 是怎么把 prompt cache 用起来的

下面按真正的请求构建顺序走一遍。

### 第 1 步：先组本轮工具池

Claude Code 会先根据当前运行态组装本轮可用工具。

主要入口在：

- `src/tools.ts`
- `src/utils/toolPool.ts`
- `src/screens/REPL.tsx`

链路大致是：

```text
toolPermissionContext
  + built-in tools
  + MCP tools
  + coordinator / REPL / brief / deferred loading 过滤
    -> assembled tool pool
```

这里最关键的不是“有哪些工具”，而是 Claude Code 为了 prompt cache 稳定性做了两件事：

1. 工具顺序稳定化
2. built-in tools 保持为连续前缀

`src/tools.ts` 和 `src/utils/toolPool.ts` 都明确写了：

- built-in tools 必须保持连续前缀
- 否则 MCP tools 插进来会破坏服务端 cache policy

这就是为什么它会刻意 partition-sort，而不是简单做一次扁平排序。

### 第 2 步：把工具转成稳定的 tool schema

接下来每个工具都会被转成 API schema，入口是：

- `src/utils/api.ts` 里的 `toolToAPISchema()`

它做了两层处理：

#### 第一层：session-stable base schema

`toolToAPISchema()` 会先生成一个基础 schema：

- `name`
- `description`
- `input_schema`
- `strict`
- `eager_input_streaming`

然后把这个结果缓存在 session 级 `toolSchemaCache` 里。

目的非常明确：

- 防止 `tool.prompt()` 文案漂移
- 防止 GrowthBook / feature gate 中途变化
- 防止工具数组字节抖动

这是 prompt cache 能稳定工作的前提之一。

#### 第二层：per-request overlay

之后它再叠本轮特有的字段：

- `defer_loading`
- `cache_control`

源码注释写得很清楚：

```text
Per-request overlay: defer_loading and cache_control vary by call
```

也就是说，Claude Code 在主动把“尽量稳定的部分”和“本轮才变化的部分”拆开。

### 第 3 步：对 deferred tools 做特殊处理

当 ToolSearch / deferred loading 开启时，不是所有工具都会真的进入当前请求。

`src/services/api/claude.ts` 会先做：

- 发现本轮哪些 deferred tools 已经被发现
- 只把已发现的 deferred tools 放进 `filteredTools`

这里最容易误解的地方是：

- “deferred tool 状态变了，缓存是不是一定破了”

Claude Code 专门处理了这个问题。

在 `promptCacheBreakDetection.ts` 里，它会把 `defer_loading` tools 从 hash 里排除。原因在 `src/services/api/claude.ts` 里也写得很直白：

```text
the API strips them from the prompt, so they never affect the actual cache key
```

这意味着：

- deferred tool 的发现状态可能影响当前可调用能力
- 但不一定影响真实 prompt cache key

这是一个非常重要的优化。

### 第 4 步：构造 system prompt blocks

system prompt 不是一整段大字符串直接塞进去。

Claude Code 会先把它拆块，再决定每一块的缓存范围。

核心入口在：

- `src/utils/api.ts` 的 `splitSysPromptPrefix()`
- `src/services/api/claude.ts` 的 `buildSystemPromptBlocks()`

它大致有三种形态：

1. 默认模式
2. 带 dynamic boundary 的 global cache 模式
3. MCP 动态工具存在时的保守模式

可以把它想成：

```text
system prompt
  -> attribution header
  -> CLI system prompt prefix
  -> static blocks
  -> dynamic blocks
```

然后 Claude Code 再给不同 block 打不同的 `cache_control(scope, ttl)`。

源码注释已经说明了它的目标：

- 静态前缀尽量复用
- 动态后缀尽量后置
- 避免让动态 MCP 部分污染本应稳定的 global prefix

### 第 5 步：给 messages 加 cache marker

光有 `tools` 和 `system` 还不够，历史消息本身也要告诉服务端：

**“从哪里开始，这一段前缀是我希望你缓存并复用的。”**

Claude Code 做这件事的入口在：

- `src/services/api/claude.ts` 的 `addCacheBreakpoints()`

这里有几个非常关键的点：

#### 1. 每个请求只放一个 message-level cache marker

源码直接写了：

```text
Exactly one message-level cache_control marker per request.
```

也就是说，它不是在每条消息上都乱打 marker，而是非常克制地只放一个。

#### 2. 这个 marker 通常放在最后一条消息

默认情况下：

- marker 在最后一条 message 上

但在某些 fire-and-forget fork 场景下：

- 它会放到倒数第二条

目的是：

- fork 共享父线程前缀
- 自己不污染新的写入尾巴

#### 3. marker 之前的 tool_result 会补 `cache_reference`

这一步也很关键。

Claude Code 会给 marker 之前的 `tool_result` 补上：

- `cache_reference: tool_use_id`

它的作用可以粗略理解为：

- 让已经在 cached prefix 内的工具结果更稳定地被复用和引用

### 第 6 步：发送请求，服务端按前缀查 cache

当 `tools[]`、`system[]`、`messages[]`、`model` 都准备好后，请求才真正发出。

这时服务端会做的事，从心智模型上可以理解成：

```text
拿本轮请求的 cache-critical prefix
  -> 算身份
  -> 看之前有没有相同前缀
  -> 有则读取 cached prefix
  -> 无则正常推理，并把新前缀写入 cache
```

客户端看不到 Anthropic 服务端的全部内部实现，但 Claude Code 本地会通过 cache break detection、cache read token 下降、prefix hash 变化去推断有没有发生 cache miss。

### 第 7 步：本轮不会帮本轮命中，只会帮下一轮变快

这是很多人第一次做 agent runtime 时最容易忽略的点。

**本轮 assistant 输出不会让本轮更快。**

真正发生的是：

- 本轮请求写热了一个新的前缀
- 下一轮如果前缀能复用，这些缓存才开始起作用

所以 prompt cache 本质上是：

**上一轮为下一轮预热。**

## 可以把 Claude Code 理解成“后台有多个 prompt cache”吗

可以，但要讲清楚“多个”的含义。

更准确地说，有三层“多个”。

### 第一层：同一个 session 会不断生成多个 cache entry

假设主线程连续跑三轮：

```text
Turn 1 -> prefix A
Turn 2 -> prefix B = A + new messages
Turn 3 -> prefix C = B + new messages
```

你可以把它理解成：

- 服务端可能已经有 A 对应的缓存
- 第二轮再写出 B
- 第三轮再写出 C

这不是一个单一缓存对象，而是一条不断增长的缓存链。

### 第二层：不同 query source 各有各的前缀序列

`src/services/api/promptCacheBreakDetection.ts` 里，Claude Code 会按 source 单独跟踪上一次状态，例如：

- `repl_main_thread`
- `sdk`
- `agent:custom`
- `agent:default`

这说明至少从客户端的检测心智看：

- 主线程有主线程的 prefix 演化
- subagent 有 subagent 的 prefix 演化
- SDK 调用也单独算

所以如果你问“是不是同时存在多个 prompt cache 轨道”，答案是：**可以这么理解。**

### 第三层：单个请求内部也会有多个 cacheable block

system prompt 并不是一个整块缓存。

在 Claude Code 的实现里，它会被拆成多个 block，并且每个 block 的 `cacheScope` 可能不同：

- `global`
- `org`
- `null`

这意味着同一个请求内部，本身就存在“分段缓存”的概念。

所以更准确的图景不是：

```text
一个 session
  -> 一个 prompt cache
```

而是：

```text
一个 session
  -> 多轮请求
  -> 每轮都有一段 cache-critical prefix
  -> prefix 内部又可能分成多个 cacheable block
```

## `tools[]` 一旦变化，到底会发生什么

这是最关键的一段。

我们分成四种场景看。

### 场景 A：mode 变了，但工具 schema 没变

例如：

- `default -> acceptEdits`
- `acceptEdits -> dontAsk`

这类变化常常只改：

- 运行时 allow / ask / deny 语义
- classifier 是否介入

而不改：

- 工具名字
- 工具输入 schema
- 工具描述文本

这时即使权限行为变了，`toolsHash` 也可能保持稳定。

换句话说：

**mode 变化不等于 tool block 必然变化。**

### 场景 B：工具集合真的变了

这类才是典型的工具缓存打碎场景，例如：

- 加了 blanket deny rule，把某个工具过滤掉
- MCP server 连接或断开，真实 MCP tools 变了
- 进入 coordinator mode，工具池被裁掉
- REPL wrapper 把 primitive tools 隐藏掉

此时发生的事情是：

```text
上一轮:
  tools[] = T1

本轮:
  tools[] = T2

结果:
  tool block 的字节变了
  -> 从 tool block 这一层开始 miss
  -> 后面的 system / messages prefix 也跟着失去同一条前缀命中
```

这就是为什么工具变化非常“贵”。

### 场景 C：deferred tools 发现状态变化

这是一个容易误判的场景。

如果变化的是：

- `defer_loading: true` 的工具是否已被发现

Claude Code 会尽量不把它算成真正的 cache break，因为 API 最终会把这类工具从实际 prompt 里剥掉。

所以：

- 可用能力可能变化
- 真实 prompt cache key 不一定变化

### 场景 D：动态 MCP 工具真正参与渲染

这时即使不是“工具名大规模变化”，缓存策略也可能变得更保守。

`src/services/api/claude.ts` 里有一个很重要的判断：

- MCP tools 是 per-user 的动态 section
- 真正会 render 时，不适合继续把 system prompt 当成全局稳定前缀来缓存

所以动态 MCP 工具不仅可能让工具数组变化，还可能改变 system prompt 的缓存策略。

## 一个容易混淆的问题：tool 被 deny 后，模型下一轮到底会看到什么

很多人会下意识地把“permission denied”理解成：

- 某个隐藏状态位变了
- 或者 transcript 里多了一条 UI 通知

从模型上下文视角看，这两种理解都不准确。

真正会进入下一轮上下文的核心变化是：

**Claude Code 会回注一条新的 `user` 消息，里面带一个失败的 `tool_result` block。**

也就是说，模型看到的不是抽象的“权限系统拒绝了”，而是一个非常具体的 tool 调用结果：

```text
assistant:
  tool_use(id=toolu_123, name=FileEdit, input=...)

user:
  tool_result(
    tool_use_id=toolu_123,
    is_error=true,
    content="Permission to use FileEdit has been denied ..."
  )
```

这是理解 tool denial 的关键。

### 从运行时到下一轮上下文，完整链路是什么

可以先看一条简化链路：

```text
模型发出 tool_use
  -> 运行时做权限判断
  -> allow: 真正执行工具
  -> ask/deny: 不执行工具
  -> 回注一条 user(tool_result, is_error=true)
  -> 下一轮模型看到这条失败结果
```

再展开一点：

```text
+----------------------------------------------------------------------------------+
| 1. assistant message                                                             |
|    tool_use(id=abc123, name=Bash/FileEdit/...)                                   |
+----------------------------------------------------------------------------------+
                                       |
                                       v
+----------------------------------------------------------------------------------+
| 2. runtime permission check                                                      |
|    hasPermissionsToUseTool()                                                     |
|    -> allow / ask / deny                                                         |
+----------------------------------------------------------------------------------+
                                       |
                                       v
+----------------------------------------------------------------------------------+
| 3. if not allow                                                                  |
|    不执行工具                                                                     |
|    生成 user message                                                              |
|    content[0] = tool_result(is_error=true, tool_use_id=abc123, content=...)      |
+----------------------------------------------------------------------------------+
                                       |
                                       v
+----------------------------------------------------------------------------------+
| 4. normalizeMessagesForAPI                                                       |
|    把 tool_use / tool_result 配成一组                                             |
|    attachments 做重排或过滤                                                      |
+----------------------------------------------------------------------------------+
                                       |
                                       v
+----------------------------------------------------------------------------------+
| 5. next API request                                                              |
|    模型看到:                                                                     |
|    - 之前的 assistant tool_use                                                   |
|    - 紧随其后的失败 tool_result                                                  |
+----------------------------------------------------------------------------------+
```

这个回注逻辑在 `src/services/tools/toolExecution.ts`。当权限结果不是 `allow` 时，它会直接构造：

- `type: 'tool_result'`
- `is_error: true`
- `tool_use_id: <当前 tool use id>`
- `content: <拒绝原因>`

然后把它包进一条新的 `user` message 里。

### 这条失败 `tool_result` 为什么这么重要

因为从模型角度，它不是“附加说明”，而是主循环的一部分。

Claude Code 的 agent loop 不是：

- 模型发 tool_use
- 本地拒绝
- 模型神奇地知道被拒绝了

而是：

- 模型发 tool_use
- 本地拒绝
- 本地显式补一条 tool_result 回去
- 模型基于这条 tool_result 再决定下一步

所以 deny 之后，模型的心智状态会发生两层变化：

1. 它知道刚才那个具体 tool call 没执行成功
2. 它知道失败原因是什么

这也是为什么系统 prompt 里会专门提醒：

- 如果用户拒绝了你调用的工具，不要原样重试
- 应该调整方案或向用户解释为什么需要这个权限

### 拒绝类型不同，回注的文本也不同

这里要分几类。

#### 1. 用户在 permission dialog 里拒绝

这种情况下，回注文本通常是：

- `REJECT_MESSAGE`
- 或 `REJECT_MESSAGE_WITH_REASON_PREFIX + 用户反馈`

如果用户填写了补充说明，这段说明会直接拼进拒绝消息正文里，成为下一轮上下文的一部分。

也就是说，模型不仅知道“被拒绝了”，还会看到：

- 用户为什么拒绝
- 用户希望你怎么改方案

#### 2. `dontAsk` 或静态规则直接 deny

这种情况下，不一定经过交互式 permission dialog。

回注文本会变成更直接的拒绝原因，例如：

- `Permission to use <tool> has been denied because Claude Code is running in don't ask mode.`
- 或基于 deny rule / working directory / safety check 生成的拒绝文本

这类 deny 的核心特点是：

- 模型能看到“拒绝原因”
- 但一般拿不到用户的自由文本反馈

#### 3. auto mode classifier 拒绝

这类拒绝在 UI 里会被压缩显示成很短的一句，例如：

- `Denied by auto mode classifier`

但那是显示层，不是模型真正看到的原文。

模型上下文里仍然会有那条错误 `tool_result`，只是 UI 为了不把 transcript 渲染得过长，做了特殊识别和简写。

#### 4. plan approval 被拒绝

这和普通 tool deny 不完全一样，但本质上仍然是：

- 用一个失败结果告诉模型“不要继续执行”

只不过它用的是另一套前缀文本，用来明确告诉模型：

- 当前仍然留在 plan mode
- 这次不是实现被批准，而是计划被拒绝了

### 除了这条 `tool_result`，还会多别的东西吗

会，但要区分“会进模型”和“只给 UI/SDK”。

#### 会进模型的

核心只有一类：

- 那条新的 `user` message，其中包含失败的 `tool_result`

在某些 `ask` 场景下，它还可能带额外的用户反馈内容块。

#### 不会进模型的

有几类是本地辅助信息，不会进入下一轮 API 请求：

- `hook_permission_decision` attachment
- 一些只为 transcript/UI 服务的显示消息
- `toolUseResult`、`sourceToolAssistantUUID` 这类本地消息对象上的附加元数据

尤其要注意 `hook_permission_decision`。

运行时在某些 hook 决策路径下会额外插一条 attachment，告诉 UI：

- 这次权限决定来自 hook
- 结果是 allow / deny

但它在 API 归一化阶段会被直接过滤掉，不会进入模型上下文。

### deny 之后，这条消息会怎么进入下一轮 API

Claude Code 在 `normalizeMessagesForAPI()` 里会做两件关键事：

1. 重排 attachment
2. 把 `tool_use` 和对应的 `tool_result` 重新配对整理

所以从下一轮模型视角看，它拿到的不是一堆松散消息，而是一条结构上完整的轨迹：

```text
assistant(tool_use X)
user(tool_result for X, is_error=true)
```

这正是 Claude 风格 tool loop 能持续推进的基础。

### deny 对 prompt cache 有什么影响

这件事经常被忽略。

很多人会只盯着 `tools[]` 有没有变化，但 deny 本身就已经会改 `messages prefix`。

因为 deny 会新增一条 `user(tool_result)`，所以下一轮请求的消息前缀一定不同于上一轮。

换句话说：

- 即使工具集合完全没变
- 一次 deny 也通常会生成一个新的 prefix

这不一定意味着“缓存彻底失效”，但至少意味着：

- 这已经不是上一轮的同一条请求前缀了

所以当你观察到某一轮被拒绝后，下一轮的 cache 行为发生变化，不要只检查：

- tools 变没变

还要检查：

- messages prefix 是不是刚多了一条失败 `tool_result`

### 一句话压缩这个问题

从上下文视角看，tool 被 deny 后，Claude Code 不是给模型塞一个抽象的“permission state changed”信号。

它做的是更简单也更可靠的事：

**把这次失败显式写成一条 `tool_result(is_error=true)`，再把它作为下一轮上下文的一部分发回给模型。**

## 一个完整例子：`acceptEdits -> plan` 时到底发生了什么

这是理解“tool 变化”和“cache 变化”不是一回事的最好例子。

### 先看权限行为

在 `acceptEdits` 下，写文件权限里有一个非常关键的快速放行分支：

- 如果路径在允许工作目录内
- 且 `mode == acceptEdits`
- 直接 allow

这个逻辑在 `src/utils/permissions/filesystem.ts`。

一旦进入 `plan`：

- 这个自动放行分支就失效了
- 只剩下 plan file 等内部可编辑路径的特殊白名单
- 普通写代码文件重新回到 ask / rule / deny 判定

所以从“权限执行”看：

- `acceptEdits` 能自由写工作区文件
- `plan` 只能写 plan file，不能直接开始实现

### 再看模型上下文

进入 `plan` 时，Claude Code 不一定会删掉写工具名字，但会注入很强的 plan-mode 约束：

- 不得编辑任何文件，除了 plan file
- 不得运行非只读工具
- 需要在最后调用 `ExitPlanMode`

这部分约束来自：

- `EnterPlanModeTool` 的 tool result
- 后续持续注入的 `plan_mode` attachments

### 最后看 cache

这里最容易讲错。

`acceptEdits -> plan` 之后，下一轮 cache miss 往往不是因为“工具一定变了”，而更可能来自这几件事：

1. `messages` 里多了 `plan_mode` attachment 或相关 meta 指令
2. `model` 可能切换了
3. `system` / message prefix 因 mode 相关指令发生变化

尤其是模型切换这一点，经常被忽略。

`src/query.ts` 会根据 `permissionMode` 调 `getRuntimeMainLoopModel()`；而 `src/utils/model/model.ts` 里明确写了：

- `opusplan` 配置下，plan mode 会改用 Opus

所以：

```text
acceptEdits -> plan

权限层:
  写工具不再自动放行

工具层:
  tools[] 可能不变

模型层:
  model 可能变

消息层:
  plan_mode attachments 会变

缓存结果:
  很可能 miss
  但 miss 的根因未必是 tools[]
```

这就是为什么不能把“工具权限变化”和“tool block 变化”混成一回事。

## Claude Code 为了少打碎 cache，做了哪些工程化处理

从源码看，它至少做了下面这些很有针对性的设计。

### 1. 工具顺序稳定化

目的：

- 保持 built-in tools 的连续前缀
- 降低 MCP 工具插入造成的下游 cache churn

### 2. tool schema session 级 memoize

目的：

- 防止 `tool.prompt()` 文案漂移
- 防止 feature gate mid-session 刷新
- 防止工具字节抖动

### 3. system prompt 分块

目的：

- 静态前缀尽量保持稳定
- 动态部分尽量后置

### 4. 只放一个 message-level cache marker

目的：

- 明确可复用前缀边界
- 避免多 marker 造成奇怪的局部 attention / eviction 行为

### 5. 对 TTL、beta headers 做 sticky / latch

`src/bootstrap/state.ts` 里专门把这些状态做成 session-stable：

- 1h TTL eligibility
- AFK mode beta header
- fast mode beta header
- cache-editing beta header

原因很简单：

- 如果这些东西 mid-session 翻来覆去变，哪怕正文没变，cache 也会被平白打碎

### 6. 对 `defer_loading` 做特殊处理

目的：

- 把“能力发现的抖动”和“真实 prompt bytes 的变化”尽量分离

这是 ToolSearch 能成立的重要前提。

## 如果你自己实现一个 Claude Code-like 系统，应该怎么做

如果你的目标不是做研究，而是真的做产品，我建议把 prompt cache 这层按下面思路设计。

### 最小可用版本

先做这四件事：

1. 把 `tools`、`system`、`messages` 显式拆开
2. 固定工具顺序
3. 把 system prompt 拆成静态前缀和动态后缀
4. 只在一个位置打 message-level cache marker

做到这里，你的缓存行为已经会比“每轮拼一大串 prompt 字符串”稳定很多。

### 第二阶段再做

再做这些：

1. session 级 tool schema cache
2. 动态工具的 deferred loading
3. `cache_reference` / `cache_edits`
4. prompt cache break detection

### 最后才做

最后再考虑：

- 多 query source 的独立 tracking
- forked agent 的 cache sharing
- 复杂的 global / org scope 策略

因为这些东西做得太早，很容易在还没把基础 request 构型理顺时就把系统复杂度炸开。

## 最后再压一句结论

Claude Code 的 prompt cache 不是一个“后台神秘加速开关”，而是它整个 request 构型设计的直接结果。

你只要记住三句话就够了：

1. `cache key` 不是 transcript，而是请求前缀
2. `tools[]` 明确是 cache-critical prefix 的一部分
3. Claude Code 的很多工程细节，本质上都在做一件事：让这个前缀尽量稳定

所以如果你以后看到：

- mode 切换
- MCP 连接
- subagent fork
- tool discovery
- prompt cache hit rate 波动

不要先问“模型是不是变笨了”。

先问：

**这次请求的 prefix，到底哪里变了。**
