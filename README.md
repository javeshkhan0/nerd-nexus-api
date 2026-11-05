# Nerd Nexus — Master API

**One OpenAI-compatible API key for every AI you are signed in to.**

Nerd Nexus is a small local server that turns your signed-in AI **web accounts**
(Muse, Google Flow, Gemini, ChatGPT, Claude, Grok) **plus** your purchased API keys and
your local agent CLIs (Claude Code, Codex, Antigravity) into **one OpenAI-compatible API
key**. Any app, script, bot, agent, or webhook that can talk to an OpenAI-style endpoint
can use all of them through a single key — you point the client at the Nerd Nexus base URL
and pick which AI with the `model` field.

> **For AI agents reading this:** this is the integration reference. To call the API you need
> (1) the **base URL** of the running server, and (2) a **Bearer key** that starts with `nn_`.
> The human operator gives you both. Everything is OpenAI-compatible: use any OpenAI SDK or a
> plain HTTP `POST`, set the base URL + key, and choose a model from the tables below.

---

## 1. Connection

| | |
|---|---|
| **Base URL** | `http://127.0.0.1:18610/v1` (or the operator's LAN/host address) |
| **Auth header** | `Authorization: Bearer nn_xxxxxxxx…` |
| **Format** | OpenAI-compatible (`/v1/chat/completions`, `/v1/images/generations`, `/v1/videos`, `/v1/models`) |

- The API key is shown (and can be rotated) in the operator's dashboard — it starts with `nn_`.
- `127.0.0.1` always means *the machine running the client*. To call the server from another
  machine, use the server's LAN IP/host (the server must be configured to allow that host).
- A route only works if the operator has at least one **eligible account** for that model's
  provider. `GET /v1/models` returns the live, currently-usable list.

---

## 2. Models

Route to a provider by setting `model`. Call `GET /v1/models` for the live list.

### Chat / coding — `POST /v1/chat/completions`
| Model | Source |
|---|---|
| `muse-spark` | Muse.ai web account |
| `gemini-chat` | Google Gemini web account |
| `chatgpt-chat` | ChatGPT web account |
| `claude-chat` | Claude.ai web account |
| `grok-chat` | Grok web account |
| `api-claude` | Operator's **purchased** Anthropic API key (direct passthrough) |
| `api-codex` | Operator's **purchased** OpenAI/Codex API key (direct passthrough) |
| `claude-code` | Local **Claude Code** CLI (signed in on the server machine) |
| `codex-cli` | Local **Codex** CLI |
| `antigravity` | Local **Antigravity** CLI |

### Images — `POST /v1/images/generations`
`muse-image`, `flow-image`, `gemini-image`, `chatgpt-image`, `grok-image`

### Video — `POST /v1/videos`, then poll `GET /v1/videos/{id}`
`muse-video`, `flow-video`

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

## 4. From your app or agent (OpenAI SDKs)

Because the endpoint is OpenAI-compatible, you change only **two settings** in any OpenAI
client: the **base URL** and the **API key**.

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

Anything that accepts an **OpenAI base URL + key** works the same way: LangChain, LlamaIndex,
n8n / Make / Zapier "OpenAI" nodes, Cline, Roo Code, Continue, aider (`--openai-api-base`),
Open WebUI, or your own agent/webhook server.

---

## 5. Images and video

```bash
# Image (synchronous — the connection stays open until the image is ready)
curl http://127.0.0.1:18610/v1/images/generations \
  -H "Authorization: Bearer nn_YOUR_KEY" -H "Content-Type: application/json" \
  --data-binary '{"model":"flow-image","prompt":"a blue ceramic cup by a window, no text","aspect_ratio":"9:16","response_format":"url","timeout":600}'

# Video (async — POST returns a task id, then poll)
curl http://127.0.0.1:18610/v1/videos \
  -H "Authorization: Bearer nn_YOUR_KEY" -H "Content-Type: application/json" \
  --data-binary '{"model":"muse-video","prompt":"clouds drift over a lake, no speech or text","aspect_ratio":"16:9","duration":10}'
# -> {"id":"task_…"} ; then GET /v1/videos/task_… until status is completed, and download the url.
```

A returned `url` may be a relative `/v1/media/…` path — join it to the **server root** (don't
add another `/v1`).

### Aspect ratios

Set `aspect_ratio` (or `size`) on any image/video request — **default `16:9`**.

| Provider | Image ratios | Video ratios |
|---|---|---|
| **Muse** (`muse-image` / `muse-video`) | `1:1`, `16:9`, `9:16`, `4:3`, `3:4` (others approximated) | `16:9`, `9:16`, `1:1` |
| **Flow** (`flow-image` / `flow-video`) | `16:9`, `9:16` only | `16:9`, `9:16` only |
| **Gemini / ChatGPT / Grok** (`*-image`) | `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3` | — |

- **Flow** natively offers only `16:9` and `9:16`; any other ratio is mapped to the nearest
  (portrait → `9:16`, otherwise → `16:9`). For `1:1`, `4:3`, etc. use `muse-image`.
- **Flow** outputs are always **720p** and a **single** result; for `flow-video`, `duration`
  snaps to the nearest of `4 / 6 / 8 / 10` seconds.
- Muse / Gemini / ChatGPT / Grok apply the ratio through the prompt, so exact pixel sizes
  vary but the orientation is always honored (e.g. Muse `16:9` ≈ 2048×1152, `9:16` ≈
  1152×2048, `1:1` ≈ 1600×1600).

---

## 6. Claude Code / Anthropic-format clients

The Master API speaks **OpenAI** format (`/v1/chat/completions`). The official **Claude Code**
CLI and the Anthropic SDK speak **Anthropic** format (`/v1/messages`, `ANTHROPIC_BASE_URL`),
which is a different shape, so pointing `ANTHROPIC_BASE_URL` here does **not** work as-is.

Working options:
1. **Use an OpenAI-compatible agent** (aider, Cline, Roo Code, Continue, Open WebUI…): set the
   base URL + `nn_` key and pick a `model`. ✅ recommended.
2. **Route Claude Code / Codex / Antigravity as a *source*** (they run on the server machine):
   call them from any OpenAI client with `model: "claude-code"` / `"codex-cli"` / `"antigravity"`.
3. **(Optional) An Anthropic `/v1/messages` shim** can be added so the real Claude Code CLI uses
   the Master key directly (an Anthropic ⇄ OpenAI translation layer).

---

## 7. Security

- The key authorizes **all** connected accounts — treat it like a password. Rotate it from the
  dashboard if it leaks (the old key stops working immediately).
- Never put the key in a URL/query string or commit it to a repo; use the `Authorization`
  header or an environment variable.
- Keep the server bound to `127.0.0.1` unless you intentionally expose it on a trusted LAN.

---

*This repository contains documentation only — no code, no keys, no account data.*
