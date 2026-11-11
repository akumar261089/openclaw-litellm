# OpenClaw + LiteLLM Gateway (v2): Kilo CLI Build & Test Spec

> **Audience:** Kilo CLI agent. Execute phases in order, run every verification step, and write results to `TEST_REPORT.md`.
> **Goal:** Put a LiteLLM proxy (plus Postgres) in front of an existing OpenClaw Docker Compose stack so OpenClaw talks only to LiteLLM, and so **the user can pick any model from Ollama or any configured cloud provider** without rebuilding anything.

```
Chat apps → OpenClaw Gateway → LiteLLM Proxy (:4000) → Ollama (local) · Gemini · OpenRouter · (OpenAI / Anthropic / Bedrock optional)
                                      │
                                  PostgreSQL (virtual keys, spend, runtime-added models)
```

---

## 0. Agent operating rules

1. **Never print, log or commit secrets.** `.env` is gitignored. In reports show keys only as `sk-...last4`.
2. **Do not modify the existing OpenClaw services.** Add LiteLLM and Postgres through a separate override file (Phase 1). The only change to an OpenClaw service is a `depends_on` on `litellm`, made in that override file.
3. **Do not invent** OpenClaw image names, flags or config schema. Anything marked **VERIFY** must be checked against `--help` output, the live system, or the docs first.
4. If a step fails: read the logs, apply one fix, retry once. If it still fails, **stop, record it in `TEST_REPORT.md`, and ask the user.** No loops.
5. **Ask the user before using `--accept-risk`** on OpenClaw onboarding. It is a risk acknowledgement that belongs to the user, not the agent.
6. Providers with no API key are **skipped**, not failed. Remove their entries from `litellm_config.yaml` (and any fallbacks that reference them), and mark them `SKIPPED (no key)`.
7. Ports are bound to `127.0.0.1` only. Nothing is exposed publicly.

### Prerequisites

| Check | Command |
|---|---|
| Docker + Compose v2 | `docker --version && docker compose version` |
| `curl` and `jq` | `curl --version && jq --version` (install `jq` if missing) |
| Shell | Git Bash or WSL on Windows |
| Existing OpenClaw compose in the working folder | `docker compose config --services` must list the OpenClaw services (expected: `openclaw-gateway`, `openclaw-socat`, `openclaw-cli`). **VERIFY exact names.** If no OpenClaw compose exists, **stop and ask the user for it** |
| Ollama running with at least one model (if using local) | `curl -s http://localhost:11434/api/tags` and `ollama list` |

**Ollama reachability from containers**
- Windows / macOS (Docker Desktop): `http://host.docker.internal:11434` works as is.
- Linux: Ollama listens on `127.0.0.1` by default, which containers cannot reach. Set `OLLAMA_HOST=0.0.0.0` for the Ollama service and **firewall port 11434** from outside the machine. The override file below already adds the `host-gateway` mapping.
- If Ollama is itself a Compose service, set `OLLAMA_API_BASE=http://<service-name>:11434`.

---

## 1. How the user chooses a model

Three mechanisms, from least to most effort. All three are built and tested in this spec.

| # | Mechanism | How the user picks a model | Needs restart? |
|---|---|---|---|
| 1 | **Wildcards** (any model, no config edit) | Use the provider-prefixed name directly: `ollama_chat/<ollama-tag>`, `gemini/<model-id>`, `openrouter/<vendor>/<model>` | No |
| 2 | **Aliases** (friendly names) | Use `local-private`, `general`, `coding`. Re-point an alias by editing `litellm_config.yaml` | Yes (`docker compose restart litellm`) |
| 3 | **Runtime add** (named entries on the fly) | `./scripts/models.sh add-ollama <tag> [alias]` or `add <alias> <model-id> <ENV_KEY>`; or use the LiteLLM admin UI at `http://localhost:4000/ui` | No |

OpenClaw's "default model" is any `model_name` LiteLLM exposes: an alias, a wildcard-form name, or a runtime-added name. Changing OpenClaw's default later: re-run onboarding or use OpenClaw's own model setting. **VERIFY in https://docs.openclaw.ai/providers/litellm.**

> **Tool-calling caveat:** agents need tool/function calling. Small local models (e.g. ~4B) may handle it poorly. Phase 3 includes an informational capability check. Prefer a larger local model or a cloud alias for agent work.
>
> **Privacy rule:** `local-private` must **never** fall back to a cloud model. No fallback is configured for it.

---

## 2. Target layout

