# Running OpenClaw + LiteLLM in one Docker Compose stack

This adds a [LiteLLM](https://docs.litellm.ai/) proxy container to your OpenClaw
compose stack, so OpenClaw talks to LiteLLM instead of talking to OpenAI/Anthropic/etc.
directly. You get centralized cost tracking, spend limits, and automatic failover
across 100+ providers — all from one gateway.

```
Chat apps → OpenClaw Gateway → LiteLLM Proxy → LLM Providers (OpenAI, Anthropic, Gemini, ...)
```

## Files in this stack

| File                  | Purpose                                                                  |
| --------------------- | ------------------------------------------------------------------------ |
| `docker-compose.yml`  | Defines `litellm`, `openclaw-gateway`, `openclaw-socat`, `openclaw-cli`  |
| `litellm_config.yaml` | Which models LiteLLM exposes and which provider keys back them           |
| `.env`                | Your secrets (master key + provider API keys) — copy from `.env.example` |

## 1. Set up the folder

```bash
mkdir openclaw-litellm && cd openclaw-litellm
# copy in docker-compose.yml, litellm_config.yaml, and .env.example from this guide
cp .env.example .env
```

## 2. Fill in `.env`

Open `.env` and set:

```bash
LITELLM_MASTER_KEY=sk-choose-any-secret-string
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
```

You only need to fill in the providers you'll actually use — leave the others blank.

## 3. Adjust `litellm_config.yaml` if needed

Each entry under `model_list` maps a friendly `model_name` (what you'll type into
OpenClaw) to a real provider model. Add, remove, or rename entries as you like —
LiteLLM hot-reloads this file, no rebuild needed for config-only changes.

## 4. Start LiteLLM and the OpenClaw gateway

```bash
docker compose up -d litellm openclaw-gateway
```

Check that LiteLLM is healthy:

```bash
docker compose ps
curl http://localhost:4000/health/liveliness
```

## 5. Onboard OpenClaw against LiteLLM

Run the interactive onboarding via the CLI profile:

```bash
docker compose run --rm openclaw-cli onboard
```

When prompted:

- **Auth method:** LiteLLM API key
- **Base URL:** `http://litellm:4000` (this is the in-network service name, not `localhost`, since OpenClaw is reaching LiteLLM over the compose network)
- **API key:** the same value you set for `LITELLM_MASTER_KEY`
- **Default model:** one of the `model_name` values from `litellm_config.yaml` (e.g. `claude-sonnet`)

Or non-interactively:

```bash
docker compose run --rm openclaw-cli onboard \
  --non-interactive --accept-risk \
  --auth-choice litellm-api-key \
  --litellm-api-key "$LITELLM_MASTER_KEY" \
  --custom-base-url "http://litellm:4000"
```

## 6. Verify it's working

```bash
docker compose exec openclaw-gateway node /app/dist/index.js agent --agent main -m "Hello, what model are you?"
```

If you get a real model response back, the wiring is correct — OpenClaw is
routing every request through LiteLLM, and LiteLLM is routing to your chosen
provider.

## 7. Day-to-day commands

```bash
docker compose up -d                 # start everything
docker compose logs -f litellm       # tail LiteLLM logs
docker compose logs -f openclaw-gateway
docker compose down                  # stop everything (config in ~/.openclaw persists)
```

Open the OpenClaw control UI at `http://127.0.0.1:18789`.

## Notes

- Config and workspace data live in `~/.openclaw` and `~/.openclaw/workspace` on
  the host, same as the original stack — nothing about persistence changes.
- If you're on a Synology NAS or hit UID/GID permission errors, uncomment the
  `user: "1000:1000"` lines in `docker-compose.yml` for the OpenClaw services.
- To add spend limits or per-key budgets, generate a scoped key against the
  running LiteLLM instead of using the master key everywhere:

  ```bash
  curl -X POST "http://localhost:4000/key/generate" \
    -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
    -H "Content-Type: application/json" \
    -d '{"key_alias": "openclaw", "max_budget": 50.00, "budget_duration": "monthly"}'
  ```

  Use the returned key as `LITELLM_API_KEY` / the OpenClaw auth key instead of
  the master key.
