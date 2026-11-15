# Promptly v1.0

A focused text workspace: **summarize**, **rewrite**, and **extract tasks**. React + Vite, Tailwind CSS, custom shadcn-inspired controls, Express, Zod, and Gemini (with optional OpenAI Responses support). One screen, one processing endpoint. No database, accounts, or payments. Past tasks are stored locally in this browser.

The writing-room interface pairs locally bundled Instrument Sans and Instrument Serif with a centered document spread. Focus view expands the editing area; clipboard paste and plain-text download are built in. On phones, switch between Original and Result. Task source quotes can be located and highlighted in the original. See [DESIGN.md](DESIGN.md) for design decisions and review evidence.

Project documentation: [Product requirements (PRD)](PRD.md) · [High-level design (HLD)](HLD.md) · [Low-level design (LLD)](LLD.md) · [Course concepts](CONCEPTS.md).

## Run locally

Install **Node.js 22.12+** (Node 24 LTS recommended). Open a terminal in this folder:

```sh
npm ci
npm run dev
```

Open **http://127.0.0.1:5173**. Express runs on port 3001; Vite proxies `/api` to it. Both start together. Stop them with Ctrl+C. No `.env` or API key is needed for the default demo.

Choose a tool → **Try a sample** → run the tool. Ctrl+Enter (or Cmd+Enter) also submits from the editor. Results can be copied or downloaded as text; tasks can be checked off and completed tasks hidden. Use Focus view to hide the workspace heading and gain reading space.

## Review and pick up later

- **Edit result** opens labeled fields for a summary, rewritten draft, or task list. Changes apply immediately to the displayed result, copy, and download. **Reset edits** restores the generated version. Original source text and task evidence are kept intact.
- **Show changes** on a rewrite compares the original source with the current draft. Added text is underlined; removed text is struck through. Very large changes are grouped into passages to keep comparison responsive.
- Task completion totals are calculated from the checklist. **Hide completed** filters the view; copying or downloading still includes the entire checklist and its completion status.
- **Past tasks** opens a drawer with the latest 20 successful results across all three tools. It retains exact source text, tool preferences, the generated version, edits, and completion. Filter by tool, reopen without another AI call, delete one result, or clear all with confirmation.

History uses `localStorage` on the same browser and site address. It is not encrypted, shared between devices, or saved on the server. Successful results are saved automatically; unfinished source-only drafts and failed requests are not saved. Older results are removed as newer ones arrive, within a bounded storage budget. Clearing browser data removes history. Avoid shared browsers for private text, or delete the results after use. If storage is blocked/full, the workspace and session history still work and show a notice. Reloading then loses unsaved session changes. Running AI again uses the original source, not the edited output.

## Enable Gemini