```
openclaw-litellm/
├── docker-compose.yml           # EXISTING OpenClaw stack (do not edit)
├── docker-compose.litellm.yml   # NEW: postgres + litellm + depends_on override
├── litellm_config.yaml          # NEW
├── .env                         # existing file: APPEND new vars, never overwrite
├── .env.example                 # NEW (documents the added vars)
├── .gitignore
├── scripts/
│   ├── models.sh                # list / add / remove / test models
│   └── smoke_test.sh
└── TEST_REPORT.md
```

---

## 3. Phase 1: Scaffold

### 3.1 `.gitignore` (append if present)

```
.env
*.log
```

### 3.2 `.env.example` (variables to ADD to `.env`)

```bash
# --- Make `docker compose` load the override automatically ---
COMPOSE_FILE=docker-compose.yml:docker-compose.litellm.yml
COMPOSE_PATH_SEPARATOR=:

# --- LiteLLM core ---
LITELLM_MASTER_KEY=sk-change-me-long-random     # admin only; must start with sk-
LITELLM_SALT_KEY=sk-change-me-another-random    # encrypts credentials stored in DB; set once, NEVER change
POSTGRES_PASSWORD=change-me-long-random
# VERIFY tag at https://github.com/BerriAI/litellm/pkgs/container/litellm ; pin the exact tested tag after Phase 3
LITELLM_TAG=main-stable

# --- Ollama (as seen from inside containers) ---
OLLAMA_API_BASE=http://host.docker.internal:11434
# --- Ollama (as seen from the host, used by scripts/) ---
OLLAMA_LOCAL_URL=http://localhost:11434

# --- Provider keys: fill only what you use, leave others blank ---
GEMINI_API_KEY=
OPENROUTER_API_KEY=
OPENAI_API_KEY=
ANTHROPIC_API_KEY=

# --- Created in Phase 5 (scoped key for OpenClaw) ---
OPENCLAW_LITELLM_KEY=
```

**Agent action:** append these to `.env` (skip any that already exist), generate strong random values for `LITELLM_MASTER_KEY`, `LITELLM_SALT_KEY` and `POSTGRES_PASSWORD` (keep `sk-` prefixes), and ask the user to fill in the provider keys they want. Do not guess keys.

### 3.3 `docker-compose.litellm.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: litellm
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: litellm
    volumes:
      - litellm_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U litellm -d litellm"]
      interval: 5s
      timeout: 5s
      retries: 12

  litellm:
    image: ghcr.io/berriai/litellm:${LITELLM_TAG:-main-stable}
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    env_file: .env
    environment:
      DATABASE_URL: postgresql://litellm:${POSTGRES_PASSWORD}@postgres:5432/litellm
      STORE_MODEL_IN_DB: "True"        # enables runtime model add/remove (mechanism 3)
    volumes:
      - ./litellm_config.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    ports:
      - "127.0.0.1:4000:4000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    healthcheck:
      # Python, because the image may not include curl
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:4000/health/liveliness')"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s

  # Make OpenClaw wait for the gateway. VERIFY the service name against `docker compose config --services`.
  openclaw-gateway:
    depends_on:
      litellm:
        condition: service_healthy

volumes:
  litellm_pgdata:
```

> If `docker compose config` errors because the existing `openclaw-gateway` defines `depends_on` as a list, convert that one entry to the map form inside this override file, or drop the `depends_on` block and note it in the report.
> The services share the stack's default network, so OpenClaw reaches LiteLLM at `http://litellm:4000`.
> On Synology or with UID/GID permission errors, the OpenClaw services may need `user: "1000:1000"` (see the existing compose comments). This is unrelated to LiteLLM.

### 3.4 `litellm_config.yaml`

Environment variables alone do not define models; this file does.

