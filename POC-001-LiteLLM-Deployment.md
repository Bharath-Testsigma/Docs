# POC-001 — LiteLLM Proxy Deployment

## What Changed in This Deployment

The original `atto-poc` called **OpenRouter** (a third-party aggregator) directly using the `litellm` Python library. The new deployment replaces that with a **self-hosted LiteLLM proxy server** running locally in Docker. The application code did not change — only where it points.

Before:
```
orchestrator.py → litellm library → OpenRouter (third-party) → Google/Anthropic/OpenAI
```

After:
```
orchestrator.py → litellm library → LiteLLM Proxy (localhost:4000, Docker) → Google/Anthropic/OpenAI
```

---

## Files Added or Changed

| File | Change |
|---|---|
| `litellm_config.yaml` | NEW — defines model aliases and which provider each routes to |
| `docker-compose.yml` | NEW — runs the LiteLLM proxy container |
| `app/config.py` | Replaced `openrouter_api_key/base` with `litellm_proxy_url` + `litellm_master_key` |
| `app/orchestrator.py` | `api_base` + `api_key` now point at `localhost:4000` |
| `ui.py` | Updated model IDs and names to match proxy aliases |
| `.env` / `.env.example` | Replaced single OpenRouter key with per-provider keys |

---

## What LiteLLM Is

LiteLLM is an open-source Python library and proxy server that gives you a **single, unified interface** to call 100+ LLM providers. It translates your request format once and handles all provider-specific differences internally.

It has two distinct modes:

| Mode | What it is | Used in this POC |
|---|---|---|
| **Library** (`import litellm`) | A Python package you call directly in code | Yes — `orchestrator.py` uses `litellm.acompletion()` |
| **Proxy server** (`litellm --config ...`) | A self-hosted HTTP server with an OpenAI-compatible API | Yes — runs in Docker on port 4000 |

---

## How LiteLLM Proxy Works in This POC

### 1. The proxy reads a config file at startup

`litellm_config.yaml` defines model **aliases** and their real provider mappings:

```yaml
model_list:
  - model_name: gemini-flash          # alias your app uses
    litellm_params:
      model: gemini/gemini-2.0-flash   # real model ID at Google
      api_key: os.environ/GOOGLE_API_KEY

  - model_name: claude-haiku
    litellm_params:
      model: anthropic/claude-haiku-4-5-20251001
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: gpt-4o-mini
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY
```

The app never knows the real model ID — it only uses the alias. To swap providers, you change the config file, not the application code.

### 2. The orchestrator calls the proxy like any OpenAI-compatible endpoint

```python
# app/orchestrator.py
response = await litellm.acompletion(
    model=active_model,           # e.g. "openai/claude-haiku"
    messages=messages,
    tools=TOOL_DEFINITIONS,
    api_base="http://localhost:4000",   # the local proxy
    api_key="sk-atto-local",            # proxy master key (local only)
)
```

The `openai/` prefix on the model name tells the litellm library to speak OpenAI-compatible format. The proxy receives the request, looks up `claude-haiku` in its config, and forwards it to Anthropic with the correct format and auth.

### 3. Request flow for one generation call

```
Streamlit UI
    │  POST /generate { query, model: "openai/claude-haiku" }
    ▼
FastAPI (port 8000)
    │  calls run_orchestrator()
    ▼
orchestrator.py
    │  litellm.acompletion(model="openai/claude-haiku", api_base="http://localhost:4000")
    ▼
LiteLLM Proxy (Docker, port 4000)
    │  looks up "claude-haiku" → anthropic/claude-haiku-4-5-20251001
    │  translates to Anthropic message format
    │  attaches ANTHROPIC_API_KEY from env
    ▼
Anthropic API
    │  returns response (possibly with tool calls)
    ▼
LiteLLM Proxy
    │  normalizes response back to OpenAI format
    ▼
orchestrator.py
    │  parses tool calls, executes WriteFile/ReadFile/etc.
    │  loops until model returns no tool calls
    ▼
FastAPI
    │  returns GenerateResponse with test cases
    ▼
Streamlit UI
    └  displays generated XML test cases
```

### 4. The agentic tool loop

The orchestrator runs a **while loop** that:

1. Sends the conversation + tools to the LLM via the proxy
2. If the LLM returns tool calls (e.g. `WriteFile`, `ReadFile`, `ListFiles`), executes them locally
3. Appends tool results to the conversation and loops again
4. Stops when the LLM sends a plain text response (the `<output>` block)
5. Falls back to `FALLBACK_MODEL` if the primary model fails

Tool calls are defined in `app/tools.py` and executed locally — the LLM only sees results, never touches the filesystem directly.

### 5. Model routing table (current config)

| UI Label | Proxy Alias | Real Model | Provider |
|---|---|---|---|
| Claude Haiku 4.5 (default) | `claude-haiku` | `claude-haiku-4-5-20251001` | Anthropic |
| Claude Sonnet 4.6 | `claude-sonnet` | `claude-sonnet-4-6` | Anthropic |
| Gemini 2.0 Flash | `gemini-flash` | `gemini-2.0-flash` | Google |
| Gemini 2.0 Flash Lite | `gemini-flash-lite` | `gemini-2.0-flash-lite` | Google |
| Gemini 2.5 Flash | `gemini-2.5-flash` | `gemini-2.5-flash` | Google |
| Gemini 2.5 Pro | `gemini-pro` | `gemini-2.5-pro` | Google |
| GPT-4o Mini | `gpt-4o-mini` | `gpt-4o-mini` | OpenAI |
| GPT-4o | `gpt-4o` | `gpt-4o` | OpenAI |

---

## Why Self-Hosted LiteLLM Instead of OpenRouter

| Concern | OpenRouter | LiteLLM Proxy (self-hosted) |
|---|---|---|
| API keys | One OpenRouter key (they hold your provider keys) | Your keys go directly to providers |
| Routing control | OpenRouter's rules | Fully yours — config file |
| Observability | OpenRouter dashboard | Proxy logs, can add Prometheus/Langfuse |
| Cost | OpenRouter markup on top of provider pricing | Direct provider pricing |
| Network path | Your app → OpenRouter → Provider | Your app → localhost → Provider |
| Future: alpha/atto-browser-agent-v2 | Would need code changes | Point `api_base` at proxy — no code changes |

---

## Running the Stack

### Start the proxy (first time pulls Docker image)
```bash
cd POC-001/atto-poc
docker compose up -d
docker compose ps   # wait for (healthy)
```

### Start the FastAPI backend
```bash
uv run python main.py
```

### Start the Streamlit UI
```bash
uv run streamlit run ui.py
```

Open `http://localhost:8501`.

### To add a new model
Edit `litellm_config.yaml`, add a new entry, then `docker compose restart`. No other files change.

```yaml
- model_name: my-new-model
  litellm_params:
    model: anthropic/claude-opus-4-7
    api_key: os.environ/ANTHROPIC_API_KEY
```

---

## Health Check Endpoints

| Endpoint | Auth required | Returns |
|---|---|---|
| `GET localhost:4000/health/liveliness` | No | `"I'm alive!"` |
| `GET localhost:4000/v1/models` | Bearer `sk-atto-local` | List of configured model aliases |
| `GET localhost:4000/health` | Bearer `sk-atto-local` | Per-model health status |
