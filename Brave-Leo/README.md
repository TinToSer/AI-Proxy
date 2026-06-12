# Brave Leo Proxy

An OpenAI-compatible HTTP proxy that routes requests to Brave Search's Ask AI
(`search.brave.com/ask`).  No Brave account, API key, or browser tokens are
required — authentication is handled transparently using Brave's own
server-signed request tokens.

**Implementation:** single compiled Go binary (`brave_proxy.exe`) — no runtime,
no interpreter, no dependencies.  Drop it on a server and run it as a daemon.

---

## Features

- **Drop-in OpenAI replacement** — works with any SDK or tool that targets the
  OpenAI API (`openai`, `liteLLM`, `Cursor`, `Open WebUI`, …)
- **Text & image (multimodal)** — pass images as base64 data URLs, remote URLs,
  or local file paths in the standard OpenAI content-list format
- **Streaming & non-streaming** — `"stream": true` returns proper SSE chunks;
  `"stream": false` returns a single JSON completion
- **Zero auth friction** — no accounts, no API keys on the Brave side; the
  server-side token is fetched automatically per request
- **Daemon-friendly** — silent by default; only upstream errors are logged to
  stderr.  The startup banner goes to stdout; everything else is quiet.
- **Robust** — exponential-backoff retry, locale auto-detection with fallback,
  connection pooling, image size validation, proper OpenAI-format error bodies
- **Single binary** — pure Go stdlib, no external packages, no CGO

---

## Quick start

```bash
./brave_proxy
```

The proxy listens on `http://127.0.0.1:9006` by default and prints a one-time
startup banner to stdout, then goes silent:

```
============================================================
  Brave Leo Proxy  v1.0.0  (Go)
  Listening : http://127.0.0.1:9006
  API base  : http://127.0.0.1:9006/v1
============================================================
```

After that the process produces **no output** unless an upstream error occurs,
making it safe to run as a background daemon or system service.

---

## Command-line options

| Flag | Default | Description |
|------|---------|-------------|
| `--host` | `127.0.0.1` | Bind address (`0.0.0.0` for LAN/Docker) |
| `--port` | `9006` | TCP port |
| `--api-key KEY` | *(none)* | Require `Authorization: Bearer KEY` on all requests |

```bash
# Expose on all interfaces, protect with a key
./brave_proxy --host 0.0.0.0 --port 9006 --api-key mysecret
```

---

## Running as a daemon

### systemd (Linux)

```ini
# /etc/systemd/system/brave-proxy.service
[Unit]
Description=Brave Leo Proxy
After=network-online.target

[Service]
ExecStart=/usr/local/bin/brave_proxy --host 127.0.0.1 --port 9006
Restart=on-failure
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now brave-proxy
journalctl -u brave-proxy -f          # tail errors only
```

### Windows Service (NSSM)

```bat
nssm install BraveProxy "C:\brave_proxy.exe" "--host 0.0.0.0 --port 9006"
nssm set BraveProxy AppStdout C:\logs\brave_proxy.log
nssm set BraveProxy AppStderr C:\logs\brave_proxy_err.log
nssm start BraveProxy
```

---

## API reference

### Base URL

```
http://localhost:9006/v1
```

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness check |
| `GET` | `/v1/models` | List available models |
| `POST` | `/v1/chat/completions` | Create a chat completion |

---

### `POST /v1/chat/completions`

#### Text completion

```bash
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "brave/leo",
    "messages": [{"role": "user", "content": "What is a black hole?"}]
  }'
```

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1749000000,
  "model": "brave/leo",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "A black hole is a region of spacetime where gravity..."
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 0,
    "completion_tokens": 42,
    "total_tokens": 42
  }
}
```

#### Streaming

```bash
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "brave/leo",
    "messages": [{"role": "user", "content": "Explain SSE"}],
    "stream": true
  }'
```

Each chunk:
```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":...,"model":"brave/leo","choices":[{"index":0,"delta":{"content":"Server"},"finish_reason":null}]}

data: [DONE]
```

#### Image input

Images are passed using the standard OpenAI content-list format.
The `url` field accepts:

| Scheme | Example |
|--------|---------|
| `data:` base64 | `data:image/jpeg;base64,/9j/4AAQ...` |
| Remote URL | `https://example.com/photo.jpg` |
| Local file | `file:///home/user/photo.jpg` |