```yaml
model_list:
  # ===== Mechanism 2: friendly aliases (edit the right-hand model IDs to re-point) =====
  - model_name: local-private
    litellm_params:
      model: ollama_chat/qwen3.5:4b            # VERIFY: must exactly match `ollama list`
      api_base: os.environ/OLLAMA_API_BASE

  - model_name: general
    litellm_params:
      model: gemini/gemini-2.5-flash           # VERIFY: model ID still available
      api_key: os.environ/GEMINI_API_KEY

  - model_name: coding
    litellm_params:
      model: openrouter/meta-llama/llama-3.3-70b-instruct:free   # VERIFY on your OpenRouter account
      api_key: os.environ/OPENROUTER_API_KEY

  # ===== Mechanism 1: wildcards. ANY model from these providers, chosen by name =====
  # VERIFY wildcard syntax: https://docs.litellm.ai/docs/wildcard_routing
  - model_name: "ollama_chat/*"
    litellm_params:
      model: "ollama_chat/*"
      api_base: os.environ/OLLAMA_API_BASE

  - model_name: "gemini/*"
    litellm_params:
      model: "gemini/*"
      api_key: os.environ/GEMINI_API_KEY

  - model_name: "openrouter/*"
    litellm_params:
      model: "openrouter/*"
      api_key: os.environ/OPENROUTER_API_KEY

  # ===== Optional providers: uncomment, set a real model ID and key =====
  # - model_name: openai-main
  #   litellm_params: { model: "openai/<MODEL_ID>", api_key: os.environ/OPENAI_API_KEY }
  # - model_name: claude-main
  #   litellm_params: { model: "anthropic/<MODEL_ID>", api_key: os.environ/ANTHROPIC_API_KEY }
  # - model_name: bedrock-main
  #   litellm_params: { model: "bedrock/<MODEL_ID>", aws_region_name: us-east-1 }   # IAM role / credential chain

router_settings:
  routing_strategy: simple-shuffle
  num_retries: 2
  timeout: 120
  allowed_fails: 3
  cooldown_time: 30
  # An alias is not a fallback policy; fallbacks are explicit. None for local-private (privacy).
  fallbacks:
    - general: ["coding"]
    - coding: ["general"]

litellm_settings:
  drop_params: true
  set_verbose: false

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
```

> Do **not** rely on config hot-reload. After editing this file run `docker compose restart litellm`. Runtime-added models (mechanism 3) need no restart.
> **VERIFY** all model IDs and the `fallbacks` schema at https://docs.litellm.ai/docs/providers/ and https://docs.litellm.ai/docs/routing.

### 3.5 `scripts/models.sh`

```bash
#!/usr/bin/env bash
# Model chooser helper. Usage: ./scripts/models.sh <command> [args]
#   ollama                          list models installed in local Ollama
#   gateway                         list model names LiteLLM currently exposes
#   info                            list registered models: id, name, underlying model
#   add-ollama <tag> [alias]        expose a local Ollama model under an alias (default: local-<tag>)
#   add <alias> <model-id> <ENVKEY> expose a cloud model, e.g. add fast gemini/gemini-2.5-flash GEMINI_API_KEY
#   remove <model-id-from-info>     remove a runtime-added model
#   test <model-name>               send one test prompt through the gateway
set -euo pipefail
cd "$(dirname "$0")/.."
set -a; source .env; set +a

GW="${GATEWAY_URL:-http://localhost:4000}"
OLLAMA_LOCAL="${OLLAMA_LOCAL_URL:-http://localhost:11434}"
OLLAMA_IN_CONTAINER="${OLLAMA_API_BASE:-http://host.docker.internal:11434}"
H=(-H "Authorization: Bearer ${LITELLM_MASTER_KEY}" -H "Content-Type: application/json")

case "${1:-help}" in
  ollama)  curl -sf "$OLLAMA_LOCAL/api/tags" | jq -r '.models[].name' ;;
  gateway) curl -sf "$GW/v1/models" "${H[@]}" | jq -r '.data[].id' ;;
  info)    curl -sf "$GW/model/info" "${H[@]}" \
             | jq -r '.data[] | [.model_info.id, .model_name, .litellm_params.model] | @tsv' ;;
  add-ollama)
    tag="${2:?ollama tag required (see: $0 ollama)}"; alias="${3:-local-$tag}"
    curl -sf "$GW/model/new" "${H[@]}" -d "$(jq -n --arg a "$alias" --arg m "ollama_chat/$tag" --arg b "$OLLAMA_IN_CONTAINER" \
      '{model_name:$a, litellm_params:{model:$m, api_base:$b}}')" | jq -c '{added: .model_name}' ;;
  add)
    alias="${2:?alias}"; mid="${3:?litellm model id}"; key="${4:?API key env var name}"
    curl -sf "$GW/model/new" "${H[@]}" -d "$(jq -n --arg a "$alias" --arg m "$mid" --arg k "os.environ/$key" \
      '{model_name:$a, litellm_params:{model:$m, api_key:$k}}')" | jq -c '{added: .model_name}' ;;
  remove)
    curl -sf "$GW/model/delete" "${H[@]}" -d "$(jq -n --arg i "${2:?model id}" '{id:$i}')" | jq -c . ;;
  test)
    curl -sf "$GW/v1/chat/completions" "${H[@]}" -d "$(jq -n --arg m "${2:?model name}" \
      '{model:$m, messages:[{role:"user",content:"Reply with the single word: pong"}], max_tokens:20}')" \
      | jq -r '.choices[0].message.content' ;;
  *) sed -n '2,11p' "$0" ;;
esac
```

