# Context + Memory 研究笔记

## 这份笔记解决什么问题

Claude Code 的“上下文”不是单一 prompt，也不是简单的消息历史。

从源码看，它至少拆成四层：

1. 每轮都会带上的静态上下文注入
2. 查询进行中异步召回的相关记忆
3. 当前长会话内部的 session memory
4. 会后或 stop hook 触发的 durable auto-memory 提取

后续写教程时，`Context` 和 `Memory` 这两章不能混成一个章节内部的大杂烩。更准确的写法应该是：

- `Context`：模型这一轮真正看到什么，以及这些内容如何拼进 query
- `Memory`：哪些信息会跨轮、跨会话保留下来，以及何时被召回

## 核心结论

Claude Code 的设计不是“把所有记忆都直接塞进 system prompt”，而是分层处理：

- 轻量、稳定、必须每轮都在的内容，走 `getSystemContext()` / `getUserContext()` 这条静态注入链
- 体积更大、相关性不稳定的历史记忆，走 `startRelevantMemoryPrefetch()` 这条异步召回链
- 当前会话需要被压缩成工作笔记时，走 `SessionMemory` hook
- 想沉淀成未来会话可复用的 durable memory 时，走 `extractMemories` forked-agent

换句话说，Claude Code 的 memory 不是一个功能点，而是一组不同时间尺度的 persistence 机制。

## 必读源码入口

### 静态上下文注入

- `../claude-code-sourcemap/restored-src/src/context.ts`
- `../claude-code-sourcemap/restored-src/src/utils/claudemd.ts`
- `../claude-code-sourcemap/restored-src/src/main.tsx`
- `../claude-code-sourcemap/restored-src/src/screens/REPL.tsx`

### Memory prompt 与记忆类型约束

- `../claude-code-sourcemap/restored-src/src/memdir/memdir.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/memoryTypes.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/paths.ts`

### 动态相关记忆召回

- `../claude-code-sourcemap/restored-src/src/utils/attachments.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/findRelevantMemories.ts`
- `../claude-code-sourcemap/restored-src/src/memdir/memoryScan.ts`
- `../claude-code-sourcemap/restored-src/src/query.ts`

### Session memory 与 durable extraction

- `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemory.ts`
- `../claude-code-sourcemap/restored-src/src/services/SessionMemory/sessionMemoryUtils.ts`
- `../claude-code-sourcemap/restored-src/src/services/SessionMemory/prompts.ts`
- `../claude-code-sourcemap/restored-src/src/services/extractMemories/extractMemories.ts`
- `../claude-code-sourcemap/restored-src/src/utils/memoryFileDetection.ts`

## 1. 静态上下文注入链

### 1.1 入口长什么样

`REPL.tsx` 在启动 query 前，会并行取三类东西：

- `getSystemPrompt(...)`
- `getUserContext()`
- `getSystemContext()`

这说明 Claude Code 从一开始就把“system prompt 主体”和“补充上下文字段”分开构造，而不是把所有内容拼成一个超大字符串后再传给 `query()`。

### 1.2 `getSystemContext()` 负责什么

`context.ts` 里的 `getSystemContext()` 是 memoized 的，当前会话内只算一次，核心内容很克制：

- git status snapshot
- branch / main branch / recent commits
- ant 内部用的 cache breaker 注入

这里有两个重要边界：

- 这是“会话开始时的 snapshot”，不会随会话自动刷新
- remote 模式或 git instructions 被关闭时，不会去抓 git status

这意味着 Claude Code 并不尝试让 prompt 永远反映“最新 git 状态”，而是把它当作首轮定位上下文。

### 1.3 `getUserContext()` 负责什么

`getUserContext()` 同样 memoized，主要产物只有两个：

- `claudeMd`
- `currentDate`

其中 `claudeMd` 不是单文件，而是 `getMemoryFiles()` 找到的一组文件经过 `filterInjectedMemoryFiles()` 和 `getClaudeMds()` 后的拼接结果。

这条链说明“用户上下文”在 Claude Code 里主要就是：

- CLAUDE.md 家族
- 自动注入的日期

而不是整个项目索引、文件树摘要之类的大上下文。

### 1.4 CLAUDE.md 的加载顺序

`getMemoryFiles()` 的处理顺序很明确：

