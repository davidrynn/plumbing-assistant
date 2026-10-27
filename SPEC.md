# PipeWise Working Prototype: Build Specification

Version 1.0, 2026-10-27. Written for an implementing engineer or coding agent. Everything in this document is a requirement unless marked *optional* or *stretch*.

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
- [ ] Asking "What hanger spacing does the spec require for 2 inch copper?" against a real spec book returns an answer with at least one citation chip. Clicking the chip opens the PDF at the cited page.
- [ ] Asking a question the documents do not answer returns an explicit "not found in project documents" statement, not an invented answer.
- [ ] Code Lookup with jurisdiction "2021 IPC" returns an answer that names the code section and includes the verification disclaimer.
- [ ] Uploading a JPEG or PNG with the question "is this hanger spacing OK?" returns 1 to 5 structured findings, each with a severity and a reference to a project document or code section.
- [ ] The second identical question to the same project costs materially fewer input tokens than the first (prompt cache hit). Log `usage` on every call so this is visible in the server console.
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
| PDF viewing | Browser's native PDF viewer via `<iframe src="...pdf#page=N">` | pdf.js is a stretch goal. |
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
      "revision": "CD Set, Aug 2"
    },
    {
      "file": "P-401-R4.pdf",
      "title": "P-401 Plumbing Plan Level 4",
      "type": "drawing",
      "revision": "Rev 4, Oct 14"
    }
  ]
}
```

`type` is one of `spec`, `drawing`, `addendum`, `submittal`, `rfi`, `other`. It is shown in the UI and included in the document title passed to Claude so the model can reason about precedence (an addendum supersedes the spec it amends).

Validate on load. A project with a missing file or malformed JSON is logged and skipped, never crashes the server.

### 6.2 Projects at startup

`projects.ts` scans `app/projects/*/project.json` once at boot and caches the result. Provide `GET /api/projects` returning `[{id, name, jurisdiction, documents:[{title,type,revision,file,pages?}]}]`. Reload on each request in dev mode is *optional*.

### 6.3 Size guard

Before building a request, sum the document file sizes. If the total exceeds 28 MB, return HTTP 413 with a message naming the project and the limit. The Claude request limit is 32 MB. Do not silently drop documents.

## 7. API endpoints

All responses are JSON. All errors use `{ "error": { "code": string, "message": string } }` with an appropriate HTTP status. `message` must be safe to show to an end user.

### 7.1 `POST /api/ask`

Request:
```json
{ "projectId": "riverside-tower-b", "question": "What hanger spacing...", "history": [] }
```
`history` is an array of prior `{role, content}` turns for this conversation, as returned by earlier responses. Cap at the last 10 turns server-side. Empty array on first question.

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

`answer` is an ordered list of text segments. A segment with a non-empty `cites` array renders with citation chips after it. The client concatenates `text` in order.

### 7.2 `POST /api/code`

Request:
```json
{ "projectId": "riverside-tower-b", "codeBook": "2021 IPC", "question": "hanger spacing for 2 inch copper" }
```
`projectId` is optional. When present, attach the project's `spec` and `addendum` documents (not drawings) so the answer can say whether the spec is stricter than code. `codeBook` is free text chosen from a fixed list in the UI: `2021 IPC`, `2018 IPC`, `2021 UPC`, `2018 UPC`, `NFPA 99 2021`.

Response: same shape as 7.1 but `citations` may be empty when no project is attached. Code references appear inline in the text as section numbers (for example "IPC 308.5"). Do not attempt to deep-link into code books.

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
`severity` is one of `bad`, `warn`, `ok`. `confidence` is one of `low`, `medium`, `high`. 1 to 5 findings.

### 7.4 `GET /api/projects/:id/docs/:file`

Streams the PDF with `Content-Type: application/pdf` and `Content-Disposition: inline`. Reject any `file` not listed in that project's `project.json` (prevents path traversal). The client opens `/api/projects/<id>/docs/<file>#page=<n>` in the viewer iframe.

## 8. Claude integration

All API calls go through `claude.ts`. One `Anthropic` client instance, constructed with no arguments so it reads `ANTHROPIC_API_KEY` from the environment.

### 8.1 Model and request defaults

| Setting | Value | Reason |
|---|---|---|
| `model` | `process.env.CLAUDE_MODEL ?? "claude-opus-5-5"` | Current Opus. Overridable for cost experiments. |
| `max_tokens` | 16000 | Non-streaming default that stays under SDK timeouts. |
| `thinking` | omit | Always on for this model. Do not send `disabled` or `budget_tokens`, both return 400. |
| `output_config.effort` | `"medium"` for ask and code, `"high"` for photo | Photo analysis benefits from more reasoning. |
| `betas` | `["server-side-fallback-2026-07-01"]` | Enables the fallback below. Requires `client.beta.messages.create`. |
| `fallbacks` | `"default"` | If a safety classifier declines, the API retries on an appropriate fallback model. |
| `temperature`, `top_p`, `top_k` | never send | Rejected on current API versions. |
| `tool_choice` | never send | Not used in this POC. |

Streaming is a *stretch* goal. The POC uses non-streaming calls with a typing indicator in the UI.

### 8.2 Document attachment and caching

Build the first user message of every ask or code request as:

```ts
const content: Anthropic.Beta.BetaContentBlockParam[] = [
  ...project.documents.map((d, i) => ({
    type: "document" as const,
    source: { type: "base64" as const, media_type: "application/pdf" as const, data: readBase64(d.file) },
    title: `${d.type.toUpperCase()}: ${d.title} (${d.revision})`,
    citations: { enabled: true },
    // cache_control only on the LAST document block
    ...(i === project.documents.length - 1 ? { cache_control: { type: "ephemeral" as const, ttl: "1h" as const } } : {}),
  })),
  { type: "text", text: question },
];
```

Rules:
- Documents first, question last. The cache breakpoint on the last document caches every document in one prefix.
- Document order must be deterministic (the order in `project.json`). Reordering breaks the cache.
- Base64 strings must contain no newlines.
- Put the project jurisdiction text in the `system` prompt, which also carries `cache_control`. Nothing time-varying (dates, request IDs) goes in `system` or before the last cache breakpoint.
- For follow-up turns, keep the documents in the *first* user message of `history` and append new turns after. Do not re-attach documents on every turn.
- Log `usage.cache_read_input_tokens` and `usage.cache_creation_input_tokens` on every call. Acceptance criterion 7 depends on the second call showing a large `cache_read_input_tokens`.

### 8.3 Citations

With `citations: { enabled: true }` on every document block, the response `content` is a list of `text` blocks. Blocks that cite carry a `citations` array. Each citation has `type: "page_location"`, `document_index`, `document_title`, `cited_text`, `start_page_number` (1-indexed) and `end_page_number`.

Map to the response shape in 7.1:
- One `answer` segment per text block.
- De-duplicate citations by `(document_index, start_page_number)` so the same page is one chip.
- `page` is `start_page_number`.
- `quote` is `cited_text` trimmed to 200 characters.

Citations are incompatible with `output_config.format` (structured outputs). That is why the photo endpoint, which uses structured output, does not enable citations. See 8.5.

### 8.4 System prompts (`prompts.ts`)

**Ask the Project**

> You are a field assistant for commercial plumbers working on the project named below. Answer only from the attached project documents. When a document answers the question, cite it. When the documents do not answer the question, say so plainly in one sentence and do not guess. When an addendum or later revision changes an earlier document, the later one governs; say which document governs and why. Where the project specification is stricter than the applicable code, say so. Use plain language a journeyman plumber would use. Keep answers under 200 words unless the question needs a list or table. Use imperial units. End every answer that affects installation with one sentence noting that the engineer of record and the inspector make the final call.
>
> Project: {name}
> Adopted code: {jurisdiction.code}
> Authority having jurisdiction: {jurisdiction.ahj}
> Other applicable standards: {jurisdiction.extra joined}

**Code Lookup**

> You are a plumbing code reference assistant. The user has selected the code edition named below. Answer from your knowledge of that edition. Name the section or table number for every requirement you state. Summarize requirements in your own words; do not reproduce code text verbatim beyond a short phrase. If the user has attached project specifications, compare the code requirement with the specification and state which is stricter. If the selected edition does not cover the question, say so. State clearly when a requirement commonly varies by local amendment. Close with one sentence: verify with the adopted code and the authority having jurisdiction.
>
> Code edition: {codeBook}
> {project block as above, if present}

**Photo Check**

> You are a field assistant reviewing a job-site photo for a commercial plumber. Identify what plumbing work is visible. Then list between one and five findings about compliance with the attached project documents and the adopted code. For each finding give a severity (bad means likely non-compliant, warn means needs verification, ok means appears compliant), a one or two sentence explanation, a reference to the governing spec section or code section, and a confidence level. A photo cannot measure distances reliably; when a finding depends on a measurement, say the value must be verified with a tape. Never report a finding you cannot see evidence for in the photo. Respond only in the required JSON format.

### 8.5 Photo request

User content: the image block (base64, after downscaling), then the project's `spec` and `submittal` documents (no drawings; too many tokens for a quick answer), then the text question or "Review this photo." Documents are attached with `citations` **disabled**.

Use structured outputs:

```ts
output_config: {
  effort: "high",
  format: {
    type: "json_schema",
    schema: {
      type: "object",
      additionalProperties: false,
      required: ["title", "findings", "caveat"],
      properties: {
        title: { type: "string" },
        caveat: { type: "string" },
        findings: {
          type: "array", minItems: 1, maxItems: 5,
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

Parse the JSON from the text block. If parsing fails, return HTTP 502 with code `bad_model_output`.

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
2. **Ask the Project** sends to `POST /api/ask` with the running `history`. Render segments in order; after any segment with cites, render one chip per cite labelled `"{title}, p. {page}"`. Remove the scripted answer data. Keep the suggested-question chips but make them plain example questions that work with any spec book (for example "What pipe material is specified for domestic water?", "What are the hanger spacing requirements?", "Which documents have been revised by an addendum?").
3. **Citation chip** opens the modal with an `<iframe>` pointing at `/api/projects/{id}/docs/{file}#page={page}`, with the quote shown above the iframe. Remove the fake rendered pages.
4. **Code Lookup** sends to `POST /api/code` with the selected code book and the current project. Render like Ask.
5. **Photo Check** replaces the sample tiles with a real `<input type="file" accept="image/*" capture="environment">`. Show the uploaded image, then the findings list using the existing numbered badge styling. Drop the fake bounding-box overlay; the model returns text findings only in this version.
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

`app/README.md` covers: install, set the key, add a project folder, run, and a note on cost (first question to a project is the expensive one; later questions hit the cache for an hour).

## 11. Build order

Implement and verify in this sequence. Each step should be runnable before the next starts.

1. Server skeleton: Express, static serving, `GET /api/projects` reading the sample project folder, PDF streaming endpoint with the allow-list check.
2. `claude.ts`: client, request builder with documents and caching, usage logging, error mapping. Test with a one-off script that asks one question and prints the raw response including `usage`.
3. `POST /api/ask` with citation mapping. Verify the acceptance criteria for citations and the "not found" behavior using a real spec PDF.
4. Front end wiring for projects, ask, citation viewer, errors.
5. `POST /api/code` and its tab.
6. `POST /api/photo` with `multer`, `sharp`, structured output, and its tab.
7. README, `.env.example`, `.gitignore`, final typecheck pass.

Commit after each step with a message naming the step.

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

- **Request size.** Base64 inflates PDFs by a third. The 32 MB request limit means roughly 24 MB of raw PDF per project. The size guard in 6.3 enforces 28 MB of raw file size as a safe ceiling for base64 plus overhead. The Files API (upload once, reference by `file_id`) removes this limit and is the first upgrade when a project is too large.
- **Page limit.** 600 pages per document. Split larger PDFs before adding them to a project.
- **Scanned drawings.** PDFs that are pure images still work (Claude reads the page image) but cost more tokens and cite less precisely. Prefer vector PDFs exported from CAD.
- **Cost.** A 200-page spec book is roughly 150K to 250K tokens. First question to a project is on the order of a dollar on Opus 5.5; cached follow-ups within the hour are a few cents plus output. Set `CLAUDE_MODEL=claude-sonnet-5-5` to run the same demo at about half the price.
- **Data retention.** Project documents are sent to the Claude API. Confirm with the subcontractor that the demo documents are not under an NDA that forbids this before the first live demo.
- **Sample project.** The repo needs at least one real or realistic PDF set in `app/projects/` for the acceptance tests. Public-domain or owner-provided documents only. Do not commit copyrighted code books.

## 14. Sample test questions

Use these during development against whatever spec book is loaded. Expected behavior is described, not exact wording.

| Question | Expected |
|---|---|
| What pipe material is specified for domestic cold water above grade? | Answer with a citation to the piping spec section. |
| What is the maximum hanger spacing for 2 inch copper? | Specific distance with citation; mention code comparison if the model knows it. |
| What does the spec say about the color of the ceiling tiles? | "Not found in the project documents" or equivalent. No invention. |
| Which sheets were revised by the latest addendum? | List drawn from the addendum document, with citation. |
| Code tab, 2021 IPC: minimum slope for a 4 inch sanitary drain | Names Table 704.1 and 1/8 inch per foot, with the disclaimer. |
| Photo of any pipe run: "Is this installed correctly?" | 1 to 5 findings, at least one marked `warn` noting measurements cannot be confirmed from a photo. |
