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

- `phpcs.yml` — PHP CodeSniffer linting
- `security-scan.yml` — security scanning
- `ai-issue-agent.yml` — AI-assisted issue triage
- `vip-sync.yml` — WordPress VIP sync
- `vip-reverse-sync.yml` — WordPress VIP reverse sync

---

### `do-deploy.yml` — Deploy to the DigitalOcean WordPress pool

Rsyncs the repository to a new release directory on a pool droplet, links the shared uploads and `wp-config.php`, flips the `current` symlink, flushes the object cache and prunes old releases. Expects the site layout created by the pool's Ansible playbook (`/var/www/<site>/{releases,current,shared}`).

**Inputs:** `site` (slug under /var/www), `environment` (`stage` or `production`), `host` (droplet), `url` (optional, smoke-tested after activation), `keep_releases` (default 5).

**Secrets:** `DEPLOY_SSH_KEY` (private key for the droplet's `deploy` user).

**Example caller:**

```yaml
on:
  push:
    branches: [stage, main]
jobs:
  deploy:
    uses: Firecrown-Media/gh-shared-workflows/.github/workflows/do-deploy.yml@main
    with:
      site: media
      environment: ${{ github.ref_name == 'main' && 'production' || 'stage' }}
      host: ${{ github.ref_name == 'main' && vars.DO_PROD_HOST || vars.DO_STAGE_HOST }}
      url: https://media.freightwaves.com
    secrets:
      DEPLOY_SSH_KEY: ${{ secrets.DO_DEPLOY_SSH_KEY }}
```
