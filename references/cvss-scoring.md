# CVSS v3.1 scoring guidance

Every finding must carry a numeric `cvss_score` (0.0–10.0), a `cvss_band` (`Critical` / `High` / `Medium` / `Low` / `None`), and a `cvss_vector` (the v3.1 vector string).

This file gives subagents enough to score consistently without needing to derive each metric from scratch.

## Bands (canonical)

| Band | Score range |
|------|-------------|
| Critical | 9.0 – 10.0 |
| High | 7.0 – 8.9 |
| Medium | 4.0 – 6.9 |
| Low | 0.1 – 3.9 |
| None | 0.0 |

These come from FIRST.org and are the same bands used by NVD.

## Base metric quick guide

- **AV** Attack Vector: `N` Network, `A` Adjacent, `L` Local, `P` Physical. Default `N` for web/API findings.
- **AC** Attack Complexity: `L` Low, `H` High (only when exploitation requires specific conditions the attacker cannot trigger).
- **PR** Privileges Required: `N` None, `L` Low (any authenticated user), `H` High (admin).
- **UI** User Interaction: `N` None, `R` Required.
- **S** Scope: `U` Unchanged, `C` Changed (e.g., XSS that hits other users, multi-tenant leakage).
- **C/I/A** Confidentiality/Integrity/Availability impact: `H` High, `L` Low, `N` None.

## Defaults by category

Use these as starting points; adjust each metric only when the code clearly justifies it.

- **A01 Broken Access Control / IDOR** — `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N` → ~8.1 High.
- **A01 SSRF (cloud, hits metadata)** — `AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:N` → ~9.3 Critical.
- **A02 Debug mode in production** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` → ~7.5 High.
- **A02 Missing security header (CSP / HSTS)** — `AV:N/AC:H/PR:N/UI:R/S:C/C:L/I:L/A:N` → ~4.7 Medium.
- **A03 Known-vulnerable dependency (no upgrade path identified)** — score from the upstream advisory if you can; otherwise default `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:L` → ~5.3 Medium with `confidence: low`.
- **A04 Weak password hash (md5/sha1)** — `AV:N/AC:H/PR:H/UI:N/S:U/C:H/I:N/A:N` → ~4.4 Medium (requires DB compromise to exploit).
- **A04 Hard-coded private key in repo** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` → ~9.1 Critical.
- **A05 SQL injection (unauth)** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` → 9.8 Critical.
- **A05 Stored XSS** — `AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N` → ~5.4 Medium (higher if it hits admin UI).
- **A05 Reflected XSS** — `AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N` → ~6.1 Medium.
- **A05 OS command injection (unauth)** — 9.8 Critical.
- **A06 Business logic — coupon replay / negative qty** — `AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N` → ~6.5 Medium (higher if monetary blast radius is large).
- **A07 Login without rate limit** — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N` → ~5.3 Medium.
- **A07 Predictable password-reset token** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` → ~9.1 Critical.
- **A08 Insecure deserialization (untrusted input)** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` → 9.8 Critical.
- **A08 Missing CSRF on state-changing endpoint** — `AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:H/A:N` → ~6.5 Medium.
- **A08 Unverified webhook** — `AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N` → ~7.5 High.
- **A09 Missing logs on auth events** — `AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:N/A:N` → low; score by downstream impact (often 3.1 Low). Score higher only when combined with another finding the logs would have caught.
- **A09 Sensitive data in logs (passwords/tokens)** — `AV:L/AC:L/PR:H/UI:N/S:U/C:H/I:N/A:N` → ~4.4 Medium; raise to High if logs are shipped off-host.
- **A10 Fail-open auth check** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` → ~9.1 Critical.
- **A10 Missing HTTP timeout** — `AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H` → ~7.5 High.
- **A10 Silent exception swallow on permission check** — same as fail-open above.

## When you cannot be sure

- Pick the **lower** of the two candidate scores.
- Set `confidence: "low"` in the finding.
- Document the uncertainty in the `description` field.

## Vector string format

Always start with the prefix `CVSS:3.1/` followed by the seven base metrics in the order `AV/AC/PR/UI/S/C/I/A`, separated by `/`. Example:

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

Do not include temporal or environmental metrics unless the context strongly justifies them.
