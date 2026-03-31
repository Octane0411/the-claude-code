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

- each tutorial chapter should answer both:
  - how Claude Code appears to work
  - how to build the same capability in a clean implementation
- prefer Simplified Chinese for tutorial-facing content unless English is explicitly requested
- keep English technical terms when they are the clearest option, but explain them in Chinese
- keep notes factual and source-cited
- separate source reading from design opinion
- call out version assumptions when relevant

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
