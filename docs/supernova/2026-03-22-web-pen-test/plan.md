# Web Pen Test Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use supernova:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `web-pen-test` skill that orchestrates authorized penetration testing of web applications using independent black-box and white-box agents with a synthesis/reporting agent.

**Architecture:** Thin `SKILL.md` orchestrator handles intake, consent gate, scope configuration, and agent dispatch. Three agent prompt templates (`black-box-recon.md`, `white-box-auditor.md`, `red-team-lead.md`) define independent attacker personas and methodologies. Agents run in parallel via `supernova:dispatching-parallel-agents`, synthesis runs sequentially after both complete.

**Tech Stack:** Markdown (skill/agent definitions), Playwright MCP (browser automation), curl (HTTP probing)

---

## File Structure

```
skills/web-pen-test/
├── SKILL.md                          # Orchestrator: intake, consent, dispatch (~400 words)
├── agents/
│   ├── black-box-recon.md            # External attacker agent prompt template
│   ├── white-box-auditor.md          # Source-informed attacker agent prompt template
│   └── red-team-lead.md              # Synthesis & reporting agent prompt template
```

---

### Task 1: Create the SKILL.md orchestrator

**Files:**
- Create: `skills/web-pen-test/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/web-pen-test/agents
```

- [ ] **Step 2: Write SKILL.md**

The skill file must contain:

**Frontmatter:**
```yaml
---
name: web-pen-test
description: Use when conducting authorized penetration testing against a web application, API, or web-accessible service to identify security vulnerabilities
---
```

**Sections to include:**

1. **Title:** `# Web Pen Test`

2. **Overview:** One-liner: Three-agent authorized pen test — black-box recon, white-box audit, red team synthesis. Non-destructive by default.

3. **When to Use / When NOT to Use:**
   - Use when: user wants to pen test, red team, vulnerability scan, or OWASP test a web app
   - NOT for: code review for security, dependency auditing (`npm audit`), configuring security headers, incident response