> **VERIFY** `/model/new`, `/model/delete` and `/model/info` payloads against the running LiteLLM version (`http://localhost:4000/` Swagger docs). A runtime-added `api_key` of the form `os.environ/NAME` requires `NAME` to exist in the LiteLLM container's environment (it does, via `env_file`).

---

## 4. Phase 2: Start the gateway first (OpenClaw not yet)

```bash
chmod +x scripts/*.sh
docker compose config > /dev/null && echo "compose OK"
docker compose up -d postgres litellm
docker compose ps
docker compose logs litellm --tail 80
curl http://localhost:4000/health/liveliness
```

**Pass:** `postgres` and `litellm` are `healthy`; no stack traces; liveness returns 200.

---

## 5. Phase 3: Smoke tests (gateway + model choice)

### `scripts/smoke_test.sh`

```bash
#!/usr/bin/env bash
# Prints PASS/FAIL/SKIP/INFO per check. Never prints full keys.
set -u
cd "$(dirname "$0")/.."
set -a; source .env; set +a

GW="http://localhost:4000"
OLLAMA_LOCAL="${OLLAMA_LOCAL_URL:-http://localhost:11434}"
P=0; F=0; S=0
ok(){ echo "PASS  $1"; P=$((P+1)); }
bad(){ echo "FAIL  $1"; F=$((F+1)); }
skip(){ echo "SKIP  $1"; S=$((S+1)); }
info(){ echo "INFO  $1"; }
code(){ curl -s -o /dev/null -w "%{http_code}" "$@"; }
chat(){ # chat <model> <bearer> [extra-json-fields]
  curl -s -o /tmp/resp.json -w "%{http_code}" "$GW/v1/chat/completions" \
    -H "Authorization: Bearer $2" -H "Content-Type: application/json" \
    -d "$(jq -n --arg m "$1" '{model:$m,messages:[{role:"user",content:"Reply with the single word: pong"}],max_tokens:20}')"; }
MK="$LITELLM_MASTER_KEY"

# 1-3 basics
[ "$(code $GW/health/liveliness)" = 200 ] && ok "liveness" || bad "liveness"
[ "$(code $GW/v1/models)" = 401 ] && ok "anonymous rejected (401)" || bad "anonymous not rejected"
[ "$(code $GW/v1/models -H "Authorization: Bearer $MK")" = 200 ] && ok "model list (master key)" || bad "model list"

# 4 aliases
t_alias(){ if [ -n "${2:-}" ] && [ -z "${!2:-}" ]; then skip "alias $1 (no $2)"; return; fi
  [ "$(chat "$1" "$MK")" = 200 ] && ok "alias $1" || bad "alias $1 (see /tmp/resp.json)"; }
t_alias local-private
t_alias general GEMINI_API_KEY
t_alias coding OPENROUTER_API_KEY

# 5 MODEL CHOICE, mechanism 1: wildcard. Any installed Ollama model, picked by name
TAG=$(curl -s "$OLLAMA_LOCAL/api/tags" | jq -r '.models[0].name // empty')
if [ -n "$TAG" ]; then
  [ "$(chat "ollama_chat/$TAG" "$MK")" = 200 ] && ok "wildcard ollama_chat/$TAG" || bad "wildcard ollama_chat/$TAG"
else skip "wildcard ollama (no Ollama models found)"; fi
if [ -n "${GEMINI_API_KEY:-}" ]; then
  [ "$(chat "gemini/gemini-2.5-flash" "$MK")" = 200 ] && ok "wildcard gemini/gemini-2.5-flash" || bad "wildcard gemini"
else skip "wildcard gemini (no key)"; fi

# 6 MODEL CHOICE, mechanism 3: runtime add -> use -> remove
if [ -n "$TAG" ]; then
  ./scripts/models.sh add-ollama "$TAG" smoke-local >/dev/null 2>&1 && ok "runtime add smoke-local" || bad "runtime add"
  [ "$(chat smoke-local "$MK")" = 200 ] && ok "runtime-added model responds" || bad "runtime-added model"
  MID=$(curl -s "$GW/model/info" -H "Authorization: Bearer $MK" | jq -r '.data[] | select(.model_name=="smoke-local") | .model_info.id' | head -1)
  [ -n "$MID" ] && ./scripts/models.sh remove "$MID" >/dev/null 2>&1 && ok "runtime remove" || bad "runtime remove"
else skip "runtime add/remove (no Ollama model)"; fi

# 7 fallback
if [ -n "${GEMINI_API_KEY:-}" ] && [ -n "${OPENROUTER_API_KEY:-}" ]; then
  C=$(curl -s -o /tmp/fb.json -w "%{http_code}" "$GW/v1/chat/completions" -H "Authorization: Bearer $MK" \
    -H "Content-Type: application/json" \
    -d '{"model":"general","mock_testing_fallbacks":true,"messages":[{"role":"user","content":"ping"}],"max_tokens":10}')
  [ "$C" = 200 ] && ok "fallback general -> coding" || bad "fallback (see /tmp/fb.json)"
else skip "fallback (needs Gemini + OpenRouter keys)"; fi

# 8 scoped key (what OpenClaw will use): all models allowed, with a budget
K=$(curl -s "$GW/key/generate" -H "Authorization: Bearer $MK" -H "Content-Type: application/json" \
  -d '{"key_alias":"smoke-scoped","max_budget":1,"budget_duration":"1d","duration":"1h"}' | jq -r '.key // empty')
if [ -n "$K" ]; then
  ok "scoped key created (sk-...${K: -4})"
  [ "$(code $GW/v1/models -H "Authorization: Bearer $K")" = 200 ] && ok "scoped key lists models" || bad "scoped key list"
  curl -s "$GW/key/delete" -H "Authorization: Bearer $MK" -H "Content-Type: application/json" -d "{\"keys\":[\"$K\"]}" >/dev/null
else bad "scoped key creation (is the database connected?)"; fi

# 9 restricted key must be denied for a non-allowed model
R=$(curl -s "$GW/key/generate" -H "Authorization: Bearer $MK" -H "Content-Type: application/json" \
  -d '{"key_alias":"smoke-restricted","models":["local-private"],"duration":"1h"}' | jq -r '.key // empty')
if [ -n "$R" ]; then
  D=$(chat general "$R"); { [ "$D" = 401 ] || [ "$D" = 403 ]; } && ok "restricted key denied non-allowed model ($D)" || bad "restricted key not enforced ($D)"
  curl -s "$GW/key/delete" -H "Authorization: Bearer $MK" -H "Content-Type: application/json" -d "{\"keys\":[\"$R\"]}" >/dev/null
else bad "restricted key creation"; fi

# 10 informational: does local-private support tool calling? (agents need it)
TC=$(curl -s "$GW/v1/chat/completions" -H "Authorization: Bearer $MK" -H "Content-Type: application/json" -d '{
  "model":"local-private","max_tokens":100,
  "messages":[{"role":"user","content":"What is the weather in Paris? Use the tool."}],
  "tools":[{"type":"function","function":{"name":"get_weather","description":"Get weather","parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}}]}' \
  | jq -r '.choices[0].message.tool_calls[0].function.name // empty')
[ "$TC" = "get_weather" ] && info "local-private made a tool call (good for agents)" \
  || info "local-private did NOT make a tool call; prefer a larger local or cloud model for OpenClaw"

echo "-----"; echo "PASS=$P FAIL=$F SKIP=$S"; [ "$F" -eq 0 ]
```

