# JEE Main Physics Question Generator (n8n Workflow)

> **Workflow:** `for gemani ai model updated`
> **Workflow ID:** `RWpigWQwMT7EBzp9`
> **Nodes:** 53 · **Trigger:** Manual

---

## 1. Project Overview

This n8n workflow is an **automated question-generation pipeline** for **JEE Main Physics**. It walks through a curriculum (categories → chapters → topics → subtopics), asks **Google Gemini** to generate exam questions in batches, validates and cleans the AI output, converts any LaTeX math into **MathML**, and finally stores the questions in a backend API. It is designed to run for **thousands of questions** (config targets ~2,130 questions across ~355 API calls) while surviving transient API failures through a bounded retry loop.

## 2. Workflow Purpose

- Generate high-quality, JEE-Main-style Physics questions at scale using an LLM.
- Guarantee questions are **well-formed** (valid LaTeX placeholders, correct option structure).
- Convert LaTeX expressions to **MathML** so the frontend can render math reliably.
- Persist valid questions to the application database through an authenticated REST API.
- Do all of this **batch-by-batch** with progress state, so a large curriculum can be processed incrementally and safely resumed.

## 3. High-Level Architecture

```
Manual Trigger
      │
      ▼
[Config + DB ID mapping]  ──►  [Build batch/loop items from curriculum JSON + state file]
      │
      ▼
┌─────────────────────────── Loop Over Batches ───────────────────────────┐
│                                                                          │
│   Build Gemini prompt ─► Wait ─► Call Gemini ─► Parse response           │
│                                     │  (on HTTP error)                   │
│                                     └─► Route by status ─► Retry (max 5) │
│                                                                          │
│   Parsed questions ─► Validate & classify into 4 groups:                 │
│        • valid + LaTeX     ─► LaTeX→MathML ─► login ─► store              │
│        • valid + no LaTeX  ─────────────────► login ─► store             │
│        • invalid + LaTeX   ─────────────────► skipped                    │
│        • invalid + no LaTeX ────────────────► skipped                    │
│                                                                          │
│   Merge 4 groups ─► Aggregate batch summary ─► next batch                │
└──────────────────────────────────────────────────────────────────────────┘
```

The workflow has **two logical stages**:

1. **Setup stage** (runs once): load config, map curriculum names to database IDs, and build the list of batch "loop items" from a JSON curriculum file plus a persisted state file.
2. **Generation loop** (runs per batch): generate → validate → transform → store → summarize, repeating until all batches are done.

## 4. Workflow Flow (step-by-step)

### Setup stage
1. **Start (Manual Trigger)** — user clicks *Execute*.
2. **Set Generation Config** — emits a static config object (exam = JEE Main, subject = Physics, total questions, batch size, difficulty / question-type ratios).
3. **Set Generation Config** feeds both **Fetch Category/Chapter/Topic IDs** and **Merge Config & DB Lookup** directly (the redundant always-true DB gate was removed).
4. **Fetch Category/Chapter/Topic IDs** *(Postgres)** — `SELECT` from `subject_categories`, `chapters`, and `topics` (UNION) to get name→ID mappings.
5. **Map Names to DB IDs** — replaces curriculum names in the config with the real database IDs.
6. **Passthrough to Loop Builder** — pass-through step.
7. **Build Loop Items & State** — reads the curriculum JSON file and a persisted `state.json`, computes the next set of batch "loop items", and advances the cursor. (A second copy, **Build Loop Items & State (Unused)**, is a disconnected duplicate — see *Important Notes*.)
8. **Flatten Loop Items** — flattens the nested `loopItems` array into individual n8n items.

### Generation loop (per batch)
9. **Loop Over Batches** *(Split In Batches)* — iterates over the batch items.
   - **Done output →** **Finalize Loop Output** (loop completion endpoint / no-op passthrough).
   - **Loop output →** continue below.
