# Contributing to hermes-memory-skills

This is a **markdown-only skills repo** — no code, no packages, no test suite.
The deliverable is Hermes Agent memory skills (SKILL.md files plus supporting
docs/specs) that route memory consolidation across backends (built-in,
Holographic, DuckBrain). "Build" and "test" here mean: quality gates pass and
the content is accurate.

## Quality gates

Two independent gates run on every change:

### 1. GitReins guard (local, on every commit)

The pre-commit hook (`.git/hooks/pre-commit`) runs `gitreins guard` and blocks
the commit if any Tier-1 guard fails. For this repo the guard is configured
(`.gitreins/config.yaml`) as:

- **secrets** — gitleaks + built-in regex scanner (sk-*, ghp_*, AKIA, …).
  Markdown content IS scanned — `.gitleaks.toml` only allows VCS/gitreins/log
  paths. Do not paste real API keys into docs, examples, or commit messages.
- **lint** — ruff (warns only; no Python here, so effectively a no-op).
- **tests** — `test_command: "true"`, `test_mode: diff` — intentional no-op for
  a docs repo. Don't invent a test suite.

Run it manually any time:

```bash
gitreins guard
```

If it fails, the summary is truncated — re-run the reported command yourself
to see the real error.

### 2. GitHub Actions docs-health (CI)

`.github/workflows/ci.yml` runs on push to `main` and on PRs. It verifies
`README.md` and `AGENTS.md` exist and greps all markdown for obvious secret
patterns. This is an independent check so GitReins verdicts can be cross-checked
against real CI. Keep docs present and secret-free and it stays green.

## Commit conventions

Use [Conventional Commits](https://www.conventionalcommits.org/). Common types
for this repo: `docs`, `fix`, `ci`, `chore`, `feat`.

```bash
git commit -m "docs: add DuckBrain backend reference"
```

The pre-commit hook runs `gitreins guard` automatically; docs-only changes pass.
Don't skip the hook. Local state (`.gitreins/tasks.yaml`, `.gitreins/history/`,
`.gitleaks.toml`, `.hilo/`) is gitignored — never commit it.

## Fork workflow

This repo is a fork of `nexus9888/hermes-memory-skills` adding DuckBrain as a
first-class backend. Remotes: `origin` = your fork, `upstream` = nexus9888.

Keep `upstream` in sync by **merging**, never rewriting history
(see AGENTS.md):

```bash
git fetch upstream
git checkout main
git merge upstream/main        # merge, do NOT rebase / force-push
git push origin main
```

`main` on the fork is the shared working branch; feature branches are fine for
larger changes, but the fork's history must stay a strict superset of upstream.

## Opening a PR

Fork → branch → commit → push → open the PR:

```bash
git checkout -b docs/my-change
# ...make edits...
git commit -m "docs: ..."
git push -u origin docs/my-change
gh pr create --title "docs: ..." --body "What/why"
```

Or after pushing, open the PR from the repo's Pull Requests tab. The CI
docs-health workflow runs on the PR. Tag the board task in the commit message
(e.g. `docs(meta): DOCS-001 — …`) so the change is traceable.
