# Claude Code 是怎么构建 harness 的

## 这篇文章要回答什么

很多人第一次接触 Claude Code，会把它理解成：

- 一个终端 UI
- 一个 prompt
- 一组工具
- 一个会反复调用模型的 loop

这些都没错，但还不够。

如果只用这几个词来理解 Claude Code，很容易忽略一个真正决定系统工程质量的层：

**harness。**

这篇文章要回答的是：

- 我理解的 `harness` 到底是什么
- Claude Code 看起来是怎样把它构建出来的
- 如果你自己实现一个 Claude Code-like 系统，应该如何把这层做得更干净

## 先给结论

在当前可见源码里，Claude Code 的 `harness` 不是一个单独的 `Harness.ts`，而是一层分散实现的 runtime shell。

它做的事情不是“替模型思考”，而是：

- 在启动期搭好运行时骨架
- 给模型准备一组受控的内部状态面
- 把工具调用变成可权限化、可持久化、可恢复的执行
- 改写模型最终能看到的输出
- 在本地宿主、远端宿主和 UI 之间维护控制协议

所以如果让我用一句话概括：

**模型是决策器，harness 是操作系统边界层。**

## 先纠正一个常见误区

如果你在源码里直接搜 `harness`，会找到不少注释，但找不到一个中心化的 `Harness` 类。

这不是因为 Claude Code 没有 harness，而是因为它把 harness 拆进了几类横切职责里：

- 启动与 session 初始化
- 内部目录和持久化
- 权限判定
- 工具输出改写
- 控制消息协议
- 远端模式适配

这恰好说明，在 Claude Code 的实际工程里，`harness` 更接近一种 runtime 组织方式，而不是一个单独模块名。

## 第一层：启动时先把运行时壳搭起来

最能说明问题的文件之一是 `src/setup.ts`。

这里最值得注意的不是参数解析，而是它在首轮 query 之前就做了很多 background runtime 的准备工作，例如：

- 注册 `initSessionMemory()`
- 初始化 `context collapse`
- 预取 `commands`
- 预加载 plugin hooks

这说明 Claude Code 不是等模型第一次要用某个能力时再临时补环境，而是先把一层运行时骨架搭起来，再让 agent loop 进去跑。

这正是 harness 思维。

如果你自己实现一个简化版系统，这里最容易偷懒成：

1. 收到用户输入
2. 调模型
3. 如果模型要用工具，再临时初始化一堆东西

Claude Code 走的显然不是这条路。它更像：

1. 先建立 session 运行时
2. 再启动 query loop

这样做的收益是很直接的：

- 生命周期更稳定
- 工具环境更一致
- session 级状态更容易持有
- 后续加入远端模式、恢复、插件和 background tasks 时不容易失控

## 第二层：harness 先准备一组内部状态通道

如果只看模型调用工具，你会以为 Claude Code 的世界就是工作区文件和 shell。

但源码显示，Claude Code 还专门维护了一批**内部路径**，用来承接运行时状态和工具副产物。

典型例子包括：

- `session-memory`
- `plans`
- `tool-results`
- `auto memory`
- `agent memory`

例如：

- `src/utils/toolResultStorage.ts` 会把 session 目录组织成 `projectDir/sessionId/tool-results`
- `src/services/SessionMemory/sessionMemory.ts` 会创建 `session-memory/summary.md`
- `src/memdir/memdir.ts` 多处直接写明 harness 会保证 memory 目录已经存在，模型可以直接写

这里最关键的工程含义是：

**Claude Code 并不是让模型直接面对整个文件系统，而是先在文件系统里开辟一批由 runtime 控制的“内部回流面”。**

这样做有两个好处：

第一，很多原本太大、太临时、太内部的结果不需要硬塞进上下文，可以先落盘再按需读取。  
第二，权限系统可以对这批路径做特殊处理，把“运行时自己的产物”和“用户真实工作区”区分开来。

## 第三层：权限系统把这些内部路径变成受控世界

这也是我认为 harness 概念最清楚的一层。

在 `src/utils/permissions/filesystem.ts` 里，可以看到权限系统显式放行了若干 internal harness paths，包括：

- `session-memory`
- `plans`
- `tool-results`
- `scratchpad`
- project temp
- agent memory
- auto memory
- tasks
- teams

这件事非常重要，因为它意味着：

Claude Code 不是简单地“给模型一个 Read 工具”，而是先定义一套 runtime 自己认可的内部世界，再让模型在这个世界里读写。

换句话说，模型并不天然拥有“随便读文件”的能力。它读到的很多东西，其实是 harness 先布置好的材料。

这和很多 toy agent 的差别非常大。后者往往只有两种状态：

- 全放开
- 全阻断

Claude Code 走的是第三条路：

**给模型一套受控的内部文件面，让它在这个面上表现得像“有长期状态”和“能读完整结果”，但底层仍然由 runtime 保持边界。**

## 第四层：工具输出不会原样回给模型

这是 harness 最容易被忽略、但又最像“产品工程”的地方。

### 大输出不会直接塞回上下文

`src/tools/BashTool/BashTool.tsx` 里，大输出会被复制到 `tool-results/`，然后只把路径和预览回给模型。

这背后其实有两个判断：

- 不应该把超长输出直接塞回上下文
- 但也不能直接丢掉，因为模型后面可能还需要继续读

于是 harness 做了一个折中：

1. 保存完整结果到内部目录
2. 把这个文件路径当成下一轮可读取对象
3. 只给当前上下文一个简化引用

