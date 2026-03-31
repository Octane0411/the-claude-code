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

## Git Rules

- commit only files inside this repository when working here
- do not modify upstream source reference trees from this repo