4. **Intake Flow** — five questions, asked one at a time:

   **Q1: Target URL**
   > "What is the URL to pen test?"
   - Validate URL is well-formed
   - Warn (don't block) if hostname lacks `staging`, `dev`, `test`, `localhost` — it may be production

   **Q2: Source Code Path**
   > "Where is the source code? Enter a path, or 'none' for black-box only."
   - If `none`: skip white-box agent, run black-box + synthesis only

   **Q3: Authentication**
   > "Does the app require authentication? Provide: test user credentials, admin credentials (optional), auth mechanism (JWT / session cookies / OAuth / API keys / magic links)"
   - If auth provided: both agents receive it
   - If no auth but source code: white-box discovers auth from source for its own use. Black-box gets nothing — unauthenticated only
   - If no auth and no source: black-box only, unauthenticated surface
   - **Critical: black-box never receives info derived from source code**

   **Q4: Scope Boundaries**
   > "Any areas to focus on or avoid? (optional)"

   **Q5: Consent Gate** — present summary table of target, source, auth, scope, and default Rules of Engagement. User must explicitly confirm `yes` before any HTTP requests are made.

5. **Rules of Engagement Defaults** — present during consent gate:

   **ALLOWED by default:**
   - GET requests for discovery, sitemap, robots.txt, directory enumeration
   - Response header/cookie/CSP/CORS inspection
   - Form submission with benign test payloads (XSS canaries, SQLi detection strings)
   - Auth flow probing with user-provided credentials only
   - IDOR testing within user's own session
   - JWT/session token inspection
   - Playwright UI interaction (clickjacking, open redirect, DOM XSS)
   - Source code analysis (white-box only)
   - Rate limit detection (max 20 requests over 10 seconds)
   - Checking for exposed files (`.env`, `.git/`, source maps)

   **BLOCKED by default (report, don't exploit):**
   - Persistent data creation (accounts, posts, uploads)
   - SQL injection beyond detection
   - Destructive mutations (DELETE, admin actions, password resets)
   - DoS/DDoS, flood testing, connection exhaustion
   - Brute force beyond 3-5 attempts
   - Triggering emails/SMS/notifications/OTP
   - Spidering beyond declared target domain
   - Aggressive WAF/CDN bypass
   - Source code or infrastructure modification
   - Exfiltrating real user data

   **REQUIRES EXPLICIT OPT-IN:**
   - Production with real user data
   - Write-based API fuzzing (POST/PUT/DELETE)
   - File upload testing
   - Aggressive rate testing (100+ burst)
   - Third-party integration testing
   - Actions that could trigger ops alerts

   **The Iron Law: "PROVE THE VULNERABILITY, DON'T CAUSE THE DAMAGE."**

6. **Orchestration** — after consent:
   - Target fingerprint: quick `curl -sI` to validate reachability and gather initial headers
   - If source code provided: dispatch black-box and white-box agents in parallel using `supernova:dispatching-parallel-agents`
   - If no source code: dispatch black-box agent only
   - After agent(s) complete: dispatch red team lead with both reports (or single report)
   - Report written to `docs/supernova/[DATE]-pen-test-report-[target-domain]-[HH-MM].md`
   - Present executive summary to user with path to full report

7. **Escalation during run:**
   - Critical/High: log real-time status update, continue testing. No stopping.
   - Medium and below: collected silently for final report.

8. **Agent References:**
   - Black-box recon: `./agents/black-box-recon.md`
   - White-box auditor: `./agents/white-box-auditor.md`
   - Red team lead: `./agents/red-team-lead.md`

- [ ] **Step 3: Review SKILL.md for token efficiency**

Verify the skill stays under ~400 words of instructional content (excluding the RoE list which is reference material). The RoE defaults are necessary reference — they're presented to the user and baked into agent prompts — so they stay. But all other prose should be lean.

- [ ] **Step 4: Commit**

```bash
git add skills/web-pen-test/SKILL.md
git commit -m "feat(web-pen-test): add skill orchestrator with intake, consent gate, and RoE defaults"
```

---

### Task 2: Create the black-box recon agent prompt

**Files:**
- Create: `skills/web-pen-test/agents/black-box-recon.md`

- [ ] **Step 1: Write black-box-recon.md**

Follow the agent prompt template pattern from `skills/frontend-smoke-testing/smoke-test-agent.md`. Use template variables for dynamic values.

**Structure:**

```markdown
# Black-Box Recon Agent Prompt Template

Use this template when dispatching the black-box recon agent for external penetration testing.
```

Then a code-fenced prompt block containing:

**Template variables:**
- `{{TARGET_URL}}` — the URL to test
- `{{AUTH_CREDENTIALS}}` — user-provided credentials (or "None — unauthenticated testing only")
- `{{SCOPE_BOUNDARIES}}` — focus areas / exclusions (or "Full application, no exclusions")
- `{{REPORT_OUTPUT_PATH}}` — path for this agent's findings file
- `{{ROE_DEFAULTS}}` — the full Rules of Engagement text

**Agent prompt sections:**

1. **Role:** "You are a black-box penetration tester. You have NO knowledge of the application's source code, architecture, or internal implementation. You are an external attacker who only knows the target URL and any credentials explicitly provided below."

2. **Inputs:** Template variables listed above.

3. **Tooling:**
   - Primary: Playwright MCP tools (browser automation), curl via Bash
   - Opportunistic: check `which nikto`, `which ffuf`, `which sqlmap`, `which nuclei` — use if available but don't depend on them
   - Never use: Read, Grep, Glob on the application source code (you are black-box)

4. **Rules of Engagement:** `{{ROE_DEFAULTS}}` — these are non-negotiable constraints. Violating them is a hard failure.

5. **Methodology — Phase 1: Reconnaissance**
   - Fingerprint tech stack: response headers (`Server`, `X-Powered-By`, `X-Frame-Options`), cookie names/attributes, error page signatures
   - Discover endpoints: `robots.txt`, `sitemap.xml`, common paths (`/.env`, `/.git/config`, `/api`, `/graphql`, `/.well-known/`), JavaScript source map files
   - API discovery from client-side JS: use Playwright to load pages, extract API calls from network requests and inline scripts
   - Directory enumeration: try common paths (admin, dashboard, api/docs, swagger, graphiql)

6. **Methodology — Phase 2: OWASP Systematic Sweep**
   For each category, describe what to test and how:
   - **Injection:** Test all discovered form fields and URL params with: `' OR 1=1--` (SQLi), `<script>alert(1)</script>` (XSS), `{{7*7}}` (SSTI), `; ls` (command injection). Check if payloads are reflected, executed, or cause errors.
   - **Broken Authentication:** Try default credentials if login found. Check cookie flags (HttpOnly, Secure, SameSite). Look for session tokens in URLs. Test session fixation. Inspect JWT structure (alg:none, weak secrets).
   - **Sensitive Data Exposure:** Check HTTP vs HTTPS. Inspect security headers (CSP, HSTS, X-Content-Type-Options). Look for exposed `.env`, `.git/`, source maps, `package.json`, stack traces in errors.
   - **Broken Access Control:** If authenticated, manipulate resource IDs (IDOR). Try forced browsing to admin paths. Test HTTP method tampering (GET→POST, POST→PUT). Check CORS configuration.
   - **Security Misconfiguration:** Check verbose error pages. Look for default pages/credentials. Test directory listing. Inspect CORS (wildcard origins, credential headers). Check CSP for unsafe-inline/eval.
   - **SSRF:** If any input accepts URLs, test with internal addresses (127.0.0.1, metadata endpoints).

7. **Methodology — Phase 3: Creative Probing**
   Based on what was discovered:
   - Business logic abuse (price manipulation, quantity overflow, step-skipping in flows)
   - Race conditions on sensitive operations (use parallel curl requests)
   - Open redirects (manipulate redirect params)
   - Clickjacking (check X-Frame-Options, try iframe embedding)
   - Subdomain/path-based auth bypass

8. **Escalation:**
   - Critical/High findings: log a status update immediately, then continue testing
   - Medium/Low/Info: collect silently for report

9. **Output format:** Write findings to `{{REPORT_OUTPUT_PATH}}` in this exact structure:

   ```markdown
   # Black-Box Recon Findings

   **Target:** {{TARGET_URL}}
   **Auth:** [authenticated/unauthenticated]
   **Run:** [ISO 8601 timestamp]
   **Tools Used:** [list of tools actually used]

   ## Findings

   ### [SEVERITY] Finding N: [Title]
   **Category:** [OWASP category or "Business Logic" / "Configuration"]
   **Endpoint:** [URL or path affected]
   **Description:** [What the vulnerability is]
   **Evidence:**
   [Request/response pairs, screenshots, or observed behavior]
   **Reproduction Steps:**
   1. [Step-by-step to reproduce]
   **Remediation:** [How to fix]

   ## Reconnaissance Summary
   - Tech stack: [discovered technologies]
   - Endpoints discovered: [count and notable ones]
   - Security headers: [present/missing summary]
   ```

10. **Rules:**
    - Never access or read application source code
    - Never exceed RoE boundaries
    - Test every applicable category — do not stop at first finding
    - Be thorough with evidence — the synthesis agent needs request/response pairs
    - One interaction at a time — check state after each probe before proceeding

- [ ] **Step 2: Commit**

```bash
git add skills/web-pen-test/agents/black-box-recon.md
git commit -m "feat(web-pen-test): add black-box recon agent prompt template"
```

---

### Task 3: Create the white-box auditor agent prompt

**Files:**
- Create: `skills/web-pen-test/agents/white-box-auditor.md`

- [ ] **Step 1: Write white-box-auditor.md**

Same template pattern as black-box-recon.md.

**Template variables:**
- `{{TARGET_URL}}` — the URL to test
- `{{SOURCE_CODE_PATH}}` — path to application source code
- `{{AUTH_CREDENTIALS}}` — user-provided credentials, or "None — discover from source code"
- `{{SCOPE_BOUNDARIES}}` — focus areas / exclusions
- `{{REPORT_OUTPUT_PATH}}` — path for this agent's findings file
- `{{ROE_DEFAULTS}}` — the full Rules of Engagement text

**Agent prompt sections:**

1. **Role:** "You are a white-box penetration tester simulating a malicious insider or an attacker who has compromised the application's source code. You have full access to the codebase at `{{SOURCE_CODE_PATH}}` and can use this knowledge to craft targeted attacks that an external attacker could not."

2. **Inputs:** Template variables listed above.

3. **Tooling:**
   - Source analysis: Read, Grep, Glob on `{{SOURCE_CODE_PATH}}`
   - Active testing: Playwright MCP tools, curl via Bash
   - Opportunistic CLI tools (same as black-box)

4. **Rules of Engagement:** `{{ROE_DEFAULTS}}` — non-negotiable.

5. **Methodology — Phase 1: Source Analysis**
   - **Route map:** Use Grep/Glob to find all route definitions, API endpoints, middleware chains. For each: note HTTP method, path, auth requirement, handler function. Look for routes that skip auth middleware.
   - **Auth implementation:** Find session/token creation, validation, invalidation logic. Check JWT signing (algorithm, secret source, expiration). Look for auth bypass patterns (commented-out checks, dev-mode skips, role validation gaps).
   - **Database queries:** Search for raw SQL strings, ORM query builders. Check parameterization — look for string concatenation in queries. Note any `raw()`, `execute()`, or template literal queries.
   - **Input validation:** Find validation middleware/functions. Check what inputs are validated vs. passed through raw. Look for sanitization gaps (validated on create but not update, validated on frontend but not backend).
   - **Secrets:** Search for hardcoded tokens, API keys, passwords. Check `.env.example` for sensitive defaults. Look for credentials in test fixtures/seeds. Check if secrets are logged or exposed in error responses.
   - **Dependencies:** Read `package.json`/`requirements.txt`/lock files. Run `npm audit` or equivalent if available. Note any packages with known CVEs.
   - **Auth discovery (if no credentials provided):** Search for test/seed credentials, default admin accounts, `.env.example` with passwords, auth configuration that reveals mechanism type. Use discovered credentials for your own testing only — never share with black-box agent.

6. **Methodology — Phase 2: Targeted Exploitation**
   Using source knowledge, craft precise attacks:
   - **Unprotected endpoints:** Hit routes identified as missing auth middleware with curl
   - **Validation bypass:** Craft payloads that bypass specific validation logic found in source (e.g., if validation checks for `<script>` but not `<img onerror>`)
   - **Race conditions:** Identify operations that lack locking/transactions (e.g., balance checks before deductions). Test with parallel curl requests.
   - **Business logic:** Find logic flaws visible in code — e.g., discount applied after total calculation, role check uses OR instead of AND, admin check looks at wrong field
   - **Privilege escalation:** If multiple roles exist, test if lower-privilege session can access higher-privilege endpoints by manipulating tokens/IDs

7. **Methodology — Phase 3: Configuration & Infrastructure**
   - **Environment variables:** Check for insecure defaults (DEBUG=true, secret=changeme). Verify production vs dev configuration separation.
   - **CORS/CSP in source:** Compare configured policy to what's actually served (may differ due to middleware ordering)
   - **Rate limiting:** Check if implemented. If so, check coverage (all endpoints or just auth?)
   - **Error handling:** Look for catch blocks that expose stack traces, internal paths, or database details to clients

8. **Escalation:** Same as black-box — Critical/High logged immediately as status updates, continue testing.

9. **Output format:** Write findings to `{{REPORT_OUTPUT_PATH}}`:

   ```markdown
   # White-Box Auditor Findings

   **Target:** {{TARGET_URL}}
   **Source:** {{SOURCE_CODE_PATH}}
   **Auth:** [method and source of credentials]
   **Run:** [ISO 8601 timestamp]

   ## Findings

   ### [SEVERITY] Finding N: [Title]
   **Category:** [OWASP category or "Business Logic" / "Configuration"]
   **Endpoint:** [URL or path affected]
   **Source Location:** [file:line where the vulnerability originates]
   **Description:** [What the vulnerability is]
   **Evidence:**
   [Source code snippet showing the flaw + request/response proving exploitability]
   **Reproduction Steps:**
   1. [Step-by-step to reproduce]
   **Remediation:** [How to fix, referencing specific code to change]

   ## Architecture Summary
   - Framework: [detected framework and version]
   - Auth mechanism: [how auth works]
   - Database: [ORM/driver and query patterns]
   - Notable patterns: [middleware, validation, error handling approach]
   ```

10. **Rules:**
    - Use source code to inform attacks but validate everything against the live target
    - Never share findings or discovered credentials with the black-box agent
    - Reference specific file:line for every source-derived finding
    - Never exceed RoE boundaries — source access doesn't grant permission to be destructive
    - Test every applicable category — be thorough

- [ ] **Step 2: Commit**

```bash
git add skills/web-pen-test/agents/white-box-auditor.md
git commit -m "feat(web-pen-test): add white-box auditor agent prompt template"
```

---

### Task 4: Create the red team lead (synthesis) agent prompt

**Files:**
- Create: `skills/web-pen-test/agents/red-team-lead.md`

- [ ] **Step 1: Write red-team-lead.md**

**Template variables:**
- `{{BLACK_BOX_REPORT_PATH}}` — path to black-box agent's findings
- `{{WHITE_BOX_REPORT_PATH}}` — path to white-box agent's findings (or "N/A — black-box only run")
- `{{TARGET_URL}}` — original target URL
- `{{SCOPE_SUMMARY}}` — original scope/RoE summary from intake
- `{{FINAL_REPORT_PATH}}` — output path (`docs/supernova/[DATE]-pen-test-report-[target-domain]-[HH-MM].md`)

**Agent prompt sections:**

1. **Role:** "You are a senior red team lead. Two independent penetration testers — one black-box (external attacker, no source code) and one white-box (insider with source access) — have completed their assessments. Your job is to synthesize their findings into a single, prioritized, actionable report."

   If `{{WHITE_BOX_REPORT_PATH}}` is "N/A": "Only the black-box assessment was run (no source code was provided). Produce the report from a single input and include a 'Recommended Follow-Up' section suggesting a white-box audit for deeper coverage."

2. **Inputs:** Template variables.

3. **Methodology:**

   **Step 1: Read both reports** (or the single black-box report).

   **Step 2: Deduplicate.** Merge findings that both agents discovered independently. When merging, keep the richer evidence set and note "Confirmed by both agents" — these are high-confidence findings.

   **Step 3: Cross-reference** (skip if black-box only). Identify the deltas:
   - **White-box found, black-box missed:** Hidden attack surface. Good if intentional (internal-only endpoint). Bad if it's an endpoint that should be protected but was only found because the attacker had source code — means the protection is "security through obscurity."
   - **Black-box found, white-box missed:** Gaps in code-level analysis. The white-box agent should have caught this from source — signals areas where code review alone is insufficient.
   - **Both found independently:** High confidence. The vulnerability is discoverable from outside AND confirmed in source.

   For each delta, add a brief note explaining what it means for the application's security posture.

   **Step 4: Severity calibration.** Assign final severity using this rubric:
   - **Critical** — Unauthenticated RCE, SQL injection with data access, auth bypass to admin, exposed database, hardcoded production secrets
   - **High** — Stored XSS, IDOR on sensitive data, privilege escalation, missing auth on state-changing endpoints, JWT with weak/hardcoded secret
   - **Medium** — Reflected XSS, CSRF on state-changing actions, verbose error messages exposing internals, missing rate limiting on auth endpoints
   - **Low** — Missing security headers (non-critical), server version disclosure, clickjacking on non-sensitive pages, cookie without Secure flag on HTTPS
   - **Info** — Best practice observations, defense-in-depth suggestions, configuration recommendations

   If agents disagree on severity, use the higher rating and note the disagreement.

   **Step 5: Remediation roadmap.** Group findings into:
   - **Quick wins** (< 1 hour): missing headers, cookie flags, config changes
   - **Short-term fixes** (1 day): input validation, auth middleware additions, CORS tightening
   - **Architectural improvements** (multi-day): auth system overhaul, query parameterization across codebase, rate limiting infrastructure

   Within each group, order by severity (critical first).

   **Step 6: Executive summary.** 3-5 sentences covering: what was tested, how many findings at each severity, the most critical risks, and overall security posture assessment.

4. **Output format:** Write to `{{FINAL_REPORT_PATH}}`:

   ```markdown
   # Penetration Test Report

   **Target:** {{TARGET_URL}}
   **Date:** [ISO 8601 date]
   **Methodology:** [Black-box only | Black-box + White-box]
   **Scope:** {{SCOPE_SUMMARY}}

   ## Executive Summary

   [3-5 sentences: what was tested, key risk areas, overall posture]

   ## Finding Summary

   | # | Severity | Category | Title | Found By |
   |---|----------|----------|-------|----------|
   | 1 | Critical | Broken Access Control | Unauthenticated admin API | Both |
   | 2 | High | Injection | Reflected XSS in search | Black-box |
   | ... | ... | ... | ... | ... |

   **Totals:** N Critical, N High, N Medium, N Low, N Info

   ## Detailed Findings

   ### Finding 1: [Title]
   **Severity:** [Critical/High/Medium/Low/Info]
   **Category:** [OWASP category]
   **Found By:** [Black-box / White-box / Both]
   **Endpoint:** [URL or path]
   **Source Location:** [file:line — white-box findings only]

   **Description:**
   [What the vulnerability is and why it matters]

   **Evidence:**
   [Request/response pairs, source code, or observed behavior]

   **Reproduction Steps:**
   1. [Step-by-step]

   **Remediation:**
   [Specific guidance on how to fix, referencing code where applicable]

   ---

   ## Cross-Reference Analysis

   [Skip this section entirely if black-box only]

   ### Confirmed by Both Agents
   [Findings both agents discovered independently — highest confidence]

   ### Hidden Attack Surface (White-box only)
   [Findings only the white-box agent found — assess whether obscurity is intentional]

   ### External Discovery Gaps (Black-box only)
   [Findings the white-box agent missed — signals code review blind spots]

   ## Remediation Roadmap

   ### Quick Wins (< 1 hour)
   - [ ] [Finding N]: [one-line description of fix]

   ### Short-Term Fixes (1 day)
   - [ ] [Finding N]: [one-line description of fix]

   ### Architectural Improvements (multi-day)
   - [ ] [Finding N]: [one-line description of fix]

   ## Recommended Follow-Up

   [If black-box only: recommend white-box audit with source code for deeper coverage]
   [If both ran: recommend areas for deeper investigation, periodic re-testing schedule]
   ```

5. **Rules:**
   - Never re-test or make HTTP requests — you are a reviewer, not a tester
   - Preserve all evidence from original reports
   - When merging duplicates, keep the more detailed evidence set
   - If severity is ambiguous, rate higher and note uncertainty
   - The report should be actionable — a developer should be able to read a finding and know exactly what to fix

- [ ] **Step 2: Commit**

```bash
git add skills/web-pen-test/agents/red-team-lead.md
git commit -m "feat(web-pen-test): add red team lead synthesis agent prompt template"
```

---

### Task 5: Register the skill in the plugin configuration

**Files:**
- Modify: `.claude-plugin/plugin.json` — add the new skill entry

- [ ] **Step 1: Read the current plugin.json to understand the registration format**

```bash
cat .claude-plugin/plugin.json
```

- [ ] **Step 2: Add the web-pen-test skill entry**

Add a new entry to the skills array following the exact pattern of existing entries. The skill should have:
- `name`: `web-pen-test`
- `description`: same as SKILL.md frontmatter
- `path`: `skills/web-pen-test/SKILL.md`

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat(web-pen-test): register skill in plugin configuration"
```

---

### Task 6: End-to-end validation

- [ ] **Step 1: Verify file structure**

```bash
find skills/web-pen-test -type f | sort
```

Expected:
```
skills/web-pen-test/SKILL.md
skills/web-pen-test/agents/black-box-recon.md
skills/web-pen-test/agents/red-team-lead.md
skills/web-pen-test/agents/white-box-auditor.md
```

- [ ] **Step 2: Verify SKILL.md frontmatter**

Check that `name` is `web-pen-test` and `description` starts with "Use when".

- [ ] **Step 3: Verify agent template variables are consistent**

Confirm that the template variables referenced in SKILL.md's orchestration section match what each agent expects:
- Black-box: `{{TARGET_URL}}`, `{{AUTH_CREDENTIALS}}`, `{{SCOPE_BOUNDARIES}}`, `{{REPORT_OUTPUT_PATH}}`, `{{ROE_DEFAULTS}}`
- White-box: `{{TARGET_URL}}`, `{{SOURCE_CODE_PATH}}`, `{{AUTH_CREDENTIALS}}`, `{{SCOPE_BOUNDARIES}}`, `{{REPORT_OUTPUT_PATH}}`, `{{ROE_DEFAULTS}}`
- Red team lead: `{{BLACK_BOX_REPORT_PATH}}`, `{{WHITE_BOX_REPORT_PATH}}`, `{{TARGET_URL}}`, `{{SCOPE_SUMMARY}}`, `{{FINAL_REPORT_PATH}}`

- [ ] **Step 4: Verify plugin.json has the new skill entry**

- [ ] **Step 5: Commit any fixes if needed**
