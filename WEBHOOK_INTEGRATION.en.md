# Nerd Nexus — Master API integration

Nerd Nexus turns your signed-in AI web accounts (Muse, Google Flow, Gemini, ChatGPT,
Claude, Grok) **plus** your purchased API keys and local agent CLIs into **one
OpenAI-compatible API key**. Any app, script, bot, or webhook that can talk to an
OpenAI-style endpoint can use all of them through a single key — you point the client at
the Nerd Nexus base URL and pick a provider with the `model` field.

> The dashboard's **Integration guide** tab has the full, always-up-to-date reference with
> copy-paste cURL / Python for every image and video parameter. This file is the short
> "how do I plug my app in" version.

---

## 1. Connection

| | |
|---|---|
| **Base URL** | `http://127.0.0.1:18610/v1` |
| **Auth header** | `Authorization: Bearer nn_xxxxxxxx…` |
| **Format** | OpenAI-compatible (`/v1/chat/completions`, `/v1/images/generations`, `/v1/videos`, `/v1/models`) |

- The API key is shown (and can be rotated) in the dashboard — it starts with `nn_`.
- Use the machine's LAN IP instead of `127.0.0.1` to call it from another computer (the
  server must be configured to allow that host). `127.0.0.1` always means *the machine
  running the client*, not the server.
- If API auth is turned off in `.env`, use the literal key `off`.
- A route only works if at least one **eligible account** exists for that model's provider.

---

## 2. Models

Route to a provider by setting `model`. `GET /v1/models` returns the live list.

### Chat / coding (`POST /v1/chat/completions`)
| Model | Source |
|---|---|
| `muse-spark` | Muse.ai web account |
| `gemini-chat` | Google Gemini web account |
| `chatgpt-chat` | ChatGPT web account |
| `claude-chat` | Claude.ai web account |
| `grok-chat` | Grok web account |
| `api-claude` | Your **purchased** Anthropic API key (direct passthrough) |
| `api-codex` | Your **purchased** OpenAI/Codex API key (direct passthrough) |
| `claude-code` | Local **Claude Code** CLI (already signed in on this machine) |
| `codex-cli` | Local **Codex** CLI |
| `antigravity` | Local **Antigravity** CLI |

### Images (`POST /v1/images/generations`)
`muse-image`, `flow-image` (Nano Banana Pro), `flow-image-lite` (Nano Banana 2 Lite),
`flow-image-2` (Nano Banana 2.1), `gemini-image`, `chatgpt-image`, `grok-image`

### Video (`POST /v1/videos`, then poll `GET /v1/videos/{id}`)
`muse-video`, `flow-video` (Omni 1.1 Flash), `flow-video-fast`, `flow-video-quality`,
`flow-video-lite` (Veo 3.1)

> **CLI models** (`claude-code`, `codex-cli`, `antigravity`) must be turned on in the
> dashboard's **LLM Assistant** tab (`claude-code` is **off by default**). `GET /v1/models`
> always reflects the live, usable set.

---

## 3. Chat — quickest test (cURL)

```bash
curl http://127.0.0.1:18610/v1/chat/completions \
  -H "Authorization: Bearer nn_YOUR_KEY" \
  -H "Content-Type: application/json" \
  --data-binary '{
    "model": "claude-chat",
    "messages": [{"role": "user", "content": "In one word, the capital of France?"}],
    "stream": false
  }'
```

Set `"stream": true` for an SSE token stream (same shape as OpenAI).

---

## 4. From your app (OpenAI SDKs — the usual webhook case)

Because the endpoint is OpenAI-compatible, you change only **two settings** in any
OpenAI client: the base URL and the API key.

**Python (`openai`):**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:18610/v1",
    api_key="nn_YOUR_KEY",
)

resp = client.chat.completions.create(
    model="gemini-chat",                       # or muse-spark, claude-chat, api-claude, claude-code…
    messages=[{"role": "user", "content": "Write a 2-line video hook."}],
)
print(resp.choices[0].message.content)
```

**Node (`openai`):**
```js
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "http://127.0.0.1:18610/v1",
  apiKey: "nn_YOUR_KEY",
});

