# Supernova: Focused Fork of Superpowers

## Summary

Supernova is a Claude-Code-only fork of superpowers (v5.0.5). It preserves all 14 skills and the core workflow (brainstorm → plan → execute → review → finish) while removing:

- All non-Claude-Code platform support (Gemini, Codex, OpenCode, Cursor)
- The visual companion feature (browser-based mockup server)
- Deprecated command wrappers
- Upstream maintenance files (tests, GitHub templates, changelogs)

## What Gets Removed

### Directories (not copied)

- `.cursor-plugin/` — Cursor plugin config
- `.codex/` — Codex installation docs
- `.opencode/` — OpenCode plugin system + installation docs
- `commands/` — Deprecated command wrappers (brainstorm.md, write-plan.md, execute-plan.md)
- `skills/brainstorming/scripts/` — Visual companion server (server.cjs, frame-template.html, helper.js, start/stop scripts)
- `skills/using-superpowers/references/` — Cross-platform tool mappings (codex-tools.md, gemini-tools.md)
- `tests/` — Upstream test infrastructure
- `.github/` — Upstream GitHub templates and CI
- `docs/plans/`, `docs/specs/` — Upstream planning docs
- `docs/README.codex.md`, `docs/README.opencode.md` — Platform-specific READMEs

### Files (not copied)

- `GEMINI.md` — Gemini context injection
- `gemini-extension.json` — Gemini CLI metadata
- `package.json` — OpenCode entry point
- `hooks/hooks.json` — OpenCode hook format
- `hooks/hooks-cursor.json` — Cursor hook format
- `hooks/run-hook.cmd` — Windows batch script
- `skills/brainstorming/visual-companion.md` — Visual companion guide
- `skills/brainstorming/spec-document-reviewer-prompt.md` — if it exists, keep (it's for spec review, not visual)
- `CHANGELOG.md`, `RELEASE-NOTES.md`, `CODE_OF_CONDUCT.md` — Upstream maintenance
- `README.md` — Will be replaced with supernova-specific version if needed
- `.claude-plugin/marketplace.json` — Marketplace listing (superpowers-specific)

## Content Edits Within Skill Files

### using-superpowers → using-supernova

- Rename skill to `using-supernova`
- Remove "Platform Adaptation" section
- Remove references to Gemini CLI, Codex, OpenCode tool mappings
- Remove `references/` subdirectory mentions
- Update "superpowers" → "supernova" in branding

### brainstorming

- Remove checklist step 2 (visual companion offer)
- Remove "Visual Companion" section entirely
- Remove visual companion references from process flow diagram
- Remove browser/server references from key principles
- Keep spec review loop, design process, and all other workflow logic

### All skills (grep pass)

- Remove any stray mentions of: Gemini, Codex, OpenCode, Cursor, Windsurf
- Remove Windows-specific conditionals (`run-hook.cmd`, `CODEX_CI` env var)
- Remove references to `gemini-tools.md`, `codex-tools.md`
- Replace "superpowers" branding with "supernova" where it appears in user-facing text

## Final Structure

```
supernova/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── brainstorming/
│   │   ├── SKILL.md
│   │   └── spec-document-reviewer-prompt.md (if present)
│   ├── writing-plans/
│   ├── subagent-driven-development/
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   ├── requesting-code-review/
│   ├── receiving-code-review/
│   ├── dispatching-parallel-agents/
│   ├── verification-before-completion/
│   ├── using-git-worktrees/
│   ├── finishing-a-development-branch/
│   ├── using-supernova/
│   └── writing-skills/
├── agents/
│   └── code-reviewer.md
└── hooks/
    └── session-start
```

## Skills Preserved (13)

All workflow logic remains intact. Platform conditionals, visual companion, and executing-plans (replaced by subagent-driven-development as sole execution method) are removed.

1. using-supernova (was using-superpowers)
2. brainstorming
3. writing-plans
4. subagent-driven-development
5. test-driven-development
6. systematic-debugging
7. requesting-code-review
8. receiving-code-review
9. dispatching-parallel-agents
10. verification-before-completion
11. using-git-worktrees
12. finishing-a-development-branch
13. writing-skills
