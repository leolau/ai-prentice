# Session Handoff — Hermes Agent Integration & Repowise Sync

**Date:** 2026-06-22/23 UTC
**Session:** https://app.devin.ai/sessions/d9e0df17721942a4b32c885640932ddb
**Repo:** https://github.com/leolau/ai-prentice (default branch: `develop`)

## Completed Work

### 1. Hermes Agent Subtree Integration
- **PR #12** (merged): Added `hermes-agent/` as a git subtree from [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) `main` branch
- **PR #13** (merged): CI config fixes to exclude `hermes-agent/` from guards that produce false positives:
  - `.pre-commit-config.yaml`: `detect-private-key` hook excludes `^hermes-agent/`
  - `scripts/check-no-conflict-markers.mjs`: skips `hermes-agent/` prefix
  - `scripts/github/dependency-guard.mjs`: skips `hermes-agent/` dependency files
  - `scripts/check-duplicates.mjs`: skips `hermes-agent/` in duplicate scan
  - `scripts/report-test-temp-creations.mjs`: excludes `hermes-agent` from git diff
  - `test/scripts/test-projects.test.ts`: ignores `hermes-agent/**` in shard discovery

### 2. Repowise Multi-Language Indexing
- Built full Repowise index locally (24,790 files, 213,656 symbols, 769MB DB)
- Language breakdown: TypeScript 70%, Python 10%, Markdown 10%, Swift 3%, +12 more
- Transferred index DB to ECS instance at `i-j6camnt3ocwlmzajthil` (8.217.86.90, cn-hongkong)
- Verified hermes-agent Python code is queryable via Repowise MCP (`get_context`, `get_overview`)
- Repowise service running on ECS: port 7337 (MCP/API), port 3000 (dashboard UI)

### 3. Deploy Workflow Update
- **PR #14** (merged): Updated `.github/workflows/deploy-repowise.yml`:
  - Switched from `main` to `develop` branch for git pull/clone
  - Added service stop before reindex to free memory on 3.5GB instance
  - Replaced removed `--non-interactive` with `--mode fast --skip-tests --skip-infra -y`
  - Increased ECS RunCommand timeout 600s -> 1800s

## Infrastructure State

### ECS Instance (cn-hongkong)
- **Instance ID:** `i-j6camnt3ocwlmzajthil` (IP: 8.217.86.90)
- **RAM:** 3.5GB — too small for full `repowise init` on 25K files (OOM-killed repeatedly)
- **Workaround:** Index was built on a 32GB VM and transferred via HTTP tunnel
- **Repowise DB:** `/opt/repowise/repos/ai-prentice/.repowise/wiki.db` (769MB, current as of commit 3f96cd05abd0)
- **Repowise service:** systemd `repowise.service`, auto-restarts

### Future Re-indexing
- The `deploy-repowise.yml` workflow runs weekly (Sundays 04:00 UTC) or on manual dispatch
- **Known issue:** The ECS instance will likely OOM during `repowise init` on the full repo. Options:
  1. Upgrade ECS instance to 8GB+ RAM
  2. Build index on a larger machine and transfer (as done in this session)
  3. Reduce repo scope before indexing

### Pulling Upstream Hermes Updates
```bash
git subtree pull --prefix=hermes-agent https://github.com/NousResearch/hermes-agent.git main --squash
```
Note: Subtree merges don't work well with rebase. Use `git merge origin/develop --no-edit` for integration.

## Pre-existing CI Failures (on every PR)
These are NOT caused by any changes and exist on the base branch:
1. **`label`** + **`auto-response`**: Missing GitHub App private key secrets (app IDs 2729701, 2971289)
2. **`check-additional-boundaries-bcd`**: Extension test files import core internals instead of using plugin-sdk subpaths

## Required GitHub Secrets
- `ALIBABA_CLOUD_ACCESS_KEY_ID` / `ALIBABA_CLOUD_ACCESS_KEY_SECRET` — Alibaba Cloud CLI auth
- `ECS_INSTANCE_ID` — Target ECS instance for deployments
- `OPENCLAW_GATEWAY_TOKEN` — Gateway auth token
- `DEEPSEEK_API_KEY` — DeepSeek LLM provider key

## Suggested Next Steps
1. **Docker Compose integration** — Run hermes-agent Python service alongside OpenClaw TypeScript gateway with HTTP communication between them
2. **Define HTTP API contract** — Decide which requests route to hermes-agent vs OpenClaw
3. **ECS instance upgrade** — Consider upgrading to 8GB+ RAM to support automated reindexing
4. **Hermes agent Dockerfile** — Create a Dockerfile for the Python service under `hermes-agent/`