```bash
B64=$(base64 -w0 photo.jpg)

curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"brave/leo\",
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"text\",      \"text\": \"What is in this image?\"},
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,$B64\"}}
      ]
    }]
  }"
```

---

## Python SDK

### With the official openai package

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:9006/v1",
    api_key="not-used",           # any non-empty string
)

# Text — non-streaming
resp = client.chat.completions.create(
    model="brave/leo",
    messages=[{"role": "user", "content": "What is Python?"}],
)
print(resp.choices[0].message.content)

# Text — streaming
stream = client.chat.completions.create(
    model="brave/leo",
    messages=[{"role": "user", "content": "Explain async/await"}],
    stream=True,
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)

# Image — non-streaming
import base64

with open("photo.jpg", "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

resp = client.chat.completions.create(
    model="brave/leo",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text",      "text": "Describe this image"},
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}},
        ],
    }],
)
print(resp.choices[0].message.content)
```

### With plain httpx (no SDK)

```python
import httpx, json

BASE = "http://localhost:9006/v1"

# Non-streaming
resp = httpx.post(f"{BASE}/chat/completions", json={
    "model": "brave/leo",
    "messages": [{"role": "user", "content": "Hello!"}],
})
print(resp.json()["choices"][0]["message"]["content"])

# Streaming
with httpx.stream("POST", f"{BASE}/chat/completions", json={
    "model": "brave/leo",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": True,
}) as r:
    for line in r.iter_lines():
        if line.startswith("data: ") and line != "data: [DONE]":
            chunk = json.loads(line[6:])
            print(chunk["choices"][0]["delta"].get("content", ""), end="", flush=True)
```

---

## JavaScript / Node.js

```js
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "http://localhost:9006/v1",
  apiKey:  "not-used",
});

// Text
const resp = await client.chat.completions.create({
  model: "brave/leo",
  messages: [{ role: "user", content: "What is JavaScript?" }],
});
console.log(resp.choices[0].message.content);

// Streaming
const stream = await client.chat.completions.create({
  model: "brave/leo",
  messages: [{ role: "user", content: "Explain closures" }],
  stream: true,
});
for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}

// Image (Node.js)
import { readFileSync } from "fs";
const b64 = readFileSync("photo.jpg").toString("base64");

const imgResp = await client.chat.completions.create({
  model: "brave/leo",
  messages: [{
    role: "user",
    content: [
      { type: "text",      text: "What is in this image?" },
      { type: "image_url", image_url: { url: `data:image/jpeg;base64,${b64}` } },
    ],
  }],
});
console.log(imgResp.choices[0].message.content);
```

---

## Using with LiteLLM

```python
import litellm

litellm.api_base = "http://localhost:9006/v1"

resp = litellm.completion(
    model="openai/brave/leo",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(resp.choices[0].message.content)
```

---



## Logging

The proxy is intentionally quiet:

| Event | Output |
|-------|--------|
| Startup | One-time banner to **stdout**, then silence |
| Normal requests | **Nothing** |
| Upstream error (5xx, rate limit, network) | Single line to **stderr** |
| Fatal startup failure (port in use, etc.) | Error to **stderr** + exit 1 |

Redirect stderr to a file to capture the error log:

```bash
./brave_proxy >> /var/log/brave_proxy.log 2>&1
```

---

## Environment

The proxy auto-detects the server's locale (country, language, geolocation)
from Brave's SvelteKit page data on the first request and caches it for the
process lifetime.

---

## Limitations

- **Single model** — only `brave/leo` is available
- **No conversation history** — each request is independent
- **Rate limits** — Brave may throttle heavy usage; the proxy returns `429`
- **Image size** — images larger than 40 KB are rejected before upload
- **Brave Search TOS** — usage is subject to Brave's terms of service

---

## Testing

```bash
# Terminal 1 — start the proxy
./brave_proxy

# Terminal 2 — run test suite (requires Python + httpx)
python test_proxy.py
python test_proxy.py --base-url http://localhost:9006 --api-key mykey
```
