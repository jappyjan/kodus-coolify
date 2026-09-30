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

