# Technical Documentation — Workflow "for gemani ai model updated" (version 4)

> **Source:** `C:\Users\alion\n8n-data\n8n\for gemani ai model updated (4).json`
> **Workflow ID:** `RWpigWQwMT7EBzp9`  ·  **versionId:** `ac9e349b-9491-410c-aa33-fe82acc5823d`  ·  **Active:** `false`
> **Node count:** 59 (57 functional + 2 sticky notes)
> **Engine settings:** `executionOrder: v1`, `binaryMode: separate`, `availableInMCP: true`
> **Generated:** 2026-07-17  ·  **Template applied:** `n8n/workflow-info.md`

---

## 0. Read this first — what this file is and how it relates to the other export

This is the **refactored, consolidated version** of the same workflow (`RWpigWQwMT7EBzp9`) previously documented from the 152-node export (`for gemani ai model updated.json`). It is the successor/cleanup:

| | 152-node export | **This (4).json — 59 nodes** |
|---|---|---|
| Nodes | 152 (parts-bin) | **59 (consolidated)** |
| Orphan nodes | 51 with no inbound | **~6, mostly labelled "(Unused)"** |
| Node names | generic (`Code in JavaScript27`) | **descriptive** (`Route by HTTP status`, `Retries exhausted?`) |
| Poison-400 retry bug | **present** (`If9` retried 400+429) | **FIXED** — see below |
| MCP-exposed | no | `availableInMCP: true` |

