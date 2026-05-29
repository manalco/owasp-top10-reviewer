# OWASP Top 10 Reviewer

> A Claude Code skill that runs an automated, parallelized security review based on the **OWASP Top 10 (2025)**.

![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)
![OWASP](https://img.shields.io/badge/OWASP-Top%2010%202025-blue.svg)
![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-orange.svg)

`OWASP Top 10 Reviewer` is a skill for [Claude Code](https://docs.claude.com/en/docs/claude-code) that performs a structured security audit of your codebase. An orchestrator agent dispatches ten subagents in parallel — one per OWASP Top 10 (2025) category — consolidates their findings, scores them with CVSS, and produces an actionable remediation plan.

> [!WARNING]
> **This skill consumes MANY tokens**, since it runs 10 agents in parallel. If token usage is a concern, you are strongly advised to scan specific directories instead of the whole repository.
>
> **Rough estimate per 100 files scanned:** approximately **1.5M – 3M tokens**.
>
> The math (illustrative — actual usage varies widely):
> - An average source file ≈ **2,000–3,000 tokens**, so 100 files ≈ **250K tokens** of code in a single pass.
> - With **10 subagents** each reading the relevant code, input is multiplied up to **~10×** → roughly **2.5M tokens**.
> - Add per-agent overhead (instructions, reasoning, findings reporting) and the orchestrator's consolidation + remediation plan: **~150K–300K tokens**.
>
> Actual consumption depends heavily on file size, code density, number of findings, and how much of the codebase each subagent needs to read. Scoping to specific paths is the single most effective way to reduce cost.

## Features

- **Parallel review** — 10 subagents, one per OWASP 2025 risk category.
- **Orchestrator pattern** — a single coordinating agent makes decisions and is the only one that talks to you.
- **CVSS scoring** — every finding is classified with a numeric CVSS score and band (Critical / High / Medium / Low / None).
- **Remediation plan** — generated for Critical, High, and Medium findings; you can edit it before any fix is applied.
- **Stack-agnostic** — works with any language or framework.
- **Multilingual UX** — all internal analysis and the report are produced in English, while communication with you happens in your own language.
- **Dual output** — a Markdown report file plus a chat/console summary.

## OWASP Top 10 (2025) Coverage

| # | Category |
|----|----------|
| A01 | Broken Access Control *(includes SSRF)* |
| A02 | Security Misconfiguration |
| A03 | Software Supply Chain Failures |
| A04 | Cryptographic Failures |
| A05 | Injection |
| A06 | Insecure Design |
| A07 | Authentication Failures |
| A08 | Software & Data Integrity Failures |
| A09 | Security Logging & Alerting Failures |
| A10 | Mishandling of Exceptional Conditions |

## How It Works

1. **Scope** — the orchestrator asks what to analyze: the whole repository, a git diff, or specific paths.
2. **Dispatch** — 10 subagents run in parallel according to the chosen scope.
3. **Consolidate** — findings are collected using a standard format: `{OWASP category, file:line, description, evidence, CVSS score, band, proposed fix}`.
4. **Classify** — by numeric CVSS: Critical 9.0–10 · High 7.0–8.9 · Medium 4.0–6.9 · Low 0.1–3.9 · None 0.
5. **Plan** — a remediation plan is generated for Critical/High/Medium findings. Low/None are listed but excluded from the plan.
6. **Review** — the full plan is presented to you; you may modify it before remediation begins.
7. **Remediate** — after approval, the fixes are applied.
8. **Deliver** — a Markdown report is saved and a summary is shown in chat/console.

## Installation

Clone this repository into your Claude Code skills directory:

```bash
git clone https://github.com/manalco/owasp-top10-reviewer.git
```

Place the skill folder where Claude Code discovers skills (e.g. `~/.claude/skills/` or your project's skills directory), then restart Claude Code.

## Usage

Inside Claude Code, simply ask for a security review:

```
Run a security review on this project
```

or

```
Audit this codebase against the OWASP Top 10
```

The skill triggers automatically and the orchestrator will guide you from there.

## Tags

`owasp` · `owasp-top-10` · `security` · `security-audit` · `static-analysis` · `appsec` · `claude-code` · `claude-skill` · `cvss` · `code-review` · `vulnerability-scanner` · `devsecops`

## License

This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/).

You are free to share and adapt the material under the following terms: **Attribution**, **NonCommercial**, and **ShareAlike**.

## Author

[Manuel Alvarado](https://github.com/manalco)

## Disclaimer

This tool assists with security review but does not replace a professional security audit. Always validate findings and remediations before deploying to production.
