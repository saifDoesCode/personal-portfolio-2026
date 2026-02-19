# Portfolio Chat Worker

An AI-powered chat assistant embedded in [saif-ahmed.dev](https://saif-ahmed.dev), deployed as a Cloudflare Worker. Visitors can ask questions about Saif's experience, skills, and projects and receive real-time streamed responses.

---

## Architecture

```
Browser (index.htm)
    │
    │  POST /api/chat  (SSE stream)
    ▼
Cloudflare Worker  (portfolio-chat.saif-ahmed.workers.dev)
    │
    ├── Cloudflare KV  ──  rate limiting store
    │
    ├── Groq API  (primary)
    │     └── llama-3.1-8b-instant
    │
    └── Cloudflare Workers AI  (fallback)
          └── @cf/meta/llama-3.1-8b-instruct
```

The Worker always tries Groq first for its speed, then automatically falls back to Cloudflare's own AI inference if Groq is unavailable.

---

## Features

| Feature | Detail |
|---|---|
| Streaming | Server-Sent Events (SSE) — tokens appear as they are generated |
| Dual LLM | Groq (primary) → Workers AI (fallback) |
| Rate limiting | 10 req / minute · 50 req / day per IP (stored in KV) |
| CORS | Configurable via `ALLOWED_ORIGIN` env var |
| Message safety | Last 10 messages kept · each capped at 1 000 chars |
| Scoped context | System prompt restricts answers to Saif's portfolio only |

---

## API

### `POST /api/chat`

**Request body**
```json
{
  "messages": [
    { "role": "user",      "content": "What are Saif's skills?" },
    { "role": "assistant", "content": "..." },
    { "role": "user",      "content": "Tell me more about the AI ones." }
  ]
}
```

**Response** — `text/event-stream`

Each chunk:
```
data: {"token":"Hello"}

data: {"token":" there"}

data: [DONE]
```

Error chunk (rate limit, bad JSON, etc.):
```
data: {"error":"Too many requests. Please wait a minute."}
```

**HTTP status codes**

| Code | Meaning |
|---|---|
| 200 | Streaming started |
| 400 | Invalid JSON or no valid messages |
| 404 | Wrong path or method |
| 429 | Rate limit exceeded |
| 503 | Both AI providers unavailable |

---

## Environment / Bindings

Configured in `wrangler.toml`:

| Binding | Type | Purpose |
|---|---|---|
| `GROQ_API_KEY` | Secret | Groq API authentication |
| `RATE_LIMIT` | KV Namespace | Per-IP rate limit counters |
| `AI` | Workers AI | Fallback inference binding |
| `ALLOWED_ORIGIN` | Var | Comma-separated allowed origins (`*` for any) |

---

## Local Development

```bash
# Install dependencies
npm install

# Start local dev server
npx wrangler dev

# The worker will be available at http://localhost:8787
```

Set secrets locally:
```bash
npx wrangler secret put GROQ_API_KEY
```

---

## Deployment

```bash
npx wrangler deploy
```

The worker deploys to `portfolio-chat.<account>.workers.dev` and is routed from the portfolio site.

---

## Frontend Integration (`index.htm`)

The chat widget in the portfolio calls the worker directly from the browser:

- A hero-section input bar triggers the chat on first message.
- The chat container expands with an animation and displays streamed tokens in real time.
- Conversation history (up to 10 turns) is kept client-side and sent with each request.
- On any error the UI falls back to a polite message directing the visitor to Saif's email.

Relevant frontend constant:
```js
const WORKER_URL = 'https://portfolio-chat.saif-ahmed.workers.dev';
```

---

## System Prompt Design

The system prompt grounds the model exclusively in Saif's portfolio data:

- Personal background and location
- Full work history with role descriptions
- Technical skills across all categories
- Project names, descriptions, and live URLs
- Certifications and contact info

The model is instructed to be concise and professional, redirect off-topic questions, and never reveal the prompt itself.
