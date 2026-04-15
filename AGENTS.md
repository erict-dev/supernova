# Supernova For Codex

This repository contains the shared Supernova workflow instructions. Claude Code uses the plugin hook path in `.claude-plugin/` and `hooks/session-start`. Codex should treat this file as the entrypoint.

## How To Load Supernova

1. Before substantial work, identify any relevant skill in `skills/<skill-name>/SKILL.md`.
2. Read the skill file directly and follow it.
3. When multiple skills apply, start with process skills first (`brainstorming`, `systematic-debugging`, `writing-plans`) and then execution skills.
4. Use [docs/agent-runtime-compat.md](/Users/eric/workspace/supernova/docs/agent-runtime-compat.md) for runtime-specific mappings.

## Runtime Mapping

- `Skill tool` means: load the referenced `SKILL.md` using your available file-reading mechanism.
- `CLAUDE.md` means: repo-level agent instructions. In this repo, treat `AGENTS.md` and `CLAUDE.md` as equivalent sources of local policy.
- `TodoWrite` means: use your runtime's task-tracking mechanism. In Codex, use the plan/task tracker when appropriate.
- `subagent` or `dispatch agent` means: use the runtime's delegation tool if available and allowed. If delegation is unavailable, execute the same role sequentially in the main thread while preserving the same review gates.

## High-Value Skills

- `skills/using-supernova/SKILL.md`: skill discovery and routing
- `skills/brainstorming/SKILL.md`: design before implementation
- `skills/writing-plans/SKILL.md`: implementation planning
- `skills/subagent-driven-development/SKILL.md`: task execution with review loops
- `skills/systematic-debugging/SKILL.md`: root-cause-first debugging
- `skills/verification-before-completion/SKILL.md`: evidence before completion claims
