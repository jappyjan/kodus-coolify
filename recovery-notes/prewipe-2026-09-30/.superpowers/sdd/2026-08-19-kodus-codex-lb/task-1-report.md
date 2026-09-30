# Task 1 Report: Private Codex LB Compose Service

## Status

Completed and committed.

## Files Changed

- `docker-compose.yml`
  - Added the private `codex-lb` service using `ghcr.io/soju06/codex-lb:1.23.0`.
  - Exposed only ports `2455` and `1455` to the Compose network.
  - Added persistent `codex-lb-data` storage at `/var/lib/codex-lb`.
  - Updated `api` and `worker` default model to `gpt-5.6-sol`.
  - Updated `api` and `worker` default OpenAI base URL to `http://codex-lb:2455/v1`.
  - Preserved both `API_OPEN_AI_API_KEY` references unchanged.

- `.superpowers/sdd/2026-08-19-kodus-codex-lb/task-1-report.md`
  - This uncommitted execution report. It is intentionally excluded from the Compose-only commit.

## Commands And Output Summary

1. `sh /tmp/validate-kodus-codex-lb.sh` before the change
   - Exited non-zero as expected because the Codex LB image and requested Kodus defaults were absent.

2. `sh /tmp/validate-kodus-codex-lb.sh` after the change
   - Exited 0. `docker compose -f docker-compose.yml config --no-interpolate` rendered successfully and all four required `rg` assertions matched.

3. `git diff --check && git diff -- docker-compose.yml`
   - Exited 0 with no whitespace errors.
   - Diff contained only the requested `codex-lb` service, `codex-lb-data` volume, and API/worker model and base-URL default changes.

4. `git add docker-compose.yml && git diff --cached --check && git diff --cached -- docker-compose.yml && git status --short && git commit -m "feat: add private Codex LB service for Kodus"`
   - Staged diff check exited 0.
   - Commit created with one changed file: 14 insertions and 4 deletions.

## Commit

`abf9d7e feat: add private Codex LB service for Kodus`

## Concerns

- No deployment or Coolify settings were changed, as required.
- Coolify must provide the dedicated Codex LB proxy key through the existing `API_OPEN_AI_API_KEY` variable after OAuth setup; this task intentionally does not configure that value.
- The pre-existing untracked `docs/superpowers/plans/` directory was left unchanged. This report is also intentionally uncommitted because the requested commit must contain only `docker-compose.yml`.