1. `Managed`
2. `User`
3. 自根目录向当前工作目录下行遍历得到的 `Project`
4. 同路径上的 `Local`
5. 可选的 `--add-dir`
6. memdir 的 `MEMORY.md`
7. team memory 的 `MEMORY.md`

这里最值得后续正文强调的不是“有很多路径”，而是两个原则：

- checked-in instruction 与 private instruction 被明确区分
- 自动记忆入口 `MEMORY.md` 也被视作一类可注入上下文，而不是完全独立于 prompt 系统

### 1.5 静态注入不是永远把 `MEMORY.md` 塞进去

`filterInjectedMemoryFiles()` 有个关键 feature flag：

- 当 `tengu_moth_copse` 关闭时，`AutoMem` / `TeamMem` 会像普通注入文件一样进 prompt
- 当它打开时，会跳过这些 entrypoint，因为相关记忆会改走动态 attachments 召回链

这点非常关键。它说明 Claude Code 自己也在迭代 memory 设计，从“全量 index 注入”逐渐转向“按需召回”。

### 1.6 启动阶段的预取策略

`main.tsx` 的 `startDeferredPrefetches()` 会在首屏渲染后启动预取：

- 总是预取 `getUserContext()`
- 只在 non-interactive 或 trust 已建立时预取 `getSystemContext()`

这体现出一个工程取舍：

- `UserContext` 更像便宜、低风险的缓存预热
- `SystemContext` 里包含 git 等潜在副作用更强的内容，所以要受 trust gate 约束

## 2. Memory prompt 其实定义了“什么叫记忆”

### 2.1 `memdir.ts` 不是工具函数文件，而是产品策略文件

如果只看架构图，很容易把 `memdir.ts` 当成普通路径/IO 辅助层。但它实际上编码了 Claude Code 对 memory 的产品定义：

- memory 是文件系统里的持久信息
- `MEMORY.md` 是 index，不是正文
- memory 要与 task / plan / 当前对话状态严格区分
- memory 不是让模型记住 repo 里本来就能推导出的东西

这些规则都直接写在 `buildMemoryLines()` / `buildMemoryPrompt()` 产出的 prompt 里。

### 2.2 `MEMORY.md` 的角色

源码里对 `MEMORY.md` 的定义很清楚：

- 它永远是入口 index
- 每条记录应该是一行 link + hook
- 它总会进上下文，但会被行数和字节数截断
- 真正的 memory 内容应该写在 topic file，不直接堆到 `MEMORY.md`

这套设计很值得复刻，因为它把“发现入口”和“正文内容”分离了，避免单文件无限膨胀。

### 2.3 记忆类型是被强约束的

`memoryTypes.ts` 只允许四类 memory：

- `user`
- `feedback`
- `project`
- `reference`

同时明确禁止把下面这些东西写成 memory：

- 代码模式、架构、文件路径、项目结构
- git 历史和最近改动
- debugging recipe
- 已在 `CLAUDE.md` 里存在的信息
- 当前任务的临时细节

这说明 Claude Code 的 memory 不是“第二个知识库”，而是补 current project state 不能直接推出的长期协作信息。

### 2.4 召回也被强约束

`WHEN_TO_ACCESS_SECTION` 和 `TRUSTING_RECALL_SECTION` 给了两个很强的行为约束：

- 用户要求忽略 memory 时，要把 `MEMORY.md` 视为空
- memory 里提到的 file / function / flag，在拿来推荐之前必须重新验证

这个设计很成熟，因为它承认 memory 会漂移，而不是假设 memory 永远可靠。

## 3. 动态相关记忆召回链

### 3.1 触发方式

`query.ts` 在每个用户 turn 的开头调用：

- `startRelevantMemoryPrefetch(state.messages, state.toolUseContext)`

它不是每轮 loop iteration 都重新跑，而是每个用户 turn 只发起一次 prefetch。

### 3.2 为什么是 prefetch

`startRelevantMemoryPrefetch()` 的设计目的很明确：

- 在主模型 streaming 和工具执行期间并行做 memory 检索
- collect point 只检查 `settledAt`
- 如果还没好，就跳过本轮，下一 iteration 再试
- 整个过程不阻塞 turn

这跟把“召回”做成 query 前的同步步骤完全不同。Claude Code 这里明显在优化首 token 延迟和主循环吞吐。

### 3.3 召回范围不是整个记忆库正文

`findRelevantMemories()` 先做的是轻量扫描：