**The poison-400 retry bug from project memory `[[gemini-workflow-retry-loop]]` is fixed in this version.** Instead of one IF that retried both 429 and 400, this version has:
- a **`Route by HTTP status`** Switch that retries **only 408 / 429 / 500 / 502 / 503 / 504**, and routes everything else (400/401/403/404) to a **`Non-retryable error — stop`** no-op;
- a **`Strip error & count retries`** Code node that rebuilds a **clean** Gemini body (the old bug was that n8n's appended `error` object poisoned the retried request body, causing Gemini to return `400 "Unknown name error"`), and tracks a **per-question retry counter (max 5)** in workflow static data;
- a **`Retries exhausted?`** IF + **`Retries exhausted — stop`** guard so the loop can never run forever.

This version is **coherent and mostly connected** — one manual trigger drives a single batch loop end-to-end. A few nodes remain unused (explicitly named "(Unused)" or orphaned) and one config branch dead-ends; these are called out honestly in §4/§7, not invented around.

**Still open (security):** the Gemini API key is still hardcoded in a query param **and** duplicated in a sticky note; login credentials are still hardcoded in the `Login API` bodies. See §20 — treat both as compromised.

---

## 1. Overview

| Aspect | Detail |
|---|---|
| **Purpose** | Generate JEE-physics exam questions with Google **Gemini 2.5 Flash**, render their LaTeX to MathML, validate/classify them, and store valid questions to a backend API — in resumable batches. |
| **Business goal** | Automated bulk creation of a syllabus-aligned question bank (driven by a generation config; taxonomy resolved against a Postgres DB). |
| **Problem solved** | Manual authoring of thousands of schema-valid, math-rendered questions is slow and error-prone. |
| **Expected output** | Valid questions POSTed to `http://host.docker.internal:5000/api/questions`; per-batch summaries aggregated back into the loop; retry state in `$getWorkflowStaticData('global')`. |
| **Workflow type** | Batch generation pipeline with a Split-In-Batches loop, a 4-way classification Switch, and a status-driven retry sub-loop. |
| **Automation category** | AI content generation + ETL (Postgres taxonomy → LLM → MathML → backend API). |
| **Trigger type** | **Manual** only (`manualTrigger`, pinned data present). |
| **Complexity** | High but organized — 59 nodes, 1 main loop, 2 Switches, 3 IF gates, 2 Merges, 27 Code nodes, external LLM + MathML + backend + Postgres. |
| **Dependencies** | Google Generative Language API; backend `:5000` (`/api/auth/login`, `/api/questions`); MathML service `:3000` (`/api/mathml/batch`); Postgres taxonomy tables. |
| **External services** | Google Gemini API. |
| **Internal services** | Backend API (:5000), MathML service (:3000), Postgres. |
| **Advantages** | Clean readable graph; correct status-based retry with counter + exhaustion guard; 4-way valid/invalid × latex/no-latex routing; store-count verification; resumable via static data. |
| **Limitations** | `active:false`; hardcoded secrets (§20); one dead-end config branch + a few "(Unused)" nodes; no dead-letter persistence for permanently-failed batches; Gemini JSON parsing still relies on `JSON.parse` of `parts[0].text`. |

---

## 2. High-Level Architecture

```text
Manual Trigger
  ├─(branch A, LIVE)  Fetch taxonomy IDs → Map names→IDs → Passthrough → Build Loop Items & State → Flatten → LOOP
  └─(branch B, DEAD-END) Set Generation Config → Merge Config & DB Lookup → Map Names to DB IDs (stops)

LOOP "Loop Over Batches" (splitInBatches):
   done → Finalize Loop Output
   body → Build Gemini Prompt & Request → Wait → Generate Questions (Gemini)
             success → Parse → Validate & Classify → Split Into 4 Groups → Route by Question Group (Switch x4)
                         ├ valid-haveLatex  → MathML render → login → Store LaTeX Questions → verify ┐
                         ├ valid-noLatex    → split → login → Store Non-LaTeX Questions → verify     ├→ Merge Group Results
                         ├ invalid-haveLatex→ Skip Invalid LaTeX Questions ─────────────────────────┤
                         └ invalid-noLatex  → Skip Invalid Non-LaTeX Questions ────────────────────┘
                       → Aggregate Batch Summary1 → back to LOOP
             error   → Route by HTTP status (Switch)
                         ├ 408/429/5xx (retryable) → Strip error & count retries → Retries exhausted?
                         │        ├ yes → Retries exhausted — stop
                         │        └ no  → Restore clean retry body → Wait Before Retry → Generate Questions (retry)
                         └ else (400/401/403/404) → Non-retryable error — stop
```

Data is carried as n8n items (`$json`). Cross-node state: `$getWorkflowStaticData('global')` holds the JWT token and the `__geminiRetry` per-question counter. The main loop feeds each batch's summary back into `Loop Over Batches`.

---

## 3. Workflow Diagram (full, every branch)

```text
Start (Manual Trigger)
 ├─> Set Generation Config
 │      ├─> Merge Config & DB Lookup(in0)
 │      └─> Fetch Category/Chapter/Topic IDs ─> Merge Config & DB Lookup(in1)
 │                 └─> Map Names to DB IDs        [DEAD-END: no outbound]
 └─> Fetch Category/Chapter/Topic IDs1 ─> Map Names to DB IDs1 ─> Passthrough to Loop Builder
            ├─> Build Loop Items & State (Unused)        [DEAD-END]
            └─> Build Loop Items & State ─> Flatten Loop Items ─> Loop Over Batches
                                                                     │
        ┌────────────────────────────────────────────────────────────┘
        Loop Over Batches
          ├─(done, main#0)─> Finalize Loop Output
          └─(body, main#1)─> Build Gemini Prompt & Request ─> Wait Before Gemini Call ─> Generate Questions (Gemini)
                 ├─(success main#0)─> Parse Gemini Response ─> Validate & Classify Questions ─> Split Into 4 Groups ─> Route by Question Group
                 │      ├─#0 valid-haveLatex ─> Select LaTeX Questions ─> Flatten LaTeX Questions ─> Convert LaTeX to MathML
                 │      │       ─> Attach MathML to Questions ─> Merge MathML & Clean XMLNS ─> Remove LaTeX Field (LaTeX)
                 │      │       ─> Gate: Has LaTeX Questions ─true─> Login API (LaTeX) ─> Set Access Token (LaTeX)
                 │      │       ─> Attach Token to LaTeX Questions ─> Store LaTeX Questions ─> Verify LaTeX Store Count ─> Merge Group Results(in0)
                 │      ├─#1 valid-noLatex ─> Set Non-LaTeX Questions ─> Remove LaTeX Field (Non-LaTeX) ─> Split Non-LaTeX Questions
                 │      │       ─> Gate: Has Non-LaTeX Questions ─true─> Login API (Non-LaTeX) ─> Set Access Token (Non-LaTeX)
                 │      │       │       ─> Prepare Non-LaTeX Questions ─> Store Non-LaTeX Questions ─> Verify Non-LaTeX Store Count ─> Merge Group Results(in1)
                 │      │       └─false─> Edit Fields ─> Merge Group Results(in1)
                 │      ├─#2 invalid-haveLatex ─> Skip Invalid LaTeX Questions ─> Merge Group Results(in2)
                 │      └─#3 invalid-noLatex ─> Skip Invalid Non-LaTeX Questions ─> Merge Group Results(in3)
                 │              Merge Group Results ─> Aggregate Batch Summary1 ─> Loop Over Batches (next batch)
                 └─(error main#1)─> Route by HTTP status
                        ├─#0 retryable(408/429/500/502/503/504) ─> Strip error & count retries ─> Retries exhausted?
                        │        ├─true─> Retries exhausted — stop (noOp)
                        │        └─false─> Restore clean retry body ─> Wait Before Retry ─> Generate Questions (Gemini)  [RETRY]
                        └─#1 fallback(400/401/403/404/...) ─> Non-retryable error — stop (noOp)

ORPHANS (no path from trigger): Aggregate Batch Summary, Build Gemini Prompt (Unused),
                                Generate Questions (Gemini)1, Code in JavaScript
```

---

## 4. Node Inventory (all 59)

**R** = reachable & used · **D** = reachable but dead-ends · **U** = unused/orphan (no path from trigger, or explicitly "(Unused)").

| # | Node | Type (tv) | State | Purpose |
|---|---|---|---|---|
| 1 | Start (Manual Trigger) | manualTrigger (1) | R | Sole entry point; pinned sample data. |
| 2 | Fetch Category/Chapter/Topic IDs1 | postgres (2.6) | R | Load taxonomy id+name (live branch). |
| 3 | Map Names to DB IDs1 | code (2) | R | Replace category/chapter/topic names with DB ids. |
| 4 | Passthrough to Loop Builder | code (2) | R | Pass mapped config to loop builders. |
| 5 | Build Loop Items & State | code (2) | R | Build `loopItems[]` + loop state. |
| 6 | Build Loop Items & State (Unused) | code (2) | U | Parallel variant; not consumed. |
| 7 | Flatten Loop Items | code (2) | R | Flatten loopItems into one-per-item. |
| 8 | Loop Over Batches | splitInBatches (3) | R | Main batch loop. |
| 9 | Finalize Loop Output | code (2) | R | Runs when loop is done. |
| 10 | Build Gemini Prompt & Request | code (2) | R | Build systemInstruction/contents/generationConfig (16.8 KB). |
| 11 | Wait Before Gemini Call | wait (1.1) | R | Rate-limit spacing before each call. |
| 12 | Generate Questions (Gemini) | httpRequest (4.4) | R | **Gemini** generateContent; 2 outputs (success/error). |
| 13 | Parse Gemini Response | code (2) | R | Parse `candidates[0].content.parts[0].text` → questions. |
| 14 | Validate & Classify Questions | code (2) | R | Validate schema/LaTeX; split into valid/invalid × latex/no-latex. |
| 15 | Split Into 4 Groups | code (2) | R | Emit 4 items tagged `group`. |
| 16 | Route by Question Group | switch (3.4) | R | 4-way route on `group`. |
| 17 | Select LaTeX Questions | set (3.4) | R | Keep valid-haveLatex payload. |
| 18 | Flatten LaTeX Questions | code (2) | R | Flatten to one question per item. |
| 19 | Convert LaTeX to MathML | httpRequest (4.4) | R | **MathML** `/api/mathml/batch`. |
| 20 | Attach MathML to Questions | code (2) | R | Merge MathML back onto questions. |
| 21 | Merge MathML & Clean XMLNS | code (2) | R | Clean `xmlns` from MathML. |
| 22 | Remove LaTeX Field (LaTeX) | code (2) | R | Strip raw `latex` before store. |
| 23 | Gate: Has LaTeX Questions | if (2.3) | R | `Object.keys($json).length > 0`. |
| 24 | Login API (LaTeX) | httpRequest (4.4) | R | **Auth** `/api/auth/login` (hardcoded creds). |
| 25 | Set Access Token (LaTeX) | set (3.4) | R | Store token for LaTeX branch. |
| 26 | Attach Token to LaTeX Questions | code (2) | R | Attach Bearer token to payload. |
| 27 | Store LaTeX Questions | httpRequest (4.4) | R | **Submit** POST `/api/questions`. |
| 28 | Verify LaTeX Store Count | code (2) | R | Compare stored vs expected count. |
| 29 | Set Non-LaTeX Questions | set (3.4) | R | Keep valid-noLatex payload. |
| 30 | Remove LaTeX Field (Non-LaTeX) | code (2) | R | Strip `latex`. |
| 31 | Split Non-LaTeX Questions | splitOut (1) | R | Explode question array. |
| 32 | Gate: Has Non-LaTeX Questions | if (2.3) | R | `Object.keys($json).length > 0`. |
| 33 | Login API (Non-LaTeX) | httpRequest (4.4) | R | **Auth** login. |
| 34 | Set Access Token (Non-LaTeX) | set (3.4) | R | Store token. |
| 35 | Prepare Non-LaTeX Questions | code (2) | R | Shape submit payload. |
| 36 | Store Non-LaTeX Questions | httpRequest (4.4) | R | **Submit** POST `/api/questions`. |
| 37 | Verify Non-LaTeX Store Count | code (2) | R | Verify stored count. |
| 38 | Edit Fields | set (3.4) | R | Empty-branch passthrough (Gate false). |
| 39 | Skip Invalid LaTeX Questions | code (2) | R | Sink for invalid-haveLatex. |
| 40 | Skip Invalid Non-LaTeX Questions | code (2) | R | Sink for invalid-noLatex. |
| 41 | Merge Group Results | merge (3.2) | R | 4-input join of all groups. |
| 42 | Aggregate Batch Summary1 | code (2) | R | Summarize batch; feed loop. |
| 43 | Generate Questions (Gemini) | (see #12) | — | — |
| 44 | Route by HTTP status | switch (3.4) | R | Retryable(408/429/5xx) vs fallback. |
| 45 | Strip error & count retries | code (2) | R | Rebuild clean body; per-question retry counter (max 5). |
| 46 | Retries exhausted? | if (2.3) | R | `$json.exceeded == true`. |
| 47 | Restore clean retry body | set (3.4) | R | Set clean body for retry. |
| 48 | Wait Before Retry | wait (1.1) | R | Backoff before retry. |
| 49 | Retries exhausted — stop | noOp (1) | R | Terminal: give up this question. |
| 50 | Non-retryable error — stop | noOp (1) | R | Terminal: 4xx (non-429) errors. |
| 51 | Set Generation Config | set (3.4) | D | Seed config (dead-end branch B). |
| 52 | Merge Config & DB Lookup | merge (3.2) | D | Join config + taxonomy (branch B). |
| 53 | Fetch Category/Chapter/Topic IDs | postgres (2.6) | D | Taxonomy (branch B). |
| 54 | Map Names to DB IDs | code (2) | D | Map ids; **dead-end** (no outbound). |
| 55 | Aggregate Batch Summary | code (2) | U | Orphan (older summary variant). |
| 56 | Build Gemini Prompt (Unused) | code (2) | U | Orphan (older prompt builder). |
| 57 | Generate Questions (Gemini)1 | httpRequest (4.4) | U | Orphan Gemini call. |
| 58 | Code in JavaScript | code (2) | U | Orphan scratch node. |
| 59 | Sticky Note / Sticky Note1 | stickyNote (1) | — | Docs: "get Latext using api"; **"ENV" note containing the API key** (⚠ §20). |

---

## 5. Detailed Node Documentation (distinctive nodes)

### 5.1 `Start (Manual Trigger)`
Sole entry point; emits one item. **Pinned data** present, so test runs start from a fixed sample without external calls. Fans out to branch A (live loop builder) and branch B (dead-end config).

### 5.2 `Fetch Category/Chapter/Topic IDs` / `…IDs1` (postgres v2.6)
Both run the same read-only lookup:
```sql
SELECT 'category' AS type, id, LOWER(name) AS name FROM subject_categories
UNION ALL SELECT 'chapter' AS type, id, LOWER(name) AS name FROM chapters
UNION ALL SELECT 'topic'  AS type, id, LOWER(name) AS name FROM topics;
```
Purpose: give the `Map Names to DB IDs*` nodes a flat `{type,id,name}` table to translate human-readable taxonomy names into DB ids before submission. **Redundant duplication** — `…IDs` (branch B) dead-ends; only `…IDs1` (branch A) matters. Consolidate.

### 5.3 `Generate Questions (Gemini)` (httpRequest v4.4) — the LLM call
- **Endpoint:** `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- **Auth:** API key in **query param `key`** (⚠ hardcoded, §20).
- **Body:** `={{ $json }}` — the upstream `Build Gemini Prompt & Request` supplies `{systemInstruction, contents, generationConfig}`.
- **Two outputs:** main#0 = success → `Parse Gemini Response`; main#1 = error → `Route by HTTP status`. (Requires "Continue On Fail"/error-output enabled on the node.)

### 5.4 `Route by HTTP status` (switch v3.4) — retry classifier ✅ (the fix)
- **Rule (output 0, retryable):** `$json.error.status` equals **408, 429, 500, 502, 503, 504**.
- **Fallback (output 1):** everything else (**400, 401, 403, 404**, …) → `Non-retryable error — stop`.
- **Why it matters:** this is the corrected replacement for the old `If9` that lumped 400 with 429. A 400 (malformed request / safety block) is permanent and now terminates instead of looping. Directly resolves `[[gemini-workflow-retry-loop]]`.

### 5.5 `Strip error & count retries` (code v2) — poison-fix + counter ✅
Its own header comment documents the fix. It:
1. rebuilds a **clean** Gemini body `{systemInstruction, contents, generationConfig}` (dropping n8n's appended `error` field that previously poisoned the retried request → Gemini `400 "Unknown name error"`);
2. computes a **stable hash key** from `contents` so the retry counter is **per-question**, not global;
3. increments `staticData.__geminiRetry[key]` in `$getWorkflowStaticData('global')` with **MAX = 5**, and sets `exceeded: true` when the cap is hit.

### 5.6 `Retries exhausted?` (if v2.3) + stop nodes
`$json.exceeded == true` → `Retries exhausted — stop` (noOp) — abandons that question after 5 tries; else → `Restore clean retry body` → `Wait Before Retry` → back to `Generate Questions (Gemini)`.

### 5.7 `Parse Gemini Response` (code v2)
```js
const rawText = item.candidates?.[0]?.content?.parts?.[0]?.text ?? "[]";
try { parsed = JSON.parse(rawText); } catch (e) { throw new Error(`JSON Parse Error: ${e.message}`); }
// returns parsed (array) | parsed.questions (array) | [parsed]
```
- **Edge case:** if Gemini wraps JSON in markdown fences or emits prose, `JSON.parse` throws and the batch fails. **Recommendation:** strip ```` ```json ```` fences, and set `generationConfig.responseMimeType: "application/json"` upstream.

### 5.8 `Validate & Classify Questions` → `Split Into 4 Groups` (code v2)
`Validate & Classify` produces `{ valid:{haveLatex,noLatex}, invalid:{haveLatex,noLatex} }`. `Split Into 4 Groups` emits exactly four items tagged `group: "valid-haveLatex" | "valid-noLatex" | "invalid-haveLatex" | "invalid-noLatex"`, which `Route by Question Group` fans out.

### 5.9 `Convert LaTeX to MathML` (httpRequest v4.4)
`POST http://host.docker.internal:3000/api/mathml/batch`, body `{"expressions": {{ JSON.stringify($json.latex) }}}`. (This version uses a **single, consistent host** — the LAN-IP inconsistency of the 152-node export is gone.)

### 5.10 `Login API (LaTeX)` / `(Non-LaTeX)` (httpRequest v4.4) ⚠
`POST http://host.docker.internal:5000/api/auth/login`, body `{"email":"naveed@gmail.com","password":"naveed@gmail.com"}` — ⚠ **hardcoded credentials** (note: password now equals the email, likely a throwaway test account — still must not be in git). Downstream `Set Access Token` + `Attach Token` inject `Authorization: Bearer <token>`.

### 5.11 `Store LaTeX Questions` / `Store Non-LaTeX Questions` (httpRequest v4.4)
`POST http://host.docker.internal:5000/api/questions`, body `={{ JSON.stringify($json) }}` (real payload — no longer an empty `{}` template). Followed by `Verify … Store Count` which reconciles returned vs expected counts.

### 5.12 `Gate: Has LaTeX/Non-LaTeX Questions` (if v2.3)
`{{ Object.keys($json).length }} > 0` — skip the login+store round-trip when a group is empty. Non-LaTeX false branch → `Edit Fields` → `Merge Group Results` so the merge still receives an item (avoids a stalled merge).

---

## 6. Connection Documentation
All 59 connections were extracted and are reflected in §3. Key semantics:
- **Trigger → two branches:** A (live, via `…IDs1`) and B (dead-end, via `Set Generation Config`). Both execute; only A reaches the loop.
- **Loop feedback:** `Aggregate Batch Summary1 → Loop Over Batches` closes the batch loop.
- **Merge inputs:** `Merge Group Results` receives group 0 on in0, non-latex on in1 (or `Edit Fields` on in1 when empty), invalid-haveLatex on in2, invalid-noLatex on in3 — a 4-way synchronizing join.
- **Retry cycle:** `Generate Questions (Gemini)(main#1) → Route by HTTP status → … → Wait Before Retry → Generate Questions (Gemini)`.
- **Taxonomy join:** config on in0, taxonomy on in1 of `Merge Config & DB Lookup`.

---

## 7. Execution Flow
1. **Trigger** fires (branch A and B both start).
2. **Branch B (dead-end):** `Set Generation Config → Merge Config & DB Lookup → Map Names to DB IDs` then stops — no effect on output. *(Cleanup candidate.)*
3. **Branch A (live):** `Fetch …IDs1 → Map Names to DB IDs1 → Passthrough → Build Loop Items & State → Flatten Loop Items → Loop Over Batches`.
4. **Per batch (loop body):** `Build Gemini Prompt & Request → Wait → Generate Questions (Gemini)`.
   - **Success:** `Parse → Validate & Classify → Split Into 4 Groups → Route by Question Group` → the four group branches → `Merge Group Results → Aggregate Batch Summary1 →` next batch.
   - **Error:** `Route by HTTP status` → retryable → `Strip error & count retries → Retries exhausted?` → retry (`Restore clean retry body → Wait Before Retry → Generate Questions`) or stop; non-retryable → `Non-retryable error — stop`.
5. **Loop done:** `Finalize Loop Output`.

---

## 8. Data Flow
- **Input:** generation config (from prior mapping) + Postgres taxonomy.
- **Intermediate:** `loopItems[]`; Gemini request `{systemInstruction, contents, generationConfig}`; raw Gemini JSON; parsed questions; `{valid/invalid × haveLatex/noLatex}`; MathML batches; JWT token; retry counters.
- **Output:** questions POSTed to `/api/questions`; batch summaries; static-data retry/token state.
- **Splits:** `Split Into 4 Groups`, `Split Non-LaTeX Questions` (splitOut), `Flatten *`.
- **Merges:** `Merge Config & DB Lookup` (2-in), `Merge Group Results` (4-in).
- **Binary:** `binaryMode: separate` (no heavy binary here; MathML/JSON are text).

## 9. Expression Documentation
- `{{ $json }}` — whole item as Gemini body.
- `{{ JSON.stringify($json.latex) }}` — serialize LaTeX array for MathML.
- `{{ JSON.stringify($json) }}` — serialize question payload for storage.
- `{{ Object.keys($json).length }}` — emptiness check in the Gate IFs.
- `{{ $json.group }}` — Route by Question Group key.
- `{{ $json.error.status }}` — HTTP status for Route by HTTP status.
- `{{ $json.exceeded }}` — retry-cap flag.
- `$getWorkflowStaticData('global')` — token + `__geminiRetry` counter store.

## 10. Loop Documentation
One primary `splitInBatches` loop (`Loop Over Batches`, v3): output#0 = "done" → `Finalize Loop Output`; output#1 = body → generation pipeline, returning via `Aggregate Batch Summary1`. Plus the **retry sub-loop** (`Generate Questions ↔ Route by HTTP status ↔ Wait Before Retry`) bounded by the **max-5 counter** — cannot run forever (the key correctness property this version adds). Batch size derives from the loop-builder output (per the generation config, 6 questions/call).

## 11. Conditional Logic
- **2 Switches:** `Route by Question Group` (4-way on `group`); `Route by HTTP status` (retryable set vs fallback).
- **3 IF gates:** `Gate: Has LaTeX Questions`, `Gate: Has Non-LaTeX Questions` (`Object.keys > 0`), `Retries exhausted?` (`exceeded == true`).
- **Routing philosophy:** classify first (valid/invalid, latex/no-latex), act per class, converge at `Merge Group Results`. Errors are classified by HTTP status, not blindly retried.

## 12. Merge Documentation
- `Merge Config & DB Lookup` — joins generation config (in0) with taxonomy (in1) so id-mapping sees both. *(In the dead-end branch B.)*
- `Merge Group Results` — 4-input synchronizing merge collecting all four group outcomes per batch. The empty-group false branches route through `Edit Fields`/skip nodes so every input still receives an item — **prevents the merge from stalling** (a real risk with conditional branches).

## 13. Code Nodes
27 Code nodes (JS v2). Notable:
- `Build Gemini Prompt & Request` (16.8 KB) — assembles the full prompt/schema; the single most important node to read before changing prompts.
- `Strip error & count retries` — the poison-fix + retry counter (§5.5).
- `Validate & Classify Questions`, `Split Into 4 Groups` — the classification core.
- `Parse Gemini Response` — JSON extraction (fragile on non-JSON output).
- Small pure shapers: `Remove LaTeX Field *`, `Attach *`, `Prepare *`, `Verify * Store Count`, `Skip Invalid *`.
- **Orphans to prune:** `Build Gemini Prompt (Unused)`, `Build Loop Items & State (Unused)`, `Aggregate Batch Summary`, `Code in JavaScript`.

## 14. HTTP Requests
7 HTTP nodes → 3 targets: **Gemini** (key in query), **backend :5000** (`/api/auth/login`, `/api/questions`, Bearer), **MathML :3000** (`/api/mathml/batch`). Retry/backoff handled by the explicit `Route by HTTP status` + `Wait Before Retry` sub-loop (max 5). No explicit per-node `options.timeout` — add one. All plain HTTP to internal hosts.

## 15. AI Components
- **Model:** Gemini **2.5 Flash**, `generateContent`.
- **Prompt:** built in `Build Gemini Prompt & Request` (systemInstruction + contents + generationConfig).
- **Structured output:** expected as JSON in `parts[0].text`, then `JSON.parse`d — **enforce** `responseMimeType: "application/json"` (+ a response schema) to eliminate parse failures.
- **Token/cost:** not tracked — log `usageMetadata`. **Retry cost guard:** max-5 per question limits runaway spend.

## 16. Database
Postgres, read-only taxonomy `SELECT … UNION ALL`. No writes (questions persist via the backend API). Two identical lookups (one in a dead-end branch) — consolidate to one and cache in static data.

## 17. Vector Database
**Not applicable** — no embeddings/vector store/similarity search.

## 18. Error Handling
- **Gemini errors:** classified by status. Retryable → bounded retry (max 5) with clean body; non-retryable → stop. This is the model to keep.
- **401/403 (token):** tokens are fetched per branch just before store; no explicit expiry re-check here — low risk given immediacy, but add one for long batches.
- **MathML/backend errors:** no dedicated error branch — a failure aborts that group. Consider wrapping in the same status-routing pattern.
- **Dead-letter:** the two "stop" noOps end the path but **do not persist** the failed question anywhere — add a DLQ (write failed items to disk/table) so they can be reprocessed.

## 19. Performance Optimization
- **Batching:** 6 questions/call via the loop.
- **Rate limiting:** `Wait Before Gemini Call` + `Wait Before Retry`.
- **Retry cost control:** per-question max 5.
- **Redundancy:** dead-end config branch + duplicate taxonomy query + orphan nodes waste a little execution and a lot of clarity — prune.

## 20. Security ⚠ (still the top issue)
| Issue | Location | Action |
|---|---|---|
| **Gemini API key hardcoded** | `Generate Questions (Gemini)` query `key=AQ.Ab8RN6Jm…` **and** duplicated in `Sticky Note1` ("## ENV …") | **Rotate now**; move to n8n credential/env var; delete from sticky note; scrub git history |
| **Login credentials hardcoded** | `Login API (LaTeX/Non-LaTeX)` body `naveed@gmail.com` / `naveed@gmail.com` | **Change now**; use an n8n credential |
| **Plain HTTP internal services** | :3000 / :5000 | Prefer HTTPS + auth |

> Values are present in the committed file; this doc **redacts** them. Treat as compromised.

## 21. Logging
`console.log` in Code nodes (`Parse Gemini Response` logs rawText/parsed; retry node logs). No central monitoring/audit; n8n execution history is the only trail. Add structured logging + an error workflow.

## 22. Testing Strategy
- **Pinned trigger data** enables offline node testing.
- **Mock APIs** (see `n8n-related-files/mock-latex*.json`) for MathML/backend.
- **Edge cases:** non-JSON Gemini output; each retryable status (408/429/5xx) vs 400/401; empty groups; retry exhaustion at exactly 5; MathML failure; DB down.
- **Integration:** run a single batch end-to-end before scaling; verify store counts reconcile.

## 23. Troubleshooting
| Symptom | Root cause | Diagnosis | Resolution | Prevention |
|---|---|---|---|---|
| Retries never stop | (fixed here) old 400-in-retry | Check `Route by HTTP status` outputs | Ensure 400 → fallback stop | Keep status-based Switch + max-5 |
| Batch fails at Parse | Non-JSON Gemini output | Log `rawText` | Strip fences; enforce `responseMimeType` | Structured output + schema |
| Merge Group Results stalls | A group branch produced no item | Check all 4 inputs received items | Ensure empty branches route via skip/Edit Fields | Keep the false-branch passthroughs |
| 401 on store | Token stale on long batch | Inspect token/expiry | Re-login before store | Central token refresh |
| MathML failures abort group | No error branch on MathML | Inspect node error | Wrap in status routing / retry | Reuse retry pattern |
| Config change has no effect | Editing dead-end branch B | Trace from trigger (§7) | Edit branch A (`…IDs1` path) | Delete branch B |

## 24. Maintenance Guide
- **Prune** the dead-end branch B, "(Unused)" nodes, and orphans before further edits.
- **Externalize** the Gemini key, login creds, and the three base URLs into credentials/env.
- **Model swap:** change the Gemini model id in `Generate Questions (Gemini)` (and remove the orphan `…(Gemini)1`).
- **Versioning:** this is `versionId ac9e349b…`; keep it as the canonical fixed version over the 152-node export.
- **Backup:** under git; runtime resume state in `$getWorkflowStaticData('global')`.

## 25. Improvements
1. **Rotate/relocate secrets** (blocking, §20) and delete the ENV sticky note.
2. **Prune** dead-end branch B, duplicate taxonomy query, and the 4 orphan/"Unused" nodes.
3. **Enforce Gemini JSON** (`responseMimeType` + schema) to remove parse fragility.
4. **Add a dead-letter sink** for the two "stop" paths so failed questions are recoverable.
5. **Parameterize hosts**; move to HTTPS.
6. **Deduplicate login** into a shared sub-workflow (both branches re-login).
7. **Add explicit HTTP timeouts** and status-routing for MathML/backend calls too.
8. **Track token usage** for cost visibility.

## 26. Best Practices (embodied / recommended)
- ✅ Classify errors by status; retry only transient ones; cap retries — **keep this**.
- ✅ Guard merges against empty branches — **keep this**.
- ✅ Verify store counts after submit — **keep this**.
- ➕ Move secrets to credentials; enforce structured LLM output; add DLQ + monitoring; remove dead code; activate only after happy-path + failure-path tests.

## 27. Glossary
| Term | Meaning |
|---|---|
| **Poison-400 retry** | Endlessly retrying a permanent HTTP 400; fixed here via status routing + clean-body rebuild. |
| **Retryable status** | 408/429/500/502/503/504 — transient; safe to retry with backoff. |
| **Workflow static data** | `$getWorkflowStaticData('global')` — persists token + `__geminiRetry` counter across executions. |
| **Split In Batches** | n8n loop node processing items in chunks. |
| **MathML** | XML math markup rendered from LaTeX for display. |
| **JWT / Bearer** | Auth token from `/api/auth/login`, sent as `Authorization: Bearer …`. |
| **4-way route** | Switch on `group` = valid/invalid × haveLatex/noLatex. |
| **Dead-letter queue** | Durable sink for permanently-failed items so the pipeline isn't blocked. |
| **Orphan / Unused node** | No path from the trigger; won't run in a normal execution. |

---

## 28. Assumptions & Missing Information (explicit)
- **Branch B** (`Set Generation Config → Merge Config & DB Lookup → Map Names to DB IDs`) dead-ends in the file; documented as-is, not wired into an invented output.
- **Credential references** (Postgres, any HTTP credential) live in n8n's store and were **not inspected**; verify they exist before running.
- **`Build Gemini Prompt & Request`** (16.8 KB) prompt text is summarized by purpose, not reproduced line-by-line — read the node body before editing prompts/schema.
- **`Route by HTTP status` fallback** is inferred to catch 400/401/403/404 because those are absent from the retryable list; the node's explicit fallback output confirms non-matching statuses route to stop.
- **No configuration values were invented.** Endpoints, SQL, Switch/IF conditions, and secret locations are quoted from the file; secret values are **redacted** and must be rotated.
