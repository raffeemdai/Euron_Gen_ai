# EURI AI — Quick Notes

documentation  refer: euri ai models documentation: 
https://docs.euri.ai/quickstart

**What it is:** EURI (by Euron) is an **OpenAI-compatible API gateway**. One API key + one base URL gives you access to 40+ models from OpenAI, Google, Anthropic, Meta, Groq, Sarvam, etc. If you already know the OpenAI SDK, you already know EURI — just swap the `base_url` and `api_key`.

**Base URL:**
```
https://api.euron.one/api/v1/euri
```

---

## 1. Quickstart (elaborated)

### Step 1 — Create an account and get an API key
1. Go to **[euron.one/euri](https://euron.one/euri)** and sign in / sign up.
2. Open **Billing & API Keys** in the dashboard.
3. Click **Create API Key** → copy it immediately (it's shown only once).
4. You don't need to add money to try it — the **free tier gives 10,000 tokens/day** on non-premium models, no wallet top-up required.

### Step 2 — Set up your environment (Python)

You can use either `pip` or `uv` (faster, drop-in compatible) — both work identically here since EURI just needs the standard `openai` or `euriai` packages.

**Using pip:**
```bash
# Option A: use EURI purely as an OpenAI-compatible endpoint (recommended, zero learning curve)
pip install openai

# Option B: use EURI's own native SDK (adds LangChain/CrewAI/AutoGen helpers etc.)
pip install euriai
```

**Using uv:**
```bash
# If you don't have uv yet:
# curl -LsSf https://astral.sh/uv/install.sh | sh        (macOS/Linux)
# powershell -c "irm https://astral.sh/uv/install.ps1 | iex"   (Windows)

# Option A: OpenAI SDK
uv add openai

# Option B: native euriai SDK
uv add euriai
```

`uv add` creates/updates a `pyproject.toml` + lockfile for a proper project. If you just want a quick throwaway script without a project, use `uv pip install` (mirrors plain pip) or the newer inline-script style:

```bash
# quick, pip-style install into current environment
uv pip install openai

# or run a standalone script with dependencies declared inline (no venv setup needed)
uv run --with openai python quickstart.py
```

Example of an inline-dependency script (`uv run` reads the header and auto-installs `openai` into an ephemeral env):
```python
# quickstart.py
# /// script
# dependencies = ["openai"]
# ///
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["EURI_API_KEY"],
    base_url="https://api.euron.one/api/v1/euri"
)
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Say hello in 3 languages."}]
)
print(response.choices[0].message.content)
```
Run it with:
```bash
uv run quickstart.py
```
No manual `pip install`, no venv activation — `uv run` handles the environment for you based on the inline `dependencies` block.

Store your key as an environment variable rather than hardcoding it:
```bash
export EURI_API_KEY="your_key_here"        # macOS/Linux
setx EURI_API_KEY "your_key_here"           # Windows
```

### Step 3 — Make your first call
The simplest path is the **OpenAI Python SDK**, just pointed at EURI's base URL:

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["EURI_API_KEY"],
    base_url="https://api.euron.one/api/v1/euri"
)

response = client.chat.completions.create(
    model="gpt-4o-mini",       # free-tier / non-premium model
    messages=[{"role": "user", "content": "Say hello in 3 languages."}],
    max_tokens=150
)

print(response.choices[0].message.content)
```

That's the entire "quickstart" — same OpenAI SDK you already know, only `base_url` and `api_key` changed.

### Step 4 (optional) — Raw HTTP / curl, no SDK at all
```bash
curl -X POST https://api.euron.one/api/v1/euri/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $EURI_API_KEY" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Say hello in 3 languages."}],
    "max_tokens": 150
  }'
```

### Step 5 (optional) — Native `euriai` Python SDK
```python
from euriai import EuriaiClient

client = EuriaiClient(api_key="YOUR_EURI_API_KEY")

response = client.chat.completions.create(
    model="gpt-4.1-nano",
    messages=[{"role": "user", "content": "Say hello in 3 languages."}],
    max_tokens=150
)
print(response.choices[0].message.content)
```
The `euriai` package also ships optional integrations (`pip install euriai[langchain]`, `[crewai]`, `[autogen]`, `[langgraph]`, `[smolagents]`, `[llama-index]`) if you want to plug EURI into agent frameworks.

---

## 2. Authentication (elaborated)

**How it works:** EURI uses simple **Bearer token** auth — same scheme as OpenAI. Every request must carry:
```
Authorization: Bearer YOUR_EURI_API_KEY
```

**Getting a key:**
1. Sign in at [euron.one/euri](https://euron.one/euri).
2. **Billing & API Keys** → **Create API Key**.
3. Copy and store it immediately — it will not be shown again.

**Using it in code — 3 equivalent ways:**

```python
# 1) OpenAI SDK — key passed at client construction
from openai import OpenAI
client = OpenAI(api_key="YOUR_EURI_API_KEY", base_url="https://api.euron.one/api/v1/euri")
```

```python
# 2) OpenAI SDK — key read from an env var (cleanest for real projects)
import os
from openai import OpenAI
client = OpenAI(
    api_key=os.environ.get("EURI_API_KEY"),
    base_url="https://api.euron.one/api/v1/euri"
)
```

```python
# 3) Plain HTTP with `requests` — full manual control
import os, requests

url = "https://api.euron.one/api/v1/euri/chat/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {os.environ['EURI_API_KEY']}"
}
payload = {
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
}

