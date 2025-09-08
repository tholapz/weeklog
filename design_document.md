# Personal Weekly Journaling Tool — Design Doc (v2)

**Author:** Kamol  
**Date:** 2025-09-08  

> This update adds: (1) real-time LLM feedback triggered on pause (5s), (2) a runtime‑switchable LLM adapter, and (3) secure user API key storage (“BYOK”).

---

## 1) Real‑Time LLM Feedback (Pause‑Triggered)

### Interaction Design
- **Trigger:** When the user stops typing for **5 seconds**, send the current section to the assistant.
- **Non‑blocking:** UI never freezes; requests are cancelable if the user resumes typing.
- **Granularity:** Per‑section (the active prompt only) to keep latency & cost down.
- **Streaming UX:** Show tokens as they arrive; allow **Insert**, **Replace**, or **Dismiss**.
- **Privacy guardrails:** Never send sections marked as *private*; show a lock icon on those fields.

### Client Logic (pseudo‑TS)
```ts
let pauseTimer: number | undefined;
let inflight: AbortController | null = null;

function onInputChange(text: string) {
  setDraft(text);
  // reset pause timer
  if (pauseTimer) clearTimeout(pauseTimer);
  // cancel inflight request if user resumes typing
  if (inflight) { inflight.abort(); inflight = null; }
  pauseTimer = window.setTimeout(() => requestFeedback(text), 5000);
}

async function requestFeedback(text: string) {
  inflight = new AbortController();
  const res = await fetch('/api/assist/feedback', {
    method: 'POST',
    body: JSON.stringify({ section: currentSectionId, text }),
    signal: inflight.signal,
    headers: { 'Content-Type': 'application/json' }
  });
  // handle SSE or fetch stream; update side panel incrementally
}
```

### Server Endpoint Contract
`POST /api/assist/feedback`
```json
{
  "section": "didThisWeek",
  "text": "raw user text",
  "model": "auto"        // optional; defaults to user preference
}
```
**Response (streamed):**
```json
{
  "polish": "refined text...",
  "insights": ["bullet1","bullet2"],
  "tags": ["growth","ops"],
  "topics": ["pricing"],
  "meta": { "provider":"openai", "model":"gpt-4.1-mini", "latencyMs": 820 }
}
```

### Concurrency & Cost Controls
- **Debounce:** 5s inactivity (configurable).
- **Throttle:** Min interval **8s** between server calls per user/section.
- **Max tokens:** Cap input/output to prevent runaway costs.
- **Diffing:** Send only **changed** deltas (optional optimization).

---

## 2) LLM Adapter (Runtime‑Switchable)

### Goals
- Swap providers/models without touching UI/business logic.
- Normalize outputs into a single schema for the side panel.
- Support system + safety prompts, JSON output, and streaming.

### Interface
```ts
export type FeedbackInput = {
  text: string;
  section: string;
  userId: string;
  locale?: string;
  maxTokens?: number;
  temperature?: number;
  stream?: boolean;
};

export type FeedbackOutput = {
  polish: string;
  insights: string[];
  tags: string[];
  topics: string[];
  oneLine?: string;
  meta: { provider: string; model: string; usage?: any };
};

export interface LLMAdapter {
  name: string;
  supportsStreaming: boolean;
  feedback(input: FeedbackInput, ctx: { signal?: AbortSignal }): AsyncIterable<FeedbackOutput> | Promise<FeedbackOutput>;
}
```

### Registry (Provider Catalog)
```ts
export const ModelRegistry = {
  openai: {
    defaultModel: "gpt-4.1-mini",
    models: ["gpt-4.1-mini", "gpt-4o-mini", "o3-mini-high"],
  },
  anthropic: {
    defaultModel: "claude-3-5-sonnet",
    models: ["claude-3-5-sonnet", "claude-3-5-haiku"],
  },
  google: {
    defaultModel: "gemini-1.5-pro",
    models: ["gemini-1.5-pro", "gemini-1.5-flash"],
  }
} as const;
```

### Runtime Selection Flow
- **User Prefs:** `userSettings.llm = { provider: 'openai', model: 'gpt-4.1-mini', mode:'balanced' }`
- **Override:** Request can pass `"model": "anthropic:claude-3-5-haiku"` to override per call.
- **Adapter Factory:**
```ts
function getAdapter(spec: string | undefined, prefs: UserLLMSettings): LLMAdapter {
  // parse "provider:model" or use prefs
  // return singleton adapter instance for that provider
}
```

### Normalized Prompting
- **System role:** “Editor/coach tone; return strict JSON fields.”
- **User content:** raw section text + optional hints (tone, length).
- **Response format:** Enforce JSON (tool schema) and post‑validate.

---

## 3) Bring‑Your‑Own‑Key (BYOK) — API Key Storage

