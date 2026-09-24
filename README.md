# gh-shared-workflows

Reusable GitHub Actions workflows shared across Firecrown Terraform repositories.

## Available Workflows

### `terraform-ci.yml` — Terraform CI checks

Runs `fmt`, `validate`, `tflint`, Trivy security scan, and terraform-docs check on a Terraform codebase.

**Inputs:**

| Input | Type | Default | Description |
|---|---|---|---|
| `terraform_dir` | string | `.` | Directory containing Terraform files |
| `docs_output_file` | string | `README.md` | File verified by `git diff --exit-code` after terraform-docs |
| `skip_terraform_validate` | boolean | `false` | Skip `terraform init` + `validate` (use when backend requires live credentials) |
| `trivy_config` | string | `''` | Path to trivy config file (empty = auto-detect `trivy.yaml`) |
| `trivyignore_path` | string | `''` | Path to `.trivyignore` file (empty = auto-detect at scan root) |

**Example caller:**

```yaml
jobs:
  terraform-ci:
    if: vars.TERRAFORM_CI_ENABLED != 'false'
    uses: Firecrown-Media/gh-shared-workflows/.github/workflows/terraform-ci.yml@main
    with:
      terraform_dir: terraform
      docs_output_file: docs/terraform.md
      skip_terraform_validate: true
      trivy_config: terraform/trivy.yaml
      trivyignore_path: terraform/.trivyignore
```

**Disabling CI per-repo:** Set the `TERRAFORM_CI_ENABLED` repository variable to `false` in GitHub Settings → Secrets and variables → Actions → Variables. Delete the variable or set it to any other value to re-enable.

---

### `infracost.yml` — Infracost cost estimation

Posts a cost estimate comment on every PR using Infracost in HCL parse mode (no AWS credentials required). Uses `behavior: update` to edit the existing comment on each push rather than creating new comments.

**Inputs:**

| Input | Type | Default | Description |
|---|---|---|---|
| `terraform_dir` | string | `terraform` | Directory containing Terraform files |
| `tfvars_file` | string | `prod.tfvars` | `.tfvars` file for cost estimates (relative to `terraform_dir`) |

**Secrets:** `INFRACOST_API_KEY` — get a free key at [cloud.infracost.io](https://cloud.infracost.io). Store as a GitHub repository Secret (Settings → Secrets and variables → Actions → New repository secret). **Never store as a plain variable — secrets are encrypted and masked in logs.**

**HCL mode limitation:** Resources inside external Terraform modules (`terraform-aws-ecs-fargate`, `terraform-aws-cicd`) may show as "not supported" in the cost breakdown. Directly-managed resources (Aurora, EFS, ECS tasks, ElastiCache, ALB) are priced accurately.

**Example caller:**

```yaml
name: Infracost

on:
  pull_request:
    branches: [main]

permissions:
  pull-requests: write
  contents: read

jobs:
  infracost:
    if: vars.TERRAFORM_CI_ENABLED != 'false'
    uses: Firecrown-Media/gh-shared-workflows/.github/workflows/infracost.yml@main
    with:
      terraform_dir: terraform
      tfvars_file: prod.tfvars
    secrets: inherit
```

**Rotating the API key:** Go to cloud.infracost.io → account settings → API keys → generate a new key, then update the `INFRACOST_API_KEY` secret in GitHub.

---

## Existing Workflows (WordPress / VIP)

The following workflows predate the Terraform migration and are used by WordPress repos (e.g., `astronomy`):

- `phpcs.yml` — PHP CodeSniffer linting (see below; delta mode skips vendored third-party plugin directories)
- `security-scan.yml` — security scanning
- `ai-issue-agent.yml` — AI-assisted issue triage
- `vip-sync.yml` — WordPress VIP sync
- `vip-reverse-sync.yml` — WordPress VIP reverse sync

### `phpcs.yml` — PHP_CodeSniffer

`delta` mode (the default) lints the `.php` and `.js` files a pull request or push changes;
`full` mode scans the paths in the repo's own ruleset and opens an issue with the results. A repo
with no ruleset file falls back to `--standard=WordPress-VIP-Go`. The job is skipped when the
`ENABLE_PHPCS` repository variable is `false`.

**Inputs:**

| Input | Type | Default | Description |
|---|---|---|---|
| `php_version` | string | `8.2` | PHP version to use |
| `scan_mode` | string | `delta` | `delta` scans changed files only; `full` scans the repo per its ruleset and opens an issue |
| `lint_vendored_plugins` | boolean | `false` | Lint vendored plugin directories anyway, instead of skipping them (see below) |

**Secrets:** callers forward `secrets: inherit` so the workflow can mint a GitHub App token
(`COMPOSER_APP_ID` / `COMPOSER_APP_PRIVATE_KEY`) for private Composer packages. Without them those
steps skip.

**Vendored-plugin skip (`PHPCS_VENDORED_SKIP`, wpvip-fleet ADR-024).** Installing a third-party
plugin into `plugins/<dir>/` would otherwise lint every file of someone else's code and turn the
PR red. In `delta` mode, the files under `plugins/<dir>/` are dropped from the scan when a commit
in the scan range carries the trailer

```
Plugin-Vendored: plugins/<dir>/
```

`garage/scripts/plugin-rollout.py` (in `wpvip-fleet`) writes it on the commits it builds. The
trailer is honoured only when:

- it is on a **non-merge** commit in the range (`origin/<base>..HEAD` on a pull request,
  `<before>..<sha>` on a push);
- it names **exactly one** directory level, matching `^plugins/[A-Za-z0-9._-]+/$`;
- **every** path that commit changes is inside that directory;
- the directory is **not first-party**: `plugins/fc-*`, `plugins/fw-*`, `plugins/firecrown-*` and
  `plugins/kserv*` are always linted, with a `::warning::`.

Once a directory is accepted, every changed file under it is skipped for that run, including
files that other commits in the same range change there. A rejected trailer is reported as a
`::warning::` with the reason, and its files are linted. Skipped files are counted in a
`::notice::`. On a push whose `before` is all zeros (a new branch), no trailer is honoured. Set
`lint_vendored_plugins: true` in a caller to lint vendored directories anyway.
