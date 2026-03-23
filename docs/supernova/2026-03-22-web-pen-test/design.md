# Web Pen Test Skill — Design Spec

## Summary

A supernova skill that orchestrates authorized penetration testing of web applications using a three-agent architecture: a black-box recon agent (external attacker perspective), a white-box auditor agent (source-informed attacker), and a red team lead agent (synthesis and reporting). The skill handles intake, consent, scope configuration, parallel agent dispatch, and final report generation.

## Architecture

```
skills/web-pen-test/
├── SKILL.md                      # Lean orchestrator (~400 words)
├── agents/
│   ├── black-box-recon.md        # External attacker agent
│   ├── white-box-auditor.md      # Source-informed attacker agent
│   └── red-team-lead.md          # Synthesis & reporting agent
```

**SKILL.md** is a thin orchestrator that handles intake, consent, scope, and agent dispatch. Agent files contain full methodology prompts (~500-800 words each).

## Intake Flow

Five questions, asked one at a time:

### Q1: Target URL
> "What is the URL to pen test?"

Validate it's a reachable URL. Warn (don't block) if it looks like production (no `staging`, `dev`, `test`, `localhost` in the hostname).

### Q2: Source Code Path
> "Where is the source code? Enter a path, or 'none' for black-box only."

If `none`: skip white-box agent entirely. Run black-box + synthesis only.

### Q3: Authentication
> "Does the app require authentication? If so, provide:
> - Test user credentials (email/password, API key, or session cookie)
> - Admin credentials (optional, enables privilege escalation testing)
> - Auth mechanism (JWT / session cookies / OAuth / API keys / magic links)"

- If auth provided: black-box agent uses it. White-box agent uses it.
- If no auth provided but source code provided: white-box agent discovers auth from source for its own use. Black-box agent gets nothing — it tests unauthenticated surface only.
- If no auth and no source code: black-box only, unauthenticated surface test.

**Critical: black-box agent never receives information derived from source code.** The agents are fully independent with no cross-contamination.

### Q4: Scope Boundaries
> "Any areas to focus on or avoid?"

Optional. Narrows or excludes specific paths/endpoints.

### Q5: Consent Gate

Present a summary of target, source code path, auth, scope, and the default Rules of Engagement. User must explicitly confirm authorization before any probing begins.

```
Target:       https://staging.myapp.com
Source code:  /path/to/source (or: none)
Auth:         [summary of provided credentials]
Scope:        [focus/exclusions or "Full app"]
Rules of Engagement: Non-destructive defaults (listed below)

Do you confirm you have authorization to test this target
and approve this scope? (yes/no)
```

No HTTP requests are made before this gate passes.

## Orchestration Flow

```
Intake (5 questions, one at a time)
    ↓
Consent Gate (explicit yes required)
    ↓
Target Fingerprint (quick curl + header check to validate target is reachable)
    ↓
┌─── Has source code? ───┐
│ yes                     │ no
│                         │
│  Parallel dispatch:     │  Single dispatch:
│  ┌──────────┐           │  ┌──────────┐
│  │Black-box │           │  │Black-box │
│  │Recon     │           │  │Recon     │
│  └──────────┘           │  └──────────┘
│  ┌──────────┐           │       │
│  │White-box │           │       │
│  │Auditor   │           │       │
│  └──────────┘           │       │
│       │                 │       │
│       ↓                 │       ↓
│  Red Team Lead          │  Red Team Lead
│  (synthesizes both)     │  (single-source report)
│       │                 │       │
└───────┴─────────────────┘───────┘
    ↓
Report written to docs/supernova/
    ↓
Present summary to user with path to full report
```

When only black-box runs, the Red Team Lead produces a full report but skips cross-reference analysis and includes a "Recommended follow-up" section suggesting a white-box audit.

## Agent Designs

### Black-Box Recon Agent

**Persona:** External attacker with no insider knowledge. Only knows the target URL and whatever credentials the user explicitly provided during intake.

**Tooling:** Playwright (browser automation), curl, and any opportunistically discovered CLI tools (`nikto`, `ffuf`, etc. — check `which` and use if available).