**Pass:** `FAIL=0`. `SKIP` only for missing keys/models. `INFO` lines are recorded, never failures.

**Persistence check**

```bash
docker compose restart litellm          # wait for healthy
./scripts/models.sh add-ollama "$(./scripts/models.sh ollama | head -1)" persist-test
docker compose restart litellm          # wait for healthy
./scripts/models.sh gateway | grep persist-test   # must still be listed (stored in Postgres)
./scripts/models.sh info                           # copy persist-test's id, then:
./scripts/models.sh remove <id>
```

---

## 6. Phase 4: Pin versions

1. Get the running LiteLLM version: `docker compose exec litellm pip show litellm`.
2. Replace `LITELLM_TAG=main-stable` in `.env` and `.env.example` with the exact tested tag.
3. `docker compose up -d postgres litellm`, then re-run `./scripts/smoke_test.sh`.
4. Record the pinned LiteLLM and Postgres tags in the report.

---

## 7. Phase 5: Connect OpenClaw

Docs (**VERIFY first**): https://docs.openclaw.ai/providers/litellm · https://docs.openclaw.ai/gateway/config-tools/custom-providers

### 7.1 Create a scoped key for OpenClaw. Do **not** use the master key

```bash
curl -s http://localhost:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H "Content-Type: application/json" \
  -d '{"key_alias":"openclaw","max_budget":50,"budget_duration":"30d"}' | jq -r .key
```

