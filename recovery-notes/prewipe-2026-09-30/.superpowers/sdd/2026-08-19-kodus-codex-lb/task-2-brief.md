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
