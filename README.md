# fleet-workflows

Reusable GitHub Actions stage workflows for the
[fleet standard](https://github.com/VelimirMueller/fleet-standard) (private). Every repo calls
these files instead of copying steps (CI-STAGE-04). The caller templates and the rollout guide
are in fleet-standard: `templates/<tier>/<variant>/` and `docs/rollout.md`.

- Pin a release tag: `uses: VelimirMueller/fleet-workflows/.github/workflows/<file>@v0.0.3`.
- This repo is public, so public and private repos can both call it.
- Every action inside is pinned by SHA. The workflows reach their own composite actions with
  `uses: $/.github/actions/<name>` (GitHub's self-repository syntax), so an action always runs at
  the same tag as the workflow. A `./` path would look in the caller's checkout instead.

## Workflows

| File | Stage | Caller job | Caller permissions |
| --- | --- | --- | --- |
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
| --- | --- | --- | --- |
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
- `_build-publish.yml`: `image`, `dockerfile`, `context`, `artifact-path`, `required-stages` (`lint unit build`). Publishes only from a dispatch on `develop`, and only a SHA (or a merged PR head with the same tree) whose newest GitHub Actions run of `conclusion` and of each required stage is green. Image tag `ghcr.io/<owner>/<repo>:<sha>`, artifact `build-dev-<sha>`.
- `_deploy-run.yml`: `environment`, `sha`, `promote-branch` (`stage` or `main`), `app-client-id`, `github-environment`, `stack`, `artifact`, `deploy-command`, `health-url`, `smoke-command`. Secrets: `PROMOTE_APP_PRIVATE_KEY` and `DEPLOY_ENV` (secret `KEY=value` lines; every value is masked).
- `_promote-preflight.yml`: `target` (`stg`, `prd`, `stg-hotfix`), `two-env`, `dev-workflow` (`deploy.yml`), `stg-workflow` (`promote.yml`). Outputs: `sha`, `branch`, `push`.
- `_tf-plan.yml` / `_tf-apply.yml`: `environment`, `working-directory`, `github-environment`, `env` (non-secret `KEY=value` lines, for example `ARM_USE_OIDC=true`), `aws-role-arn`, `aws-region`, `app-client-id`. Secrets: `TF_ENV`, `APP_PRIVATE_KEY`. The plan outputs `exitcode` (0 or 2), `destroys`, `artifact` and `plan-sha256`. Apply also takes `plan-run-id`, `plan-workflow` (`infra.yml`), `plan-sha256` and `allow-destroy` (false).

## Conformance check action

`.github/actions/fleet-check` runs `fleet check` in a repo's own `fleet-check.yml` (job `fleet-check`,
the required check on `develop`, CI-CHECK-06). fleet-standard is private, so a public repo cannot
check it out. This action ships the checker as one bundled file instead.

- `dist/`, `rules/` and `schema/` are generated: `pnpm bundle:action <this repo>/.github/actions/fleet-check`
  in fleet-standard. The first line of `dist/index.min.js` names the source commit. Do not edit them here.
- Runtime `node24`: no toolchain install, no shell step.
- On `pull_request` it diffs the checked merge commit against its base (`git diff HEAD^1 HEAD`; the
  caller checks out with `fetch-depth: 2`; no API, no token). It passes at once only when every changed
  file is a document or image outside `.github/`, `.githooks/`, `.gitlab/`, the dirs the rules name and
  every dir with Terraform. Any other file, or a diff it cannot compute, runs the check (fail closed).
- The report is printed inside `::stop-commands::`, so repo-controlled names cannot run workflow commands.
- Exit 0 conforms, 1 drift, 2 config error. Each finding of a failing rule is an `::error` annotation; the report
  goes to the job summary.
- Inputs: `strict` (`false`; `true` also fails on SHOULD findings), `path` (`.`). Caller job
  permissions: `contents: read`.
- The caller job and its rules (no `paths` filter, no draft guard, `pull_request` into `develop` only)
  are in fleet-standard `docs/rollout.md`.

## Saved Terraform plans

A plan can hold secret values, and the artifacts, logs and summaries of a public repo are public.

- `_tf-plan.yml` keeps the plan text out of the log. The summary lists only resource addresses and actions.
- The plan is sealed with age (`.github/actions/age`, v1.3.2, checksum pinned) for the required secret `PLAN_AGE_RECIPIENT`. The artifact holds only `tfplan.age`, `tfplan.sha256` and `plan-sha.txt`. With no recipient, the job fails and uploads nothing.
- `_tf-apply.yml` opens the plan with the required secret `PLAN_AGE_IDENTITY`. With `plan-run-id`, it first checks through the API that the run is a successful `workflow_dispatch` of `plan-workflow` in this repo, on this branch, for this SHA.
- It downloads the exact artifact `tfplan-<env>-<sha>`, checks the sha256 (and `plan-sha256` when given), and takes the destroy guard from `terraform show -json`.

## Release

1. Merge the PR into `main`. `ci.yml` lints every workflow on the PR.
2. Tag the merge commit `v0.0.<n>` and create a GitHub release. Bump every caller pin together
   (fleet-standard `templates/`).
3. A breaking change to an input, a secret or a permission needs `v2`.