10. **Build Gemini Prompt & Request** — builds the full Gemini request body: a detailed JEE Physics system prompt plus the `generationConfig` and response schema for the batch.
11. **Wait Before Gemini Call** — small delay to pace/throttle requests.
12. **Generate Questions (Gemini)** *(HTTP)* — `POST` to Gemini `gemini-2.5-flash:generateContent`.
    - **Success →** **Parse Gemini Response**.
    - **Error →** **Route by HTTP status** (retry sub-flow, see §9).
13. **Parse Gemini Response** — extracts `candidates[0].content.parts[0].text`, `JSON.parse`s it, and normalizes into a list of question items.
14. **Validate & Classify Questions** — validates LaTeX placeholders (`{{ latex[n] }}`), option structure, etc., and classifies every question into `valid`/`invalid` × `haveLatex`/`noLatex`, plus a summary count.
15. **Split Into 4 Groups** — emits four items keyed by `group`: `valid-haveLatex`, `valid-noLatex`, `invalid-haveLatex`, `invalid-noLatex`.
16. **Route by Question Group** *(Switch)* — routes each group to its branch:

    **A. `valid-haveLatex` branch (questions containing math):**
    - **Select LaTeX Questions** → **Flatten LaTeX Questions** → **Convert LaTeX to MathML** *(HTTP `POST /api/mathml/batch`)* → **Attach MathML to Questions** → **Merge MathML & Clean XMLNS** (merges converted MathML back into each question and strips `xmlns`) → **Remove LaTeX Field (LaTeX)** → **Gate: Has LaTeX Questions** *(If — proceeds only when the item has fields)* → **Login API (LaTeX)** → **Set Access Token (LaTeX)** → **Attach Token to LaTeX Questions** → **Store LaTeX Questions** *(HTTP `POST /api/questions`)* → **Verify LaTeX Store Count** → **Merge Group Results** (input 0).

    **B. `valid-noLatex` branch (plain questions):**
    - **Set Non-LaTeX Questions** → **Remove LaTeX Field (Non-LaTeX)** → **Split Non-LaTeX Questions** → **Gate: Has Non-LaTeX Questions** *(If)* → **Login API (Non-LaTeX)** → **Set Access Token (Non-LaTeX)** → **Prepare Non-LaTeX Questions** → **Store Non-LaTeX Questions** *(HTTP `POST /api/questions`)* → **Verify Non-LaTeX Store Count** → **Merge Group Results** (input 1).

    **C. `invalid-haveLatex` branch:** **Skip Invalid LaTeX Questions** → **Merge Group Results** (input 2).

    **D. `invalid-noLatex` branch:** **Skip Invalid Non-LaTeX Questions** → **Merge Group Results** (input 3).

17. **Merge Group Results** *(Merge, 4 inputs)* — recombines the four branch outputs.
18. **Aggregate Batch Summary** — computes per-batch totals (generated, valid, invalid) and an overall `success` flag, then feeds back into **Loop Over Batches** for the next batch.

## 5. Node Responsibilities

> Nodes have been renamed to action-based names. Node **IDs and connections are unchanged**.

### Setup & curriculum
| Node | Type | Responsibility |
|------|------|----------------|
| Start (Manual Trigger) | Manual Trigger | Starts the workflow on demand. |
| Set Generation Config | Set (raw JSON) | Emits the static generation config (exam, subject, totals, ratios, batch size). |
| Fetch Category/Chapter/Topic IDs | Postgres | UNION query over `subject_categories`, `chapters`, `topics` to get name→ID rows (runs directly from Set Generation Config). |
| Merge Config & DB Lookup | Merge | Combines config with DB lookup rows. |
| Map Names to DB IDs | Code | Replaces curriculum names in config with real DB IDs. |
| Passthrough to Loop Builder | Code | Pass-through step between mapping and loop building. |
| Build Loop Items & State | Code | Reads curriculum JSON + `state.json`, builds next batch loop items, advances state. |
| Build Loop Items & State (Unused) | Code | Disconnected duplicate of the loop builder (vestigial). |
| Flatten Loop Items | Code | Flattens `loopItems` into individual n8n items. |

