# Report template

Write to `docs/security-reviews/owasp-review-<YYYYMMDD-HHMMSS>.md` (create the directory if it does not exist). English only. Never add this file to version control — keep `docs/security-reviews/` in `.gitignore`.

Use this exact skeleton; trim empty sections only if they would be misleading.

```markdown
# OWASP Top 10 Security Review — <YYYY-MM-DD HH:MM>

## Scope
- **Kind:** repo | diff | paths
- **Target:** <repo name / branch / range / list of paths>
- **Files reviewed:** <count> (<short summary, e.g., 42 source + 6 configs + 3 manifests>)
- **Frameworks detected:** <list>

## Methodology
Ten subagents, one per OWASP Top 10 (2025) category, dispatched in parallel. Each returned findings in a standard JSON shape. The orchestrator merged duplicates, dropped unverifiable items, sorted by CVSS, and classified by the standard bands (Critical 9.0–10 / High 7.0–8.9 / Medium 4.0–6.9 / Low 0.1–3.9 / None 0.0). Only Critical/High/Medium findings entered the remediation plan.

## Summary by band

| Band | Count |
|---|---|
| Critical | N |
| High | N |
| Medium | N |
| Low | N |
| None | N |

## Findings (sorted by CVSS, descending)

| ID | Category | Title | File:Line | CVSS | Band |
|---|---|---|---|---|---|
| a05-001 | A05 Injection | Raw SQL concatenation in user lookup | src/api/user.ts:42 | 9.8 | Critical |
| ... | ... | ... | ... | ... | ... |

## Per-finding detail

### a05-001 — Raw SQL concatenation in user lookup
- **Category:** A05 Injection
- **File:line:** `src/api/user.ts:42`
- **CVSS:** 9.8 (Critical) — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- **Confidence:** high
- **Description:** <what + why it matters>
- **Evidence:**
  ```ts
  const q = `SELECT * FROM users WHERE email = '${req.query.email}'`;
  ```
- **Proposed fix:** <concrete steps>
- **Blast radius:** <files + likely tests>
- **Status:** fixed | approved-pending | deferred | report-only
- **Notes:** <if any>

(repeat per finding)

## Remediation plan

For each in-plan finding, show: id, title, status, who proposed (skill / user), order.

| Order | ID | Title | Band | Status |
|---|---|---|---|---|
| 1 | a05-001 | Raw SQL concatenation in user lookup | Critical | fixed |
| ... | ... | ... | ... | ... |

## What was fixed in this session

Concise list of `file:line` edits applied, grouped by finding id.

## Deferred / manual actions required

- <finding id> — <reason>

## Methodology notes (if any)

- Subagent for A0X returned incomplete: <reason>.
- N findings dropped as false positives after orchestrator re-verification.

## Appendix — Low / None findings (not in plan)

| ID | Category | Title | File:Line | CVSS | Band |
|---|---|---|---|---|---|

## Appendix — Pattern roll-ups (if noise control was applied)

- "Missing input validation in 14 handlers" — representatives: a05-014, a05-015; remainder omitted for brevity.
```

Keep the report skimmable. The per-finding sections are the heart of it; everything else is navigation.
