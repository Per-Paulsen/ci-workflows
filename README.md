# ci-workflows

Central **reusable GitHub Actions workflows** for Per Paulsen's repositories. Define the PR-check logic once here; every other repo references it with a tiny caller file (no copy-paste, no drift). Update a workflow here → all consuming repos pick it up on their next run.

## What's here

| Reusable workflow | Purpose | Stack |
|---|---|---|
| `node-ci.reusable.yml` | `npm ci` → ESLint (non-blocking) + Vitest/`npm test` (blocking) | Node |
| `autofix.reusable.yml` | `eslint --fix` and auto-commit fixes back to the PR branch | Node |
| `claude-review.reusable.yml` | Independent fresh-context AI review via `anthropics/claude-code-action` | any |
| `gitleaks.reusable.yml` | Secret scan on the diff | any |

## How to consume it

Add **one** file to the consuming repo at `.github/workflows/pr-checks.yml` (the `/ci-init` skill does this automatically). Example for a Node repo:

```yaml
name: PR Checks
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
jobs:
  ci:
    uses: Per-Paulsen/ci-workflows/.github/workflows/node-ci.reusable.yml@main
  gitleaks:
    uses: Per-Paulsen/ci-workflows/.github/workflows/gitleaks.reusable.yml@main
  review:
    if: github.event_name == 'pull_request'
    uses: Per-Paulsen/ci-workflows/.github/workflows/claude-review.reusable.yml@main
    secrets: inherit
  autofix:
    if: github.event_name == 'pull_request'
    uses: Per-Paulsen/ci-workflows/.github/workflows/autofix.reusable.yml@main
```

For a non-Node repo, drop the `ci` and `autofix` jobs (keep `review` + `gitleaks`).

## One-time setup per consuming repo

- Add an `ANTHROPIC_API_KEY` repository secret (Settings → Secrets and variables → Actions). `secrets: inherit` forwards it to the review workflow. Without it, the review step skips cleanly.
- Install the Claude GitHub App once (https://github.com/apps/claude, "All repositories"). Required for `claude-review`; an API key alone is not enough.
- `claude-review` only runs for real once `pr-checks.yml` is on the repo's **default branch** (a security check skips it on the PR that first introduces it).

## Notes

- Pin to `@main` for always-latest, or to a tag/SHA for stability.
- `node-ci` / `autofix` accept an optional `node-version` input (default `"20"`).
