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
    uses: Per-Paulsen/ci-workflows/.github/workflows/claude-review.reusable.yml@v1
    secrets: inherit
  autofix:
    if: github.event_name == 'pull_request'
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

## Gotchas / Troubleshooting

Hard-won lessons — each of these cost a failed run to discover.

- **`claude-review` needs the Claude GitHub App installed**, not just the secret. Without
  https://github.com/apps/claude on the repo you get `401: Claude Code is not installed
  on this repository`. The `ANTHROPIC_API_KEY` secret alone is not enough.
- **Caller must declare a top-level `permissions:` block.** A reusable workflow cannot have
  more permission than its caller. If the caller grants nothing, the run fails at startup
  (the reusables need `id-token: write` + `contents/pull-requests/issues: write`).
- **`continue-on-error` is NOT valid on a job that `uses:` a reusable workflow** — it causes
  a `startup_failure` ("workflow file issue"). To make a reusable advisory, put
  `continue-on-error` on the **step inside the reusable** (that's how `claude-review` is
  non-blocking).
- **`claude-review` skips/fails on workflow-changing PRs by design.** Its anti-tampering
  check requires the running workflow file to match the default branch. On the PR that
  first *adds* it → skips (green). On a PR that *modifies* `pr-checks.yml` → the action
  errors; the reusable's `continue-on-error` step keeps the job green. Real reviews run on
  normal PRs that don't touch the workflow.
- **Pin consumers to `@v1`, never `@main`.** To publish: `git tag -f v1 && git push -f origin v1`.
- **`node-ci`/`autofix` set dummy CI env** (DATABASE_URL/AUTH_SECRET/ENCRYPTION_KEY) so
  Prisma/Auth.js/encryption module-load doesn't crash. CI never touches a real DB.
- **`autofix` only fixes auto-fixable rules.** `no-unused-vars` and `react-hooks/*` have no
  fixer, so `eslint --fix` may change nothing. To auto-remove unused imports, add
  `eslint-plugin-unused-imports` to the repo's eslint config.
- **Type-check for Next + Vitest:** repo script `"type-check": "next typegen && tsc --noEmit"`
  (bare `tsc` fails without `next-env.d.ts`), plus a `src/test/vitest-globals.d.ts` with
  `/// <reference types="vitest/globals" />` so `tsc` sees the test globals (don't set
  `compilerOptions.types`, it drops the auto-included @types).
- **`gh run rerun --failed` does not reliably re-resolve `@v1`/`@main` reusables** to their
  latest commit. Push a new commit (or move the tag) to force a fresh resolution.
- **Build is intentionally not run here** — Vercel's per-PR preview builds the app.
