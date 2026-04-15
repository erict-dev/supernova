# Agent Runtime Compatibility

Supernova's workflow is shared across Claude Code and Codex. The workflow is stable; only the runtime affordances differ.

## Core Rule

Preserve the workflow intent when tooling names differ:

- load the right skill before substantial work
- maintain explicit task tracking
- preserve review and verification gates
- use isolated workers when the runtime supports them

## Mapping Table

| Supernova term | Claude Code | Codex | Notes |
| --- | --- | --- | --- |
| Skill tool | Invoke `Skill` | Read `skills/<name>/SKILL.md` directly | Same skill content, different loader |
| `CLAUDE.md` | Local Claude instructions | `AGENTS.md` or repo-local agent instructions | Treat both as the same policy tier |
| TodoWrite | `TodoWrite` | Plan/task tracker | Checklists are still required; only the tool name changes |
| Subagent | Claude subagent/task | Delegated agent if allowed, otherwise sequential role execution | Keep review roles distinct even if one agent does them serially |
| Session-start hook | `hooks/session-start` | `AGENTS.md` entrypoint | Claude gets automatic injection; Codex reads repo instructions directly |

## Delegation Guidance

When a skill says to dispatch a fresh subagent:

1. Use the runtime's delegation tool if it exists and policy allows it.
2. Give the worker scoped context rather than raw session history.
3. If delegation is unavailable, perform the same role locally, then switch cleanly to the next role instead of collapsing the review loop.

Example:

- implementer role
- spec review role
- code quality review role

Those can run as separate agents or as separate sequential passes by the main agent. The separation of concerns matters more than the exact API.

## Local Policy Files

If a skill says to check `CLAUDE.md`, follow this order:

1. `AGENTS.md`
2. `CLAUDE.md`
3. direct user instruction

Use whichever exists. If both exist, prefer the file that matches the active runtime, but merge any non-conflicting project conventions.

## Scope Of This Compatibility Layer

This document is meant for workflow portability. It does not replace runtime-specific docs for:

- Claude plugin installation
- Claude session hooks
- Codex-specific tool policy

Those remain runtime-specific on purpose.
