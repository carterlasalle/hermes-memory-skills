# Security Policy

## Reporting a vulnerability

Please report vulnerabilities **privately** — do not open a public issue.

- **Preferred:** GitHub Security Advisories — use the repo's **Security tab →
  "Report a vulnerability"**, or from the CLI:

  ```bash
  gh security-advisory create
  ```

  (For an existing private advisory: `gh security-advisory edit`.)

- **Alternative:** email the maintainers directly if you know them; otherwise
  use Security Advisories so the report reaches the right inbox.

## What to include

- Repository/commit where the issue appears (or "latest `main`").
- A minimal description of the vulnerability and its impact.
- Any reproduction steps or example payload. If it involves a secret you
  believe is real (API key, token), **redact it** — don't paste live secrets
  into the advisory.

## Expected response time

- **Acknowledgement:** within 72 hours of the report.
- **Triage/initial response:** within 7 days — we will confirm or dispute
  scope and outline next steps.
- **Fix/remediation:** depends on severity; we aim for a fix on `main` within
  30 days for confirmed issues.

## Scope

This repository is a **markdown/skills-only** repo: there is no runtime code or
build pipeline of its own. However, the skill content is executed as
instructions by Hermes Agent during memory operations — so prompt-injection
style issues in skill text, or secrets accidentally committed to markdown, are
in scope.

- Secrets scanning is enforced by `.gitleaks.toml` (local `gitreins guard`)
  and `.github/workflows/ci.yml` (docs-health grep on every push/PR) — a
  committed secret is a valid report.
- Issues in the underlying tools (Hermes Agent, DuckBrain, Holographic) belong
  in those projects' own security channels.

## Safe harbor

Research done in good faith and reported privately per this policy is
welcome; we will not pursue legal action for such research.
