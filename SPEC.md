# PipeWise Working Prototype: Build Specification

Version 1.1, 2026-10-27. Written for an implementing engineer or coding agent. Everything in this document is a requirement unless marked *optional* or *stretch*. Version 1.1 incorporates an independent review; see section 15 for what changed.

## 1. Purpose

Turn the clickable mockup at the repo root (`index.html`, live at https://davidrynn.github.io/plumbing-assistant/) into a working demo that answers real questions about a real set of construction documents. The demo must convince a mechanical subcontractor that an AI assistant grounded in *their* project's documents is useful to a plumber in the field.

**What the POC must prove**

1. A plumber can ask a plain-language question and get an answer drawn from this project's spec book, drawings and addenda, with page-level citations they can click to see the source.
2. The assistant can answer a plumbing code question for a stated jurisdiction and code edition, and say when the project spec is stricter than code.
3. A plumber can upload a job-site photo with a question and get findings tied back to the project documents.

**What it does not need to prove**

Multi-user auth, Procore or ACC integration, offline use, billing, or production hardening. See section 12 for the full non-goals list.

## 2. Acceptance criteria

The build is done when all of the following pass on a laptop with a valid `ANTHROPIC_API_KEY`:

- [ ] `npm install && npm run dev` in `app/` starts a server on `http://localhost:3000` that serves the UI.
- [ ] Dropping 1 to 10 PDFs into `app/projects/<id>/docs/` and editing `app/projects/<id>/project.json` makes that project selectable in the UI with no code changes.
- [ ] Asking "What hanger spacing does the spec require for 2 inch copper?" against a real spec book returns an answer with at least one citation chip. Clicking the chip renders the cited PDF page, with the quoted text shown above it, on both a desktop browser and iOS Safari.
- [ ] `POST /api/projects/<id>/warm` completes and logs a non-zero `cache_creation_input_tokens`. The next question to that project logs `cache_read_input_tokens` of the same order and returns in well under half the time of a cold question.
- [ ] Asking a question the documents do not answer returns an explicit "not found in project documents" statement, not an invented answer.
- [ ] Code Lookup with jurisdiction "2021 IPC" returns an answer that names the code section and includes the verification disclaimer.
- [ ] Uploading a JPEG or PNG with the question "is this hanger spacing OK?" returns 1 to 5 structured findings, each with a severity and a reference to a project document or code section.
- [ ] The second identical question to the same project costs materially fewer input tokens than the first (prompt cache hit). Log `usage` on every call so this is visible in the server console.
- [ ] A Code Lookup answer is visually labelled as coming from model knowledge, not from project documents.
- [ ] `GET /api/health` reports whether an API key is present, how many projects loaded, and the token count of each.
- [ ] A Claude API error (bad key, rate limit, refusal) shows a readable message in the UI instead of a hung spinner.
- [ ] The UI works at phone width (375 px) with no horizontal scroll.

## 3. Architecture

Single Node.js process. No database, no vector store, no background workers.

```
Browser (app/public, vanilla HTML/JS adapted from the mockup)
   │  fetch JSON
   ▼
Express server (app/server, TypeScript)
   │  reads project.json + PDFs from disk
   │  calls Claude with the PDFs attached as document blocks
   ▼
Claude API (@anthropic-ai/sdk)
```

**Why no retrieval pipeline.** Claude accepts PDFs directly as `document` content blocks, up to 600 pages each, with native page-level citations, inside a 1M-token context window. A Division 22 spec book plus a P-series drawing set fits. Attaching the documents on every request and marking them for prompt caching gives correct citations with roughly one tenth the input cost after the first call. Chunking, embeddings and a vector database would add a week of work and a new class of bugs for no demo benefit. Revisit only if a project's documents exceed about 150 MB or 2,000 pages total.

**Why one process.** The front end is static files served by the same Express app, so there is no CORS, no build step for the client, and one command to run.

## 4. Technology

| Layer | Choice | Notes |
|---|---|---|
| Runtime | Node.js 20 or later | |
| Language | TypeScript, run with `tsx` in dev | No bundler. |
| Server | Express 4 | `multer` for photo upload. |
| Claude SDK | `@anthropic-ai/sdk`, latest | Use the SDK's own types. Do not define parallel interfaces for messages or content blocks. |
| Front end | Vanilla HTML, CSS, JS | Start from the root `index.html`. No framework. |
| PDF viewing | pdf.js (`pdfjs-dist`, pinned version, loaded from cdnjs) rendering one page to a `<canvas>` | Required, not optional. Phone browsers do not honor `#page=N` in iframes and iOS Safari does not render PDFs in iframes reliably. Provide an "Open full PDF" link as a fallback. |
| Image resize | `sharp` | Photo downscaling. |
| PDF page count and token count | `pdf-lib` for page count; `client.messages.countTokens` for tokens | Both at boot, see 6.2. |
| Config | `.env` loaded with `dotenv` | Never commit `.env`. Ship `.env.example`. |

## 5. Repository layout

Keep the existing mockup at the repo root so GitHub Pages keeps working. Add the working app under `app/`.

```
/
├── index.html                  # existing mockup, untouched
├── SPEC.md                     # this file
├── .gitignore                  # add: app/node_modules, app/.env, app/uploads
└── app/
    ├── package.json
    ├── tsconfig.json
    ├── .env.example            # ANTHROPIC_API_KEY=, PORT=3000, CLAUDE_MODEL=claude-opus-5-5
    ├── README.md               # how to run, how to add a project
    ├── server/
    │   ├── index.ts            # Express bootstrap, static serving, routes
    │   ├── projects.ts         # load project.json files, resolve doc paths, serve PDFs
    │   ├── claude.ts           # the single Anthropic client, shared request builder, usage logging, error mapping
    │   ├── ask.ts              # POST /api/ask
    │   ├── code.ts             # POST /api/code
    │   ├── photo.ts            # POST /api/photo
    │   └── prompts.ts          # all system prompt text, exported as constants
    ├── public/
    │   ├── index.html          # adapted from root mockup
    │   ├── app.js
    │   └── styles.css
    └── projects/
        └── riverside-tower-b/
            ├── project.json
            └── docs/
                ├── 22-05-29-hangers.pdf
                └── ...
```

## 6. Data model

### 6.1 `project.json`

```json
{
  "id": "riverside-tower-b",
  "name": "Riverside Medical Center, Tower B",
  "jurisdiction": {
    "code": "2021 International Plumbing Code with state amendments",
    "ahj": "City Building Department, Plumbing Division",
    "extra": ["NFPA 99 (2021) for medical gas"]
  },
  "documents": [
    {
      "file": "22-05-29-hangers.pdf",
      "title": "Spec 22 05 29 Hangers and Supports",
      "type": "spec",
      "revision": "CD Set",
      "date": "2026-08-02"
    },
    {
      "file": "P-401-R4.pdf",
      "title": "P-401 Plumbing Plan Level 4",
      "type": "drawing",
      "revision": "Rev 4",
      "date": "2026-10-14"
    }
  ]
}
```

`type` is one of `spec`, `drawing`, `addendum`, `submittal`, `rfi`, `other`. `date` is an ISO date (issue or revision date) and is required. Both are included in the document title passed to Claude as `"{TYPE}: {title} ({revision}, {date})"` so the model can order documents and reason about precedence (an addendum supersedes the spec it amends; a later revision supersedes an earlier one).

`jurisdiction.codeBook` is *optional* and, when present, must be one of the Code Lookup list in 7.2. It sets the default selection in the Code Lookup tab. Without it the default is `2021 IPC`.

Validate on load. A project with a missing file, missing `date`, or malformed JSON is logged and skipped, never crashes the server.

### 6.2 Projects at startup

`projects.ts` scans `app/projects/*/project.json` once at boot and caches the result. For each document it reads the file once, stores the base64 string in memory (so 20 MB is not re-read and re-encoded per request), and records the page count with `pdf-lib`. For each project it then calls `client.messages.countTokens` with the full document set and the Ask system prompt, and logs one line per project: name, document count, total pages, total raw MB, input tokens. If a project exceeds 800K tokens, log a warning and still load it.

Provide `GET /api/projects` returning `[{id, name, jurisdiction, tokenCount, documents:[{title,type,revision,date,file,pages}]}]`. Reload on each request in dev mode is *optional*.

Provide `GET /api/health` returning `{ ok, hasApiKey, model, projects: [{id, name, tokenCount}] }`. It must respond even when the key is missing.

### 6.3 Size guard

Base64 inflates a file by about one third, and the Claude request limit is 32 MB after encoding. Before building any request, sum the raw sizes of the documents being attached plus, for photo requests, the downscaled image. If the total exceeds 22 MB, return HTTP 413 with error code `project_too_large` and a message naming the project, the total, and the limit. Do not silently drop documents.

## 7. API endpoints

All responses are JSON. All errors use `{ "error": { "code": string, "message": string } }` with an appropriate HTTP status. `message` must be safe to show to an end user. Error codes are a closed set:

| Code | Status | When |
|---|---|---|
| `unknown_project` | 404 | `projectId` does not match a loaded project. |
| `unknown_document` | 404 | PDF endpoint asked for a file not in the project's list. |
| `empty_question` | 400 | `question` missing or whitespace, except on photo where it is optional. |
| `invalid_code_book` | 400 | `codeBook` not in the fixed list. |
| `invalid_photo` | 400 | Missing file, unsupported type, or over 10 MB (from `multer`). |
| `project_too_large` | 413 | Size guard in 6.3. |
| `api_key_missing` | 500 | Startup check or `AuthenticationError`. |
| `rate_limited` | 429 | `RateLimitError`. |
| `upstream_bad_request` | 400 | `BadRequestError`; include the API message. |
| `upstream_unreachable` | 503 | `APIConnectionError`. |
| `upstream_error` | 502 | Any other `APIError`. |
| `bad_model_output` | 502 | Structured output did not parse. |

### 7.1 `POST /api/ask`

Request:
```json
{ "projectId": "riverside-tower-b", "question": "What hanger spacing...", "history": [] }
```
`history` is an array of prior plain-text turns, `{ "role": "user" | "assistant", "content": string }`, exactly as returned by earlier responses. It never contains documents or images. The server caps it at the last 10 turns, then builds the request as: documents in a fresh first user message, the capped text history, the new question (see 8.2). Empty array on first question. The client resets `history` when the project changes.

Response:
```json
{
  "answer": [
    { "text": "For 2\" Type L copper, the spec requires hangers at 8 ft max.", "cites": [0] },
    { "text": " The IPC would allow 12 ft, but the spec governs.", "cites": [1] }
  ],
  "citations": [
    { "id": 0, "docIndex": 0, "title": "Spec 22 05 29 Hangers and Supports", "file": "22-05-29-hangers.pdf", "page": 6, "quote": "NPS 1-1/2 to NPS 2, 8 feet" },
    { "id": 1, "docIndex": 0, "title": "Spec 22 05 29 Hangers and Supports", "file": "22-05-29-hangers.pdf", "page": 6, "quote": "..." }
  ],
  "history": [ ...updated history including this turn... ],
  "usage": { "input_tokens": 1200, "cache_read_input_tokens": 180000, "cache_creation_input_tokens": 0, "output_tokens": 410 }
}
```

`answer` is an ordered list of text segments. A segment with a non-empty `cites` array renders with citation chips after it. `cites` holds `id` values from `citations`. `history` in the response is the text-only history including this turn, where the assistant turn is the concatenated answer text without chips.

### 7.2 `POST /api/code`

Request:
```json
{ "projectId": "riverside-tower-b", "codeBook": "2021 IPC", "question": "hanger spacing for 2 inch copper" }
```
`projectId` is optional. `codeBook` must be one of `2021 IPC`, `2018 IPC`, `2021 UPC`, `2018 UPC`, `NFPA 99 2021`; anything else is `invalid_code_book`.

When `projectId` is present, the request attaches the **same document set and the same system prompt as Ask**, so it hits the same cache. The code lookup instruction and the code edition go in the user turn (see 8.4). Without a project, send only the Code Lookup system prompt and the question.

Response: same shape as 7.1 plus `"source": "model_knowledge"`. `citations` is non-empty only when a project document was cited. Code references appear inline in the text as section numbers (for example "IPC 308.5"). Do not attempt to deep-link into code books. The UI must label these answers as unverified model knowledge (see 9.4).

### 7.3 `POST /api/photo`

`multipart/form-data` with fields `projectId`, `question` (optional), `photo` (JPEG, PNG or WebP, max 10 MB). Downscale server-side so the longest edge is at most 1568 px before sending to Claude. Use `sharp`.

Response:
```json
{
  "title": "Copper domestic water run with clevis hangers",
  "findings": [
    {
      "severity": "bad",
      "text": "Visible hanger spacing appears to exceed the 8 ft maximum in Spec 22 05 29.",
      "reference": "Spec 22 05 29 Para 3.2.A",
      "confidence": "medium"
    }
  ],
  "caveat": "Distances estimated from a single photo. Verify with a tape before correcting.",
  "usage": { ... }
}
```
`severity` is one of `bad`, `warn`, `ok`. `confidence` is one of `low`, `medium`, `high`. The server enforces 1 to 5 findings: truncate past 5; if the model returns none, return one `warn` finding reading "No specific findings. Verify against the drawings for this location." Each finding also carries `"docFile": string | null`, resolved server-side by matching `reference` against document titles (case-insensitive substring on the spec section number or title); `null` when nothing matches.

If the model refuses (`stop_reason: "refusal"`), return 200 with `findings: []`, `title: ""`, and `caveat: "The assistant declined to analyze this photo."`

### 7.4 `GET /api/projects/:id/docs/:file`

Streams the PDF with `Content-Type: application/pdf` and `Content-Disposition: inline`. Reject any `file` not listed in that project's `project.json` with `unknown_document` (prevents path traversal). The client fetches this URL with pdf.js and renders the cited page.

### 7.5 `POST /api/projects/:id/warm`

Pre-warms the prompt cache for a project. Sends the Ask request shape (system prompt plus all documents, question text "warm") with `max_tokens: 0`. The API writes the cache and returns immediately with empty content. Respond with `{ "ok": true, "usage": {...} }`. Call this on server boot for every project *(optional)* and expose it so the demo runner can hit it within the hour before a meeting. Do not use structured outputs or streaming on this request; both are rejected with `max_tokens: 0`.

## 8. Claude integration

All API calls go through `claude.ts`. One `Anthropic` client instance, constructed with no arguments so it reads `ANTHROPIC_API_KEY` from the environment. The first line of `server/index.ts` must be `import "dotenv/config";` so the variable exists before the client module loads. At startup, if the key is missing, log one clear line and keep serving the UI and `/api/health`; API routes then return `api_key_missing`.

### 8.1 Model and request defaults

| Setting | Value | Reason |
|---|---|---|
| `model` | `process.env.CLAUDE_MODEL ?? "claude-opus-5-5"` | Current Opus. Overridable for cost experiments. |
| `max_tokens` | 16000 | Non-streaming default that stays under SDK timeouts. |
| `thinking` | omit | Always on for this model. Do not send `disabled` or `budget_tokens`, both return 400. |
| `output_config.effort` | `"medium"` for ask, code and photo | Raise photo to `"high"` only if findings are weak in testing; it costs latency on top of a large document prefix. |
| `betas` | `["server-side-fallback-2026-07-01"]` | Enables the fallback below. Requires `client.beta.messages.create`. |
| `fallbacks` | `"default"` | If a safety classifier declines, the API retries on an appropriate fallback model. |
| `temperature`, `top_p`, `top_k` | never send | Rejected on current API versions. |
| `tool_choice` | never send | Not used in this POC. |

Streaming is a *stretch* goal. The POC uses non-streaming calls with a typing indicator in the UI.

### 8.2 Document attachment and caching

Every ask, code (with project) and warm request shares one prefix: the Ask system prompt, then a first user message holding all project documents. Build it as:

```ts
// Type names are illustrative. Use whatever the installed SDK exports and the compiler accepts.
const docBlocks = project.documents.map((d, i) => ({
  type: "document",
  source: { type: "base64", media_type: "application/pdf", data: d.base64 }, // cached in memory at boot
  title: `${d.type.toUpperCase()}: ${d.title} (${d.revision}, ${d.date})`,
  citations: { enabled: true },
  // cache_control on the LAST document block only
  ...(i === project.documents.length - 1 ? { cache_control: { type: "ephemeral", ttl: "1h" } } : {}),
}));

const messages = [
  { role: "user", content: [...docBlocks, { type: "text", text: "Project documents attached. Answer the questions that follow from them." }] },
  { role: "assistant", content: "Understood. I will answer from the attached project documents." },
  ...cappedTextHistory,              // plain {role, content: string} turns from the client
  { role: "user", content: question } // for code lookup, the user turn described in 8.4
];
```

Rules:
- `system` is a single text block with `cache_control: { type: "ephemeral", ttl: "1h" }`. The API requires longer TTLs to come before shorter ones, so a 5-minute `system` entry ahead of a 1-hour document entry is rejected with a 400. Both breakpoints use `"1h"`.
- Documents first, everything variable after the last document. The breakpoint on the last document caches the system prompt and every document in one prefix.
- Document order is the order in `project.json`. Reordering breaks the cache.
- Base64 strings contain no newlines. Encode once at boot (6.2), never per request.
- Nothing time-varying (dates, request IDs, the question) goes in `system` or before the last cache breakpoint.
- Follow-up turns re-send the same document message; the cache makes it cheap. The client never sees the documents, only text history (7.1).
- Ask and Code use the identical system prompt and identical document set so they share one cache entry. Photo uses the same prefix too (8.5).
- Log `usage.cache_read_input_tokens` and `usage.cache_creation_input_tokens` on every call. Acceptance criteria 3 and 8 depend on the second call showing a large `cache_read_input_tokens`.

### 8.3 Citations

With `citations: { enabled: true }` on every document block, the response `content` is a list of `text` blocks. Blocks that cite carry a `citations` array. Each citation has `type: "page_location"`, `document_index`, `document_title`, `cited_text`, `start_page_number` (1-indexed) and `end_page_number`.

Map to the response shape in 7.1:
- Keep the array of documents attached to *this request* (`attachedDocs`). `document_index` indexes that array, which for Ask and Code is the full `project.documents` list in order. Resolve `file` and `title` from it, never from a differently filtered list.
- One `answer` segment per text block.
- De-duplicate by the key `(document_index, start_page_number)`. Assign `id` values in first-seen order starting at 0. A block's `cites` lists the ids of its (de-duplicated) citations.
- `page` is `start_page_number`. `quote` is `cited_text` trimmed to 200 characters.
- Scanned pages produce little or no `cited_text`; still emit the chip with an empty `quote`.

Citations are incompatible with `output_config.format` (structured outputs). That is why the photo endpoint, which shares the citation-enabled document prefix, asks for JSON in the prompt instead of using structured outputs. See 8.5.

### 8.4 System prompts (`prompts.ts`)

**Ask the Project**

> You are a field assistant for commercial plumbers working on the project named below. Answer only from the attached project documents. When a document answers the question, cite it. When the documents do not answer the question, say so plainly in one sentence and do not guess. When an addendum or later revision changes an earlier document, the later one governs; say which document governs and why. Where the project specification is stricter than the applicable code, say so. Use plain language a journeyman plumber would use. Keep answers under 200 words unless the question needs a list or table. Use imperial units. End every answer that affects installation with one sentence noting that the engineer of record and the inspector make the final call.
>
> Project: {name}
> Adopted code: {jurisdiction.code}
> Authority having jurisdiction: {jurisdiction.ahj}
> Other applicable standards: {jurisdiction.extra joined}

**Code Lookup, with a project attached.** Uses the Ask system prompt unchanged (to share the cache). The final user turn is:

> Code lookup. Code edition: {codeBook}. Answer the following from your knowledge of that edition. Name the section or table number for every requirement you state. Summarize in your own words; do not reproduce code text verbatim beyond a short phrase. Compare the code requirement with the attached project specifications and addenda and state which is stricter, citing the project document. If the edition does not cover the question, say so. Note when a requirement commonly varies by local amendment. Close with one sentence: verify with the adopted code and the authority having jurisdiction.
>
> Question: {question}

**Code Lookup, no project.** System prompt:

> You are a plumbing code reference assistant. The user has selected the code edition named below. Answer from your knowledge of that edition. Name the section or table number for every requirement you state. Summarize requirements in your own words; do not reproduce code text verbatim beyond a short phrase. If the selected edition does not cover the question, say so. State clearly when a requirement commonly varies by local amendment. Close with one sentence: verify with the adopted code and the authority having jurisdiction.
>
> Code edition: {codeBook}

**Photo Check**

> You are a field assistant reviewing a job-site photo for a commercial plumber. Identify what plumbing work is visible. Then list between one and five findings about compliance with the attached project documents and the adopted code. For each finding give a severity (bad means likely non-compliant, warn means needs verification, ok means appears compliant), a one or two sentence explanation, a reference to the governing spec section or code section, and a confidence level. A photo cannot measure distances reliably; when a finding depends on a measurement, say the value must be verified with a tape. Never report a finding you cannot see evidence for in the photo. Respond only in the required JSON format.

### 8.5 Photo request

Photo requests reuse the shared prefix from 8.2 (Ask system prompt, all documents in the first user message with the 1-hour breakpoint) so they hit the same cache and do not pay a second document cache write.

That prefix has citations enabled, and the API rejects `output_config.format` (structured outputs) on a request with citation-enabled documents. So the photo endpoint does **not** use structured outputs. Instead the final user turn asks for JSON and the server parses it:

- Final user turn content, in order: the Photo Check instruction text (8.4) followed by "Respond with a single JSON object matching this schema and nothing else:" and the schema below as text; then the image block (base64 JPEG after downscaling); then the user's question or "Review this photo."
- Parse: concatenate all text blocks, strip code fences, take the substring from the first `{` to the last `}`, `JSON.parse` it, and validate against the schema with a small hand-written check (types, enums, required keys). On failure, retry the request once with an extra user turn "Your previous reply was not valid JSON. Reply with only the JSON object." On a second failure return `bad_model_output`.
- Ignore any citations the model attaches to its text blocks.

Schema (used as prompt text and as the server-side validation contract). `minItems`/`maxItems` are omitted; the server enforces the count (7.3).

```ts
{
  effort: "medium",   // output_config.effort still applies; only format is omitted
  schema: {
      type: "object",
      additionalProperties: false,
      required: ["title", "findings", "caveat"],
      properties: {
        title: { type: "string" },
        caveat: { type: "string" },
        findings: {
          type: "array",
          items: {
            type: "object", additionalProperties: false,
            required: ["severity", "text", "reference", "confidence"],
            properties: {
              severity: { enum: ["bad", "warn", "ok"] },
              text: { type: "string" },
              reference: { type: "string" },
              confidence: { enum: ["low", "medium", "high"] }
            }
          }
        }
      }
    }
  }
}
```

Parsing and the retry rule are described above. After two failures return HTTP 502 with code `bad_model_output`.

### 8.6 Error handling

In `claude.ts`, catch in this order and map to HTTP:

| SDK error | HTTP | User message |
|---|---|---|
| `Anthropic.AuthenticationError` | 500 | "Server is missing a valid API key." |
| `Anthropic.RateLimitError` | 429 | "The assistant is busy. Try again in a few seconds." |
| `Anthropic.BadRequestError` | 400 | Include the API's message; usually an oversized request. |
| `Anthropic.APIConnectionError` | 503 | "Could not reach the assistant." |
| any other `Anthropic.APIError` | 502 | "The assistant returned an error." |

Also check `response.stop_reason` before reading content:
- `"refusal"`: return HTTP 200 with a single answer segment: "The assistant declined to answer this question." Include `stop_details.category` in the server log only.
- `"max_tokens"`: append " [Answer was cut short.]" to the last segment.

Never match on error message strings. Never swallow errors into an empty answer.

## 9. Front end

Start from the root `index.html`. Copy it to `app/public/index.html`, split the CSS and JS into `styles.css` and `app.js`, and make these changes. Keep the look identical.

1. **Project selector** populates from `GET /api/projects`. The "Indexed for this project" card and the "Documents" tab list the real documents from that response. The "Jurisdiction" card shows the real jurisdiction.
2. **Ask the Project** sends to `POST /api/ask` with the running text `history`. Changing the project clears the chat and resets `history` to `[]`. Remove the scripted answer data. Suggested-question chips come from the demo script in section 14 (first four questions).
   - **Rendering.** The model returns markdown (lists, bold, occasional tables). Concatenate the segments into one markdown string, inserting a placeholder token such as `[[cite:3]]` after each segment that has cites. Render with a pinned `marked` build from cdnjs, sanitize the resulting HTML with a pinned `DOMPurify`, then replace each placeholder with a chip element labelled `"{title}, p. {page}"`. Never inject model text with `innerHTML` unsanitized.
   - **Slow-response message.** If no reply has arrived after 10 seconds, change the typing indicator caption to "Reading {n} documents. The first question on a project can take up to a minute." Abort the fetch at 120 seconds and show an error.
3. **Citation chip** opens the modal, shows the `quote` (if any) above a `<canvas>`, fetches `/api/projects/{id}/docs/{file}` with pdf.js, and renders `page` at a width that fits the modal. Show a page caption "{title}, page {page} of {pages}", and an "Open full PDF" link that opens the URL in a new tab. Remove the fake rendered pages.
4. **Code Lookup** sends to `POST /api/code` with the selected code book and the current project. The code-book selector defaults to the project's `jurisdiction.codeBook` or `2021 IPC`. Render like Ask, but with a visible banner above the answer: "From the assistant's knowledge of {codeBook}. Not checked against the code book. Verify before relying on it." Any project citations still render as chips; inline code section numbers render as plain text, never as chips.
5. **Photo Check** replaces the sample tiles with a real `<input type="file" accept="image/*" capture="environment">`. Show the uploaded image, then the findings list using the existing numbered badge styling. Each finding shows its `reference`; when `docFile` is non-null, the reference is a link that opens the citation modal at page 1 of that document. Drop the fake bounding-box overlay; the model returns text findings only in this version. Show the `caveat` under the findings.
6. **Error display**: any non-2xx response replaces the typing indicator with the `error.message` in a red-tinted bot bubble.
7. **Usage line** *(optional)*: show a small muted line under each answer: "{output_tokens} out, {cache_read_input_tokens} cached in". Useful during demos to show caching works.
8. Keep the footer disclaimer and the prototype badge.

Remove all demo-only data (`ASK`, `CODE`, `PHOTOS`, `DOCS` constants).

## 10. Configuration and running

`app/.env.example`:
```
ANTHROPIC_API_KEY=sk-ant-...
PORT=3000
CLAUDE_MODEL=claude-opus-5-5
```

`package.json` scripts:
- `dev`: `tsx watch server/index.ts`
- `start`: `tsx server/index.ts`
- `typecheck`: `tsc --noEmit`

`app/README.md` covers: install, set the key, add a project folder (including the `date` field and the text-layer requirement), run, the warm endpoint, the demo-day checklist from 14.3, and a note on cost (the warm call or first question is the expensive one; later questions hit the cache for an hour).

## 11. Build order

Implement and verify in this sequence. Each step should be runnable before the next starts.

1. Server skeleton: Express, static serving, `GET /api/projects` reading the sample project folder, PDF streaming endpoint with the allow-list check.
2. `claude.ts`: client, request builder with documents and caching, usage logging, error mapping. Test with a one-off script that asks one question and prints the raw response including `usage`.
3. `POST /api/ask` with citation mapping. Verify the acceptance criteria for citations and the "not found" behavior using a real spec PDF.
4. Front end wiring for projects, ask, citation viewer, errors.
5. `POST /api/code` and its tab.
6. `POST /api/photo` with `multer`, `sharp`, structured output, and its tab.
7. README, `.env.example`, `.gitignore`, final typecheck pass.

Also, within step 1, add `GET /api/health` and the boot-time token count, and within step 2 add `POST /api/projects/:id/warm`.

Committing after each step is recommended, not required. The `typecheck` script is a convenience.

**Prerequisite before step 3.** The sample document set must be in `app/projects/<id>/docs/` with its `project.json` written. The developer and architect supply it before handoff. It must contain, at minimum: one Division 22 spec section with a numeric requirement (hanger spacing or pipe slope), one addendum that changes a value in that spec, and one or two P-series sheets. All PDFs must have a text layer (vector export or OCR'd); image-only scans do not cite reliably. Without this set, steps 3 to 6 cannot be verified and the builder should stop and ask.

## 12. Non-goals for this version

- Authentication, user accounts, multi-tenant separation.
- Procore, Autodesk Construction Cloud, Bluebeam or any document source integration. Documents are files on disk.
- Vector search, embeddings, chunking.
- Streaming responses.
- Bounding boxes or annotated images from photo check.
- Reproducing copyrighted code text. The code lookup summarizes and cites section numbers only.
- Offline or low-signal operation.
- Voice input (keep the mic button, make it show a "coming soon" tooltip).
- Mobile app packaging. The web page at phone width is the mobile story.
- Deployment. Runs locally. Hosting on a small VM or Render is a follow-on.

## 13. Known constraints and assumptions

- **Request size.** Base64 inflates PDFs by a third. The 32 MB request limit means roughly 24 MB of raw PDF per project. The size guard in 6.3 enforces 22 MB of raw file size, image included, as a safe ceiling. The Files API (upload once, reference by `file_id`) removes this limit and is the first upgrade when a project is too large.
- **Page limit.** 600 pages per document. Split larger PDFs before adding them to a project.
- **Scanned drawings.** PDFs that are pure images still work (Claude reads the page image) but cost more tokens and return little or no `cited_text`, so chips lose their quotes. The sample set must have a text layer (see section 11 prerequisite).
- **Cost.** Each PDF page is billed as extracted text plus a page image, so a 200-page spec book is roughly 200K to 300K tokens and a drawing set adds more. The boot-time token count (6.2) gives the real number. On Opus 5.5 at $4 per million input tokens, a 300K-token project costs about $1.20 to read uncached, and the 1-hour cache write bills at twice that, so the first question or the warm call is roughly $2.40. Cached follow-ups within the hour bill reads at $0.20 per million, about $0.06 for the same prefix, plus output. Setting `CLAUDE_MODEL=claude-sonnet-5-5` halves uncached input and output cost but cache reads are priced the same, so it saves little once warm. Budget on the order of $5 to $10 for a one-hour demo with one warm and a dozen questions.
- **Data retention.** Project documents are sent to the Claude API. Confirm with the subcontractor that the demo documents are not under an NDA that forbids this before the first live demo.
- **Sample project.** The repo needs at least one real or realistic PDF set in `app/projects/` for the acceptance tests. Public-domain or owner-provided documents only. Do not commit copyrighted code books.

## 14. Demo script and test questions

### 14.1 Generic development questions

Use these while building, against whatever spec book is loaded. Expected behavior is described, not exact wording.

| Question | Expected |
|---|---|
| What pipe material is specified for domestic cold water above grade? | Answer with a citation to the piping spec section. |
| What is the maximum hanger spacing for 2 inch copper? | Specific distance with citation; mention code comparison if the model knows it. |
| What does the spec say about the color of the ceiling tiles? | "Not found in the project documents" or equivalent. No invention. |
| Which sheets were revised by the latest addendum? | List drawn from the addendum document, with citation. |
| Code tab, 2021 IPC: minimum slope for a 4 inch sanitary drain | Names Table 704.1 and 1/8 inch per foot, with the disclaimer. |
| Photo of any pipe run: "Is this installed correctly?" | 1 to 5 findings, at least one marked `warn` noting measurements cannot be confirmed from a photo. |

### 14.2 Demo script (fill in before the first live demo)

Once the real document set is loaded, the developer and architect write `app/DEMO.md` with five or six questions whose answers they have verified by hand against the PDFs, including the expected page number for each citation. Code Lookup questions in the script must have their section numbers checked against the code book by a person, because those answers come from model knowledge. The first four questions become the suggested-question chips in the UI. The script should show, in order: a spec fact with a citation, an addendum that changed a value (precedence), a code comparison, a question the documents do not answer, and one photo.

### 14.3 Demo-day checklist (also goes in `app/README.md`)

1. `GET /api/health` returns `hasApiKey: true` and lists the project with its token count.
2. Hit `POST /api/projects/<id>/warm` within the hour before the meeting. Confirm `cache_creation_input_tokens` is non-zero in the log.
3. Ask the first scripted question and confirm `cache_read_input_tokens` is large and the answer returns in seconds.
4. Open one citation on the phone that will be used in the demo.
5. Have the photo from the script already on that phone.
6. Keep the mockup URL open in another tab as a fallback if the API is unavailable.

## 15. Changes in version 1.1

Based on an independent review of version 1.0. Accepted and applied:

- Conversation history is text-only; documents are rebuilt server-side each turn (7.1, 8.2). Version 1.0 would have shipped the PDFs to the browser and back.
- Both cache breakpoints use the 1-hour TTL. Version 1.0 mixed a default 5-minute `system` entry ahead of a 1-hour document entry, which the API rejects.
- Size guard lowered from 28 MB to 22 MB raw, image included (6.3). 28 MB raw exceeds the 32 MB limit after base64.
- Ask, Code and Photo share one cached prefix (7.2, 8.2, 8.5). Version 1.0 created three separate caches.
- Photo asks for JSON in the prompt instead of structured outputs, because structured outputs reject citation-enabled documents and the shared prefix has them enabled. Array length constraints removed from the schema; the server enforces the count.
- Photo attaches addenda, and the image goes after the documents, not before.
- Pre-warm endpoint, health endpoint, boot-time token count, in-memory base64 cache (6.2, 7.5).
- pdf.js page rendering is required; iframe PDF viewing does not work on phones (section 4, 9.3).
- Code Lookup answers are labelled as unverified model knowledge (7.2, 9.4).
- ISO `date` on every document for precedence reasoning (6.1).
- Closed error-code table (section 7). Markdown rendering with sanitizing (9.2). Slow-response message and timeout (9.2).
- Cost estimates restated with the cache-write multiplier and the fact that Sonnet does not reduce cached read cost (13).
- Sample document set is a named prerequisite with content requirements and a text-layer rule (11).
- Demo script, demo-day checklist (14). Commit-per-step relaxed to a recommendation.

Reviewed and not applied:

- Collapsing the error-mapping table to three rows. The five-row table is small and the codes are already enumerated; collapsing saves nothing.
- Hiding the `rfi` document type in the UI. Document types only appear when a document of that type is loaded, so there is nothing to hide.
- Server-side page rendering to PNG as an alternative to pdf.js. Both solve the phone problem; pdf.js avoids a native rendering dependency on the server.
