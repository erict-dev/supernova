# Frontend Smoke Testing Skill + Docs Reorganization — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use supernova:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a frontend smoke testing skill that automates the "open the app and see if it works" workflow after implementation, and reorganize supernova docs into per-feature folders.

**Architecture:** New `frontend-smoke-testing` skill with two-agent architecture (smoke test agent + bugfix agent) orchestrated by the main conversation in a fix loop (max 3 iterations). Docs move from `docs/specs/` + `docs/plans/` to `docs/supernova/YYYY-MM-DD-<feature>/` containing `design.md`, `plan.md`, `smoke-tests.md`, and `smoke-test-results.md`. Existing skills updated to use new paths and invoke smoke testing after implementation.

**Tech Stack:** Markdown (skill definitions), Playwright (browser automation via MCP)

---

### Task 1: Create the `frontend-smoke-testing` skill directory and SKILL.md

**Files:**
- Create: `skills/frontend-smoke-testing/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/frontend-smoke-testing
```

- [ ] **Step 2: Write SKILL.md**

The skill definition must cover:
- **Frontmatter:** name `frontend-smoke-testing`, description starting with "Use when..." focused on post-implementation frontend verification
- **Overview:** Automates the manual "open the app and click around" workflow. Two-agent architecture: smoke test agent reports errors, bugfix agent fixes them. Max 3 fix-and-retry loops before escalating to user.
- **When to use:** After implementation of a feature that has frontend/UI work. Before claiming "done." Invoked automatically by subagent-driven-development after all tasks complete.
- **When to skip:** No frontend in the feature. No dev server feasible (e.g., static library, CLI tool, backend-only). Dev server requires environment config that isn't available locally.
- **The process (detailed below)**
- **Agent prompts:** References `./smoke-test-agent.md` and `./smoke-test-bugfix-agent.md`
- **Integration:** Called by subagent-driven-development. References supernova:systematic-debugging as escalation path for bugfix agent.

**The process to encode in the skill:**

```
1. WRITE SMOKE TEST PLAN
   - Analyze what was built (read spec + plan + code)
   - Write comprehensive scenarios split into two sections:
     a. "Testable locally" — pages to visit, interactions to perform,
        expected results. Only things feasible on localhost with Playwright.
     b. "Requires manual testing" — scenarios needing production env,
        third-party services, special config, or complex harnessing.
        Include WHY each can't be tested locally.
   - Save to: docs/supernova/<feature-folder>/smoke-tests.md

2. START DEV SERVER
   - Auto-detect how to run it from project (package.json scripts,
     Makefile, etc.)
   - Common patterns: npm run dev, yarn dev, pnpm dev, next dev, etc.
   - Wait for server to be ready (watch for "ready" / "listening" output
     or poll the URL)
   - If no dev server can be detected or started: skip smoke testing,
     report to user with reason

3. SMOKE TEST LOOP (max 3 iterations):
   a. Dispatch fresh SMOKE TEST agent (./smoke-test-agent.md)
      - Input: path to smoke-tests.md
      - Agent uses Playwright to walk through "Testable locally" scenarios
      - Agent writes results to docs/supernova/<feature-folder>/smoke-test-results.md
        (overwritten each run — always reflects current state)
      - Agent returns: PASS or FAIL

   b. If PASS: break loop, proceed to finish

   c. If FAIL: Dispatch fresh BUGFIX agent (./smoke-test-bugfix-agent.md)
      - Input: path to smoke-test-results.md + smoke-tests.md + plan.md
      - Agent reads source files from stack traces/errors
      - Agent fixes errors and commits
      - Agent returns: summary of fixes
      - Loop back to (a)

   d. If 3 failures: stop dev server, escalate to user with full error
      report and what was attempted

4. POST-FIX CODE REVIEW (conditional)
   - If bugfix agent made code changes during the loop: dispatch a code
     quality review scoped to just the bugfix commits (base SHA from
     before first fix, head SHA from after last fix)
   - Provide the spec and plan as context so the reviewer understands
     the feature — don't make them re-review the full implementation
   - If smoke tests passed on first try (no fixes): skip this step
   - Use the same code-quality-reviewer-prompt.md from subagent-driven-development
   - After review, proceed to finish (no need to re-run smoke tests)

5. STOP DEV SERVER
   - Kill the dev server process
   - Report results to user
```

**Red flags to include:**
- Never skip the smoke test plan writing step
- Never claim "done" if smoke tests haven't passed (or been explicitly skipped)
- Never run more than 3 fix iterations without escalating
- Never leave the dev server running after completion

- [ ] **Step 3: Commit**

```bash
git add skills/frontend-smoke-testing/SKILL.md
git commit -m "feat: add frontend-smoke-testing skill definition"
```

