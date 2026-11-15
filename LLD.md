# Low-Level Design (LLD): Promptly

Version: 1.0, including review and browser-local history improvements

Document date: 15 November 2026

Status: Describes the current code; example payloads are illustrative

## 1. Repository structure and module boundaries

```text
client/src/App.jsx                   Screen orchestration and request lifecycle
client/src/components/               Editor, result, edit, comparison, history, guide, controls
client/src/lib/api.js                HTTP client, output formatting, copy, download
client/src/lib/workspace.js          Pure reducer and displayed-result selector
client/src/lib/history.js            Snapshot validation, retention, persistence
client/src/lib/diff.js               Bounded word comparison
shared/schemas.js                   Request/output/response contracts and limits
shared/samples.js                   Tool-specific demonstration source text
server/index.js                     Startup, listening, graceful shutdown
server/config.js                    Environment validation and engine selection settings
server/app.js                       Express factory, middleware, endpoint, static assets
server/services/ai.js                Real-provider factory
server/services/gemini.js            Gemini HTTP transport, parsing, error mapping
server/services/openai.js            OpenAI Responses adapter
server/services/demo.js              Local fixtures and conservative text rules
server/prompts/templates.js          Operation-specific prompt builder
server/middleware/errors.js          AppError and centralized HTTP error formatting
tests/                              API, provider, workspace, and browser checks
```

The code uses JavaScript ES modules. `createApp(config, { processor, limit })` supports processor/rate-limit injection for tests. Provider factories accept injectable transport, allowing SDK/HTTP behavior to be tested without paid generation.

## 2. Request contract

`POST /api/process` accepts `Content-Type: application/json`.

```json
{
  "mode": "actions",
  "text": "Maya will review the copy.",
  "preferences": {}
}
```

| Field | Validation and behavior |
| --- | --- |
| `mode` | Required enum: `summarize`, `rewrite`, or `actions`. |
| `text` | Required string; trimmed by Zod; 10–20,000 characters; at most 4,000 words counted by whitespace separation. |
| `preferences` | Optional strict object, default `{}`. |
| `preferences.length` | Optional `concise` or `detailed`; relevant to summarize. |
| `preferences.tone` | Optional `professional`, `casual`, `concise`, or `polished`; relevant to rewrite. |

Unknown request or preference keys are rejected. Both defined preference keys are accepted by the shared schema; the selected operation uses only its relevant preference. Express also limits the JSON body to 100 KB. The UI checks raw input length as well as word count before normal submission. The source snapshot retains original whitespace separately from the normalized API request.

## 3. Response and result contracts

All successful responses use `{ mode, result, meta }`. The browser uses a mode-discriminated Zod union and confirms the returned mode matches the submitted one.

| Operation | Strict result fields |
| --- | --- |
| Summarize | `summary: string`, `keyPoints: string[]`, `takeaway: string` |
| Rewrite | `rewrittenText: string`, `changes: string[]` |
| Actions | `overview: string`, `actionItems: ActionItem[]` |

An action item requires `task: string`, `assignee: string | null`, `dueDate: string | null`, `priority: high | medium | low | unspecified`, and `evidence: string`. Nullable fields must exist explicitly. Empty task arrays are valid. Dates remain source wording rather than inferred calendar dates.

Metadata contains `provider: demo | gemini | openai`, `inputWords: number`, `durationMs: number`, and `requestId: string`.

```json
{
  "mode": "actions",
  "result": {
    "overview": "One review task was identified.",
    "actionItems": [
      {
        "task": "Review the copy",
        "assignee": "Maya",
        "dueDate": null,
        "priority": "unspecified",
        "evidence": "Maya will review the copy."
      }
    ]
  },
  "meta": {
    "provider": "demo",
    "inputWords": 5,
    "durationMs": 700,
    "requestId": "illustrative-request-id"
  }
}
```

## 4. Backend execution

`readConfig` validates environment variables with Zod. Demo defaults to true. Real mode defaults to Gemini with `gemini-3.5-flash-lite`; OpenAI is selectable. A missing selected provider key fails startup. Invalid configuration reports variable names without echoing their values.

The Express request path is:

1. Assign a UUID request ID and `X-Request-Id` header.
2. Apply Helmet policies; production CSP restricts scripts, styles, fonts, and connections to the same origin.
3. For `/api`, set `Cache-Control: no-store`, check a supplied Origin, apply CORS, and enforce the default 30-request/minute/IP in-memory limit.
4. Parse JSON with a 100 KB limit.
5. Require JSON media type and validate the request.
6. Time and await the selected processor, then validate its mode-specific output.
7. Return result/metadata, or use centralized error formatting.