这不是小优化，而是运行时设计。

它把“工具执行结果”从一次性文本，变成了 session 里的可寻址资源。

### 某些输出只给用户，不给模型

`src/utils/claudeCodeHints.ts` 的注释更直接：  
CLI 或 SDK 可以发出 `<claude-code-hint />` 标记，harness 会扫描并剥离这些标签，然后只给用户弹安装提示，模型本身看不到。

这说明 harness 还有一个很重要的职责：

**区分哪些信号属于模型上下文，哪些信号属于宿主/UI 侧通道。**

如果没有这层分离，很多工程提示、推荐、遥测相关信息都会污染模型上下文。

所以从工程角度看，Claude Code 的工具系统不是“执行完命令，把 stdout 喂回去”。

它更像：

1. 执行命令
2. 清洗结果
3. 判断哪些部分该持久化
4. 判断哪些部分只给 UI
5. 最后再生成模型可见版本

这整条链，本质上都属于 harness。

## 第五层：harness 还包含 control plane

如果说前面几层更多是在处理 I/O，那么 `control_request` 这一层说明 harness 还负责控制面。

在 `src/cli/structuredIO.ts` 里，权限协商会通过 `control_request` / `control_response` 跟宿主通信。这里不是模型自己决定“我要不要执行这个工具”，而是 runtime 把请求送到宿主侧，再等待回应。

远端模式下，这个控制面更明显。

`src/utils/teleport.tsx` 会在远端 session 创建时预先写入 `set_permission_mode` 事件，确保远端 CLI 在第一轮用户输入之前，就已经处在正确的权限模式里。

这说明 Claude Code 里的 harness 并不只是本地命令执行包装器，它还是：

- 本地 CLI 与宿主之间的控制层
- 远端容器与本地主界面之间的协商层
- permission mode、tool approval、interrupt 等行为的统一协议层

如果没有这层，远端运行时和本地 UI 会很快漂移成两套系统。

## 第六层：harness 也影响“模型看到的上下文”

这篇文章虽然重点不是 context，但还是要补一个关键判断：

`src/context.ts` 里，system context 和 user context 也体现了 harness 的边界意识。

例如：

- 远端模式下会跳过某些本地 git status 注入
- `CLAUDE.md`、memory 文件和日期信息会被统一组织进用户上下文

这说明 harness 不只是拦截工具输出，它还在决定：

- 哪些宿主环境信息应该注入
- 哪些环境下要省略注入
- 哪些上下文应该缓存

也就是说，harness 同时控制了：

- 模型的输入边界
- 模型的输出回流边界

这正是它和“普通工具调用框架”的差别。

## 如果你自己实现一个更干净的版本

Claude Code 当前源码把 harness 分散在不少模块里。研究它很有价值，但如果是自己实现，我建议把这层显式收束成一个独立概念。

例如，你完全可以先定义一个：

```ts
type HarnessRuntime = {
  session: SessionRuntime
  internalPaths: InternalPathRegistry
  toolExecutor: ToolExecutor
  permissionGate: PermissionGate
  hostControl: HostControlChannel
  outputRewriter: OutputRewriter
}
```

然后按下面顺序做最小实现。

### 1. 先做 session runtime

至少要有：

- session id
- project root
- internal state dir
- transcript / tool-results / memory 等内部目录

这一步不要先想 UI，要先把运行时地基打好。

### 2. 再做 tool execution + result persistence

核心不是“能跑 shell”，而是：

- 能执行
- 能限权
- 能把大结果存盘
- 能给模型一个受控引用

只要这一步做好，你的系统就已经比很多 demo agent 更接近真实产品。

### 3. 再做 permission gate

不要只有二选一：

- 完全允许
- 完全拒绝

更合理的是三层边界：

- 工作区路径
- harness 自己的内部路径
- 工作区外部路径

其中内部路径应该是最早被定义出来的能力之一。

### 4. 最后做 host control protocol

无论是本地 UI、SDK host 还是远端容器，最好都不要让工具审批和模式切换散落在业务代码里。

把它们统一抽成一个 control channel，会让你后面做：

- remote mode
- IDE bridge
- background tasks
- multi-agent

时轻松很多。

## 我对 harness 的最终理解

结合当前源码，我更愿意把 harness 理解成下面这句话：

**harness 是 Claude Code 包在模型外面的运行时边界层。它决定模型能看到什么、能调用什么、哪些结果如何回流、哪些状态如何跨回合保存，以及宿主在什么时候接管控制权。**

所以 Claude Code 真正难复制的部分，不只是 prompt，也不只是 loop。

真正难的是这一层：

- 既要让模型像在操作真实世界
- 又不能真的把整个世界毫无边界地交给模型

harness 就是解决这个矛盾的工程答案。

## 关键源码入口

- `src/setup.ts`
- `src/utils/toolResultStorage.ts`
- `src/tools/BashTool/BashTool.tsx`
- `src/utils/claudeCodeHints.ts`
- `src/utils/permissions/filesystem.ts`
- `src/services/SessionMemory/sessionMemory.ts`
- `src/memdir/memdir.ts`
- `src/cli/structuredIO.ts`
- `src/utils/teleport.tsx`
- `src/context.ts`

## 版本假设

- 本文基于 `../claude-code-sourcemap/restored-src/src` 在 2026-03-31 可见的还原源码
- 文中的“看起来如何工作”以当前源码为准
- 涉及已安装客户端的运行时差异，仍建议再用 `../extracts/` 或真实运行观察交叉验证