**Phased methodology:**

1. **Reconnaissance** — Fingerprint target: tech stack (response headers, `X-Powered-By`, cookie names, error page signatures), sitemap/robots.txt, directory enumeration, JavaScript source map discovery, API endpoint discovery from client-side JS.
2. **OWASP Systematic Sweep** — Methodically test each applicable category:
   - Injection (SQLi, XSS, command injection, template injection) via form fields and URL params
   - Broken auth (default creds, session fixation, cookie flags, token leakage in URLs)
   - Sensitive data exposure (HTTP vs HTTPS, missing security headers, exposed `.env`/`.git`/source maps)
   - Broken access control (IDOR via ID manipulation, forced browsing, method tampering)
   - Security misconfiguration (CORS, CSP, verbose errors, directory listing, default pages)
   - SSRF (if any URL-accepting inputs are discovered)
3. **Creative Probing** — Based on discoveries: business logic abuse, race conditions, open redirects, clickjacking, subdomain/path-based auth bypass.

**Output:** Structured findings list with severity, evidence (request/response pairs), and reproduction steps. Surfaces critical/high findings as real-time status updates.

### White-Box Auditor Agent

**Persona:** Malicious insider or attacker who has compromised the source code. Full access to codebase and credentials (user-provided or self-discovered from source).

**Tooling:** Source code analysis (Grep, Read, Glob) + Playwright + curl for validation.

**Phased methodology:**