### Generation loop
| Node | Type | Responsibility |
|------|------|----------------|
| Loop Over Batches | Split In Batches | Iterates over batches; `reset: false` so it resumes across iterations. |
| Finalize Loop Output | Code | Loop-completion endpoint (no-op passthrough on the "done" output). |
| Build Gemini Prompt & Request | Code | Builds the Gemini request (system prompt + generationConfig + schema). |
| Wait Before Gemini Call | Wait | Paces requests before calling Gemini. |
| Generate Questions (Gemini) | HTTP | Calls Gemini `gemini-2.5-flash:generateContent`. |
| Parse Gemini Response | Code | Parses the LLM text into a JSON array of questions. |
| Validate & Classify Questions | Code | Validates LaTeX/options and classifies into valid/invalid × latex/no-latex + summary. |
| Split Into 4 Groups | Code | Emits the four `group`-keyed items. |
| Route by Question Group | Switch | Routes each group to its processing branch. |

### `valid-haveLatex` branch (math questions)
| Node | Type | Responsibility |
|------|------|----------------|
| Select LaTeX Questions | Set | Picks the `haveLatex` question array. |
| Flatten LaTeX Questions | Code | Flattens questions into individual items. |
| Convert LaTeX to MathML | HTTP | `POST /api/mathml/batch` — converts LaTeX expressions to MathML. |
| Attach MathML to Questions | Code | Recombines original questions with the MathML responses. |
| Merge MathML & Clean XMLNS | Code | Merges MathML back into each question and removes `xmlns` attributes. |
| Remove LaTeX Field (LaTeX) | Code | Strips the raw `latex` field from each question. |
| Gate: Has LaTeX Questions | If | Real guard — proceeds only when the item has fields (`Object.keys($json).length > 0`), mirroring the non-LaTeX branch. |
| Login API (LaTeX) | HTTP | `POST /api/auth/login` to obtain an access token. |
| Set Access Token (LaTeX) | Set | Extracts `accessToken` from the login response. |
| Attach Token to LaTeX Questions | Code | Attaches token/questions; also caches token in workflow static data. |
| Store LaTeX Questions | HTTP | `POST /api/questions` with `Authorization: Bearer <token>`. |
| Verify LaTeX Store Count | Code | Compares generated vs. valid counts; sets a `success` flag. |

### `valid-noLatex` branch (plain questions)
| Node | Type | Responsibility |
|------|------|----------------|
| Set Non-LaTeX Questions | Set | Picks the `noLatex` question array. |
| Remove LaTeX Field (Non-LaTeX) | Code | Strips the `latex` field from each question. |
| Split Non-LaTeX Questions | Split Out | Splits the `noLatex` array into individual items. |
| Gate: Has Non-LaTeX Questions | If | Proceeds only when questions exist (`keys > 0`). |
| Login API (Non-LaTeX) | HTTP | `POST /api/auth/login` to obtain an access token. |
| Set Access Token (Non-LaTeX) | Set | Extracts `accessToken` from the login response. |
| Prepare Non-LaTeX Questions | Code | Prepares the split questions for storage. |
| Store Non-LaTeX Questions | HTTP | `POST /api/questions` with `Authorization: Bearer <token>`. |
| Verify Non-LaTeX Store Count | Code | Compares generated vs. valid counts; sets a `success` flag. |

### Invalid branches & merge
| Node | Type | Responsibility |
|------|------|----------------|
| Skip Invalid LaTeX Questions | Code | No-op passthrough for invalid LaTeX questions (not stored). |
| Skip Invalid Non-LaTeX Questions | Code | No-op passthrough for invalid plain questions (not stored). |
| Merge Group Results | Merge (4 inputs) | Recombines all four branches. |
| Aggregate Batch Summary | Code | Aggregates per-batch counts and overall success, then loops back. |

