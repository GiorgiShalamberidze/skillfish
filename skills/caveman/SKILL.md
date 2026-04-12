---
name: caveman
description: Cut ~75% of output tokens by talking like caveman. Full technical accuracy. Lite, full, ultra, and 文言文 intensity levels. Works on Claude Code, Cursor, Copilot, Gemini CLI, and 40+ agents.
externalRepo: JuliusBrussee/caveman
---

# Caveman

> **Community skill** — source and full install at [github.com/JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman)

Ultra-compressed communication mode. Cuts token usage ~65–75% by having the agent talk like caveman — dropping articles, filler, and pleasantries — while keeping every bit of technical accuracy.

## Install

```bash
# Any agent (auto-detect)
npx skills add JuliusBrussee/caveman

# Claude Code
claude plugin marketplace add JuliusBrussee/caveman && claude plugin install caveman@caveman

# Cursor
npx skills add JuliusBrussee/caveman -a cursor

# Windsurf
npx skills add JuliusBrussee/caveman -a windsurf

# Gemini CLI
gemini extensions install https://github.com/JuliusBrussee/caveman
```

## Usage

```
/caveman          — activate (default: full)
/caveman lite     — drop filler, keep grammar
/caveman full     — classic caveman (fragments OK, drop articles)
/caveman ultra    — maximum compression, arrows for causality
/caveman wenyan   — classical Chinese 文言文 mode
stop caveman      — back to normal
```

## Companion Skills

- `caveman-commit` — terse commit messages, ≤50 char subject
- `caveman-review` — one-line PR comments
- `/caveman:compress` — compress CLAUDE.md / memory files (~46% input token savings)
