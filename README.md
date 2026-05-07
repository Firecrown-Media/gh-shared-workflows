# shared-workflows

Centralized GitHub Actions reusable workflows for Firecrown WordPress VIP site repositories.

## Usage

All site repos call these workflows from their `.github/workflows/` directory using the `uses:` key. Do not copy workflow logic into site repos — update it here and all sites pick up the change automatically.

## Workflows

| Workflow | Purpose |
|---|---|
| `vip-sync.yml` | Push a Firecrown branch to a target branch on the VIP repo |
| `vip-reverse-sync.yml` | Detect VIP hotfixes and open a reverse-sync PR back to Firecrown |
| `phpcs.yml` | PHP_CodeSniffer with WordPress-VIP-Go rules |
| `security-scan.yml` | PHP vuln scan, dependency audit, secret scanning |
| `ai-issue-agent.yml` | Claude Code CLI AI issue agent |

---

## `vip-sync.yml`

Pushes a branch from the calling repo to a target branch on a `wpcomvip` repo using a deploy SSH key. Intended for `stage → VIP stage` and `main → VIP production` flows.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `target_repo` | string | yes | — | VIP repo, e.g. `wpcomvip/astronomy-com` |
| `target_branch` | string | yes | — | Branch to push to, e.g. `stage` or `production` |
| `php_version` | string | no | `8.2` | PHP version for optional pre-sync steps |
| `node_version` | string | no | `20` | Node version for optional pre-sync steps |
| `check_vip_divergence` | boolean | no | `true` | Whether to fail if VIP has commits not in the source branch (see below) |

### Secrets

| Secret | Description |
|---|---|
| `deploy_token` | SSH private key with write access to the VIP repo |

### Divergence guard

Before pushing, the workflow checks whether the target VIP branch has commits that are not present in the source branch. If it does, the push fails with an error and lists the missing commits. This prevents silently overwriting VIP hotfixes.

**The guard is skipped automatically when `target_branch` is `production`.** The rationale: VIP hotfixes on `production` are surfaced by the daily reverse sync (`vip-reverse-sync.yml`) and flow back through a PR to `main`. By the time `main` is ready to deploy to production the hotfix is already included, so the divergence check would block unnecessarily.

The guard can also be disabled explicitly by passing `check_vip_divergence: false` for non-production branches where the check is not wanted.

**Summary of divergence check behaviour:**

| `target_branch` | `check_vip_divergence` | Check runs? |
|---|---|---|
| `stage` | `true` (default) | Yes |
| `stage` | `false` | No |
| `production` | any | No (auto-skip) |

---

## `vip-reverse-sync.yml`

Runs on a schedule (or `workflow_dispatch`) from a site repo. Compares the VIP branch to the Firecrown branch and, if VIP is ahead, opens a PR back to Firecrown so the hotfix can be reviewed and merged.

- If no open reverse-sync PR exists: creates a new branch (`vip-hotfix/YYYY-MM-DD`) and opens a PR labelled `vip-hotfix`.
- If an open PR already exists: force-pushes the branch to include any new VIP commits and posts a comment on the existing PR.

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `source_repo` | string | yes | — | VIP repo to check, e.g. `wpcomvip/astronomy-com` |
| `source_branch` | string | yes | — | VIP branch to check, e.g. `production` |
| `target_branch` | string | no | `main` | Firecrown branch to PR into |

### Secrets

| Secret | Description |
|---|---|
| `deploy_token` | SSH private key with read access to the VIP repo |
| `gh_token` | GitHub token for creating PRs and posting comments |

### Feature flag

Set `ENABLE_VIP_REVERSE_SYNC=false` as a repository variable on the calling repo to disable the scheduled check without removing the workflow file.

---

## Feature flags (all workflows)

Set any variable to `'false'` in the site repo's **Settings → Variables** to disable that job:

| Variable | Workflow | Effect when `'false'` |
|---|---|---|
| `ENABLE_VIP_SYNC_STAGE` | caller `deploy.yml` | Skip stage → VIP sync |
| `ENABLE_VIP_SYNC_PRODUCTION` | caller `deploy.yml` | Skip main → VIP production sync |
| `REQUIRE_PRODUCTION_APPROVAL` | caller `deploy.yml` | Skip the production approval gate |
| `ENABLE_VIP_REVERSE_SYNC` | caller `vip-reverse-sync.yml` | Disable the daily hotfix check |
| `ENABLE_PHPCS` | `phpcs.yml` | PHP_CodeSniffer |
| `ENABLE_PHP_SECURITY` | `security-scan.yml` | PHP vulnerability scan |
| `ENABLE_DEPENDENCY_AUDIT` | `security-scan.yml` | npm/composer audit |
| `ENABLE_SECRET_SCAN` | `security-scan.yml` | TruffleHog secret scan |
