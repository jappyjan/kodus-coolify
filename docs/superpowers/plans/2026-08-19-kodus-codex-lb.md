# Kodus Codex LB Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route Kodus API and worker LLM traffic through a private codex-lb service using `gpt-5.6-sol` and the owner's ChatGPT/Codex subscription.

**Architecture:** Add codex-lb to the existing Coolify Docker Compose application so it shares the Kodus service network and is addressed as `http://codex-lb:2455/v1`. Temporarily expose codex-lb's dashboard and OAuth callback only while the owner signs in and creates a dedicated proxy key; remove both public routes before routing Kodus traffic through it.

**Tech Stack:** Coolify CLI 1.6.2, Docker Compose, `ghcr.io/soju06/codex-lb:1.23.0`, Codex OAuth, Kodus self-hosted Docker images.

## Global Constraints

- Modify and deploy only through Coolify; do not SSH to the server or invoke Docker on it.
- Keep `codex-lb` on Kodus's existing Compose network; the steady-state proxy has no public route.
- Use `gpt-5.6-sol` for both the `api` and `worker` services.
- Store OAuth state in the named `codex-lb-data` volume mounted at `/var/lib/codex-lb`.
- Store the codex-lb dashboard bootstrap credential and Kodus proxy API key as Coolify secrets, never in Git or shell history.
- Preserve the current Alibaba values until the controlled review succeeds; restore them and redeploy on any failed health check or failed controlled review.
- Pin codex-lb to `1.23.0`; do not deploy the mutable `latest` tag.

---

## File Structure

- Modify: `docker-compose.yml` - adds the private codex-lb service and its persistent volume; changes only the API and worker LLM environment-variable defaults.
- Create: `docs/superpowers/specs/2026-08-19-kodus-codex-lb-design.md` - approved design record.
- Create: `docs/superpowers/plans/2026-08-19-kodus-codex-lb.md` - this runbook.

### Task 1: Add the Private Codex LB Compose Service

**Files:**
- Modify: `docker-compose.yml:89-92`
- Modify: `docker-compose.yml:173-176`
- Modify: `docker-compose.yml:191-250`
- Modify: `docker-compose.yml:316-320`

**Interfaces:**
- Consumes: Kodus's existing default Compose network and Coolify-managed application environment variables.
- Produces: The DNS name `codex-lb`, accepting authenticated OpenAI-compatible requests at `http://codex-lb:2455/v1` from `api` and `worker`.

- [ ] **Step 1: Add the failing Compose service assertion**

Create a temporary validation script at `/tmp/validate-kodus-codex-lb.sh`:

```sh
#!/bin/sh
set -eu
docker compose -f docker-compose.yml config --no-interpolate > /tmp/kodus-compose.rendered.yml
rg -q 'ghcr.io/soju06/codex-lb:1.23.0' /tmp/kodus-compose.rendered.yml
rg -q 'codex-lb-data:/var/lib/codex-lb' /tmp/kodus-compose.rendered.yml
rg -q 'API_OPENAI_FORCE_BASE_URL=http://codex-lb:2455/v1' /tmp/kodus-compose.rendered.yml
rg -q 'API_LLM_PROVIDER_MODEL=gpt-5.6-sol' /tmp/kodus-compose.rendered.yml
```

- [ ] **Step 2: Run the validation to verify it fails**

Run: `sh /tmp/validate-kodus-codex-lb.sh`

Expected: non-zero exit because codex-lb and the new Kodus LLM defaults do not exist yet.

- [ ] **Step 3: Add the codex-lb service and persistent volume**

Insert this service before `webhooks` in `docker-compose.yml`:

```yaml
  codex-lb:
    image: ghcr.io/soju06/codex-lb:1.23.0
    restart: unless-stopped
    expose:
      - "2455"
      - "1455"
    volumes:
      - codex-lb-data:/var/lib/codex-lb
```

Add the named volume alongside the existing volume declarations:

```yaml
  codex-lb-data:
```

Replace the API and worker defaults exactly:

```yaml
      - API_LLM_PROVIDER_MODEL=${API_LLM_PROVIDER_MODEL:-gpt-5.6-sol}
      - API_OPENAI_FORCE_BASE_URL=${API_OPENAI_FORCE_BASE_URL:-http://codex-lb:2455/v1}
```

Do not change the `API_OPEN_AI_API_KEY` Compose reference. Coolify supplies the dedicated codex-lb proxy key after OAuth setup.

- [ ] **Step 4: Re-run the validation**

Run: `sh /tmp/validate-kodus-codex-lb.sh`

Expected: exit code 0 and a fully rendered Compose file.

- [ ] **Step 5: Inspect the targeted diff**

Run: `git diff --check && git diff -- docker-compose.yml`

Expected: only the codex-lb service, volume, and API/worker model/base-URL defaults change.

- [ ] **Step 6: Commit the Compose change**

```bash
git add docker-compose.yml
```

### Task 2: Deploy Codex LB and Complete OAuth Setup

**Files:**
- Modify: `docker-compose.yml:191-205` temporarily, then restore its steady-state form.

**Interfaces:**
- Consumes: the `codex-lb` service from Task 1, the target Coolify application UUID, and the owner's interactive ChatGPT/Codex login.
- Produces: a persistent authenticated codex-lb account with `gpt-5.6-sol` available and a dedicated Kodus proxy API key.

