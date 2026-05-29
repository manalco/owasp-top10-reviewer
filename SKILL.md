---
name: owasp-top-10-reviewer
description: Runs a full security audit of the codebase against the OWASP Top 10 (2025 edition) by dispatching 10 specialized subagents in parallel, then proposes a remediation plan scored by CVSS. Use this skill whenever the user asks for a "security review", "security audit", "OWASP audit", "OWASP review", "revisión de seguridad", "auditoría de seguridad", "pentest the code", "check for vulnerabilities", "find security bugs", "harden this code", or any equivalent phrasing — even if the user does not name OWASP explicitly. Also trigger when the user uploads a diff, branch, or PR and asks whether it is "safe to merge from a security standpoint". Works on any stack or language. Internal analysis and the written report are always produced in English; communication with the user happens in the user's original language.
---

# OWASP Top 10 Reviewer

Automated security reviewer aligned with the **OWASP Top 10 — 2025** categorization. One orchestrator + ten category subagents working in parallel. The orchestrator is the only agent that talks to the human; subagents only report findings back.

This skill produces three things:

1. A consolidated list of findings, each tagged with an OWASP category, a numeric CVSS score, and a CVSS severity band.
2. A remediation plan for Critical/High/Medium findings (Low/None are listed but not planned).
3. After the human approves the plan, the actual fixes applied to the code, plus a Markdown report file.

---

## Language rules (read first, they apply everywhere)

- **Internal work is always in English.** Every subagent prompt, every reasoning trace, every finding, every line of the report file: English. This holds regardless of the project's primary language or the user's language.
- **User-facing messages are in the user's original language** — the language they wrote in this conversation. The scope question (Step 1), the plan presentation (Step 6), and the final chat summary (Step 8) must be in that language.
- If code, comments, or strings are in another language, quote them verbatim in the English analysis; do not translate them.

The split exists because reasoning quality is highest in English while user trust is highest in their own language.

---

## High-level flow