---

### Task 2: Create `smoke-test-agent.md` prompt template

**Files:**
- Create: `skills/frontend-smoke-testing/smoke-test-agent.md`

- [ ] **Step 1: Write the smoke test agent prompt**

Follow the same template pattern as `subagent-driven-development/implementer-prompt.md`. The prompt must instruct the agent to:

**Input it receives:**
- Path to `smoke-tests.md`
- Dev server URL (e.g., `http://localhost:3000`)

**What it does:**
1. Read the "Testable locally" section of smoke-tests.md
2. For each scenario, use Playwright (via MCP tools) to:
   - Navigate to the page
   - Perform the described interactions (clicks, form fills, etc.)
   - Check for: page load errors, console errors, network failures, missing elements, unexpected error states
   - Capture evidence: console output, network errors, screenshot if helpful
3. Write results to `smoke-test-results.md` (overwrite, not append)

**Results document format:**
```markdown
# Smoke Test Results

**Run:** [timestamp]
**Status:** PASS | FAIL
**Dev server:** [URL]

## Results

### Scenario: [name]
**Status:** PASS | FAIL
**Page:** [URL visited]
**Errors:** (if any)
- [Console error with full message]
- [Network error: URL, status code]
- [Page crash / component error boundary]
**Evidence:** [relevant details]

### Scenario: [name]
...

## Summary
- Total: N scenarios
- Passed: N
- Failed: N
- Errors: [list of error types encountered]
```

**Return format:**
- **Status:** PASS (all scenarios passed) or FAIL (any scenario failed)
- Brief summary of what failed

**Key instructions for the agent:**
- Don't fix anything — only observe and report
- Be thorough in capturing error details (full console messages, stack traces)
- A page that renders but shows an error boundary or error message is a FAIL
- A page with console errors but visually correct is still a FAIL (report the errors)
- Network 4xx/5xx responses on API calls the feature depends on are FAILs
- If Playwright can't perform an interaction (element not found), that's a FAIL

- [ ] **Step 2: Commit**

```bash
git add skills/frontend-smoke-testing/smoke-test-agent.md
git commit -m "feat: add smoke test agent prompt template"
```

---

### Task 3: Create `smoke-test-bugfix-agent.md` prompt template

**Files:**
- Create: `skills/frontend-smoke-testing/smoke-test-bugfix-agent.md`

- [ ] **Step 1: Write the bugfix agent prompt**

Follow the same template pattern. The prompt must instruct the agent to:

**Input it receives:**
- Path to `smoke-test-results.md` (current errors)
- Path to `smoke-tests.md` (what the feature should do)
- Path to `plan.md` (implementation intent)
- Iteration number (1, 2, or 3)

**What it does:**
1. Read the smoke test results — understand what's failing and why
2. Read the smoke test plan and implementation plan for context on intent
3. For each error:
   - Read the source files referenced in stack traces / error messages
   - Diagnose the root cause
   - If the fix is obvious from the error + source: fix it directly
   - If the fix isn't obvious: use `supernova:systematic-debugging` approach (trace backward from error, form hypothesis, test minimally)
4. After fixing all errors, run the project's test suite to make sure fixes didn't break anything
5. Commit all fixes

**Return format:**
- **Status:** FIXED | PARTIALLY_FIXED | BLOCKED
- What was fixed (file:line, what the issue was, what was changed)
- What remains broken (if PARTIALLY_FIXED)
- What's blocking (if BLOCKED)

**Key instructions for the agent:**
- Fix the actual root cause, not symptoms
- Don't add workarounds or hacks — fix it properly
- If a fix would require architectural changes beyond the scope of the feature, report BLOCKED
- Run the test suite after fixes — don't break existing tests
- Commit fixes with clear messages (e.g., "fix: resolve null ref in dashboard component")
- On iteration 2+, read previous results carefully — don't re-introduce fixes that didn't work

- [ ] **Step 2: Commit**

```bash
git add skills/frontend-smoke-testing/smoke-test-bugfix-agent.md
git commit -m "feat: add smoke test bugfix agent prompt template"
```

---

### Task 4: Update `brainstorming/SKILL.md` — new docs path

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

- [ ] **Step 1: Update spec output path**

Change all references from `docs/specs/YYYY-MM-DD-<topic>-design.md` to `docs/supernova/YYYY-MM-DD-<topic>/design.md`.

Specific lines to change:

1. Checklist item 5 (line 28):
   - Old: `docs/specs/YYYY-MM-DD-<topic>-design.md`
   - New: `docs/supernova/YYYY-MM-DD-<topic>/design.md`

