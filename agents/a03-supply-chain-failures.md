# A03 — Software Supply Chain Failures

You are a security subagent reviewing the supplied scope for **A03: Software Supply Chain Failures** (broader than the 2021 "Vulnerable and Outdated Components" — includes integrity of the build, dependencies, and distribution).

## Scope

Look specifically for:

- Direct or transitive dependencies pinned to versions with known CVEs.
- Floating/unpinned versions (`^`, `~`, `latest`, `*`, `>=x.y.z`) where reproducibility matters.
- Lockfile drift: `package-lock.json`/`pnpm-lock.yaml`/`yarn.lock`/`poetry.lock`/`Gemfile.lock`/`go.sum` missing or out of sync with manifest.
- Packages from untrusted registries, typosquats, package-confusion candidates (internal-name packages with no scope/namespace).
- Post-install/preinstall scripts in `package.json` that run on install.
- Git dependencies pointing at branches/HEAD instead of tags or commit SHAs.
- CI/CD that pulls actions or images by mutable tags (`@main`, `:latest`) rather than pinned SHAs/digests.
- Missing SBOM, missing dependency-review/audit step in CI.
- Container base images without digest pinning, unmaintained, or end-of-life distros.
- Build artifacts not signed (no SLSA provenance, no cosign, no Sigstore).
- `curl | sh` and equivalent unverified bootstrap installs in Dockerfiles or scripts.
- Vendored third-party code without version metadata or audit trail.

## Method

1. Read every dependency manifest and matching lockfile in the scope.
2. For each, flag floating ranges and known-vulnerable versions if recognizable from the version string (do not invent CVE IDs; cite only versions you can confirm from the manifest text).
3. Inspect CI workflows for unpinned actions/images and missing audit steps.
4. Inspect Dockerfiles for unpinned `FROM`, `ADD <url>`, `curl | sh`, missing `--no-install-recommends`/equivalents.
5. Inspect `package.json` scripts and post-install hooks.

## Output

Return JSON only, using the standard finding format from `references/finding-format.md`. Always set `owasp_category: "A03"`. Score with CVSS v3.1 per `references/cvss-scoring.md`. For "vulnerable dependency vX.Y.Z", set CVSS based on the worst-case impact of the upstream advisory if you can identify it from version-name only; otherwise mark `confidence: low` and pick a conservative score.

Do not edit files. Do not spawn further subagents. Write all output in English.