Save the result as `OPENCLAW_LITELLM_KEY` in `.env`. No `models` restriction is set on purpose, so the user can switch between any model the gateway exposes. To lock OpenClaw to specific models instead, add `"models":["local-private","general","coding"]`.
(Ollama usage costs $0, so the budget only constrains cloud spend. **VERIFY** the `budget_duration` format against LiteLLM docs.)

### 7.2 Start the OpenClaw services

```bash
docker compose up -d          # starts the rest of the existing stack (gateway, socat, ...)
docker compose ps
```

### 7.3 Onboard OpenClaw against LiteLLM

Interactive (preferred; the user answers the prompts):

```bash
docker compose run --rm openclaw-cli onboard
#   Auth method:    LiteLLM API key
#   Base URL:       http://litellm:4000        <- service name, NOT localhost
#   API key:        $OPENCLAW_LITELLM_KEY
#   Default model:  any model name from `./scripts/models.sh gateway`
#                   (e.g. local-private, general, coding, ollama_chat/<tag>, or a runtime-added name)
```

Non-interactive (**VERIFY flags with `docker compose run --rm openclaw-cli onboard --help`; ask the user before adding `--accept-risk`**):

```bash
docker compose run --rm openclaw-cli onboard \
  --non-interactive --accept-risk \
  --auth-choice litellm-api-key \
  --litellm-api-key "$OPENCLAW_LITELLM_KEY" \
  --custom-base-url "http://litellm:4000"
```

### 7.4 Verify end to end

```bash
docker compose exec openclaw-gateway node /app/dist/index.js agent --agent main -m "Hello, what model are you?"
docker compose logs litellm --tail 50     # the request must appear under key alias "openclaw"
```

Repeat once with a different default model (a local one, then a cloud one) to prove model switching works. Control UI: `http://127.0.0.1:18789`. OpenClaw config persists in `~/.openclaw`.

---

## 8. Security & operations checklist

- [ ] `.env` gitignored and in no commit; `LITELLM_SALT_KEY` set once and never changed
- [ ] LiteLLM and OpenClaw ports bound to `127.0.0.1`; Ollama firewalled if bound to `0.0.0.0`
- [ ] Master key used only for admin calls; OpenClaw uses its own scoped key
- [ ] Images pinned (no `latest` / `main-latest`)
- [ ] Postgres volume survives `docker compose down` (never use `-v` unless resetting on purpose)
- [ ] `local-private` has no cloud fallback
- [ ] Bedrock (if enabled) uses IAM role / credential chain
- [ ] Any non-local exposure goes behind a TLS reverse proxy with network restrictions

**Day-to-day**

```bash
docker compose up -d                      # start everything
docker compose logs -f litellm            # gateway logs
docker compose logs -f openclaw-gateway   # OpenClaw logs
docker compose down                       # stop (data persists)
./scripts/models.sh ollama                # what Ollama has installed
./scripts/models.sh gateway               # what the gateway exposes
```

---

## 9. Deliverable: `TEST_REPORT.md`

```markdown
# Gateway Test Report
Date / Host OS:
## Environment: Docker+Compose versions · pinned LiteLLM & Postgres tags · Ollama models found · OpenClaw service names found
## Results (PASS / FAIL / SKIPPED / INFO + notes)
Compose config valid · Containers healthy · Liveness · Anonymous rejected · Aliases (local-private, general, coding)
· Wildcard Ollama · Wildcard Gemini · Runtime add/use/remove · Persistence after restart · Fallback
· Scoped key · Restricted key · Tool-calling (INFO) · OpenClaw onboarding · OpenClaw via local model · OpenClaw via cloud model
## Deviations from this spec
## Open issues / questions for the user
## VERIFY items and what was found
```

## 10. Definition of done

1. Postgres + LiteLLM healthy; `./scripts/smoke_test.sh` shows `FAIL=0`.
2. User can select a model via wildcard, alias and runtime add, and each is proven by a passing test.
3. Runtime-added models survive a restart.
4. OpenClaw answers through LiteLLM with a scoped key, on both a local and a cloud model (or the report says exactly why not).
5. Versions pinned, no secrets committed, `TEST_REPORT.md` complete.
