# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Centralized GitHub Actions reusable workflows for all Firecrown WordPress VIP site repositories. Site repos call these workflows via `uses:` — logic lives here, not in site repos, so a single change here propagates everywhere.

## Workflows

| File | Trigger | Purpose |
|---|---|---|
| `vip-sync.yml` | `workflow_call` | Push a branch to `wpcomvip/<repo>` via SSH deploy key |
| `vip-reverse-sync.yml` | `workflow_call` | Detect VIP hotfixes and open a reverse-sync PR back |
| `phpcs.yml` | `workflow_call` | PHP_CodeSniffer with `WordPress-VIP-Go` ruleset |
| `security-scan.yml` | `workflow_call` | PHP vuln scan, npm audit, TruffleHog secret scan |
| `ai-issue-agent.yml` | `workflow_call` | Claude Code CLI AI issue agent |

## Key Design Points

**VIP sync pattern**: `vip-sync.yml` does a plain `git push --force` from the caller repo to the target `wpcomvip` repo using an SSH deploy key. VIP then auto-deploys from the pushed branch. No asset building — VIP expects source code.

**Divergence guard** (`vip-sync.yml`): Before pushing to stage, checks if the VIP branch has commits not in the source branch (prevents overwriting VIP hotfixes). Auto-skipped for `production` target — hotfixes on production flow back via reverse sync instead.

**Feature flags**: All jobs have an `if: vars.ENABLE_* != 'false'` guard. Flags are set as repository variables on each caller repo. Variables not set default to `'true'` (enabled). `ENABLE_DEPENDENCY_AUDIT` and `ENABLE_SECRET_SCAN` are set to `'false'` at scaffold time.

**`repository_owner` guard**: Caller workflows in site repos include `if: github.repository_owner == 'Firecrown-Media'` on every job. This prevents the workflows from firing on the `wpcomvip` side when VIP sync copies `.github/workflows/` into the VIP repo.

## Editing Workflows

Changes to any workflow here automatically apply to all site repos on their next run — no per-repo updates needed. Test changes against the astronomy repo before merging.

The `README.md` in this repo is the canonical reference for inputs, secrets, and feature flags. Keep it in sync when adding inputs or changing behaviour.