- `scanMemoryFiles()` 递归扫描 `.md`
- 跳过 `MEMORY.md`
- 只读 frontmatter 与文件头部
- 最多保留 200 个，按 `mtime` 新到旧排序

然后它把 manifest 发给 `sideQuery()`，让一个 Sonnet 侧查询挑最多 5 个最相关文件。

也就是说，Claude Code 的动态召回不是 embedding 检索，而是：

1. 文件级 manifest 预筛
2. 小模型或 side model 选择文件
3. 再读取被选中文件内容

这是很工程化、也很容易复刻的一条路线。

### 3.4 召回结果如何进主对话

召回结果不是直接改 system prompt，而是形成 `relevant_memories` attachment。

在 `query.ts` 的 post-tools 阶段，如果 prefetch 已经完成，就会：

1. `await pendingMemoryPrefetch.promise`
2. 经过 `filterDuplicateMemoryAttachments(...)`
3. `yield` attachment message
4. 把内容记进 `readFileState`

所以从主模型视角看，这些记忆更像“系统补充材料”，而不是 prompt 前缀的一部分。

### 3.5 去重逻辑很讲究

这里有两层去重：

- `collectSurfacedMemories()` 扫历史 attachment，避免同一轮会话里反复 surfacing 同一个 memory
- `filterDuplicateMemoryAttachments()` 再对 `readFileState` 去重，避免模型已经通过 Read/Edit/Write 看过的文件又被重复注入

更重要的一点是，源码明确把“写入 `readFileState`”放在过滤之后，避免 prefetch 自己把自己判成重复。

### 3.6 工具上下文也会影响记忆召回

`collectRecentSuccessfulTools()` 会把最近成功调用的工具名传给 `findRelevantMemories()`。

选择器 prompt 明确要求：

- 如果一个工具已经被成功使用，不要再把它的一般用法文档当成相关记忆返回
- 但如果记忆里写的是 gotcha / known issue，仍然可以召回

这说明 Claude Code 在做的不是“语义相似度最高的文档召回”，而是“对当前推理最有边际价值的补充”。

## 4. Session memory 不是 durable memory

### 4.1 它解决的不是跨会话记忆，而是长会话压缩

`setup.ts` 会在启动时调用 `initSessionMemory()`，它注册的是一个 post-sampling hook。

这条链只在下面条件满足时工作：

- 不是 bare mode 的正常启动路径
- auto-compact 开启
- query source 是 `repl_main_thread`
- gate 开启

这说明 session memory 主要服务于主 REPL 长会话，而不是所有 agent 或后台任务。

### 4.2 触发阈值是比较保守的

`shouldExtractMemory()` 的默认阈值在 `sessionMemoryUtils.ts`：

- 初始化阈值：10000 tokens
- 两次更新最少间隔：5000 tokens
- 工具调用阈值：3 次

并且它不是“只要工具调用够多就提取”，而是：

- token 增长阈值一定要满足
- 然后再看工具调用数量，或者当前 turn 是否刚好处在无工具调用的自然断点

这说明 session memory 被当作一种有成本的压缩操作，而不是每轮都跑的 summarizer。

### 4.3 提取方式

session memory 的更新流程是：

1. 建立或读取 session memory markdown 文件
2. 生成 update prompt
3. 用 `runForkedAgent()` 跑一个隔离 agent
4. 限制它只在 memory 文件上做编辑

这条路径的本质是“让一个子 agent 维护当前会话工作笔记”，而不是主 agent 自己直接总结。

### 4.4 prompt 模板的目标

`buildSessionMemoryUpdatePrompt()` 不是单纯“帮我总结一下”，而是维护一个固定模板的 dense notes。

源码注释和模板逻辑说明它要保留的是：

- 当前状态
- task specification
- files / functions
- workflow
- errors / corrections

所以它更像“长会话 working memory”，不是用户偏好或 durable collaboration memory。

## 5. Durable auto-memory 提取链

### 5.1 触发点

`extractMemories.ts` 的公开入口是 `executeExtractMemories()`，注释说明它由 stop hooks 在 query loop 结束后 fire-and-forget 调用。

这说明 durable memory 提取不是 query 主循环内部的同步步骤，而是“主任务完成后的后台收尾”。

### 5.2 为什么要单独 fork 一个 agent

`initExtractMemories()` 里的实现非常说明问题：

