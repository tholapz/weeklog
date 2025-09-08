# Weekly Journal App

A minimalist weekly journaling tool for founders. Structured prompts, real-time AI assistance, searchable timeline, and optional public sharing. **API is powered by Cloud Functions for Firebase (HTTP, 2nd gen).**

---

## ✨ Features
- Typeform-style **one-question-at-a-time** flow
- **Real-time AI feedback** after 5s pause (streamed to side panel)
- **Geist Mono** for writing, **Geist Sans** for reading
- Timeline, filters, tags, topics, and full‑text search (Typesense)
- **Public links** via short Base64URL keys (`?k=Ab3xYz`)
- **BYOK**: secure per-user provider keys (OpenAI/Anthropic/Google) with KMS
- **Runtime model switcher** via adapter registry

---

## 🏗 Architecture

**Frontend**
- Vite + React (SPA), React Router
- Calls **Cloud Functions for Firebase (HTTP)** under `/api/*`

**Backend**
- Firebase Auth
- Firestore (entries, settings)
- **Cloud Functions (HTTP 2nd gen)** for API:
  - `POST /api/assist/feedback` (SSE streaming on pause)
  - `POST /api/assist/polish`
  - `POST /api/keys/add`, `GET /api/keys/list`, `POST /api/keys/revoke`
  - `POST /api/share/enable`, `POST /api/share/disable`
- Cloud KMS for BYOK encryption
- Typesense extension for search (optional)

**Hosting**
- Firebase Hosting serves SPA and proxies `/api/**` to the `api` function.

**firebase.json (snippet)**
```json
{
  "hosting": {
    "rewrites": [
      { "source": "/api/**", "function": "api" },
      { "source": "!/@(api)/**", "destination": "/index.html" }
    ]
  }
}
```

---

## 🔌 LLM Adapter & Prompt

- Adapter interface normalizes providers and models; user can switch at runtime.
- JSON output contract: `{ polish, insights[], tags[], topics[], oneLine?, meta }`.
- See `docs/journal_prompt.md` for the system prompt and examples.

---

## 🔒 Security

- Owner-only access to `journalEntries` via Firestore rules.
- `userApiKeys` is **server-only** (no client reads/lists).
- LLM requests are proxied through Functions; keys decrypted via KMS.
- Public view is read-only and only enabled when `shareKey` exists.

---

## 🧪 E2E Checklist

- Pause → feedback streamed; typing cancels inflight call
- Model switch reflects immediately in responses
- BYOK add/list/revoke flows
- Public share enable/disable + read-only rendering
- Cross-user access is denied; `userApiKeys` is not readable

---

## 🚀 Getting Started (high level)

1. Initialize Firebase (Auth, Firestore, Functions, Hosting).  
2. Configure **Cloud Functions (2nd gen)** in your preferred region.  
3. Set up **Cloud KMS** key ring & grant Functions service account encrypt/decrypt.  
4. Deploy `api` function (Express) and add Hosting rewrites.  
5. Install Typesense extension (optional) and backfill index.  
6. Run Playwright E2E tests.
