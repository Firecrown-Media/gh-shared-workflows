# shared-workflows

Centralized GitHub Actions reusable workflows for Firecrown WordPress VIP site repositories.

## Usage

All site repos call these workflows from their `.github/workflows/` directory using the `uses:` key. Do not copy workflow logic into site repos — update it here and all sites pick up the change.

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `vip-deploy.yml` | `workflow_call` | Build assets and push to VIP built branch |
| `phpcs.yml` | `workflow_call` | PHP_CodeSniffer with WordPress-VIP-Go rules |
| `security-scan.yml` | `workflow_call` | PHP vuln, dependency audit, secret scanning |
| `ai-issue-agent.yml` | `workflow_call` | Claude Code CLI AI issue agent |

## Feature Flags

Security and analysis jobs check repository variables on the caller repo. Set any variable to `'false'` in the site repo's Settings → Variables to disable that job:

| Variable | Job |
|---|---|
| `ENABLE_PHPCS` | PHP_CodeSniffer |
| `ENABLE_PHP_SECURITY` | PHP vulnerability scan |
| `ENABLE_DEPENDENCY_AUDIT` | npm/composer audit |
| `ENABLE_SECRET_SCAN` | TruffleHog secret scan |