Create a key in [Google AI Studio](https://aistudio.google.com/apikey). Copy `.env.example` to `.env` in the project root if `.env` does not already exist. On PowerShell:

```powershell
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
```

Edit these settings, then restart the server:

```dotenv
DEMO_MODE=false
AI_PROVIDER=gemini
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-3.5-flash-lite
```

Stop `npm run dev` with Ctrl+C and run it again after editing `.env`. A successful real request displays **Gemini result**, rather than **Demo result**. The model is configurable; choose a Gemini model your API account can access that supports structured JSON output. [Gemini 3.5 Flash-Lite](https://ai.google.dev/gemini-api/docs/models/gemini-3.5-flash-lite) is the default. The key is read **only by Express**. Never add a `VITE_` API key, paste it into the editor, or commit `.env`. Real mode fails at startup if the selected provider's key is absent; it never silently substitutes demo output.

The server calls Gemini's `generateContent` REST API with JSON Schema generated from the shared Zod contracts, validates the returned JSON, and enforces a 30-second timeout. It handles safety blocks, incomplete output, bad response shapes, connection failures, key/model errors, and quota limits. The key travels in a server-side header, never in a URL. See [Google's generation API reference](https://ai.google.dev/api/generate-content) and [structured output guide](https://ai.google.dev/gemini-api/docs/structured-output). API access and quota depend on your Google account and selected model.

OpenAI remains available: set `AI_PROVIDER=openai`, `OPENAI_API_KEY`, and optionally `OPENAI_MODEL` (default `gpt-4.1-mini`). This adapter uses `responses.parse`, `zodTextFormat`, and `store: false`. Neither provider makes automatic retries or switches to another provider on failure.

## Demo behavior

- Summarize and rewrite samples use carefully written fixtures. All four tones and both summary lengths have distinct results.
- Other summaries select source sentences. Other rewrites apply simple spacing, capitalization, and tone substitutions. These are limited local rules, not AI.
- Task extraction conservatively recognizes explicit `Name will/must/needs to …`, `Please …`, `We/I need to …`, and `TODO:/Action:` statements. It retains source quotes, recognizes simple stated deadlines, skips common negated/completed/speculative statements, and leaves missing fields null or unspecified.
- Nuanced prose, other languages, complex dates, and indirect commitments need real AI. Every demo result is labeled. Schema validation guarantees shape, not factual accuracy; review results before using them.

## Production build

```sh
npm run build
npm start
```

Open **http://127.0.0.1:3001**. Express serves the built frontend and the API together. For hardened production headers, set `NODE_ENV=production` in your hosting environment (PowerShell: `$env:NODE_ENV='production'` before `npm start`). Fonts are bundled locally.

`PORT`, `HOST`, `CLIENT_ORIGIN`, `AI_TIMEOUT_MS`, and `TRUST_PROXY_HOPS` are documented in `.env.example`. If changing `PORT` locally, restart both processes; Vite reads it for the API proxy. The default host binds locally. For hosting, configure `HOST=0.0.0.0`, `CLIENT_ORIGIN` to the exact public origin, TLS at your proxy, and the exact trusted proxy hop count. The in-memory 30 requests/minute/IP limit is for one server process; a shared limiter and stronger access controls are needed before exposing a paid API broadly.

## API contract

The only application API is **POST `/api/process`**, with `Content-Type: application/json`:

```json
{
  "mode": "summarize",
  "text": "Paste a meaningful passage here, at least ten characters long.",
  "preferences": { "length": "concise" }
}
```

Modes: `summarize`, `rewrite`, `actions`. Preferences are optional: `length` is `concise|detailed`, `tone` is `professional|casual|concise|polished`. Each tool uses its relevant preference. Limits: 10–20,000 trimmed characters, at most 4,000 whitespace-separated words, and a 100 KB JSON request body. Unknown keys and invalid preference values are rejected.

Successful responses use `{ mode, result, meta }`:

| Mode | Result fields |
| --- | --- |
| `summarize` | `summary`, `keyPoints[]`, `takeaway` |
| `rewrite` | `rewrittenText`, `changes[]` |
| `actions` | `overview`, `actionItems[{task, assignee, dueDate, priority, evidence}]` |

Metadata contains `provider` (`demo`, `gemini`, or `openai`), `inputWords`, `durationMs`, and `requestId`. Errors use `{ "error": { "code", "message", "requestId" } }`. Status codes: 200 success; 400 validation/JSON; 403 origin; 404 route; 413 size; 415 media type; 422 refusal; 429 request/provider limit; 500 internal; 502 upstream or malformed output; 503 provider configuration; 504 timeout. API responses are not cached. Logs omit source text, credentials, and raw provider errors.

## Verify

```sh
npm run check
```

This runs API/service tests and builds the frontend. Gemini's HTTP requests/responses and the OpenAI SDK transport are mocked in tests; tests do not make paid API calls.

For browser checks, install Python 3.10+ and Playwright, start the app **in demo mode** in another terminal, then run:

```sh
python -m pip install playwright
python -m playwright install chromium
npm run test:ui
```

The suite covers the three tools, settings, loading, validation, copy, tasks, errors/retry, cancellation, escaping, and five viewport widths. Screenshots and a report go to ignored `test-results/`. Optional environment variables: `BASE_URL` (default `http://127.0.0.1:5173`) and `BROWSER_CHANNEL` (e.g. `msedge`). See [TESTING.md](TESTING.md) for the completed checks and [CONCEPTS.md](CONCEPTS.md) for evaluation markers.

## Where things live

```text
client/src/       React screen, review/history components, state reducer, comparison, storage, API client
shared/          Zod request/output contracts and sample text
server/app.js    Express middleware, endpoint, static serving
server/services/ Provider selection, Gemini, OpenAI, and local demo engines
server/prompts/  Per-tool instructions
tests/           API, provider transport, and browser integration checks
```

The Promptly server does not persist source or result text. Past tasks are saved locally in the browser as described above. In real mode, source text is sent to the selected provider (Google for Gemini, or OpenAI); provider retention policies still apply. Canceling discards the browser request and prevents stale output; it does not guarantee that an upstream generation or its billing stops.

If a port is occupied, stop the earlier app process. If the browser cannot reach the API, confirm both dev processes are running and use the exact URL above. A provider limit error can indicate API billing or quota, not only request frequency.
