# Standard finding format

Every subagent returns a single JSON object with this shape. The orchestrator parses these and merges them.

```json
{
  "owasp_category": "A05",
  "owasp_category_name": "Injection",
  "status": "complete",
  "findings": [
    {
      "id": "a05-001",
      "title": "Raw SQL concatenation in user lookup",
      "file": "src/api/user.ts",
      "line": 42,
      "line_end": 44,
      "description": "User-controlled query string is interpolated directly into a SQL statement, allowing arbitrary SQL execution against the users table.",
      "evidence": "const q = `SELECT * FROM users WHERE email = '${req.query.email}'`;",
      "cvss_score": 9.8,
      "cvss_band": "Critical",
      "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
      "confidence": "high",
      "proposed_fix": "Use a parameterized query: `db.query('SELECT * FROM users WHERE email = $1', [req.query.email])`. Reject non-string emails before the query and apply a length cap.",
      "fix_blast_radius": {
        "files": ["src/api/user.ts"],
        "tests_likely_affected": ["test/api/user.spec.ts"],
        "manual_action_required": false
      },
      "depends_on": []
    }
  ]
}
```

## Field reference

- `owasp_category` — exactly one of `A01`..`A10`. The subagent's assigned category.
- `owasp_category_name` — human-readable category name from `references/owasp-top-10-2025.md`.
- `status` — `"complete"` on success. If the subagent had to bail (missing input, ambiguous scope), return `"incomplete"` and add a `reason` field at the top level.
- `findings` — array; may be empty.
- `id` — `<category-lowercase>-<3-digit>`, e.g. `a05-001`. Unique within the subagent's output.
- `title` — short, single sentence, English.
- `file` / `line` / `line_end` — must point to a real location in the scope. `line_end` optional.
- `description` — what the bug is and why it matters, English, 1–4 sentences.
- `evidence` — verbatim quote from the source (≤ 5 lines). Quote strings in their original language; do not translate.
- `cvss_score` — number, 0.0–10.0.
- `cvss_band` — `Critical` / `High` / `Medium` / `Low` / `None`. Must match `cvss_score`.
- `cvss_vector` — full v3.1 vector string per `references/cvss-scoring.md`.
- `confidence` — `high` / `medium` / `low`. `low` flags findings the orchestrator should re-verify before plan inclusion.
- `proposed_fix` — concrete, minimal change. Mention library/function names where helpful.
- `fix_blast_radius.files` — files the fix touches.
- `fix_blast_radius.tests_likely_affected` — best-guess test files; empty array if unknown.
- `fix_blast_radius.manual_action_required` — `true` if the fix needs human action (rotate a key, restart a service, change infra). The orchestrator will not auto-apply these.
- `depends_on` — array of `id`s of other findings that must be fixed first. Usually empty.

## Rules

- Return JSON only. No prose around it. No code fences.
- All values in English (evidence may quote non-English code/strings verbatim).
- If unsure of a CVSS metric, pick the lower-scoring option and mark `confidence: "low"`.
- Never fabricate `file:line` references. If you cannot pin the exact line, return the closest line and note the uncertainty in `description`.
- Never propose a fix you cannot justify from the evidence shown.
- Do not include findings outside your assigned category. If you suspect overlap, leave the cross-category tag to the orchestrator.
