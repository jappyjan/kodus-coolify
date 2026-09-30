# Kodus Codex LB Deployment Design

## Goal

Run Kodus code reviews through `gpt-5.6-sol` using the owner's ChatGPT/Codex subscription rather than the current Alibaba Token Plan endpoint.

## Architecture

Deploy `ghcr.io/soju06/codex-lb` as a separate Coolify application with persistent storage. The proxy is reachable only on the Coolify internal network and exposes an authenticated OpenAI-compatible `/v1` endpoint to Kodus.

Authenticate the proxy once with the owner's ChatGPT/Codex account. Create a dedicated proxy API key for Kodus. Set the Kodus API and worker services to use the proxy URL, this proxy API key, and `gpt-5.6-sol`.

## Components

- **codex-lb:** Terminates Kodus's OpenAI-compatible requests, uses the configured Codex OAuth account upstream, and manages subscription rate limits.
- **Kodus API:** Uses the proxy for synchronous LLM operations.
- **Kodus worker:** Uses the same proxy for asynchronous review jobs.
- **Coolify secrets:** Store the codex-lb dashboard bootstrap credential and the dedicated Kodus proxy API key. Do not place them in Git.

## Configuration

The codex-lb application uses a persistent volume at `/var/lib/codex-lb` and no public domain or exposed proxy port. Its dashboard is used only for initial OAuth login and proxy API-key creation.

The Kodus API and worker share these environment values:

```text
API_LLM_PROVIDER_MODEL=gpt-5.6-sol
API_OPENAI_FORCE_BASE_URL=http://<private-codex-lb-address>/v1
API_OPEN_AI_API_KEY=<dedicated-codex-lb-api-key>
```

The exact private address is determined from the Coolify application's internal service name after it is created. The existing Alibaba key and base URL remain stored only as a documented rollback configuration; they are not active after the cutover.

## Data Flow

1. A Kodus API or worker request targets codex-lb's internal `/v1` endpoint with the dedicated proxy API key.
2. codex-lb validates the key and forwards the request using the stored Codex OAuth session.
3. The ChatGPT/Codex subscription executes `gpt-5.6-sol` and returns the completion through the proxy to Kodus.

## Failure Handling

- If the proxy is unhealthy, authentication expires, or subscription quota is exhausted, Kodus requests fail rather than silently switching providers.
- Verify that the proxy lists `gpt-5.6-sol` before changing Kodus configuration.
- If the deployment is unhealthy or a controlled review fails, restore the three Kodus LLM environment variables to the Alibaba values and redeploy Kodus.
- The codex-lb volume preserves OAuth and dashboard state across restarts; back it up before upgrades.

## Validation

1. Confirm codex-lb is healthy and `gpt-5.6-sol` is available after OAuth authentication.
2. Confirm its `/v1/models` and a minimal authenticated generation work from the Coolify network.
3. Redeploy Kodus and confirm API, worker, and webhook health checks pass.
4. Trigger one controlled pull-request review and verify the corresponding codex-lb request completes with `gpt-5.6-sol`.
5. Retain the current configuration values until the review is successful, so rollback is immediate.

## Constraints

- Use Coolify only; do not install or run services directly over SSH.
- Keep codex-lb private to the Coolify network. Do not publish its dashboard or `/v1` endpoint.
- Use `gpt-5.6-sol` for both Kodus API and worker services.
- Treat the proxy as a community integration: it depends on undocumented Codex OAuth behavior and can be affected by upstream changes or subscription limits.
