# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Centralized GitHub Actions reusable workflows for all Firecrown WordPress VIP site repositories. Site repos call these workflows via `uses:` — logic lives here, not in site repos, so a single change here propagates everywhere.

## Workflows

| File | Trigger | Purpose |
|---|---|---|
| `vip-sync.yml` | `workflow_call` | Build deploy artifacts, then force-push a branch to `wpcomvip/<repo>` via SSH deploy key |
| `vip-reverse-sync.yml` | `workflow_call` | Detect VIP hotfixes and open a reverse-sync PR back |
| `phpcs.yml` | `workflow_call` | PHP_CodeSniffer with `WordPress-VIP-Go` ruleset |
| `security-scan.yml` | `workflow_call` | PHP vuln scan, npm audit, TruffleHog secret scan |
| `ai-issue-agent.yml` | `workflow_call` | Claude Code CLI AI issue agent |

## Key Design Points

**VIP sync pattern**: `vip-sync.yml` builds deploy artifacts, commits them, and then force-pushes (`git push --force`) the result from the caller repo to the target `wpcomvip` repo using an SSH deploy key. VIP then auto-deploys from the pushed branch. It is not a plain push of source:

- **Composer:** `composer install --no-dev --no-scripts --optimize-autoloader` in every directory with a `composer.lock` found at `-maxdepth 3` under `plugins/`, `themes/`, `mu-plugins/`, `client-mu-plugins/` or `private/`, skipping paths containing `/vendor/`. A `composer.json` without a lock is only reported (`::notice::`); its shipped `vendor/` is used as-is.
- **npm:** `npm ci && npm run build` in every directory with a `package-lock.json` at `-maxdepth 3` under `plugins/`, `themes/`, `mu-plugins/` or `client-mu-plugins/`, skipping `/node_modules/`, whose `package.json` defines `scripts.build`.
- A failed install or build fails the deploy. The resulting `vendor/`, `build/` and `dist/` directories are force-added (source `.gitignore` notwithstanding) into one commit by `github-actions[bot]`, `ci: build deploy artifacts (vendor, build)`, on top of the source commit. That message is part of `vip-reverse-sync.yml`'s filter contract; change both together.

**Divergence guard** (`vip-sync.yml`): Before **every** push, `production` included, checks whether the VIP branch has commits not in the source branch (prevents overwriting VIP hotfixes), ignoring the workflow's own `ci: build deploy artifacts` commits by `github-actions[bot]`. Any other VIP-only commit fails the sync; merge the reverse-sync PR first. It is on by default for every `target_branch` and has applied to production since `4caf9a2` (2026-05-14); a caller can opt out only by passing `check_vip_divergence: false` (not recommended).

**Feature flags**: All jobs have an `if: vars.ENABLE_* != 'false'` guard. Flags are set as repository variables on each caller repo. Variables not set default to `'true'` (enabled). `ENABLE_DEPENDENCY_AUDIT` and `ENABLE_SECRET_SCAN` are set to `'false'` at scaffold time.

**`repository_owner` guard**: Caller workflows in site repos include `if: github.repository_owner == 'Firecrown-Media'` on every job. This prevents the workflows from firing on the `wpcomvip` side when VIP sync copies `.github/workflows/` into the VIP repo.

## Editing Workflows

Changes to any workflow here automatically apply to all site repos on their next run — no per-repo updates needed. Test changes against the astronomy repo before merging.

The `README.md` in this repo is the canonical reference for inputs, secrets, and feature flags. Keep it in sync when adding inputs or changing behaviour.
