# Security Policy

## Reporting a Vulnerability

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, please report them via email to: **security@gsd.build** (or DM @glittercowboy on Discord/Twitter if email bounces)

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Any suggested fixes (optional)

## Response Timeline

- **Acknowledgment**: Within 48 hours
- **Initial assessment**: Within 1 week
- **Fix timeline**: Depends on severity, but we aim for:
  - Critical: 24-48 hours
  - High: 1 week
  - Medium/Low: Next release

## Scope

Security issues in the GSD codebase that could:
- Execute arbitrary code on user machines
- Expose sensitive data (API keys, credentials)
- Compromise the integrity of generated plans/code

## Recognition

We appreciate responsible disclosure and will credit reporters in release notes (unless you prefer to remain anonymous).

---

## Dependency Integrity Verification

### Purpose

The `scripts/check-npm-integrity.sh` gate detects three classes of dependency
drift that can silently introduce security or reliability risk:

- **Invalid** — an installed package version does not satisfy the declared semver
  range (e.g., `ws@8.20.0` installed when `8.20.1` is declared). This was the
  original incident that prompted this gate.
- **Missing** — a declared dependency is absent from `node_modules/`.
- **Extraneous** — a package is present in `node_modules/` but not declared as a
  dependency.

This aligns with NIST SSDF PW.4.1 (use components from well-governed, secure
sources: https://csrc.nist.gov/publications/detail/sp/800-218/final) and the
OpenSSF Scorecard "Pinned-Dependencies" check
(https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies).

### Invoking locally

```bash
./scripts/check-npm-integrity.sh
# or via npm script:
npm run check:integrity
```

The script exits 0 on a clean install and 1 on any finding, with a structured
report to stderr listing every offender and both the declared and installed
versions for invalid packages.

Options:
- `--ignore-extraneous` — suppress extraneous-only failures (useful when
  intentionally adding packages before updating the lockfile)
- `--help` — print usage and exit 0

### Remediation

The canonical fix for any drift is:

```bash
rm -rf node_modules && npm ci
```

Then verify with `npm run check:integrity` before committing.

### Bypass policy

There is no bypass flag. If the gate must be skipped for a specific commit
(e.g., during a lockfile migration), document the reason in the commit message.
CI workflow steps can be skipped via `if: false` with a comment explaining why
and a follow-up issue number. Any such skip must be reversed in a subsequent
commit before the PR is merged.

### Scope

The gate runs `npm ls --all --json` at the repository root. The `sdk/`
sub-directory is a separate, non-workspace package and is out of scope for this
single invocation. If `sdk/` is ever declared as a workspace in root
`package.json`, it will be covered automatically (npm >=7 traverses workspaces
by default).

### CI coverage

The gate runs in:
- `test.yml` — all matrix jobs and the coverage job, after `npm ci`
- `release.yml` — rc and finalize jobs, after `npm ci`
- `security-scan.yml` — before all diff-based source scans
