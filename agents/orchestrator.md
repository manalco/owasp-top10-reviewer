# Orchestrator

The orchestrator is the only agent that talks to the human. Subagents only report findings.

## Responsibilities

1. Ask scope (Step 1 in SKILL.md). Always in the user's language.
2. Assemble the scope bundle (Step 2). Concrete file list, contents, manifests, configs.
3. Dispatch ten category subagents **in a single batch** so they run in parallel (Step 3).
4. Consolidate, deduplicate, sort, and classify findings by CVSS band (Steps 4–5).
5. Build the remediation plan for Critical/High/Medium and present it in the user's language. Wait for explicit approval (Step 6).
6. Apply approved fixes one by one (Step 7).
7. Write the Markdown report and post the chat summary (Step 8).

## Dispatch mechanics

- Spawn all ten subagents in a single message containing ten parallel tool calls. Do **not** spawn them serially.
- Each spawn passes:
  - The category-specific prompt file from `agents/`.
  - The full scope bundle from Step 2.
  - The constraint block from SKILL.md ("evaluate only your assigned category", "do not spawn further subagents", "do not edit files", "return JSON only", "write all output in English", "score every finding with CVSS v3.1").
  - The finding format from `references/finding-format.md`.
  - The CVSS scoring guidance from `references/cvss-scoring.md`.

## Handling subagent failures

If a subagent returns malformed JSON, returns nothing, or errors out:

- Retry **once** with the same prompt.
- If it fails again, record the category as `status: "incomplete"` in the report and continue. Do not block the other nine.

If a subagent appears to have fanned out further (spawned its own subagents), record the violation in the report under "methodology notes" and use only its top-level findings.

## Consolidation rules

- Merge findings that point to the same `file:line` (or to overlapping line ranges within ±3 lines) when their `proposed_fix` is compatible. Union the OWASP categories. Keep the highest CVSS.
- Drop findings where the orchestrator re-reads the cited `file:line` and the evidence does not match. Log the drop count in the report.
- Sort findings by CVSS descending; ties broken by category code ascending; further ties broken by file path.

## Noise control

If any single category returns more than ~20 findings:

- Keep every Critical and High.
- For Medium and Low, group by pattern (e.g., "missing input validation in 14 handlers"). Keep one representative per pattern in the main report and list the rest in an appendix.

## Plan presentation

The plan must show, for each in-scope finding:

- Finding id, category, title, file:line.
- CVSS score and band.
- Proposed fix in concrete steps.
- Estimated blast radius (files touched, tests likely affected).
- Dependencies on other findings.

Ask the user, in their language, whether to: approve as-is, remove items, reorder, add items, or move items between in-plan and report-only. Wait for explicit approval before any edit.

## Safety rules

- Never edit files before plan approval.
- Never delete files, drop database objects, or revoke credentials from a fix step. If a finding requires such action, surface it as a "manual action required" item in the plan, not as an auto-fix.
- Apply each approved fix as the smallest viable edit; re-read the file after each edit to confirm.

## Report and summary

- Write the Markdown report in English to `docs/security-reviews/owasp-review-<YYYYMMDD-HHMMSS>.md` (create the directory if needed). Use `references/report-template.md`.
- The report must never enter version control: ensure `docs/security-reviews/` is in `.gitignore` (add it, or create `.gitignore`, if missing) and never `git add`/commit the report. It maps live vulnerabilities, so committing it would expose the unfixed weaknesses and persist them in history.
- Post the chat summary in the user's language: scope, counts by band, what was fixed, what was deferred, path to the report.
