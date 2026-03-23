---
name: web-pen-test
description: Use when conducting authorized penetration testing against a web application, API, or web-accessible service to identify security vulnerabilities
---

# Web Pen Test

Three-agent authorized pen test — black-box recon, white-box audit, red team synthesis. Non-destructive by default.

## When to Use

- User wants to pen test, red team, vulnerability scan, or OWASP test a web app
- Target is a web application, API, or web-accessible service
- User has authorization to test the target

## When NOT to Use

- **Code review for security** — use a code review skill instead
- **Dependency auditing** — use `npm audit`, `snyk`, etc.
- **Configuring security headers** — that's implementation, not testing
- **Incident response** — active incidents need containment, not pen testing

## Intake Flow

Ask these five questions **one at a time**. Do not batch them.

### Q1: Target URL

> "What is the URL to pen test?"

- Validate URL is well-formed
- If hostname lacks `staging`, `dev`, `test`, `localhost`: warn it may be production. Do not block — just flag it.

### Q2: Source Code Path

> "Where is the source code? Enter a path, or 'none' for black-box only."

- If `none`: skip white-box agent. Run black-box + synthesis only.

### Q3: Authentication

> "Does the app require authentication? Provide: test user credentials, admin credentials (optional), auth mechanism (JWT / session cookies / OAuth / API keys / magic links)."

Resolution matrix:
| Auth provided? | Source code? | Result |
|---|---|---|
| Yes | Yes/No | Both agents receive credentials |
| No | Yes | White-box discovers auth from source for its own use. Black-box gets nothing — unauthenticated only |
| No | No | Black-box only, unauthenticated surface |

**Critical: black-box NEVER receives information derived from source code.**

### Q4: Scope Boundaries

> "Any areas to focus on or avoid? (optional)"

### Q5: Consent Gate

Present a summary table:

| Field | Value |
|---|---|
| Target | (URL) |
| Source code | (path or none) |
| Auth | (mechanism + what's provided) |
| Scope | (focus/exclusions or "no restrictions") |
| Rules of engagement | (defaults below) |

User must explicitly confirm **yes** before any HTTP requests are made. No implicit consent. No proceeding on silence.

## Rules of Engagement Defaults

Present these during the consent gate.

### ALLOWED by default

- GET requests for discovery, sitemap, robots.txt, directory enumeration
- Response header / cookie / CSP / CORS inspection
- Form submission with benign test payloads (XSS canaries, SQLi detection strings)
- Auth flow probing with user-provided credentials only
- IDOR testing within user's own session
- JWT / session token inspection
- Playwright UI interaction (clickjacking, open redirect, DOM XSS)
- Source code analysis (white-box only)
- Rate limit detection (max 20 requests over 10 seconds)
- Checking for exposed files (`.env`, `.git/`, source maps)

### BLOCKED by default (report, don't exploit)

- Persistent data creation (accounts, posts, uploads)
- SQL injection beyond detection
- Destructive mutations (DELETE, admin actions, password resets)
- DoS / DDoS, flood testing, connection exhaustion
- Brute force beyond 3-5 attempts
- Triggering emails / SMS / notifications / OTP
- Spidering beyond declared target domain
- Aggressive WAF / CDN bypass
- Source code or infrastructure modification
- Exfiltrating real user data

### REQUIRES EXPLICIT OPT-IN

- Production with real user data
- Write-based API fuzzing (POST / PUT / DELETE)
- File upload testing
- Aggressive rate testing (100+ burst)
- Third-party integration testing
- Actions that could trigger ops alerts

### The Iron Law

```
PROVE THE VULNERABILITY, DON'T CAUSE THE DAMAGE.
```

## Orchestration

After consent is confirmed:

### 1. Target Fingerprint

```bash
curl -sI <TARGET_URL>
```

Validate reachability. Gather initial headers (server, framework hints, security headers).

### 2. Dispatch Agents

Read each agent template and substitute template variables before dispatching. Use `supernova:dispatching-parallel-agents` for parallel dispatch.

**Black-box recon** (`./agents/black-box-recon.md`):

| Variable | Value |
|----------|-------|
| `{{TARGET_URL}}` | URL from Q1 |
| `{{AUTH_CREDENTIALS}}` | Credentials from Q3, or "None — unauthenticated testing only" |
| `{{SCOPE_BOUNDARIES}}` | Scope from Q4, or "Full application, no exclusions" |
| `{{REPORT_OUTPUT_PATH}}` | `docs/supernova/[DATE]-pen-test-report-[domain]-[HH-MM]-black-box.md` |
| `{{ROE_DEFAULTS}}` | The Rules of Engagement Defaults section above |

**White-box auditor** (`./agents/white-box-auditor.md`) — skip if no source code:

| Variable | Value |
|----------|-------|
| `{{TARGET_URL}}` | URL from Q1 |
| `{{SOURCE_CODE_PATH}}` | Path from Q2 |
| `{{AUTH_CREDENTIALS}}` | Credentials from Q3, or "None — discover from source code" |
| `{{SCOPE_BOUNDARIES}}` | Scope from Q4, or "Full application, no exclusions" |
| `{{REPORT_OUTPUT_PATH}}` | `docs/supernova/[DATE]-pen-test-report-[domain]-[HH-MM]-white-box.md` |
| `{{ROE_DEFAULTS}}` | The Rules of Engagement Defaults section above |

**Critical:** black-box agent prompt must NOT contain any information from source code analysis. The two agents are information-siloed.

### 3. Synthesis

After agent(s) complete, dispatch red team lead (`./agents/red-team-lead.md`):

| Variable | Value |
|----------|-------|
| `{{BLACK_BOX_REPORT_PATH}}` | Path to black-box findings file |
| `{{WHITE_BOX_REPORT_PATH}}` | Path to white-box findings file, or "N/A — black-box only run" |
| `{{TARGET_URL}}` | URL from Q1 |
| `{{SCOPE_SUMMARY}}` | Scope from Q4 + the Rules of Engagement Defaults section |
| `{{FINAL_REPORT_PATH}}` | `docs/supernova/[DATE]-pen-test-report-[domain]-[HH-MM].md` |

### 4. Report

Write final report to: `docs/supernova/[DATE]-pen-test-report-[target-domain]-[HH-MM].md`

Present executive summary to user with path to full report.

## Escalation During Run

- **Critical / High:** log real-time status update, continue testing. No stopping.
- **Medium and below:** collected silently for final report.

## Agent References

- `./agents/black-box-recon.md` — External recon, no source code access
- `./agents/white-box-auditor.md` — Source-informed audit, full code access
- `./agents/red-team-lead.md` — Synthesizes findings, writes final report
