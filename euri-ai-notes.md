# EURI AI — Quick Notes

**What it is:** EURI (by Euron) is an **OpenAI-compatible API gateway**. One API key + one base URL gives you access to 40+ models from OpenAI, Google, Anthropic, Meta, Groq, Sarvam, etc. If you already know the OpenAI SDK, you already know EURI — just swap the `base_url` and `api_key`.

**Base URL:**
```
https://api.euron.one/api/v1/euri
```

---

## 1. Quickstart

1. Sign in at **euron.one/euri** → go to **Billing & API Keys** → **Create API Key**.
2. Call it like OpenAI:

```bash
curl -X POST https://api.euron.one/api/v1/euri/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_EURI_API_KEY" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Say hello in 3 languages."}],
    "max_tokens": 150
  }'
```

3. Or drop straight into the **OpenAI SDK** (Python example):

```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_EURI_API_KEY",
    base_url="https://api.euron.one/api/v1/euri"
)

response = client.chat.completions.create(
    model="gemini-2.5-flash",   # any EURI model id works here
    messages=[{"role": "user", "content": "Explain quantum computing simply."}]
)
print(response.choices[0].message.content)
```

There's also a native `euriai` SDK for Python/JS, but the OpenAI SDK approach is the simplest if you're migrating existing code.

---

## 2. Authentication

- Every request needs: `Authorization: Bearer YOUR_EURI_API_KEY`
- Get the key from **euron.one/euri → Billing & API Keys**. It's shown only once — save it.
- Best practices: keep it server-side only, use env vars, rotate periodically, use separate keys for dev/prod.
- Bad key → `401`-style JSON error with `code: "invalid_api_key"`.

---

## 3. Pricing (simple summary)

- **No subscription** — pay-per-use, wallet-based (topped up in INR).
- **Free tier:** 10,000 tokens/day on non-premium models, no wallet needed.
- **Euron Plus:** 100,000 tokens/day on non-premium models.
- **Premium models** (GPT-5 family, Claude, o3/o4-mini, Gemini 3.x, etc.) always draw from wallet balance — daily free quota doesn't apply to them.
- EURI adds a **15% markup** over the provider's raw price.

Cheap/free-tier-friendly picks: `gpt-4o-mini`, `gpt-4.1-mini`, `gemini-2.0-flash`, `gemini-2.5-flash`, `llama-3.1-8b-instant`, `sarvam-m` (free).

Pricier/premium picks: `gpt-5.4`, `o3`, `claude-opus-4-6`, `gemini-3.1-pro`.

(Full per-model $/M-token table is on the [Pricing page](https://docs.euri.ai/pricing) — costs range roughly from $0.05–$15 input / $0.08–$75 output per million tokens depending on model tier.)

---

## 4. Models (highlights — 40+ total)

| Provider | Example model IDs | Notes |
|---|---|---|
| OpenAI | `gpt-4o-mini`, `gpt-4.1`, `gpt-5`, `gpt-5.4`, `o3`, `o4-mini`, `gpt-oss-20b/120b` | `gpt-5.4` has ~1M context |
| Google | `gemini-2.0-flash`, `gemini-2.5-flash`, `gemini-2.5-pro`, `gemini-3-flash`, `gemini-3.1-pro` | All ~1M context |
| Anthropic | `claude-sonnet-4-6`, `claude-opus-4-6`, `claude-haiku-4-5`, etc. | 200K context, all premium |
| Meta (via Groq) | `llama-4-scout-17b-16e-instruct`, `llama-3.3-70b-versatile`, `llama-3.1-8b-instant` | Fast, non-premium |
| Other | `qwen/qwen3-32b`, `groq/compound` (agentic/tool-use), `sarvam-m` (Indian languages, free) | |

Also available: **embeddings** (`text-embedding-3-small`, `gemini-embedding-001`, etc.), **image generation** (`gemini-3-pro-image-preview`), **speech-to-text** (Whisper, Sarvam), **text-to-speech** (Orpheus, PlayAI, Sarvam).

List all models programmatically:
```bash
curl https://api.euron.one/api/v1/euri/models \
  -H "Authorization: Bearer YOUR_EURI_API_KEY"
```

---

## 5. Rate Limits

- Applies only to **non-premium** models; resets midnight UTC.
- Free: 10,000 tokens/day. Plus: 100,000 tokens/day.
- Hit the limit → `403 Forbidden`, `code: "token_limit_exceeded"`.
- To go beyond: upgrade to Euron Plus, or use premium models (billed from wallet, no daily cap).

---

## 6. Cursor Setup

1. Cursor → **Settings → Models**.
2. Under **OpenAI**, paste your EURI API key.
3. Enable **"Override OpenAI Base URL (when using key)"**.
4. Set base URL to `https://api.euron.one/api/v1/euri`.
5. Add a custom model ID if your Cursor version supports it (e.g. `gpt-4o-mini`, `claude-sonnet-4-6`).
6. Test with a chat prompt.

Tip: if verification fails, try switching Cursor's network compatibility mode to HTTP/1.1, and test first with a non-premium model like `gpt-4o-mini`.

---

## 7. VS Code + Cline Setup

1. Install the **Cline** extension in VS Code.
2. Open Cline panel → gear icon → Settings.
3. **API Provider → "OpenAI Compatible"** (⚠️ not the plain "OpenAI" option).
4. Fill in:
   - **Base URL:** `https://api.euron.one/api/v1/euri`
   - **API Key:** your EURI key
   - **Model ID:** e.g. `gpt-4o-mini`, `gemini-2.5-flash`, `claude-sonnet-4-6`
5. Save, then test with a prompt in a new Cline chat.

Recommended default models for coding (both Cursor & Cline):
| Use case | Model |
|---|---|
| Fast everyday coding | `gpt-4o-mini` |
| Large context tasks | `gemini-2.5-flash` |
| Best reasoning quality | `claude-sonnet-4-6` |
| Open-source/free-ish | `llama-4-scout-17b-16e-instruct` |

---

## TL;DR
EURI = one key + one URL (`https://api.euron.one/api/v1/euri`) → OpenAI SDK-compatible → access GPT, Gemini, Claude, Llama and more. Free daily quota for lighter models, wallet-based billing for premium ones. Works as a drop-in "OpenAI Compatible" provider in Cursor and Cline (VS Code).