### Retry sub-flow (Gemini error output)
| Node | Type | Responsibility |
|------|------|----------------|
| Route by HTTP status | Switch | Retryable statuses (408/429/500/502/503/504) → retry branch; else → stop. |
| Strip error & count retries | Code | Rebuilds a **clean** request body and increments a per-question retry counter (max 5) in static data. |
| Retries exhausted? | If | Branches on whether the retry limit was reached. |
| Restore clean retry body | Set | Restores the clean request body for the retry. |
| Wait Before Retry | Wait | Back-off delay before retrying. |
| Retries exhausted — stop | No-Op | Terminates a request that used all retries. |
| Non-retryable error — stop | No-Op | Terminates on a non-retryable HTTP error. |

### Unused / annotation nodes
| Node | Type | Responsibility |
|------|------|----------------|
| Build Gemini Prompt (Unused) | Code | Alternate/legacy Gemini prompt builder; **disconnected** (not part of the live flow). |
| Sticky Note / Sticky Note1 | Sticky Note | Canvas annotations (`get LaTeX using api`, and an `ENV`/API-key note). |

## 6. External Services Used

| Service | Endpoint | Used by |
|---------|----------|---------|
| **Google Gemini** | `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent` | Generate Questions (Gemini) |
| **MathML conversion service** | `POST http://host.docker.internal:3000/api/mathml/batch` | Convert LaTeX to MathML |
| **Application backend API** | `POST http://host.docker.internal:5000/api/auth/login`, `POST http://host.docker.internal:5000/api/questions` | Login + store questions |
| **PostgreSQL** | `subject_categories`, `chapters`, `topics` tables | Fetch Category/Chapter/Topic IDs |
| **Local filesystem** | `/home/node/.n8n-files/update-another-deepseek.json`, `/home/node/.n8n-files/state.json` | Build Loop Items & State |

## 7. Input Data

- **Curriculum JSON file** — `/home/node/.n8n-files/update-another-deepseek.json`: the categories → chapters → topics → subtopics structure with per-topic question counts, difficulty distribution, and question-type distribution.
- **State file** — `/home/node/.n8n-files/state.json`: cursor tracking which category/chapter/topic to process next (enables incremental / resumable runs).
- **Generation config** (from *Set Generation Config*): e.g. `exam: jee main`, `subject: physics`, `totalquestions: 2130`, `batchsize: 6`, `questionsperapicall: 6`, `totalapicalls: 355`, difficulty ratio `30/50/20`, type ratio `60/20/20`.
- **PostgreSQL rows** — real IDs for categories, chapters, and topics.

## 8. Output Data

- **Stored questions** — valid questions (both math and plain) are POSTed to `POST /api/questions` on the backend. Math questions carry pre-rendered **MathML** instead of raw LaTeX.
- **Per-batch summary** (from *Aggregate Batch Summary*): `generatedQuestions`, `validQuestions`, `invalidQuestions`, and an overall `success` boolean.
- Invalid questions are **counted but not stored**.

## 9. Error Handling

- **Gemini retry loop:** `Generate Questions (Gemini)` uses the node's error output. On error, **Route by HTTP status** checks `error.status`:
  - **Retryable** (`408, 429, 500, 502, 503, 504`) → **Strip error & count retries** rebuilds a **clean** request body (dropping the `error` field n8n appends — which otherwise poisons the retry and makes Gemini return `400 "Unknown name error"`), increments a **per-question retry counter (max 5)** stored in workflow static data, then **Retries exhausted?** decides:
    - not exhausted → **Restore clean retry body** → **Wait Before Retry** → retry the Gemini call.
    - exhausted → **Retries exhausted — stop** (No-Op).
  - **Non-retryable** → **Non-retryable error — stop** (No-Op).
