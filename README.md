# Supernova

An opinionated development workflow plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Fork of [superpowers](https://github.com/obra/superpowers), trimmed for focus.

## What it does

Supernova gives Claude Code a structured workflow for building software:

**brainstorm** → **write plan** → **execute with subagents** → **review** → **finish**

It ships 12 skills that enforce discipline at each stage:

| Skill | Purpose |
|-------|---------|
| **using-supernova** | Skill discovery and routing |
| **brainstorming** | Collaborative design → spec document |
| **writing-plans** | Spec → bite-sized implementation plan |
| **subagent-driven-development** | Per-task subagent dispatch + two-stage review |
| **test-driven-development** | RED-GREEN-REFACTOR enforcement |
| **systematic-debugging** | 4-phase root cause investigation |
| **requesting-code-review** | Dispatch code review subagent |
| **receiving-code-review** | Evaluate feedback with technical rigor |
| **dispatching-parallel-agents** | Concurrent multi-domain investigation |
| **verification-before-completion** | Evidence-before-claims enforcement |
| **using-git-worktrees** | Isolated workspace creation |
| **writing-skills** | TDD applied to writing new skills |

## What changed from superpowers

- **Claude Code only** — removed all Gemini, Codex, OpenCode, and Cursor support
- **No visual companion** — removed the browser-based mockup server
- **No inline execution** — subagent-driven development is the only execution method
- **Trimmed bloat** — compressed redundant sections, removed dead references, cut ~1800 tokens of non-rendering `<img>` tags
- **Full fork** — will diverge from upstream

## Install

```bash
claude plugins add /path/to/supernova
```

Or if you've cloned it to a standard location:

```bash
claude plugins add ~/workspace/supernova
```

Restart Claude Code after installing. The `using-supernova` skill loads automatically via the session-start hook.

## Usage

Just use Claude Code normally. Supernova skills activate automatically when relevant — you don't need to invoke them manually. If you want to trigger a specific skill:

```
/brainstorming
/writing-plans
/systematic-debugging
```

## License

MIT
