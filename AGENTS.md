# AGENTS.md

## Purpose

This repository is for building a serious tutorial and research corpus around Claude Code internals.

Primary deliverable:

- a practical "how to build a Claude Code" tutorial in Simplified Chinese first

Secondary deliverables:

- structured notes
- architecture maps
- implementation checklists

## Source Rules

- primary source reference: `../claude-code-sourcemap/restored-src/src`
- validation source for installed-runtime claims: `../extracts/`
- do not treat extracted bundle text as the primary reading source when restored source is available

## Writing Rules

### Content Requirements

- each tutorial chapter should answer both:
  - how Claude Code appears to work
  - how to build the same capability in a clean implementation
- prefer Simplified Chinese for tutorial-facing content unless English is explicitly requested
- keep English technical terms when they are the clearest option, but explain them in Chinese
- keep notes factual and source-cited
- separate source reading from design opinion
- call out version assumptions when relevant

### Tone and Style

参考风格：Anthropic engineering blog（https://www.anthropic.com/engineering）。核心特征：

- 像一个工程师读完源码后跟同事讲，不像教科书或分析报告
- 用具体的源码证据（函数签名、文件路径、代码模式）锚定技术判断，不做没有出处的空泛断言
- 主观设计意见用"我"来表达，不伪装成客观事实
- 按问题推进（先讲为什么会坏，再讲怎么解决），不按源码目录平铺
- 承认限制和代价，不只讲好处。具体数字和 honest tradeoff 比乐观总结更有说服力

### 禁止的写作模式

- 不用"换句话说""也就是说""简单来说"这类过渡模板——如果前面说清楚了就不需要换一种方式再说一遍
- 不用"关键设计一/二/三/四/五"这种机械列举框架——用问题和叙事驱动结构
- 不密集使用粗体强调——粗体只用于真正需要读者注意的关键判断，每节最多一处
- 不在每段开头用问句再自己回答（"那么 X 是什么？X 是……"）
- 不写空洞的总结段落（"综上所述，我们可以看到……"）
- 不用"本质上""从根本上说""毫无疑问"这类增加气势但不增加信息的修饰语
- 每篇结尾不需要"本篇结论"或"总结"段——如果正文写到位了，读者自己能得出结论

## Directory Conventions

- `tutorial/`: reader-facing tutorial chapters
- `notes/`: working notes, decomposition, and raw findings

## Planning Rules

- keep stable operational rules in `AGENTS.md`
- keep execution plans, parallel work breakdowns, and chapter production queues in `notes/`
- keep the current executable backlog in `notes/02-work-queue.md`
- keep a durable execution diary in `notes/03-work-log.md`
- if a planning convention becomes durable across tasks, promote that rule into `AGENTS.md`

## Autonomous Execution Rules

- these rules apply when the user asks to continue, resume, or start the long-running tutorial/research effort
- before choosing the next task, read:
  - `AGENTS.md`
  - `notes/01-parallel-work-plan.md`
  - `notes/02-work-queue.md`
  - `notes/03-work-log.md`
- default behavior: pick the highest-priority task in `notes/02-work-queue.md` whose status is `ready` or `in_progress`, then execute it end-to-end
- keep only one primary tutorial-writing task in flight at a time; parallelizable side work should usually land in `notes/` or `scripts/`
- prefer producing durable artifacts in the repository over giving planning-only chat responses
- after each substantial work block, update both:
  - `notes/02-work-queue.md` with status changes, blockers, or newly discovered tasks
  - `notes/03-work-log.md` with what was done, what evidence was used, and what should happen next
- if a task depends on unresolved terminology, chapter structure, or a still-moving core conclusion, update the prerequisite task first instead of pushing speculative downstream prose
- stop and ask the user only when:
  - multiple plausible directions would materially change the tutorial structure
  - source evidence conflicts in a way that affects a key claim
  - required local artifacts or environment access are missing for a high-confidence conclusion
- if the uncertainty is low-risk, record the assumption in `notes/03-work-log.md` and continue
- do not claim fully autonomous background execution; these rules govern behavior after the agent is invoked, not outside an active session

## Git Rules

- commit only files inside this repository when working here
- do not modify upstream source reference trees from this repo
