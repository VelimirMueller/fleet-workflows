# fleet-workflows

Reusable GitHub Actions stage workflows for the
[fleet standard](https://github.com/VelimirMueller/fleet-standard) (private). Every repo calls
these files instead of copying steps (CI-STAGE-04). The caller templates and the rollout guide
are in fleet-standard: `templates/<tier>/<variant>/` and `docs/rollout.md`.

- Pin a tag: `uses: VelimirMueller/fleet-workflows/.github/workflows/<file>@v1`.
- This repo is public, so public and private repos can both call it.
- Every action inside is pinned by SHA. The workflows reach their own composite actions with
  `uses: $/.github/actions/<name>`, so an action always runs at the same tag as the workflow.
- `v1` is a moving major tag. A breaking input change needs `v2`.

## Workflows

| File | Stage | Caller job | Caller permissions |
|---|---|---|---|
| `_ci-lint.yml` | `lint` | `ci.yml` `lint` | `contents: read` |
| `_ci-unit.yml` | `unit` | `ci.yml` `unit` | `contents: read` |
| `_ci-e2e.yml` | `e2e` | `ci.yml` `e2e`, `promote.yml` `e2e` | `contents: read` |
| `_ci-build.yml` | `build`, verify only | `ci.yml` `build` | `contents: read` |
| `_build-publish.yml` | `build`, publish | `deploy.yml` `build` | `contents: read`, `packages: write`, `checks: read`, `pull-requests: read` |
| `_deploy-run.yml` | deploy, optional fast-forward | `deploy.yml` `deploy-dev`, `promote.yml` `promote-stg`/`promote-prd` | `contents: read`, `id-token: write`, `actions: read` |
| `_promote-preflight.yml` | promotion preflight | `promote.yml` `preflight` | `contents: read`, `actions: read` |
| `_tf-check.yml` | Terraform static checks | a repo whose lint is Terraform only | `contents: read` |
| `_tf-plan.yml` | Terraform plan | `infra.yml` `plan` | `contents: read`, `id-token: write` |
| `_tf-apply.yml` | Terraform apply of a saved plan | `infra.yml` `apply`, `tf-apply-prd.yml` `apply` | `contents: read`, `id-token: write`, `actions: read` |

A caller job must grant every permission the called job asks for. If it does not, GitHub fails
the whole run at startup with no jobs and no log.

## Inputs

The four CI stages share these inputs:

| Input | Type | Default | Meaning |
|---|---|---|---|
| `stack` | string | required | `node` (pnpm), `bun`, `go`, `python` (uv), `rust`, `generic` (needs `command`). `_ci-lint.yml` also takes `docs` (markdownlint + offline link check) and `none` (only workflow and Terraform checks). |
| `working-directory` | string | `.` | Directory that holds the project manifest |
| `tool-version` | string | `''` | Empty reads `.nvmrc`/`.node-version`, `.bun-version`, `go.mod`, `.python-version`, `rust-toolchain.toml`; otherwise node 24, bun latest, rust stable |
| `install-command` | string | `''` | Replaces the default install (`pnpm install --frozen-lockfile`, `bun install --frozen-lockfile`, `go mod download`, `uv sync --frozen`, `cargo fetch --locked`) |
| `command` | string | `''` | Replaces the default stage command |

Stage-specific inputs:

- `_ci-lint.yml`: `workflow-lint` (true: actionlint + zizmor; zizmor accepts tag pins only for `VelimirMueller/*`), `terraform` (false), `terraform-directory` (`terraform`).
- `_ci-unit.yml`: `coverage-floor` (80; 0 turns the CI check off). The check reads `coverage/lcov.info` (Go: `coverage/cover.out`).
- `_ci-e2e.yml`: `environment` (`pr`), `ref` (''). Node and bun default to `playwright test --project=chromium`. The command must write JUnit XML to `e2e-junit.xml`, because a run where no test ran fails.
- `_ci-build.yml`: `pack` (false; true for tier `library`), `image` (false), `dockerfile`, `context`.
- `_build-publish.yml`: `image`, `dockerfile`, `context`, `artifact-path`. Publishes only from a dispatch on `develop`, and only a SHA whose PR `conclusion` check passed. Image tag `ghcr.io/<owner>/<repo>:<sha>`, artifact `build-dev-<sha>`.
- `_deploy-run.yml`: `environment`, `sha`, `promote-branch` (`stage` or `main`), `app-client-id`, `github-environment`, `stack`, `artifact`, `deploy-command`, `health-url`, `smoke-command`. Secrets: `PROMOTE_APP_PRIVATE_KEY` and `DEPLOY_ENV` (secret `KEY=value` lines; every value is masked).
- `_promote-preflight.yml`: `target` (`stg`, `prd`, `stg-hotfix`), `two-env`, `dev-workflow` (`deploy.yml`), `stg-workflow` (`promote.yml`). Outputs: `sha`, `branch`, `push`.
- `_tf-plan.yml` / `_tf-apply.yml`: `environment`, `working-directory`, `github-environment`, `env` (non-secret `KEY=value` lines, for example `ARM_USE_OIDC=true`), `aws-role-arn`, `aws-region`, `app-client-id`. Secrets: `TF_ENV`, `APP_PRIVATE_KEY`. The plan outputs `exitcode` (0 or 2), `destroys` and `artifact`. Apply also takes `plan-run-id` and `allow-destroy` (false).

## Release

1. Merge the PR into `main`. `ci.yml` lints every workflow on the PR.
2. Tag the merge commit `v1.<minor>.<patch>`, then move `v1` to the same commit.
3. A breaking change to an input, a secret or a permission needs `v2`.
