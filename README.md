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

- it is on a **non-merge** commit in the range (`refs/remotes/origin/<base>..HEAD` on a pull
  request, `<before>..<sha>` on a push);
- it names **exactly one** directory level, matching `^plugins/[A-Za-z0-9._-]+/$`;
- the directory is **not first-party by name**: `plugins/fc-*`, `plugins/fw-*`,
  `plugins/firecrown-*` and `plugins/kserv*` are always linted, with a `::warning::`;
- **every** path that commit changes is inside that directory;
- the directory **does not exist on the scan base** (`refs/remotes/origin/<base>`, or
  `<before>`). The skip is for **new installs only**, so a trailer can never exempt code that is
  already in the repo, whatever its name. In-house plugins such as `astronomy-core`,
  `nexus-analytics`, `bonnier-*`, `wp-omeda` or `tcc-*` carry none of the first-party prefixes,
  and this rule is what keeps them linted;
- that commit **creates** the directory, so it is absent from the commit's parent;
- at the scanned endpoint the directory is **exactly the tree that commit wrote**. If any other
  change in the range touches it, the whole directory is linted. That covers a trailer-less
  patch, a `git mv` into it, a merge that edits it, and a mode change.

Together these mean every skipped file holds exactly the bytes the trailer commit added. No
commit, other than the one carrying the trailer, can get a file skipped. A rejected trailer is
reported as a `::warning::` with the reason, and the files under its directory are linted. Skipped
files are counted in a `::notice::`. Set `lint_vendored_plugins: true` in a caller to lint vendored
directories anyway.

Known limits:

- The skip cannot tell in-house code from third-party code by content. A **new** directory whose
  own creating commit carries the trailer is skipped whatever it holds. The trailer is a
  declaration, reviewed in the PR like the rest of the commit.
- Updating a plugin that is already installed is **not** covered: the directory exists on the
  base, so the whole update is linted. Extending the skip to updates is a separate decision
  (wpvip-fleet ADR-024 covers installs only).

**The diff base.** On a pull request the base is the full remote-tracking ref,
`refs/remotes/origin/<base>`. A short `origin/<base>` would resolve a tag of that name first, and
checkout fetches every tag, so a pushed tag named `origin/develop` could replace the base.

- A delta scan whose base **cannot be resolved** fails with `::error::`. That covers a push whose
  `before` is not in the clone (the branch's history was rewritten) and a pull request whose base
  does not resolve.
- A delta scan with **no base at all** warns and exits 0 **without linting**. That happens on a
  push or `workflow_dispatch` whose `before` is empty or all zeros: a manual run, or the first
  push of a branch. The `::warning::` says no file is linted. Use `scan_mode: full` for a
  whole-repo scan.

**How files reach phpcs.** The changed `.php`/`.js` paths are written to a file and passed with
`--file-list`, one path per line, in a single phpcs run. They are never passed as arguments, so
a name with a space, an apostrophe, or the spelling of an option such as `--ignore=*.php` is
linted as the path it is. A delta of any size produces one report. Names are read with
`core.quotePath=false`, so non-ASCII names are linted too.

- A changed `.php`/`.js` file whose name git must still quote (it holds `"`, `\` or a control
  character) or that starts with a space fails the step with `::error::` naming it, even inside a
  vendored directory. phpcs cannot be handed it by name, and the alternative is a green scan that
  skipped it.
- A linted name that is not valid UTF-8 also fails the step, because phpcs cannot write it into
  its XML report.