1. **Ask scope.** In the user's language, ask what to audit: whole repo, a `git diff` (specify the range), or specific paths.
2. **Assemble inputs.** Resolve the scope to a concrete file list with contents and metadata (language, framework hints, dependency manifests, config files).
3. **Dispatch 10 subagents in parallel**, one per OWASP 2025 category. Pass each one the full scope bundle and its dedicated prompt from `agents/`.
4. **Consolidate findings.** Merge, deduplicate, sort by CVSS.
5. **Classify** by CVSS band (Critical 9.0–10.0 / High 7.0–8.9 / Medium 4.0–6.9 / Low 0.1–3.9 / None 0.0).
6. **Build a remediation plan** covering only Critical, High, and Medium findings. Present the full plan to the user (in their language). The user may edit, reorder, drop, or add items.
7. **Apply fixes** only after explicit approval of the plan. Apply per finding; verify each edit landed cleanly before moving on.
8. **Deliver** a Markdown report file and a short chat-level summary (in the user's language).

The flow is strict: no fixes happen before plan approval. Subagents never apply edits.

---

## Step 1 — Ask scope

Always ask the user before launching. Use their language. Offer three options explicitly:

- **Whole repository** — every source file under the project root, excluding vendor/build artifacts (`node_modules`, `dist`, `build`, `.venv`, `target`, `vendor`, etc.).
- **A diff** — a specific `git` range. Ask for the range (e.g., `main..HEAD`, last commit, current branch vs. `develop`, a specific PR number).
- **Specific paths** — a list of files or directories.

If the user has already specified the scope in the message that triggered this skill, skip the question and confirm in one short line: "Auditing X. Launching." (translated).

Do not assume. An unclear scope leads to either useless coverage gaps or an audit that takes far too long.

## Step 2 — Assemble inputs

Resolve the scope to a concrete bundle the subagents can reason over:

```yaml
scope_kind: "repo" | "diff" | "paths"
files:
  - path: src/api/user.ts
    language: typescript
    content: <full content; for very large files, head/tail + a note>
manifests:
  - package.json
  - requirements.txt
  - go.mod
  - pom.xml
  - Gemfile.lock
  - composer.json
configs:
  - .env.example
  - dockerfile / docker-compose.yml
  - nginx / apache / caddy configs
  - terraform / k8s manifests
  - ci/cd workflows (.github/workflows, .gitlab-ci.yml, etc.)
framework_hints:
  - react / vue / next / express / fastapi / django / rails / spring / laravel / etc.
git_context:
  - branch, base, range (if diff)
session_language: <user's language>
```

For diff mode, include both the diff hunks and the **full file content** of each affected file so subagents can reason about context, not just hunks.

Exclude obvious noise (lockfile bodies beyond their names, generated code, minified assets, binaries).

## Step 3 — Dispatch 10 category subagents in parallel

Spawn all ten in a **single batch** so they run concurrently. Each subagent receives:

- The full scope bundle from Step 2.
- The category-specific prompt from `agents/<category>.md`.
- A reminder of the standard finding format (see `references/finding-format.md`).
- A hard rule: **do not spawn further subagents**, do not edit files, return only findings as JSON.

Categories and their prompts:

| Code | Category | Subagent file |
|------|----------|---------------|
| A01 | Broken Access Control (includes SSRF) | `agents/a01-broken-access-control.md` |
| A02 | Security Misconfiguration | `agents/a02-security-misconfiguration.md` |
| A03 | Software Supply Chain Failures | `agents/a03-supply-chain-failures.md` |
| A04 | Cryptographic Failures | `agents/a04-cryptographic-failures.md` |
| A05 | Injection | `agents/a05-injection.md` |
| A06 | Insecure Design | `agents/a06-insecure-design.md` |
| A07 | Authentication Failures | `agents/a07-authentication-failures.md` |
| A08 | Software & Data Integrity Failures | `agents/a08-integrity-failures.md` |
| A09 | Security Logging & Alerting Failures | `agents/a09-logging-alerting-failures.md` |
| A10 | Mishandling of Exceptional Conditions | `agents/a10-mishandling-exceptional-conditions.md` |

Orchestration mechanics, batching, and recovery from a failed subagent are documented in `agents/orchestrator.md`. The 2025 category definitions and scope notes are in `references/owasp-top-10-2025.md`.

Each subagent returns a JSON array of findings using the standard format:

```json
{
  "owasp_category": "A05",
  "owasp_category_name": "Injection",
  "findings": [
    {
      "id": "a05-001",
      "title": "Raw SQL concatenation in user lookup",
      "file": "src/api/user.ts",
      "line": 42,
      "description": "User-controlled query string interpolated directly into SQL.",
      "evidence": "const q = `SELECT * FROM users WHERE email = '${req.query.email}'`;",
      "cvss_score": 8.1,
      "cvss_band": "High",
      "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N",
      "proposed_fix": "Use parameterized query: db.query('SELECT * FROM users WHERE email = $1', [req.query.email]).",
      "confidence": "high"
    }
  ]
}
```

The full finding schema (including the `confidence` field and the rules for CVSS scoring) lives in `references/finding-format.md` and `references/cvss-scoring.md`.

## Step 4 — Consolidate findings

- **Merge duplicates.** A single bug surfaced by two categories (e.g., a misconfigured CORS that A01 and A02 both flagged) becomes one finding tagged with both categories. Keep the highest CVSS and union the proposed fixes.
- **Drop false positives** the orchestrator can quickly verify by re-reading the cited `file:line`. If the cited evidence does not match what the file actually contains, the finding is dropped.
- **Sort** by CVSS descending, then by category code ascending for stable ordering.
- **Cap noise.** If a single category returns more than ~20 findings, keep all Critical/High, sample the rest by representative pattern, and note "N additional Low/Medium findings of the same pattern omitted; see report appendix".

## Step 5 — Classify by CVSS band

Apply the standard bands from FIRST.org CVSS v3.1:

| Band | Score range |
|------|-------------|
| Critical | 9.0 – 10.0 |
| High | 7.0 – 8.9 |
| Medium | 4.0 – 6.9 |
| Low | 0.1 – 3.9 |
| None | 0.0 |

Scoring guidance, base-metric defaults per category, and how to handle "no exploit known yet" cases are in `references/cvss-scoring.md`. When in doubt between two scores, the subagent picks the lower one and notes the reasoning.

## Step 6 — Build and present the remediation plan

The plan covers only **Critical, High, and Medium** findings. Low/None findings appear in the report but **not** in the plan (they would dilute attention).

Plan structure:

```
For each finding in the plan:
  - finding id
  - category, title, file:line
  - CVSS score + band
  - proposed fix (concrete steps)
  - estimated blast radius (files touched, tests likely to break)
  - dependency on other findings (if any)
```

Present the **full plan** to the user in their language. Make it clear they can:

- Approve as-is.
- Remove specific items ("skip A05-003, that endpoint is internal").
- Reorder priority.
- Add their own items.
- Promote/demote items between in-plan and report-only.

Wait for explicit approval. Do not apply any fix before approval. Approval is per-plan, not per-finding; once given, work through the approved items in order.

## Step 7 — Apply fixes

For each approved finding:

1. Re-read the file at the cited location to confirm current state.
2. Apply the smallest edit that resolves the finding.
3. Re-read the file to confirm the edit landed cleanly.
4. If a test suite is present and obviously cheap to run for the touched area, run it; otherwise note "tests not run automatically" in the report.
5. If an edit fails or has a side effect the orchestrator did not anticipate, stop, mark that finding as "deferred" with the reason, and continue with the next approved item.

Do not bundle unrelated fixes. Each finding gets its own focused edit.

## Step 8 — Deliver report + chat summary

**Report file** — Markdown, English, written to `owasp-review-<YYYYMMDD-HHMMSS>.md` at repo root (or under `docs/security-reviews/` if that directory already exists). Use the template in `references/report-template.md`. The report includes:

- Scope (what was audited)
- Methodology (10 subagents, CVSS bands)
- Findings table sorted by CVSS
- Per-finding detail (evidence, CVSS vector, fix)
- Remediation plan (with approval status per item)
- What was actually fixed in this session
- Appendix: Low/None findings

**Chat summary** — in the user's language, terse:

- One line on scope.
- Counts by band (e.g., "2 Critical, 5 High, 8 Medium, 11 Low").
- What was fixed.
- What was deferred and why.
- Path to the report file.

---

## Inputs the orchestrator passes to every subagent

```yaml
session_language: <e.g., "es">  # used only for Step 8 and plan presentation
scope_kind: "repo" | "diff" | "paths"
files: [<path + content + language>, ...]
manifests: [<dependency manifests with content>]
configs: [<runtime/infra configs with content>]
framework_hints: [...]
git_context: { branch, base, range }
constraints:
  - "Evaluate only your assigned OWASP category."
  - "Do not spawn further subagents."
  - "Do not edit files."
  - "Return findings as JSON using the standard format."
  - "Write all output in English."
  - "Score every finding with CVSS v3.1."
```

`session_language` is informational for the subagents (e.g., to handle non-English strings in evidence); they must still write all output in English.

## CVSS bands (canonical reference)

| Band | Score range |
|------|-------------|
| Critical | 9.0 – 10.0 |
| High | 7.0 – 8.9 |
| Medium | 4.0 – 6.9 |
| Low | 0.1 – 3.9 |
| None | 0.0 |

Only Critical, High, and Medium enter the remediation plan. Low and None are listed in the report for transparency but never block delivery.

## Anti-patterns to avoid

- **Do not skip the scope question.** Auditing the wrong thing wastes the most expensive part of the run (the parallel dispatch).
- **Do not let subagents fan out further.** They evaluate one category each and stop. Recursive subagents make consolidation impossible.
- **Do not apply any fix before plan approval.** The human owns the call.
- **Do not auto-add Low findings to the plan** because they are "easy". They dilute attention from the items that matter.
- **Do not translate findings or the report.** English is non-negotiable for the analysis surface. Translate only the chat-level summary and the scope/plan questions.
- **Do not invent CVSS scores.** If a subagent cannot justify a score from the base metrics, mark `confidence: low` and pick the more conservative (lower) value.
- **Do not redesign the system.** "Insecure Design" findings under A06 should propose the smallest change that mitigates the specific risk found, not a re-architecture.

## Files in this skill

```
owasp-top-10-reviewer/
├── SKILL.md                                       ← you are here
├── agents/
│   ├── orchestrator.md                            ← how to dispatch + consolidate
│   ├── a01-broken-access-control.md               ← includes SSRF
│   ├── a02-security-misconfiguration.md
│   ├── a03-supply-chain-failures.md
│   ├── a04-cryptographic-failures.md
│   ├── a05-injection.md
│   ├── a06-insecure-design.md
│   ├── a07-authentication-failures.md
│   ├── a08-integrity-failures.md
│   ├── a09-logging-alerting-failures.md
│   └── a10-mishandling-exceptional-conditions.md
└── references/
    ├── owasp-top-10-2025.md                       ← canonical category definitions
    ├── cvss-scoring.md                            ← how to score, base-metric defaults
    ├── finding-format.md                          ← JSON schema for findings
    └── report-template.md                         ← Markdown report layout
```

Load reference files lazily — only the step that actually needs them.
