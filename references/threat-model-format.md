# THREAT_MODEL.md — format and lifecycle

`THREAT_MODEL.md` lives at the **root of the audited repository**. It is the skill's memory of **known non-issues**: false positives and patterns that another analysis could flag as a vulnerability but that have been verified as safe, each with its evidence. Its purpose is to save tokens on later runs — entries that still match the code are passed to subagents as "do not report" and are never re-analyzed.

It is a **tracked file** (meant to be committed), unlike the reports in `docs/security-reviews/`. It must contain **only non-issues**. Never write a real, unfixed vulnerability into it — that belongs in the (gitignored) report.

English only, like every other artifact of this skill.

## File skeleton

Create the file with this exact header when it does not exist. Append entries below `## Entries`; never reorder or renumber existing ones.

```markdown
# Threat Model — Known Non-Issues

This file is maintained by the OWASP Top 10 Reviewer skill. Each entry documents something a security analysis could flag but that has been verified as **not** a vulnerability, with its evidence. Future scans skip entries whose anchor still matches the code.

- Do not record real vulnerabilities here.
- An entry whose anchor no longer matches the code is stale and is re-analyzed.
- Edit or delete entries by hand at any time; keep IDs stable.

## Entries

### TM-001 — Admin route without auth middleware is bound to localhost only
- **Status:** active
- **Category:** A01
- **Location:** `src/server/admin.ts:12-18`
- **Anchor:**
  ```ts
  adminApp.get('/debug/state', (req, res) => res.json(state));
  ```
- **Would be flagged as:** Missing authorization check on an admin endpoint.
- **Why it is not an issue:** `adminApp` only listens on `127.0.0.1` and the port is not exposed by the container.
- **Evidence:**
  - `src/server/admin.ts:30` — `adminApp.listen(9001, '127.0.0.1')`
  - `docker-compose.yml:14` — only port `8080` is published.
- **Source:** orchestrator-verified | user-confirmed
- **Recorded:** 2026-11-10 @ `abc1234`
```

## Field rules

- **ID** — `TM-<3-digit>`, sequential, never reused (even after an entry is deleted).
- **Status** — `active` or `stale`. Only `active` entries suppress findings.
- **Category** — one or more OWASP codes (`A01`..`A10`), comma-separated, or `ANY` when the non-issue applies to every category.
- **Location** — `path:line` or `path:start-end`, relative to the repo root.
- **Anchor** — verbatim code (≤ 5 lines) at the location. This is what keeps entries honest: it is matched as text, so line drift alone does not invalidate an entry, but a real code change does.
- **Would be flagged as** — one line with the finding title a scanner would produce. Used for matching new findings.
- **Why it is not an issue** — 1–3 sentences.
- **Evidence** — concrete `file:line` references and short quotes for the mitigating control (config, middleware, network boundary, type guarantee, test). "Trust me" is not evidence; an entry without verifiable evidence must not be written.
- **Source** — `orchestrator-verified` (the orchestrator proved the finding false by reading the code) or `user-confirmed` (the user rejected the finding as not a vulnerability and gave the reason).
- **Recorded** — date and short commit SHA at the time of writing (`git rev-parse --short HEAD`; omit the SHA if not a git repo).

## Loading (before dispatch)

1. If `THREAT_MODEL.md` does not exist, skip — there is nothing to load.
2. Parse the `active` entries.
3. **Validate each anchor** against the current code: search for the anchor text in the file at `Location` (whitespace-insensitive). Do not read whole files for this — a targeted search is enough.
   - Found → the entry is **valid**. Update its effective line range to where the anchor now is.
   - File missing or anchor not found → the entry is **stale** for this run. It is not passed to subagents; the code is analyzed normally. Remember it for the update step.
4. Only validate entries whose `Location` is inside the current scope. Out-of-scope entries are ignored (neither valid nor stale).
5. Build a compact `known_non_issues` list per category (entries tagged `ANY` go to every category):

```yaml
known_non_issues:
  - id: TM-001
    category: [A01]
    location: src/server/admin.ts:12-18
    would_be_flagged_as: "Missing authorization check on an admin endpoint."
    why_not: "adminApp only listens on 127.0.0.1; port not published."
```

Pass only the summary above, never the full evidence block — the evidence was already verified when the entry was written, and re-sending it would defeat the token savings.

## Matching (after consolidation)

As a safety net, after Step 4 drop any finding that matches a valid entry:

- same file, **and**
- the finding's line range overlaps the entry's effective range (±3 lines) or the finding's `evidence` contains the anchor text, **and**
- the finding's category is in the entry's categories (or the entry is `ANY`), **and**
- the finding's title describes the same issue as `Would be flagged as`.

Count suppressed findings for the report. If a finding sits at the same location but describes a **different** issue, keep it — the entry only covers what it documents.

## Updating (after the report)

Candidates for new entries:

- Findings the orchestrator dropped in Step 4 because the code **was read and the issue is mitigated** (record the mitigating evidence). Findings dropped because the subagent cited a line that does not exist or quoted code that is not there are hallucinations, not non-issues — never record those.
- Findings the user rejected during plan review **as not being a vulnerability**, with a reason. A finding the user merely deprioritized, postponed, or accepted as a known risk is still a real issue and must not be recorded.

Procedure:

1. Build the candidate list (title, category, location, why, evidence) and the stale list from the loading step.
2. Show both to the user in their language. For stale entries propose: refresh the anchor (the reason still holds), or remove the entry. Ask for confirmation; the user may drop or edit any candidate.
3. If no candidates were confirmed and no stale entry changed, do not create or touch the file.
4. Otherwise, create `THREAT_MODEL.md` from the skeleton if missing, append the confirmed entries with the next free IDs, and apply the confirmed stale changes (refresh anchor + `Recorded`, or delete). Unconfirmed stale entries are set to `Status: stale`.
5. `THREAT_MODEL.md` is meant to be committed. Offer to commit it on its own — never in the same commit as code fixes, and never together with the report.