2. After the Design section (line 106):
   - Old: `docs/specs/YYYY-MM-DD-<topic>-design.md`
   - New: `docs/supernova/YYYY-MM-DD-<topic>/design.md`

3. The user review gate message (line 120) references `<path>` which is fine — it's dynamic.

Also update the spec-document-reviewer-prompt.md dispatch line (line 8 of that file) which says "Dispatch after: Spec document is written to docs/specs/" — change to `docs/supernova/<feature>/`.

- [ ] **Step 2: Update `brainstorming/spec-document-reviewer-prompt.md`**

Change the "Dispatch after" line from `docs/specs/` to `docs/supernova/<feature>/`.

- [ ] **Step 3: Commit**

```bash
git add skills/brainstorming/SKILL.md skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "refactor: update brainstorming to use new docs/supernova/ path"
```

---

### Task 5: Update `writing-plans/SKILL.md` — new docs path

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1: Update plan output path**

Change the "Save plans to" line (line 19):
- Old: `docs/plans/YYYY-MM-DD-<feature-name>.md`
- New: `docs/supernova/YYYY-MM-DD-<feature-name>/plan.md`

Also update the User Review Gate message (line 143) and Execution Handoff message (line 152):
- Old: `docs/plans/<filename>.md`
- New: `docs/supernova/<feature-folder>/plan.md`

- [ ] **Step 2: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "refactor: update writing-plans to use new docs/supernova/ path"
```

---

### Task 6: Update `subagent-driven-development/SKILL.md` — add smoke test step

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

- [ ] **Step 1: Add smoke test invocation after all tasks complete**

In the process flow, after "Dispatch final code reviewer subagent for entire implementation" and before "Use supernova:finishing-a-development-branch", add a new step:

```
"Does feature include frontend work?" [shape=diamond];
"Invoke supernova:frontend-smoke-testing" [shape=box style=filled fillcolor=lightyellow];
```

With edges:
- "Dispatch final code reviewer..." → "Does feature include frontend work?"
- "Does feature include frontend work?" → "Invoke supernova:frontend-smoke-testing" [label="yes"]
- "Does feature include frontend work?" → "Use supernova:finishing-a-development-branch" [label="no"]
- "Invoke supernova:frontend-smoke-testing" → "Use supernova:finishing-a-development-branch"

Also update the example workflow (around line 121) which references `docs/plans/feature-plan.md` — change to `docs/supernova/<feature>/plan.md` for consistency with the new folder structure.

Also add to the text flow after the "After all tasks" section. Add a paragraph:

```markdown
## Frontend Smoke Testing

After the final code review, check if the feature includes frontend/UI work. If yes:

- **REQUIRED SUB-SKILL:** Use supernova:frontend-smoke-testing
- This runs BEFORE finishing-a-development-branch
- The smoke testing skill handles the full loop (write plan, start server, test, fix, retry)
- If smoke tests pass or the skill determines testing isn't feasible: proceed to finish
- If smoke tests fail after 3 attempts: the skill will escalate to the user
- **If bugfix agent made code changes:** dispatch a code quality review scoped to the bugfix commits (with spec + plan as context) before proceeding to finish. Skip this review if smoke tests passed on the first try (no fixes needed). No need to re-run smoke tests after the review.
```

Also add to the Integration section:
```markdown
- **supernova:frontend-smoke-testing** - Smoke test frontend after all tasks (if applicable)
```

- [ ] **Step 2: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat: invoke frontend smoke testing after implementation in subagent-driven-development"
```

---

### Task 7: Update `verification-before-completion/SKILL.md` — add smoke test verification

**Files:**
- Modify: `skills/verification-before-completion/SKILL.md`

- [ ] **Step 1: Add smoke test row to Common Failures table**

Add a new row to the table at line 42-50:

```markdown
| Frontend works | Smoke test results: PASS | Unit tests pass, "it should work" |
```

- [ ] **Step 2: Commit**

```bash
git add skills/verification-before-completion/SKILL.md
git commit -m "feat: add frontend smoke test to verification-before-completion requirements"
```

---

### Task 8: Migrate existing spec to new folder structure

**Files:**
- Move: `docs/specs/2026-03-21-supernova-trim-design.md` → `docs/supernova/2026-03-21-supernova-trim/design.md`
- Remove: `docs/specs/` (empty after migration)

- [ ] **Step 1: Create the new folder and move the file**

```bash
mkdir -p docs/supernova/2026-03-21-supernova-trim
mv docs/specs/2026-03-21-supernova-trim-design.md docs/supernova/2026-03-21-supernova-trim/design.md
rmdir docs/specs
```

- [ ] **Step 2: Commit**

```bash
git add -A docs/
git commit -m "refactor: migrate existing spec to new docs/supernova/ folder structure"
```
