# GitHub Environments deploy-gate test

Minimal, reproducible test that a GitHub **Environment deployment branch policy**
actually blocks "deploy to prod from any branch" at the platform level — including
adversarial bypasses — before we apply the pattern to
`fluentlabs-xyz/rollup-bridge-services`.

The real `deploy-prod.yml` is `workflow_dispatch`-only and its deploy job calls a
reusable workflow via `uses:`, so `environment:` cannot be attached to the deploy job
itself. This repo reproduces that shape with echo-only jobs.

> Note: this repo's default branch is `main` (org policy prevents changing it), so
> here `main` plays the role of the protected prod branch. In the real repo the
> default is `devel` and prod deploys only from `main`. This difference is cosmetic —
> the environment branch policy is evaluated against the run's ref regardless of which
> branch is default.

## Files

| File | Role |
|---|---|
| `.github/workflows/deploy-test.yml` | `gate` job (`environment: production`) + `deploy` job (`needs: gate`, no `environment:`) — T1–T4 |
| `.github/workflows/_deploy-reusable.yml` | reusable workflow whose job declares `environment: production` — stand-in for `rancher-config/deploy.yml` — T6 |
| `.github/workflows/t6-reusable.yml` | caller: job with **no** `environment:`, `uses:` the reusable + `secrets: inherit` — T6 |

## One-time setup (via `gh`)

```bash
REPO=fluentlabs-xyz/tmp-workflow-test

# environment with a custom deployment branch policy allowing only `main`
gh api -X PUT repos/$REPO/environments/production --input - <<'JSON'
{"deployment_branch_policy":{"protected_branches":false,"custom_branch_policies":true}}
JSON
gh api -X POST repos/$REPO/environments/production/deployment-branch-policies -f name=main

# secrets: PROD_MARKER at the environment, REPO_MARKER at the repo
gh secret set PROD_MARKER --repo $REPO --env production --body env-secret-value
gh secret set REPO_MARKER --repo $REPO --body repo-secret-value
```

## Branches

| Branch | Meaning |
|---|---|
| `main` | allowed prod branch (also default here); carries the workflows |
| `devel` | a not-allowed branch, workflow identical to `main` |
| `attacker` | from `main`, **removes** `environment: production` from the gate job |
| `attacker-t4` | from `main`, keeps `environment:` but renames the gate job |

`main` is landed via PR (org rule requires it). The other branches are created from
`main` and pushed directly.

## Test matrix

Dispatch each, wait for completion, record run conclusion + per-job status + exact
error text.

| Test | Command | Expected |
|---|---|---|
| T1 | `gh workflow run deploy-test.yml --ref main` | gate passes, deploy runs, `PROD_MARKER` non-empty in gate |
| T2 | `gh workflow run deploy-test.yml --ref devel` | gate **rejected** by branch policy; deploy does not run |
| T3 | `gh workflow run deploy-test.yml --ref attacker` | run completes (no env reference → policy N/A); deploy sees `PROD_MARKER` **EMPTY** but `REPO_MARKER` **non-empty** — the leak |
| T4 | `gh workflow run deploy-test.yml --ref attacker-t4` | still blocked — policy binds to the environment reference, not the job name |
| T6a | `gh workflow run t6-reusable.yml --ref main` | reusable job (env inside it) runs; `PROD_MARKER` reachable |
| T6b | `gh workflow run t6-reusable.yml --ref devel` | blocked — policy evaluated against the caller's ref |

(Optional T5: add a required reviewer on `production`, dispatch from `main`, confirm
the run pauses for approval, then remove the reviewer.)

## Run & read results

```bash
gh run list --repo fluentlabs-xyz/tmp-workflow-test --limit 15
gh run view <run-id> --repo fluentlabs-xyz/tmp-workflow-test --log
```

## What we are proving

- **a)** Branch policy blocks at the GitHub level, before the branch's YAML is trusted
  — unlike an in-file `if: github.ref` guard, which the branch can simply delete (T3).
- **b)** Remaining bypass surface: any job that does **not** reference the environment
  runs freely from any branch and can read **repo-level** secrets (T3). So every secret
  the deploy needs must live in the **environment**, not at repo level.
- **c)** Migration for `rollup-bridge-services` — see the PR description / follow-up.
tttt