- [ ] **Step 1: Add temporary Coolify routes for dashboard and OAuth callback**

Temporarily add the following two environment entries to the `codex-lb` service:

```yaml
      - SERVICE_FQDN_CODEX_LB_2455
      - SERVICE_FQDN_CODEX_LB_1455
```

The first route is the dashboard and proxy API. The second is required for the OAuth callback. Do not add either value to `api` or `worker`.

- [ ] **Step 2: Deploy the Compose application**

Run: `coolify deploy uuid <coolify-app-uuid> --force`

Expected: Coolify deploys codex-lb with the existing Kodus services; no database volumes are recreated.

- [ ] **Step 3: Confirm codex-lb starts cleanly**

Run: `coolify app logs <coolify-app-uuid> --lines 200`

Expected: the codex-lb container is listening on ports 2455 and 1455 and reports no filesystem or database initialization error.

- [ ] **Step 4: Complete interactive Codex OAuth and create a Kodus proxy key**

Open the temporary `SERVICE_FQDN_CODEX_LB_2455` dashboard URL. Use its bootstrap flow to set dashboard credentials, sign in with the owner's ChatGPT/Codex account, confirm the model list contains `gpt-5.6-sol`, and create one API key named `kodus`.

Do not paste the proxy key into chat, Git, or a command line. Keep it available only for the next Coolify secret update.

- [ ] **Step 5: Save the proxy key as the Kodus runtime secret**

Use Coolify's secret UI for the target application to replace `API_OPEN_AI_API_KEY` with the new codex-lb `kodus` API key. Mark it runtime-only and literal.

If using the CLI with a secret held in the current shell variable, run exactly:

```bash
read -rs KODUS_PROXY_KEY
coolify app env update <coolify-app-uuid> API_OPEN_AI_API_KEY --value "$KODUS_PROXY_KEY" --runtime --is-literal
unset KODUS_PROXY_KEY
```

- [ ] **Step 6: Remove the temporary public routes**

Remove these two lines from `docker-compose.yml` after OAuth succeeds:

```yaml
      - SERVICE_FQDN_CODEX_LB_2455
      - SERVICE_FQDN_CODEX_LB_1455
```

Redeploy using `coolify deploy uuid <coolify-app-uuid> --force`.

Expected: codex-lb remains reachable as `http://codex-lb:2455` only from the Compose network. Its dashboard and proxy endpoint are no longer publicly routable.

### Task 3: Cut Kodus Over and Validate a Review

**Files:**
- Modify: Coolify runtime environment variables for the target application.

**Interfaces:**
- Consumes: authenticated private codex-lb from Task 2 and its `kodus` proxy API key in `API_OPEN_AI_API_KEY`.
- Produces: Kodus API and worker requests routed through codex-lb with model `gpt-5.6-sol`.

- [ ] **Step 1: Set the three active Kodus LLM values in Coolify**

In Coolify application environment variables, set these runtime literal values:

```text
API_LLM_PROVIDER_MODEL=gpt-5.6-sol
API_OPENAI_FORCE_BASE_URL=http://codex-lb:2455/v1
API_OPEN_AI_API_KEY=<the codex-lb kodus API key created in Task 2>
```

The model and base URL may be set with the CLI:

```bash
coolify app env update <coolify-app-uuid> API_LLM_PROVIDER_MODEL --value gpt-5.6-sol --runtime --is-literal
coolify app env update <coolify-app-uuid> API_OPENAI_FORCE_BASE_URL --value http://codex-lb:2455/v1 --runtime --is-literal
```

- [ ] **Step 2: Deploy the cutover**

Run: `coolify deploy uuid <coolify-app-uuid> --force`

Expected: API and worker start with the codex-lb model/base URL values.

- [ ] **Step 3: Verify Kodus health and proxy traffic**

Run:

```bash
curl --fail --silent --show-error <api-public-url>/health
coolify app logs <coolify-app-uuid> --lines 300
```

Expected: health endpoint exits 0, API and worker logs have no authentication, connection-refused, or unknown-model errors, and codex-lb records the model request.

- [ ] **Step 4: Trigger one controlled pull-request review**

Use the Kodus UI to request a review on a small non-production pull request. Inspect the codex-lb request log in the private dashboard through a temporary route only if necessary.

Expected: the review completes and the proxied request records `gpt-5.6-sol`.

- [ ] **Step 5: Execute rollback if validation fails**

If the health endpoint, logs, or controlled review fails, restore these prior Coolify values and redeploy immediately:

```text
API_LLM_PROVIDER_MODEL=qwen3.8-max
API_OPENAI_FORCE_BASE_URL=https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1
API_OPEN_AI_API_KEY=<the prior Alibaba Token Plan secret>
```

Run: `coolify deploy uuid <coolify-app-uuid> --force`

Expected: Kodus returns to its current Alibaba-backed behavior without data loss.

- [ ] **Step 6: Record operational outcome**

Add a short dated entry to `README.md` documenting whether the controlled review succeeded, the codex-lb image version, and that proxy state is stored in `codex-lb-data`. Do not record URLs that are no longer public or any credentials.

- [ ] **Step 7: Commit the outcome note**

```bash
git add README.md
```
