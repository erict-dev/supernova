# Supernova

An opinionated agent workflow for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and Codex. Fork of [superpowers](https://github.com/obra/superpowers), trimmed for focus.

## What it does

Supernova gives coding agents a structured workflow for building software:

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

- **Claude Code + Codex focused** — removed the older multi-runtime sprawl and kept one shared workflow with thin runtime adapters
- **No visual companion** — removed the browser-based mockup server
- **No inline execution** — subagent-driven development is the only execution method
- **Trimmed bloat** — compressed redundant sections, removed dead references, cut ~1800 tokens of non-rendering `<img>` tags
- **Full fork** — will diverge from upstream

## Runtime Support

- **Claude Code** uses the plugin metadata in `.claude-plugin/` plus `hooks/session-start`.
- **Codex** uses [AGENTS.md](/Users/eric/workspace/supernova/AGENTS.md) as the repo entrypoint and reads the same skill files directly.
- Shared runtime mappings live in [docs/agent-runtime-compat.md](/Users/eric/workspace/supernova/docs/agent-runtime-compat.md).

## Install In Claude Code

```bash
claude plugins add /path/to/supernova
```

Or if you've cloned it to a standard location:

```bash
claude plugins add ~/workspace/supernova
```

Restart Claude Code after installing. The `using-supernova` skill loads automatically via the session-start hook.

## Use In Codex

Open the repository in Codex. `AGENTS.md` routes Codex to the same skill files used by Claude Code.

## Usage

Use the workflow normally in your active runtime. Claude Code can auto-load `using-supernova`; Codex uses `AGENTS.md` to route into the skill set. In Claude Code, you can trigger a skill with shortcuts like:

```
/brainstorming
/writing-plans
/systematic-debugging
```

In Codex, open the corresponding `skills/<name>/SKILL.md` file directly.

## License

MIT