`createAIService` selects a real-provider factory. Demo selection occurs separately in `createApp`. There is no automatic provider retry/fallback.

### 4.1 Gemini adapter

`createGeminiService(config, fetchImpl)` returns an async processing function. It converts the selected output schema with `z.toJSONSchema`, removes the dialect field, and sends the resulting schema with JSON response MIME type and `maxOutputTokens: 8192` to the model's `generateContent` endpoint.

The request contains a system instruction from `buildInstructions(input)` and separate user source content. The key travels in `x-goog-api-key`. `AbortSignal.timeout(config.timeoutMs)` defaults to 30 seconds.

The adapter checks HTTP failures, prompt blocks, candidate finish reasons, and content presence. It joins non-thought text parts, parses JSON, and validates it with the output schema. Raw provider bodies are never returned to the browser.

### 4.2 Prompt and demo behavior

Base instructions treat source as untrusted material, preserve supported facts/caveats, and forbid claiming task execution. Summarize chooses concise/detailed guidance; rewrite selects tone and preserves requests/deadlines; actions omits completed, negated, or speculative actions and requires exact evidence. These are model instructions, not a proof of semantic correctness.

Demo summarize/rewrite use curated results for their samples and limited local transformations for other text. Demo actions conservatively recognizes explicit requests/commitments, keeps evidence, and leaves unsupported metadata unspecified. Every demo response identifies its provider as demo.

## 5. Error model

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Add at least 10 characters of text.",
    "requestId": "illustrative-request-id"
  }
}
```

| HTTP status | Example code/reason |
| --- | --- |
| 400 | `VALIDATION_ERROR` or `INVALID_JSON` |
| 403 | `ORIGIN_NOT_ALLOWED` |
| 404 | `NOT_FOUND` for an unsupported API route/page |
| 413 | `PAYLOAD_TOO_LARGE` |
| 415 | `UNSUPPORTED_MEDIA_TYPE` |
| 422 | `AI_REFUSAL` |
| 429 | `RATE_LIMITED` or `AI_RATE_LIMIT` |
| 500 | `INTERNAL_ERROR`, with a generic user message |
| 502 | `AI_UNAVAILABLE`, `AI_INVALID_OUTPUT`, or `AI_INCOMPLETE` |
| 503 | `AI_CONFIGURATION` |
| 504 | `AI_TIMEOUT` |

`AppError` carries status, code, and a safe message. JSON-parser errors are normalized by `errorHandler`. Server failures log only request ID, code, and status. The client preserves safe messages/request IDs and provides retry; an unreachable server produces a readable network error.

## 6. Frontend components and state

| Component | Responsibility |
| --- | --- |
| `App` | Source/preferences, request lifecycle, reducer dispatch, source navigation, and history coordination. |
| `Header` | Brand, Focus toggle, history opener, and native guide dialog. |
| `ModeSelector` | Accessible three-tool selection and moving selected indicator. |
| `TextEditor` | Controlled input, paste/sample/clear, counts/limits, keyboard submit, and exact-source reading view. |
| `ResultPanel` | Empty/loading/error/result states, editing/comparison controls, derived task views, copy/download feedback. |
| `ResultEditor` | Labeled controlled summary/rewrite/task fields; source evidence is read-only. |
| `RewriteComparison` | Memoized comparison displayed through ordinary React text, `ins`, and `del`. |
| `HistoryDrawer` | Native modal, tool filter, reopen, individual delete, confirmed clear-all, and privacy/storage feedback. |

`useState` holds source, mode, preferences, Focus/mobile navigation, history entries/notices, and local control feedback. `useReducer` holds the processing domain:

```text
status: idle | loading | success | error
request: active request version
data: untouched generated response or null
editedResult: optional user-edited result
completed: task-index → boolean
error: safe display error or null
entryId: saved snapshot ID or null
savedInput / sourceText / createdAt: snapshot metadata after success/restore
```

### 6.1 Reducer events and request lifecycle

| Event | Effect |
| --- | --- |
| `reset` | Return to idle with a new request version. |
| `request` | Enter loading and clear the previous result/error. |
| `success` | Accept only while loading and only for the matching request version; retain generated data and snapshot metadata. |
| `failure` | Accept only for the matching request version in an eligible state; show the error. |
| `edit` | Set edited content in success state; collapse to null when it equals generated content. |
| `resetEdits` | Remove the edited version while retaining generated data/completion. |
| `toggleTask` | Update a valid action item's completion immutably. |
| `restore` | Restore a successful history snapshot using the new request version. |
| `detachHistory` | Keep open work but remove its saved-entry association. |

`App.run` rejects overlapping active submissions, validates input, increments the version, creates an abort controller, and sets a 45-second browser timeout. Completion/error handlers check the current version. Reset, source/preference changes, tool switching, reopening, and unmount abort the relevant client fetch. Timeout timers are cleared. Browser cancellation does not cancel the independently timed provider call through Express.

### 6.2 Editing, evidence, and export

`visibleData` derives the displayed response from `editedResult || data.result`; generated `data` is not overwritten. Copy, download, and word counts use that same displayed data. Task totals count completion only for existing items. Hide completed affects rendering, not exports.

Editing allows summary fields, rewritten text, and task wording/owner/deadline/priority. It does not change evidence or source. For source location, `text.indexOf(quote)` finds the first exact nonempty match; a reader highlights that range and scrolls/focuses it. An unavailable quote is not approximately matched. Returning to editing restores editor focus. On phones, finding evidence switches to Original and generation opens Result.

`formatResult` produces plain text. Copy prefers the Clipboard API and falls back to a temporary textarea. Download creates a UTF-8 text Blob/object URL, clicks a local download link, and revokes the URL afterward. Feedback timers are cleaned up. The current task-completion map controls exported checkbox marks.

## 7. Browser history design

Storage key: `promptly.history.v1`. Serialized shape:

```text
{ version: 1, entries: HistoryEntry[] }

