# Task 2 Non-Interactive Preparation Report

## Status

BLOCKED.

The local `docker-compose.yml` now has only these temporary entries in the
`codex-lb` service:

```yaml
environment:
  - SERVICE_FQDN_CODEX_LB_2455
  - SERVICE_FQDN_CODEX_LB_1455
```

They are intentionally uncommitted. No OAuth authentication, dashboard
bootstrap, or proxy-key creation was attempted.

## Public URLs

No public URLs were generated. Exact URLs: unavailable for both routes.

| Route | Expected use | Exact public URL |
| --- | --- | --- |
| `SERVICE_FQDN_CODEX_LB_2455` | codex-lb dashboard and proxy API | Not generated / unavailable |
| `SERVICE_FQDN_CODEX_LB_1455` | OAuth callback | Not generated / unavailable |

Coolify deployed the remote GitHub `main` commit, not the local working tree.
The deployed commit was `b0370033f20b90e7f4144463e45ec78ee0653866`, whose
Compose file has neither the `codex-lb` service nor the temporary route
entries. Therefore Coolify had no values from which to generate either URL.

## Commands And Output Summary

| Command | Result |
| --- | --- |
| `docker compose config --quiet` | Succeeded. Local-only warnings reported unset deployment secrets; no Compose syntax error. |
| `coolify deploy uuid <coolify-app-uuid> --force` | Queued deployment. |
| `coolify deploy get <deployment-id> --format json` | Finished successfully at `2026-08-19T12:17:07Z`, but from remote commit `b0370033...`. Deployment logs show startup of `web`, `api`, `worker`, `webhooks`, `kodus-mcp-manager`, `rabbitmq`, `db_postgres`, and `db_mongodb` only. |
| `coolify app logs <coolify-app-uuid> --lines 200` | Shows the existing web application ready on port 3000. It contains no `codex-lb` startup or port 2455/1455 listener output. |
| `git log --oneline --decorate -3` | Local `main` is two commits ahead of `origin/main`: `abf9d7e feat: add private Codex LB service for Kodus`, `b2ecf16 docs: add Kodus Codex LB deployment design`; remote is `b037003`. |
| `git diff --check` | Succeeded with no whitespace errors. |

## Deployment Status

The Coolify deployment itself finished successfully, restoring the existing
remote application version. `codex-lb` was not deployed, so its startup,
port listeners 2455 and 1455, and public routes are unverified and unavailable.

## Required User Action

Publish the existing local Task 1 commit `abf9d7e` to the Git ref that Coolify
deploys (`origin/main`), then provide or authorize a deployment mechanism that
can include the two temporary route entries without committing them. The current
Git-backed Coolify deployment clones `origin/main` and cannot see uncommitted
local Compose changes.

After a deployment actually includes `codex-lb` and both route entries, open
the generated port-2455 dashboard URL, complete the owner's interactive
ChatGPT/Codex OAuth login, confirm `gpt-5.6-sol`, and create the `kodus` proxy
key. Do not share that key in chat, Git, or a command line. Then the runtime
secret update and route removal can be completed.

## 2026-08-19 Decision Update

The dashboard and OAuth callback routes are now permanent public routes. The
`codex-lb` service retains `SERVICE_FQDN_CODEX_LB_2455` and
`SERVICE_FQDN_CODEX_LB_1455`, and adds
`CODEX_LB_DASHBOARD_AUTH_MODE=standard` to use the documented built-in dashboard
authentication.

Validation after this change:

| Command | Result |
| --- | --- |
| `docker compose config --quiet` | Succeeded. The local renderer emitted warnings only for unset deployment secrets. |
| `git diff --check` | Succeeded with no whitespace errors. |

No deployment, push, OAuth authentication, Coolify secret change, or proxy-key
creation was performed.

## 2026-08-19 Domain Assignment Update

The permanent public `codex-lb` route values are deployment-specific and must
be configured outside the repository.

`CODEX_LB_DASHBOARD_AUTH_MODE=standard` remains unchanged.

Validation after assigning the deployment-specific routes:

| Command | Result |
| --- | --- |
| `docker compose config --quiet` | Succeeded. The local renderer emitted warnings only for unset deployment secrets. |
| `git diff --check` | Succeeded with no whitespace errors. |

No deployment, push, OAuth authentication, or Coolify secret modification was
performed.
