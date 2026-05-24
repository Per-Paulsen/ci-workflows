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
# Cancel superseded runs on the same ref (don't waste minutes on rapid pushes).
concurrency:
  group: pr-checks-${{ github.ref }}
  cancel-in-progress: true
# A called workflow cannot exceed its caller's permissions; grant the union here.
permissions:
  contents: write # autofix
  pull-requests: write # claude-review
  issues: write # claude-review
  id-token: write # claude-review OIDC
jobs:
  ci:
    uses: Per-Paulsen/ci-workflows/.github/workflows/node-ci.reusable.yml@v1
  gitleaks:
    uses: Per-Paulsen/ci-workflows/.github/workflows/gitleaks.reusable.yml@v1
  review:
    if: github.event_name == 'pull_request'
    continue-on-error: true # advisory; skips on workflow-changing PRs — never blocks
    uses: Per-Paulsen/ci-workflows/.github/workflows/claude-review.reusable.yml@v1
    secrets: inherit
  autofix:
    if: github.event_name == 'pull_request'
    continue-on-error: true # advisory, never blocks
    uses: Per-Paulsen/ci-workflows/.github/workflows/autofix.reusable.yml@v1
```

For a non-Node repo, drop the `ci` and `autofix` jobs (keep `review` + `gitleaks`).

## One-time setup per consuming repo

- Add an `ANTHROPIC_API_KEY` repository secret (Settings → Secrets and variables → Actions). `secrets: inherit` forwards it to the review workflow. Without it, the review step skips cleanly.
- Install the Claude GitHub App once (https://github.com/apps/claude, "All repositories"). Required for `claude-review`; an API key alone is not enough.
- `claude-review` only runs for real once `pr-checks.yml` is on the repo's **default branch** (a security check skips it on the PR that first introduces it).

## Versioning

Consumers pin to the moving major tag **`@v1`** (recommended), not `@main` — `@main`
would break every consumer on a bad push. To publish changes here: commit to `main`,
then move the tag:

```bash
git tag -f v1 && git push -f origin v1
```

Pin to a full commit SHA instead of `@v1` for maximum immutability.

## Notes

- `node-ci` runs a repo-defined `type-check` script if present (blocking), `eslint`
  (non-blocking), and `npm test` (blocking).
- `node-ci` / `autofix` accept an optional `node-version` input (default `"20"`).
