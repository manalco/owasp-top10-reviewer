# OWASP Top 10 — 2025 (reference)

The 2025 edition reorganizes the list as follows. This file is the canonical mapping the orchestrator and subagents share. Each subagent owns exactly one category and must not reach into another's.

| Code | Category | Notes |
|------|----------|-------|
| **A01** | Broken Access Control | Now explicitly absorbs **SSRF** (previously A10 in 2021). |
| **A02** | Security Misconfiguration | Unsafe defaults, missing hardening, exposed admin surfaces. |
| **A03** | Software Supply Chain Failures | Renamed from "Vulnerable & Outdated Components" — broader: integrity of dependencies, builds, and distribution. |
| **A04** | Cryptographic Failures | Weak algorithms, bad key/IV/nonce handling, plaintext sensitive data. |
| **A05** | Injection | SQL/NoSQL/OS/LDAP/XPath/SSTI/XSS/CRLF/log/expression-language. **XSS lives here.** |
| **A06** | Insecure Design | Missing or flawed *controls at the design level*, not implementation bugs. |
| **A07** | Authentication Failures | Identification & authentication: rate limiting, MFA, sessions, OAuth/OIDC. |
| **A08** | Software & Data Integrity Failures | Deserialization, unverified webhooks/updates, CSRF, CI/CD integrity. |
| **A09** | Security Logging & Alerting Failures | Logging + the alerting/escalation that makes logging useful. |
| **A10** | Mishandling of Exceptional Conditions | Fail-open behavior, silent catches, missing timeouts, partial writes, race conditions. |

## Boundaries between similar categories

Several real-world bugs sit on a boundary. Use these tiebreakers; if a finding still touches two categories, tag both and keep the higher CVSS.

- **CORS misconfig** → A02 (config) primarily; A01 if it directly enables unauthorized access to data.
- **SSRF** → **A01**, not A10.
- **XSS** → **A05** (Injection), not A02 or A06.
- **CSRF** → **A08** (Integrity), not A01 or A07.
- **Insecure deserialization** → **A08** (Integrity).
- **Hard-coded credentials in source** → A02 (Misconfig) primarily; A04 if the credential is a cryptographic key.
- **Weak password hashing** → A04 (Crypto) for the algorithm choice; A07 if the registration/login flow itself is the issue.
- **Verbose stack trace returned to client** → A02 primarily; A09 if it indicates missing structured logging; A10 if it is the result of an unhandled exception that also fails open.
- **Log injection** → A05 (Injection) for the sink; A09 for the missing sanitizer/structured-logging story.
- **Missing rate limit on login** → A07.
- **Missing rate limit on a paid AI endpoint** → A06 (Insecure Design — business-logic abuse).
- **Mass assignment** → A01 (privilege escalation surface).
- **CI/CD uses `@main` action** → A03 (supply chain) primarily; A08 if it is specifically about integrity of the build pipeline.

## What every subagent gets, regardless of category

- Full file contents in scope, with `file:line` references.
- Dependency manifests + lockfiles.
- Runtime/infra configs (Dockerfile, compose, k8s, nginx, CI workflows, framework settings).
- Framework hints (so a subagent knows whether `app.run(debug=True)` is Flask or something else).
- The standard finding format and the CVSS scoring guidance.