- **Validation:** `Validate & Classify Questions` separates malformed questions into `invalid-*` groups so they are **never stored**.
- **Count checks:** `Verify LaTeX Store Count` / `Verify Non-LaTeX Store Count` compare generated vs. valid counts and flag mismatches via `success`.
- **Gates:** `Gate: Has Non-LaTeX Questions` avoids calling the store API for empty batches.

## 10. Environment Variables & Credentials

This workflow currently uses **hard-coded values** rather than environment variables — items to externalize:

| Item | Where | Note |
|------|-------|------|
| Gemini API key | Generate Questions (Gemini) | Passed as a `key` query parameter on the request URL (no n8n credential; a key is also noted in the `ENV` sticky note). Should be a credential/secret. |
| Backend login email/password | Login API (LaTeX) / (Non-LaTeX) | Currently hard-coded `naveed@gmail.com`. Move to credentials. |
| PostgreSQL connection | Fetch Category/Chapter/Topic IDs | n8n Postgres credential. |
| Service base URLs | HTTP nodes | `host.docker.internal:3000` (MathML) and `:5000` (backend) are hard-coded. |
| Curriculum / state file paths | Build Loop Items & State | Hard-coded `/home/node/.n8n-files/*.json`. |
| Access token | Set Access Token (LaTeX)/(Non-LaTeX) | Stored in a Set field — n8n flags this; prefer the credential system. |

## 11. Important Notes

- **Node renaming:** every node was renamed to an action-based name; node IDs and connections were preserved. `$('node name')` references inside Code nodes and HTTP headers were updated to match the new names so behavior is identical.
- **Always-true gate cleanup** *(applied)*: the redundant `Gate: Run DB Lookup` (`true == true`) was **removed** and `Set Generation Config` now feeds the Postgres lookup directly; `Proceed to Store (LaTeX)` (`1 == 1`) was converted into a real guard, **Gate: Has LaTeX Questions** (`Object.keys($json).length > 0`), mirroring the non-LaTeX branch. Verified after the change: **53 nodes, 55 connections, zero stale references**.
- **Vestigial / no-op nodes** (safe to remove later):
  - **Build Gemini Prompt (Unused)** — disconnected alternate prompt builder.
  - **Build Loop Items & State (Unused)** — disconnected duplicate of the loop builder.
  - **Finalize Loop Output**, **Passthrough to Loop Builder**, **Skip Invalid LaTeX Questions**, **Skip Invalid Non-LaTeX Questions** — pass-through / no-op nodes (some still contain the default `myNewField = 1` boilerplate).
- **State-file coupling:** `Loop Over Batches` uses `reset: false` and the run advances `state.json`, so consecutive executions continue where the previous one stopped.
- **Poison-body fix:** the retry branch deliberately reconstructs `{ systemInstruction, contents, generationConfig }` to strip n8n's appended `error` object before retrying Gemini.

## 12. Future Improvements

- Move all secrets (Gemini key, backend credentials, DB, service URLs, file paths) into **n8n credentials / environment variables**.
- Store the access token with the credential system instead of a `Set` field (removes the `SET_CREDENTIAL_FIELD` warnings).
- **Remove the vestigial nodes** (`Build Gemini Prompt (Unused)`, `Build Loop Items & State (Unused)`, and the boilerplate no-ops) to reduce clutter.
- ~~Replace the always-true `Proceed to Store (LaTeX)` gate with a real guard mirroring the non-LaTeX branch.~~ ✅ **Done** — now `Gate: Has LaTeX Questions`; the redundant `Gate: Run DB Lookup` was also removed.
- Log in **once per batch** and reuse the token across both store branches instead of logging in separately in each branch.
- Add explicit handling/alerting when `success` is `false` in the batch summary (e.g. notify or re-queue the batch).
- Parameterize the Gemini model (`gemini-2.5-flash`) and generation limits so they can be tuned without editing code nodes.
