# 如何构建一个 Claude Code

## 教程目标

这套教程的目标，是让一个有工程经验的读者真正理解如何构建一个生产级的 Claude Code 风格 CLI，而不是只会拼一个套壳聊天工具。

## 文章结构原则

这套教程不按源码目录顺序展开，而按“先总览，再抓住主循环，再拆外围模块”的方式展开。

核心原因：

- 读者先需要一个全局地图，否则后面看到的模块会失去上下文
- `agent loop` 是 Claude Code 的心脏，工具系统、上下文管理、记忆、subagent 基本都围绕它组织
- 把模块放回主循环里解释，比单独讲目录结构更容易让人建立真正可实现的心智模型

## 暂定章节

1. [总览：Claude Code 到底是什么](./01-overview.md)
2. [Agent Loop：Claude Code 真正的核心循环](./02-agent-loop.md)
3. [上下文管理：模型每次到底看到了什么](./03-context-management.md)
4. [上下文压缩：太长以后 Claude Code 怎么缩上下文](./04-context-compaction.md)
5. [Prompt Cache：Claude Code 到底缓存了什么](./05-prompt-cache.md)
6. 工具系统：从工具注册到实际执行
7. 权限控制：为什么它敢帮你跑命令
8. 记忆系统：上下文窗口之外的信息如何保存与回流
9. Subagent 与任务系统：复杂工作是如何被拆开的
10. CLI 入口与命令分发：用户如何进入主循环
11. 用 React 和 Ink 构建终端 UI
12. MCP、插件、IDE Bridge 与外部系统
13. 配置、持久化与产品化细节
14. 构建一个最小可用克隆
15. 把它打磨成真正可用的产品

## 当前已完成

- [01-overview.md](./01-overview.md)
- [02-agent-loop.md](./02-agent-loop.md)
- [03-context-management.md](./03-context-management.md)
- [04-context-compaction.md](./04-context-compaction.md)
- [05-prompt-cache.md](./05-prompt-cache.md)

## 独立专题

- [Claude Code 是怎么构建 harness 的](./claude-code-harness.md)

## 每章模板

每一章都应尽量包含：

- 这个子系统解决了什么问题
- 应该先读哪些源码文件
- 运行时控制流是什么
- 关键设计取舍是什么
- 如何用更干净的方式复现同样能力
- 做最小克隆时哪些部分可以先简化

## 写作提醒

- overview 章节负责给全局地图，不陷入过深源码细节
- `agent loop` 章节应作为整套教程的支点，后续章节不断回勾它
- 工具、上下文、记忆、subagent 不要孤立讲，要明确它们在主循环中的插入点