- 有 closure-scoped cursor：`lastMemoryMessageUuid`
- 有 overlap guard：避免并发提取
- 有 trailing run：提取过程中如果又来了新上下文，结束后再补一轮
- 用 `runForkedAgent()` 跑独立 agent
- `skipTranscript: true`
- `maxTurns: 5`

这套结构表明 durable memory 提取被当作一个独立后台任务，而不是主 agent 的附属函数调用。

### 5.3 与主 agent 的职责边界

源码里有个关键判断：

- 如果主 agent 在这段消息范围里已经自己写过 memory 文件，背景提取器就跳过这段消息

也就是说 Claude Code 的策略是：

- 主 agent 可以直接写 memory
- 但如果主 agent 没写，后台 extractor 会补提取

这是一个“主线程优先，后台兜底”的分层设计。

### 5.4 Durable extractor 看到什么

在真正跑 forked agent 前，提取器会：

- 扫 memory dir，生成 manifest
- 把 manifest 预注入 prompt
- 限制可用工具权限

`createAutoMemCanUseTool(memoryDir)` 的意图很明确：

- 允许读和搜索
- 只允许在 memory dir 内 Edit / Write

这条边界值得后面写工具与权限章节时单独回勾。

## 6. 与 `agent loop` 的连接点

最关键的连接点有四个：

1. `REPL.tsx` 在调用 `query()` 前组装 `systemPrompt + userContext + systemContext`
2. `query.ts` 在 turn 开始时发起 relevant memory prefetch
3. `query.ts` 在 post-tools 阶段消费 prefetched memory attachments
4. stop hooks 在 query loop 结束后触发 session memory / durable extraction 之类的后台持久化

因此教程里不应该把 context/memory 写成“loop 外部的静态配置”。更准确的说法是：

- 有些 context 在 loop 前注入
- 有些 context 在 loop 中途补进来
- 有些 memory 在 loop 结束后异步沉淀

## 7. 如果自己实现，可以先简化什么

最小可用克隆不需要一开始就复刻四层机制，可以按这个顺序做：

### 第一阶段

- 只实现 `CLAUDE.md` 类静态注入
- 加一个简单的 `currentDate`
- 可选加 git snapshot

### 第二阶段

- 增加 `MEMORY.md` index + topic files
- 先不做 side model relevance 选择
- 只做显式读取或简单关键词过滤

### 第三阶段

- 再补动态 relevant memory prefetch
- 让它以 attachment 形式注入，而不是每轮重写 system prompt

### 第四阶段

- 最后再补 session memory 和 stop-hook durable extraction

如果一开始就把“长期记忆”和“当前会话压缩”做成一个系统，后面通常会很难解释边界，也很难控制 token 成本。

## 8. 写正文时最值得强调的设计 takeaway

- Claude Code 把“每轮必带上下文”和“按需召回记忆”明确拆开，减少首轮 token 压力
- `MEMORY.md` 是 index，不是正文仓库
- memory 被定义为“当前项目状态里推不出来，但未来会话仍值得知道的信息”
- session memory 与 durable memory 是两种不同 persistence，不应混为一谈
- 相关记忆召回不是向量检索，而是 manifest + sideQuery + attachment 注入
- memory 召回从一开始就把“可能过期”当作系统事实处理

## 9. 仍需运行时验证的问题

- `tengu_moth_copse` 在当前已安装客户端上是否已默认打开，以及打开后 `/context` 或可视化上下文到底怎么展示 `AutoMem`
- `relevant_memories` attachment 在本机 transcript / UI 中的真实呈现形式
- session memory 文件在本机运行时的实际落盘路径、命名方式和清理策略
- `extractMemories` stop-hook 在当前客户端版本里是否总是开启，还是受更多 gate / product mode 约束
- KAIROS daily-log 模式在当前安装版本里是否真实可达，还是仅保留在源码中

## 10. 对后续正文的建议拆法

后续写教程时，建议拆成两章而不是一章：

### `03-context.md`

- `systemPrompt`
- `userContext`
- `systemContext`
- CLAUDE.md 发现与注入
- query 前静态上下文组装

### `06-memory.md`

- `MEMORY.md` index 设计
- relevant memory prefetch
- session memory
- durable extract memories
- memory 类型约束与验证原则

这样叙事上会更清楚，也更贴近源码真实分层。
