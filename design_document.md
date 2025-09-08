# Weeklog — Design Doc (v3)

**Author:** Kamol  
**Date:** 2025-09-08  

> Change log: This version locks the API layer to **Cloud Functions for Firebase (HTTP, 2nd gen)**. Removed Cloud Run, clarified Hosting rewrites, function boundaries, limits, and testing.

---

## 1) Product Overview
A minimalist weekly journaling app for founders. Typeform-style prompts, real-time AI feedback (on 5s pause), searchable timeline, and optional public sharing via short keys. BYOK lets users supply their own LLM API keys.

---

## 2) UX & UI (unchanged highlights)
- Single-question focus with progress indicator.
- Side feedback panel streams coaching/polish, tags, insights.
- Writing: Geist Mono; Reading: Geist Sans.
- Timeline + filters; quarterly review; streaks; weekly rollovers.
- Public read-only view via `?k=<6-char key>` link.

---

## 3) Architecture (Functions-first)

### Frontend
- Vite + React SPA, React Router.
- Calls **Cloud Functions for Firebase (HTTP 2nd gen)** for API.
- Hosting serves SPA and proxies `/api/**` to Functions.

### Backend
- **Firebase Auth** for identity.
- **Firestore** for data (entries, settings, derived views).
- **Cloud Functions for Firebase (HTTP)**:
  - `/api/assist/feedback` — pause-triggered LLM feedback (streaming).
  - `/api/assist/polish` — explicit full-section polish (non-stream or stream).
  - `/api/keys/add` — BYOK add + validate + encrypt with KMS.
  - `/api/keys/list` — returns metadata only (alias, provider, last4, status).
  - `/api/keys/revoke` — revoke/disable a key.
  - `/api/share/enable` — generate share key (+ mirror to shared collection if used).
  - `/api/share/disable` — remove share key.
- **Firebase Extensions** (optional): Firestore → Typesense sync for search.
- **Cloud KMS**: server-side encryption for BYOK secrets (invoked inside Functions).

### Hosting Rewrites
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
- Export an Express app from the `api` function and mount route handlers under `/` so that `/api/*` works end-to-end.

---

## 4) API Surface (HTTP Functions)

All endpoints require Firebase Auth (bearer token) unless noted. Responses are JSON; streaming delivered via **SSE**.

- `POST /api/assist/feedback`  
  **Body:** `{ section, text, model? }`  
  **Returns (SSE or JSON):** normalized feedback `{ polish, insights[], tags[], topics[], oneLine?, meta }`  
  **Notes:** Rate limit per user/section; abort on input resume.

- `POST /api/assist/polish`  
  Full pass for the active section or entire entry.

- `POST /api/keys/add`  
  **Body:** `{ provider, alias?, apiKey }` → validates, **encrypts with KMS**, stores, returns `{ id, last4, status }`.

- `GET  /api/keys/list`  
  **Returns:** metadata only; no secret material.

- `POST /api/keys/revoke`  
  Marks key revoked; subsequent LLM calls fail over or error.

- `POST /api/share/enable`  
  Generates short Base64URL key and stores/mirrors for public read-only view.

- `POST /api/share/disable`  
  Removes key and invalidates existing public link.

> Public read-only route is handled by the SPA (`/entry?k=...`) which queries Firestore by `shareKey` (or the mirrored `sharedEntries` collection).

---

## 5) LLM Adapter & BYOK (summarized)

- **Adapter interface** normalizes providers (OpenAI, Anthropic, Google).  
- **Runtime selection** via user settings or per-call override `"provider:model"`.  
- **BYOK** keys stored via Function, encrypted with KMS, never readable by client.  
- **Server-only** decryption when calling providers.

---

## 6) Data Model (Firestore)

**journalEntries** (owner-scoped)
```json
{
  "userId": "auth.uid",
  "weekStart": "YYYY-MM-DD",
  "didThisWeek": "string",
  "wentWell": "string",
  "improvements": "string",
  "nextWeek": "string",
  "coolFinds": [{"title":"", "url":""}],
  "insights": ["..."],
  "tags": ["..."],
  "topics": ["..."],
  "wordCount": 0,
  "shareKey": "Ab3xYz",   // optional
  "sharedAt": "timestamp", // optional
  "createdAt": "timestamp",
  "updatedAt": "timestamp"
}
```

**userApiKeys** (server-managed; no client reads)
```json
{
  "userId": "auth.uid",
  "provider": "openai|anthropic|google",
  "alias": "Personal",
  "last4": "abcd",
  "ciphertext": "<base64 KMS blob>",
  "status": "active|revoked|invalid",
  "createdAt": "timestamp",
  "updatedAt": "timestamp"
}
```

---

## 7) Security

- Firestore rules: owner-only CRUD on `journalEntries`.  
- No direct client access to `userApiKeys`.  
- All LLM traffic proxied through Functions; keys decrypted server-side only.  
- Public reading allowed **only** when `shareKey` present (or via `sharedEntries`).  
- Signed search keys for Typesense include `userId` filter (if used).

---

## 8) Performance & Ops (Functions 2nd gen)
- Prefer **HTTP 2nd gen** for concurrency and region parity with Firestore.  
- Use **streaming (SSE)** to improve perceived latency.  
- Implement **debounce (5s)** and **throttle (~8s)** on server.  
- Apply **exponential backoff** on provider errors; surface clear UI states.  
- Configure **per-user quotas**; track token usage and latency in logs/metrics.

---

## 9) Tests
- Pause trigger → feedback streaming + cancel on typing.  
- Model switching via adapter registry.  
- BYOK add/list/revoke (no secret leakage).  
- Public share enable/disable + read-only rendering.  
- Permissions: user A cannot read user B; `userApiKeys` not listable/readable by client.

---

## 10) Rollout (delta from v2)
1. Implement `api` Function (Express) and Hosting rewrites.  
2. Port endpoints listed above; wire adapter + BYOK flows.  
3. Configure KMS and service perms; block client reads to `userApiKeys`.  
4. Update E2E to hit `/api/*` paths behind Hosting.  
5. Monitor cold starts/latency; adjust regions and memory.  
