# Gemini Proxy

A local proxy that exposes Gemini (gemini.google.com) as an OpenAI-compatible REST API.
Point any tool that speaks the OpenAI chat format at `http://localhost:9006` and it will talk to Gemini under the hood — no API key required, uses your Google account session.

## Quick start

### 1. Run the proxy

```
gemini-proxy.exe
```

On first launch a browser window opens so you can sign in to your Google account.
After sign-in the browser closes automatically and the proxy starts.

```
+--------------------------------------------------+
|                                                  |
|  Gemini Proxy (Go)                               |
|  Chat   : POST http://127.0.0.1:9006/v1/chat/completions  |
|  Models : GET  http://127.0.0.1:9006/v1/models  |
|  Health : GET  http://127.0.0.1:9006/health      |
|  Re-auth: POST http://127.0.0.1:9006/auth/login  |
|                                                  |
+--------------------------------------------------+
```

### 2. Test it is working

```
curl http://localhost:9006/health
```
```json
{"session": true, "status": "ok"}
```

---

## CLI flags

| Flag | Default | Description |
|------|---------|-------------|
| `-host` | `127.0.0.1` | Address to bind on |
| `-port` | `9006` | Port to listen on |
| `-browser` | `auto` | Browser for login: `auto`, `chrome`, `edge`, `firefox` |
| `-data-dir` | `~/.gemini_proxy` | Where to store cookies and session data |

Examples:

```
# Listen on all interfaces (so other machines on your network can reach it)
gemini-proxy.exe -host 0.0.0.0 -port 8080

# Force Chrome for login
gemini-proxy.exe -browser chrome

# Store session files next to the exe
gemini-proxy.exe -data-dir .
```

---

## API reference

### GET /health

Returns whether the session is ready.

```
curl http://localhost:9006/health
```
```json
{"status": "ok", "session": true}
```

---

### GET /v1/models

Lists available Gemini models.

```
curl http://localhost:9006/v1/models
```
```json
{
  "object": "list",
  "data": [
    {"id": "gemini-3.1-flash-lite", "name": "3.1 Flash-Lite", "object": "model", "owned_by": "google"},
    {"id": "gemini-3-flash",        "name": "3 Flash",        "object": "model", "owned_by": "google"},
    {"id": "gemini-pro",            "name": "Pro",            "object": "model", "owned_by": "google"},
    {"id": "gemini-web",                                       "object": "model", "owned_by": "google"},
    {"id": "gemini-2.5-flash",                                 "object": "model", "owned_by": "google"}
  ]
}
```

The model IDs listed above are what you pass in the `model` field of chat requests.
`gemini-web` always routes to whatever Gemini picks as its default.

---

### POST /v1/chat/completions

Standard OpenAI-format chat endpoint.

#### Plain chat (non-streaming)

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-web",
    "messages": [{"role": "user", "content": "What is the capital of France?"}],
    "stream": false
  }'
```
```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "model": "2.5 Flash",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "The capital of France is Paris."},
    "finish_reason": "stop"
  }]
}
```

> **Note:** The `model` field in the response shows the model Gemini actually used,
> which may differ from what you requested.

#### Streaming

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-web",
    "messages": [{"role": "user", "content": "Tell me a short joke."}],
    "stream": true
  }'
```

Returns Server-Sent Events (SSE) in OpenAI chunk format:

```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","model":"gemini-web","choices":[{"index":0,"delta":{"role":"assistant"}}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","model":"gemini-web","choices":[{"index":0,"delta":{"content":"Why"}}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","model":"gemini-web","choices":[{"index":0,"delta":{"content":" don't scientists trust atoms?"}}]}

data: [DONE]
```

#### Multi-turn conversation

Use the `X-Conversation-Id` header to keep context across multiple requests.
The first response includes `x_conversation_id` — pass that back in subsequent calls.