const r = await client.chat.completions.create({
  model: "chatgpt-chat",
  messages: [{ role: "user", content: "Give me 3 thumbnail ideas." }],
});
console.log(r.choices[0].message.content);
```

Anything that accepts an **OpenAI base URL + key** works the same way: LangChain,
LlamaIndex, n8n / Make / Zapier "OpenAI" nodes, Cline, Roo Code, aider (`--openai-api-base`),
Open WebUI, SillyTavern, your own webhook server, etc.

---

## 5. Images and video (summary)

```bash
# Image (synchronous — connection stays open until the image is ready)
curl http://127.0.0.1:18610/v1/images/generations \
  -H "Authorization: Bearer nn_YOUR_KEY" -H "Content-Type: application/json" \
  --data-binary '{"model":"flow-image","prompt":"a blue ceramic cup by a window, no text","aspect_ratio":"9:16","response_format":"url","timeout":600}'

# Video (async — POST returns a task id, then poll)
curl http://127.0.0.1:18610/v1/videos \
  -H "Authorization: Bearer nn_YOUR_KEY" -H "Content-Type: application/json" \
  --data-binary '{"model":"muse-video","prompt":"clouds drift over a lake, no speech or text","aspect_ratio":"16:9","duration":10}'
# -> {"id":"task_…"} ; then: GET /v1/videos/task_… until status completed, download the url.
```

The response `url` may be a relative `/v1/media/…` path — join it to the server root (don't
add another `/v1`). Full parameter tables and a ready-made `nerdnexus_media.py` are in the
dashboard's **Integration guide** tab.

### Aspect ratios

Set `aspect_ratio` (or `size`) on any image/video request — **default `16:9`**.

| Provider | Image ratios | Video ratios |
|---|---|---|
| **Muse** (`muse-image` / `muse-video`) | `1:1`, `16:9`, `9:16`, `4:3`, `3:4` (others approximated) | `16:9`, `9:16`, `1:1` |
| **Flow** (`flow-image` / `flow-video`) | `16:9`, `9:16` only | `16:9`, `9:16` only |
| **Gemini / ChatGPT / Grok** (`*-image`) | `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3` | — |

- **Flow** natively offers only `16:9` and `9:16`; any other ratio is mapped to the nearest
  (portrait → `9:16`, otherwise → `16:9`). For `1:1`, `4:3`, etc. use `muse-image`.
- **Flow** outputs are always **720p** and a **single** result. Choose the Flow model with the
  `model` id: video — `flow-video` (Omni, variable `duration` 4/6/8/10s), `flow-video-fast`,
  `flow-video-quality`, `flow-video-lite` (Veo 3.1; ~8s fixed length); image — `flow-image`
  (Nano Banana Pro), `flow-image-lite` (Nano Banana 2 Lite), `flow-image-2` (Nano Banana 2.1).
- Muse / Gemini / ChatGPT / Grok apply the ratio through the prompt, so exact pixel sizes
  vary but the orientation is always honored (e.g. Muse `16:9` ≈ 2048×1152, `9:16` ≈
  1152×2048, `1:1` ≈ 1600×1600).

### Image-to-video & reference image

Add `"reference_image"` to a request — a `data:image/...;base64,…` URI, or an image URL the
server can reach:
- On **`muse-video`** it animates that image as the first frame (image-to-video).
- On **`/v1/images/generations`** (Muse / Gemini / ChatGPT / Grok) it is used as a style / edit
  reference. (`/v1/images/edits` also accepts a multipart `image` file.)

Flow does not use a reference image.

---

## 6. Concurrency & limits

**None of these are hard caps — they are operator settings you can raise, and any request over
the current limit *queues* (it is never rejected while workers exist).** The gateway runs
browser **workers** shared across the web providers (Muse, Flow, Gemini, ChatGPT, Claude, Grok).

Three settings (dashboard → **Settings**) decide how much runs at once:

| Setting | Default | Max | What it controls |
|---|---|---|---|
| **Concurrent workers** | 8 | 32 | Total jobs running in parallel across all web providers. |
| **AI models at a time** | 2 | 8 | How many *different* AIs run simultaneously (bounds RAM). |
| **Accounts per provider** | — | — | Each account runs **one** job at a time. |

**How they combine (read this to avoid surprises):**
- One provider's parallelism = **`min(its accounts, Concurrent workers)`**. So **Muse with 1
  account runs 1 job at a time even if workers = 8** — raising workers alone does **not** speed
  up a single-account provider. To run that provider's jobs in parallel, **add more accounts for
  it** (e.g. more Muse accounts). With 4 Muse accounts and workers ≥ 4, up to 4 Muse jobs run at
  once.
- Across providers, at most **"AI models at a time"** different AIs run together; a further
  provider's jobs wait for a free slot.
- Over the active limit, requests **queue** (task status `queued` — not an error) until a worker
  frees, up to the queue timeout (default 1 hour); only then a `503`. So an agent can safely fire
  many requests — they drain in batches, nothing is lost.

**To get more throughput** (all in dashboard → Settings, mostly live/no restart):
1. **Raise Concurrent workers** (up to 32) → more jobs in parallel. Each worker ≈ one Chrome
   window, so it costs RAM — set it to what the machine can handle.
2. **Raise AI models at a time** (up to 8) → run more *different* AIs together.
3. **Add more accounts** for any provider you want to parallelise (one account = one concurrent
   job for that provider).

**Not bound by the worker pool at all:**
- **Direct-API** models (`api-claude`, `api-codex`) go straight to your purchased upstream key —
  limited only by that upstream's own rate limits.
- **Local CLI** models (`claude-code`, `codex-cli`, `antigravity`) run local processes.

Images are **synchronous** (the HTTP call returns the result); videos are **async** (POST returns
a task id, then poll), so long jobs never hold a connection open.

---

## 7. Claude Code / Anthropic-format clients — read this

**Short answer: not directly today.** The Nerd Nexus Master API speaks **OpenAI** format
(`/v1/chat/completions`). The official **Claude Code** CLI and the Anthropic SDK speak
**Anthropic** format (`/v1/messages`, `ANTHROPIC_BASE_URL`), which is a different
request/response shape. Pointing `ANTHROPIC_BASE_URL` at Nerd Nexus will **not** work as-is.

You have three working options:

1. **Use an OpenAI-compatible coding tool instead** (recommended now). Tools like
   **aider**, **Cline**, **Roo Code**, **Continue**, **Open WebUI** let you set a custom
   OpenAI base URL + key. Point them at `http://127.0.0.1:18610/v1`, key `nn_…`, and pick a
   `model` (`api-claude`, `claude-chat`, `claude-code`, `codex-cli`, `antigravity`, …).
2. **Route Claude Code / Codex through Nerd Nexus as a *source***, not a *client*: add them
   under the dashboard's **LLM Assistant** tab (they're already signed in on this machine),
   then call them from any OpenAI client with `model: "claude-code"` / `"codex-cli"`. This
   is the reverse direction and already works.
3. **(Planned) An Anthropic `/v1/messages` shim** so the real Claude Code CLI and Anthropic
   SDK can use the Master key directly. This is a translation layer (Anthropic ⇄ OpenAI,
   including SSE streaming) — tracked in `ISSUES.md` as a follow-up. Say the word and it can
   be built.

---

## 8. Security notes

- The key authorizes **all** your connected accounts — treat it like a password. Rotate it
  from the dashboard if it leaks (old key 401s immediately).
- Never put the key in a URL/query string or commit it to a repo; use the `Authorization`
  header or an environment variable.
- Keep the server bound to `127.0.0.1` unless you intentionally expose it on a trusted LAN.