HistoryEntry = {
  id, createdAt, input, sourceText,
  data, editedResult, completed
}
```

`readHistory(storage)` parses defensively and validates records with Zod. It checks request/response mode agreement, trimmed source agreement with normalized request text, valid edited output, and unchanged action count/evidence for edited tasks. It skips invalid/duplicate entries and returns a notice rather than breaking the workspace.

`upsertHistory(entries, entry)` replaces the matching ID, sorts by original generation time descending, and keeps at most 20 entries. It removes older entries until serialized JSON fits the 2,000,000 JavaScript string-length budget. The constant is named `MAX_BYTES`, but implementation uses `.length`; it is not a precise encoded-byte limit or a guarantee of available browser quota.

`writeHistory` removes the key when history is empty, otherwise serializes the envelope. Storage failures return a notice while React retains session history. Successful result, edit, and completion changes update the associated snapshot. Reopening restores state without a request. Deleting the active snapshot or clearing all dispatches `detachHistory`, preventing later edits from recreating it.

There is no cross-device sync, encryption, or multi-tab merge. Browser clearing removes persistence, and blocked/full storage can cause session changes to be lost on reload.

## 8. Rewrite comparison algorithm

`compareWords(original, revised)` is a pure function. Tokens keep trailing whitespace with each word, preserving exact text and making replacements readable. The algorithm removes common prefixes/suffixes before comparing the remaining tokens.

For middle lengths `n` and `m`, a longest-common-subsequence table uses `Uint16Array`. It runs in `O(n*m)` time and space only when `(n + 1) * (m + 1) <= 1,000,000`. Larger comparisons emit grouped removed/added passages instead of allocating the table. Adjacent segments of the same type are merged. Returned parts have `equal`, `removed`, or `added` type plus text, and a `grouped` flag explains the fallback.

Dropping added parts reconstructs the original; dropping removed parts reconstructs the revision. Rendering uses text nodes rather than HTML interpretation. Tests cover whitespace, Unicode, markup-looking text, empty sides, exact equality, and long unrelated content.

## 9. Build, testing, and operational boundaries

`npm run dev` starts Express and Vite. `npm run build` emits `dist/`; `npm start` serves the build and API. `NODE_ENV=production` enables production CSP. Configuration stays in server environment variables; [.env.example](.env.example) contains placeholders only.

`npm test` runs Node/Supertest API, provider, and pure workspace tests. `npm run test:ui` runs the Python Playwright suite against a running demo app. `npm run check` combines automated tests and build. The last application verification recorded 68 automated checks and 25 browser scenarios; see [TESTING.md](TESTING.md) for evidence and historical live-provider limitations.

The server has no authentication or content database. Its default rate limiter is process-local, and CORS is not access control for arbitrary clients. These limits are intentional in the current scope. [HLD.md](HLD.md) discusses deployment/growth boundaries, and [PRD.md](PRD.md) defines product acceptance.