1. **Source Analysis** — Map the application architecture:
   - Route map: every endpoint, auth requirements, HTTP methods
   - Auth implementation: session/token creation, validation, invalidation
   - Database queries: raw SQL or ORM patterns, parameterization (or lack thereof)
   - Input validation: where it happens (or doesn't), sanitization patterns
   - Secrets: hardcoded tokens, API keys, credentials, `.env.example` leaking info
   - Dependencies: known vulnerable packages (check lock files against known CVEs)
2. **Targeted Exploitation** — Use source knowledge to craft precise attacks:
   - Hit endpoints that skip auth middleware
   - Craft payloads that bypass specific validation logic found in source
   - Test race conditions on operations lacking proper locking
   - Probe business logic flaws visible in code but invisible from outside
3. **Configuration & Infrastructure** — Review deployment-relevant source:
   - Environment variable handling (insecure defaults)
   - CORS/CSP configuration in source vs what's actually served
   - Rate limiting implementation (or absence)
   - Error handling that leaks stack traces or internal state

**Output:** Same structured format as black-box, but each finding also references the specific source file and line number where the vulnerability originates.

### Red Team Lead (Synthesis Agent)

**Persona:** Senior red team lead reviewing findings from two independent operators.

**Inputs:** Both agent reports + original scope/RoE.

**Methodology:**

1. **Deduplicate** — Merge findings discovered by both agents independently.
2. **Cross-reference** — Identify deltas:
   - White-box found, black-box missed → hidden attack surface
   - Black-box found, white-box missed → gaps in code review methodology
   - Both found → high-confidence finding
3. **Severity calibration** — Final severity using consistent rubric:
   - **Critical** — Unauthenticated RCE, SQL injection with data access, auth bypass to admin
   - **High** — Stored XSS, IDOR on sensitive data, privilege escalation, hardcoded production secrets
   - **Medium** — Reflected XSS, CSRF on state-changing actions, verbose errors, missing rate limiting
   - **Low** — Missing security headers, info disclosure (server version), clickjacking on non-sensitive pages
   - **Info** — Observations, best practice recommendations, defense-in-depth suggestions
4. **Remediation roadmap** — Group by effort, suggest fix order (quick wins first, then architectural fixes).
5. **Executive summary** — Top-line risk assessment suitable for sharing with stakeholders.

## Rules of Engagement (Defaults)

Presented to user during consent gate. Baked into every agent prompt as non-negotiable constraints.

### ALLOWED by default
- GET requests for endpoint discovery, sitemap, robots.txt, directory enumeration
- Response header/cookie/CSP/CORS inspection
- Form submission with benign test payloads (XSS canaries, SQLi detection strings)
- Auth flow probing with user-provided credentials only
- IDOR testing by manipulating IDs within the user's own session
- JWT/session token inspection and analysis
- Playwright-driven UI interaction (clickjacking, open redirect, DOM XSS)
- Source code analysis (white-box agent only)
- Rate limit detection with modest bursts (max 20 requests over 10 seconds)
- Checking for exposed files (`.env`, `.git/`, source maps, `package.json`)

### BLOCKED by default (report, don't exploit)
- Persistent data creation (accounts, posts, uploads, comments)
- SQL injection beyond detection — no exfiltration, no mutation
- Destructive mutations (DELETE endpoints, admin actions, password resets on real accounts)
- DoS/DDoS patterns, flood testing, connection exhaustion
- Brute force beyond 3-5 attempts (enough to detect absence of lockout)
- Triggering emails/SMS/notifications/OTP through the app
- Spidering beyond the declared target domain
- Aggressive WAF/CDN bypass attempts
- Source code or infrastructure modification
- Exfiltrating real user data — proving access is possible is the finding

### REQUIRES EXPLICIT OPT-IN
- Testing against production with real user data
- Write-based API fuzzing (POST/PUT/DELETE with malformed payloads)
- File upload testing
- Aggressive rate testing (100+ burst)
- Testing third-party integrations (payment, OAuth, webhooks)
- Any action that could trigger ops alerts/pages

**The Iron Law: "PROVE THE VULNERABILITY, DON'T CAUSE THE DAMAGE."**

If an agent discovers it can achieve something destructive, it documents: (1) what it could do, (2) exact steps to reproduce, (3) the underlying weakness, (4) how to fix it. It never executes the destructive action.

## Critical Finding Escalation

- **Critical** — Log a real-time status update and continue testing. No stopping.
- **High** — Log a real-time status update and continue testing.
- **Medium and below** — Collected silently, included in final report only.

All findings get full write-ups in the final synthesis report regardless of when they were discovered. No gates, no pauses, no interruptions. Agents run their full methodology to completion.

## Output

**Report path:** `docs/supernova/[DATE]-pen-test-report-[target-domain]-[HH-MM].md`

**Report structure:**
1. Executive summary (3-5 sentences)
2. Scope & methodology
3. Finding summary table (severity, category, status)
4. Detailed findings — each with:
   - Description
   - Severity (Critical/High/Medium/Low/Info)
   - Evidence (request/response pairs, screenshots)
   - Reproduction steps
   - Affected source file and line (white-box findings only)
   - Remediation guidance
5. Cross-reference analysis (black-box vs white-box delta) — omitted if black-box only
6. Remediation roadmap (prioritized by severity and effort)
7. Recommended follow-up (if black-box only: suggest white-box audit)

## Tooling

**Baseline (zero-dependency):** curl + Playwright MCP server. The skill is fully functional with these alone.

**Opportunistic:** Agents check for installed security tools (`which nikto`, `which sqlmap`, `which ffuf`, `which nuclei`, etc.) and use them if present. Core logic never depends on external tools.

## Skill Frontmatter

```yaml
---
name: web-pen-test
description: Use when conducting authorized penetration testing against a web application, API, or web-accessible service to identify security vulnerabilities
---
```

**Trigger keywords:** pen test, penetration test, red team, vulnerability scan, OWASP, security testing.

**When NOT to use:** Code review for security, dependency auditing only (`npm audit`), configuring security headers, incident response for active breaches.

## Adaptive Target Handling

The skill adapts to whatever web-accessible surface it encounters:
- **Full-stack web app (SPA + API)** — Full browser + API testing
- **Server-rendered app** — Focus on form-based attacks, session handling
- **Standalone API** — Skip Playwright, focus on curl-based endpoint testing
- **Static site + serverless** — Test function endpoints, check for exposed config
- **Marketing site with forms** — Focus on injection, data handling, third-party script risks

Target type is determined during the fingerprint step (post-consent) and shapes the agent test plans.