r = requests.post(url, headers=headers, json=payload)
r.raise_for_status()
print(r.json()["choices"][0]["message"]["content"])
```

**Security best practices:**
- Never hardcode the key in client-side / browser / mobile code — server-side only.
- Store in env vars or a secrets manager (`.env` + `python-dotenv`, GitHub Actions secrets, etc.).
- Use **separate keys** for dev and prod so you can revoke one without breaking the other.
- Rotate keys periodically from the dashboard.

**Handling auth errors in code:**
```python
from openai import OpenAI, AuthenticationError

client = OpenAI(api_key="bad-key", base_url="https://api.euron.one/api/v1/euri")

try:
    client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Hi"}]
    )
except AuthenticationError as e:
    print("Auth failed:", e)
```
A missing/invalid key returns:
```json
{
  "error": {
    "message": "Invalid API key provided.",
    "type": "authentication_error",
    "code": "invalid_api_key"
  }
}
```

---

## 2a. Which models are free?

Important distinction EURI makes:

- **Truly $0 / free model:** `sarvam-m` — priced at $0 input and $0 output. This one costs nothing no matter how much you use it.
- **"Non-premium" models:** everything else below. These are **not free forever** — they have real per-token prices — but EURI covers them under your **daily free token quota** (10,000 tokens/day free users, 100,000 tokens/day Euron Plus users). As long as you stay under that quota, you pay nothing and don't need wallet balance. Go over the quota and requests are blocked (`403 token_limit_exceeded`) until it resets — they do **not** silently start billing your wallet.
- **"Premium" models** (Claude family, GPT-5 family, o3/o4-mini, Gemini 3.x, etc.) always draw from wallet balance, regardless of your daily quota — the free tier never applies to them.

### Free-tier-eligible text generation models (`premium: false`)

| Model | Provider | Context | Notes |
|---|---|---|---|
| `sarvam-m` | Sarvam AI | 8K | **Actually $0 — no quota needed at all** |
| `gpt-4o-mini` | OpenAI | 128K | Fast, cheap, multimodal |
| `gpt-4.1-nano` | OpenAI | 128K | Fastest/cheapest GPT-4.1 |
| `gpt-4.1-mini` | OpenAI | 128K | Balanced quality/cost |
| `gpt-4o` | OpenAI | 128K | Multimodal, vision |
| `openai/gpt-oss-20b` | OpenAI | 128K | Open-source 20B |
| `openai/gpt-oss-120b` | OpenAI | 128K | Open-source 120B |
| `gemini-2.0-flash` | Google | 1M | Fast, large context |
| `gemini-2.5-flash` | Google | 1M | Latest flash model |
| `gemini-2.5-flash-lite-preview-06-17` | Google | 1M | Ultra-light |
| `gemini-2.5-pro` | Google | 1M | High quality, still free-tier |
| `gemini-2.5-pro-preview-06-05` | Google | 1M | Preview version |
| `llama-4-scout-17b-16e-instruct` | Meta/Groq | 128K | Fast open-source |
| `llama-3.3-70b-versatile` | Meta/Groq | 128K | Versatile tasks |
| `llama-3.1-8b-instant` | Meta/Groq | 128K | Ultra-fast |
| `qwen/qwen3-32b` | Qwen | 128K | Multilingual |
| `groq/compound` | Groq | 128K | Agentic, tool use |
| `groq/compound-mini` | Groq | 128K | Light agentic |

### Free-tier-eligible embedding models (`premium: false`)
| Model | Provider |
|---|---|
| `text-embedding-3-small` | OpenAI |
| `togethercomputer/m2-bert-80M-32k-retrieval` | Together |
| `gemini-embedding-001` | Google |

### Free-tier-eligible image generation model
| Model | Provider |
|---|---|
| `gemini-3-pro-image-preview` | Google |

### Free-tier-eligible text-to-speech models
| Model | Provider |
|---|---|
| `canopylabs/orpheus-v1-english` | Canopy Labs |
| `canopylabs/orpheus-arabic-saudi` | Canopy Labs |
| `playai-tts` | PlayAI |
| `playai-tts-arabic` | PlayAI |

Note: all **speech-to-text** models (`whisper-large-v3`, `whisper-large-v3-turbo`, `sarvam-stt`) are premium — none are free-tier eligible.

**Full working example — free model, no wallet required:**
```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["EURI_API_KEY"],
    base_url="https://api.euron.one/api/v1/euri"
)

def ask(prompt: str, model: str = "gpt-4o-mini") -> str:
    resp = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": "You are a concise, helpful assistant."},
            {"role": "user", "content": prompt}
        ],
        max_tokens=300,
        temperature=0.7
    )
    return resp.choices[0].message.content

print(ask("Explain photosynthesis in 2 sentences."))
print(ask("Write a haiku about the ocean.", model="gemini-2.0-flash"))
print(ask("What's 15% of 240?", model="llama-3.1-8b-instant"))
```

**Streaming response with a free model:**
```python
stream = client.chat.completions.create(
    model="gemini-2.5-flash",
    messages=[{"role": "user", "content": "Count from 1 to 10 slowly."}],
    stream=True
)

for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

⚠️ Once you exceed 10,000 tokens/day on free models, you'll get a `403` with `code: "token_limit_exceeded"`. Either wait for the daily UTC reset, upgrade to Euron Plus, or switch to a premium model backed by wallet balance.

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