```
# First turn
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-web","messages":[{"role":"user","content":"My name is Alice."}],"stream":false}'

# Response includes: "x_conversation_id": "c_abc123..."

# Second turn — reference the same conversation
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-Conversation-Id: c_abc123..." \
  -d '{"model":"gemini-web","messages":[{"role":"user","content":"What is my name?"}],"stream":false}'
```

#### Temporary chat (no history saved)

Add the `-temp` suffix to the model name, or pass the `X-Temporary-Chat: true` header.
Gemini will not save this conversation to your account history.

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-web-temp",
    "messages": [{"role": "user", "content": "This chat is off the record."}],
    "stream": false
  }'
```

Or via header:

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-Temporary-Chat: true" \
  -d '{"model":"gemini-web","messages":[{"role":"user","content":"Sensitive question here"}],"stream":false}'
```

#### Image input

Pass images as base64 data URLs or as HTTP/HTTPS URLs.

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-web",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "What is in this image?"},
        {"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg"}}
      ]
    }],
    "stream": false
  }'
```

Base64 example (data URL):

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-web",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "Describe this image."},
        {"type": "image_url", "image_url": {"url": "data:image/png;base64,iVBORw0KGgo..."}}
      ]
    }],
    "stream": false
  }'
```

#### System prompt

```
curl http://localhost:9006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-web",
    "messages": [
      {"role": "system", "content": "You are a helpful pirate. Respond in pirate speak."},
      {"role": "user",   "content": "What time is it?"}
    ],
    "stream": false
  }'
```

---

### POST /auth/login

Re-authenticate without restarting the proxy. Opens a browser window.

```
curl -X POST http://localhost:9006/auth/login
```
```json
{"status": "ok", "message": "Session initialised."}
```

### POST /auth/logout

Clears the saved session. The proxy will require re-login on the next request.

```
curl -X POST http://localhost:9006/auth/logout
```
```json
{"status": "ok", "message": "Logged out."}
```

---

## Using with the OpenAI Python SDK

The proxy is fully compatible with the `openai` Python package — just point it at the local server.

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:9006/v1",
    api_key="unused",          # required by the SDK but ignored by the proxy
)

# Plain chat
response = client.chat.completions.create(
    model="gemini-web",
    messages=[{"role": "user", "content": "Explain quantum entanglement simply."}],
)
print(response.choices[0].message.content)

# Streaming
stream = client.chat.completions.create(
    model="gemini-web",
    messages=[{"role": "user", "content": "Write a haiku about Go."}],
    stream=True,
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

---

## Using with Open WebUI

Open WebUI can talk to any OpenAI-compatible endpoint.

1. In Open WebUI go to **Settings → Connections → OpenAI API**
2. Set the URL to `http://localhost:9006/v1`
3. Set the API key to anything (e.g. `unused`)
4. Click **Save** — models from `/v1/models` will appear in the model picker

---

## Session and cookie storage

By default the proxy stores cookies and session tokens in `~/.gemini_proxy/`:

```
~/.gemini_proxy/
  cookies.json          # Google session cookies
  session_tokens.json   # WIZ_global_data tokens (at / fsid / bl)
  browser_profile/      # Persistent Chromium profile for session init
```

Use `-data-dir` to change the location, for example to keep everything next to `gemini-proxy.exe`:

```
gemini-proxy.exe -data-dir .
```

Sessions typically last several weeks. When cookies expire the proxy detects the
failure on the next request and you can re-login via `POST /auth/login` or by restarting.

---

## Troubleshooting

**`session: false` in /health**
Run `curl -X POST http://localhost:9006/auth/login` to open a fresh login browser window.

**Image upload fails with error 1100**
The session cookies are stale for uploads. Open [gemini.google.com](https://gemini.google.com) in a real browser on the same network to clear any unusual-traffic check, then re-login via `/auth/login`.

**`StreamGenerate auth error`**
Cookies expired. Re-login: `curl -X POST http://localhost:9006/auth/login`.

**Browser doesn't open on login**
Try forcing a specific browser: `gemini-proxy.exe -browser chrome` or `-browser edge`.