### Requirements
- Users can add **provider API keys** (OpenAI/Anthropic/Google).
- Keys are **write‑only** from the client; never readable by the client after submission.
- Keys are **encrypted at rest** using a server‑side key (GCP **Cloud KMS**).
- Only **Cloud Functions** may decrypt to call providers.
- Support **multiple keys** (e.g., personal vs. org), with **alias + last4** display.
- Allow **test & validate**, **rotate**, **revoke**.

### Data Model
**Collection:** `userApiKeys` (one doc per key) — *server‑managed only*
```json
{
  "id": "auto",
  "userId": "auth.uid",
  "provider": "openai|anthropic|google",
  "alias": "Personal",
  "last4": "sk-...abcd",
  "ciphertext": "<base64 KMS-encrypted blob>",
  "createdAt": "timestamp",
  "updatedAt": "timestamp",
  "status": "active|revoked|invalid"
}
```

### Firestore Rules
- **Client cannot read or list** `userApiKeys`.
- Clients may **call HTTPS Callable Function** to create/update/delete keys.
```js
// Pseudocode: in rules, deny any direct read; allow only via CFs.
match /userApiKeys/{id} {
  allow read, write: if false; // fully blocked from client
}
```

### Key Handling Flow
1. **Client** posts `{provider, alias, apiKey}` to `/api/keys/add` (callable function).
2. **Function** validates format, encrypts with KMS, stores doc with `last4`, returns masked confirmation.
3. **Optional:** Immediately run a **test request** against the provider to mark status `active` or `invalid`.
4. **Usage:** When making LLM calls, the server selects the user’s active key for the chosen provider.
5. **Rotation/Revocation:** Endpoints `/api/keys/rotate`, `/api/keys/revoke` update docs and clear caches.

### UI
- **Settings → API Keys**
  - Provider dropdown
  - Alias (optional)
  - Key input (masked)
  - **[Test & Save]** button
  - Table of keys: Provider, Alias, Last4, Status, Updated, [Revoke]
- **Never** display full keys. Copy is not supported after save.

### Logging & Compliance
- Never log raw keys.
- Keep request/response metadata only (latency, token counts).
- Consider per‑user rate caps and monthly soft budgets.

---

## 4) Server APIs (Sketch)

### `/api/assist/feedback` (POST)
- Auth required (unless public anonymous mode is later added).
- Reads provider/model from request or user settings.
- Fetches user key from `userApiKeys` via server (no client exposure).
- Calls adapter; supports streaming via **SSE** or **Fetch readable stream**.

### `/api/keys/add` (Callable/POST)
- Auth required; body `{provider, alias, apiKey}`.
- Validate; encrypt with **KMS**; store; run **verify**; return `{id,last4,status}`.

### `/api/keys/list` (Callable/GET)
- Returns metadata only (no ciphertext, no key).

### `/api/keys/revoke` (Callable/POST)
- Marks key `revoked` and stops using it immediately.

---

## 5) E2E and Integration Tests

- **Pause trigger:** typing → stop 5s → feedback arrives; resume typing cancels prior call.
- **Streaming render:** partial tokens appear; final JSON passes schema validation.
- **Model switch:** change provider/model in settings; next pause uses new adapter.
- **BYOK flow:** add key → test → active; revoke key → subsequent calls fail gracefully.
- **Security:** client cannot read `userApiKeys` documents; keys never present in local storage or logs.

---

## 6) Operational Considerations

- **Fallback models:** If user key is invalid/ratelimited, fall back to app’s default provider (if allowed) or surface a clear error with retry.
- **Backoff & retry:** Exponential backoff; jitter; abort on user input.
- **Observability:** Capture latency, token counts, error rates per provider/model; surface in an internal dashboard.
- **Quotas:** Per‑user rate limits (e.g., 20/min) and daily soft caps with UI warnings.
- **Privacy modes:** Per‑section “Do not send to AI” toggle respected by client/server.

---

## 7) UI Additions (Concise)

- **Side Panel:** small “Model” chip (e.g., `Claude 3.5 Sonnet`) with a dropdown; changing it updates preference instantly.
- **Settings → API Keys:** add/verify/revoke; usage stats per provider.
- **Toast States:** “Analyzing…”, “Suggestion ready”, “Model switched to X”, “API key invalid”.

---

## 8) Rollout Steps (Delta)

1. Build **LLM adapter** and registry; implement at least **OpenAI** initially.  
2. Add **pause‑trigger** and **cancellation** on input resume; wire streaming UI.  
3. Implement **BYOK endpoints** with **KMS encryption** and blocked client reads.  
4. Ship **Settings → API Keys** UI and **Model switcher** in side panel.  
5. Add tests and dashboards; set sensible quotas and backoff.
