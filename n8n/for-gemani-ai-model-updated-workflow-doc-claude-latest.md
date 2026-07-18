# Workflow Documentation — "for gemani ai model updated"

> Source of truth: n8n MCP (`n8n_get_workflow`), workflow ID `RWpigWQwMT7EBzp9`.
> Generated for engineering handoff — accurate enough to reproduce, maintain, and troubleshoot.

---

## 1. Metadata

| Field | Value |
|-------|-------|
| **Name** | for gemani ai model updated |
| **ID** | `RWpigWQwMT7EBzp9` |
| **Active** | `false` (runs **on demand only**) |
| **Trigger count** | `0` — manual trigger, no schedule/webhook activation |
| **Archived** | `false` |
| **Created** | 2026-07-07 |
| **Updated** | 2026-07-17 |
| **Node count** | 59 |
| **Execution order** | `v1` |
| **Binary mode** | `separate` |
| **Owner project** | Naveed Shaikh (personal project `5fiPKQorCahRgeZE`) |
| **Version counter** | 466 (`versionId` ac9e349b…) |
| **Builder meta** | `aiBuilderAssisted: true`, `builderVariant: mcp` |

**Purpose (in one paragraph):** This workflow mass-generates JEE Main Physics multiple-choice/numerical questions using Google Gemini 2.5 Flash, then post-processes them. It walks a hierarchical syllabus (categories → chapters → topics → subtopics) stored in a JSON file on disk, keeps a resumable cursor in `state.json`, and per iteration asks Gemini for a batch of questions with a strict JSON schema. Generated questions are classified into four buckets (valid-with-LaTeX, valid-without-LaTeX, invalid-with-LaTeX, invalid-without-LaTeX). LaTeX-bearing questions have their math converted to MathML via a local service and re-injected. Both valid buckets are pushed to a local backend API (with a login step to obtain a bearer token). A batch summary loops back to drive the next iteration. A dedicated **error subsystem** handles Gemini HTTP failures with a poison-payload fix and a bounded retry counter (max 5).

> ⚠️ **`active: false` + manual trigger + `triggerCount: 0`** — nothing runs this automatically. It is executed by clicking **Execute Workflow**.

---

## 2. Security Findings (read first)

The prompt mandates never exposing secrets. The following secrets are **hardcoded inside the workflow** and should be treated as compromised/rotated. Values are masked here deliberately.

| Secret | Location (node) | Masked value | Recommendation |
|--------|-----------------|--------------|----------------|
| Gemini API key #1 | `Generate Questions (Gemini)` (query param `key`) | `AQ.Ab8RN6LL…` | Move to n8n credential / env var; rotate |
| Gemini API key #2 | `Generate Questions (Gemini)1` (orphaned) | `AQ.Ab8RN6Kv…` | Remove node or rotate; disconnected but key still leaked |
| Gemini API key #3 | Sticky Note "ENV" | `AQ.Ab8RN6Jm…` | Delete from sticky note; rotate |
| Backend login creds | `Login API (LaTeX)` & `Login API (Non-LaTeX)` | email `naveed@gmail.com`, password identical to email | Move to credential store; the password equalling the email is itself a weakness |
| Postgres credential | `Fetch Category/Chapter/Topic IDs` / `…IDs1` | credential id `b1sfIH8F19ybgksK` ("Postgres account") | Properly stored as an n8n credential ✔ (the correct pattern) |

Additional security notes:
- All backend/service calls use **plain HTTP** to `host.docker.internal` (`:3000`, `:5000`) — unencrypted, acceptable only for host-local Docker traffic.
- No authentication on the local MathML batch service (`:3000`).
- Bearer tokens are also written into **workflow static data** (`accessToken`) by `Attach Token to LaTeX Questions` — persists in the n8n DB between runs.

---

## 3. Environment & Dependencies (reproducibility)

To run this workflow elsewhere you must provide:

**External services**
- **Google Gemini** — `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=…`
- **LaTeX→MathML batch service** — `POST http://host.docker.internal:3000/api/mathml/batch`, body `{ "expressions": [...] }`, returns `{ success, data: [{ latex, mathml }] }`.
- **Backend question API** — `http://host.docker.internal:5000`
  - `POST /api/auth/login` → `{ accessToken }`
  - `POST /api/questions` (Bearer auth) → stores a question.
- **PostgreSQL** — tables `subject_categories`, `chapters`, `topics` (each with `id`, `name`). Credential id `b1sfIH8F19ybgksK`.

**Filesystem** (must exist / be writable inside the n8n container)
- `/home/node/.n8n-files/update-another-deepseek.json` — the active syllabus config (read + rewritten each run; IDs and remaining counts mutate in place).
- `/home/node/.n8n-files/state.json` — resumable cursor `{ currentCategoryIndex, currentChapterIndex, currentTopicIndex }`.
- (Commented-out legacy paths in code: `deep-seek-json.json`, `update-initial-json.json`, `update-deepseek.json`.)

**Runtime assumptions**
- `host.docker.internal` DNS resolution (Docker Desktop / host-gateway). On Linux Docker add `--add-host=host.docker.internal:host-gateway`.
- n8n **filesystem access** enabled for Code nodes (`require("fs")` is used).
- **Workflow static data** keys used: `__geminiRetry` (per-request retry counters), `accessToken`.

---

## 4. High-Level Execution Flow

```
Start (Manual Trigger)
├─► Set Generation Config ──► Merge Config & DB Lookup ──► Map Names to DB IDs ──► [DEAD END]   (live but discarded)
│        └─► Fetch Category/Chapter/Topic IDs ──► (Merge input 1)
│
└─► Fetch Category/Chapter/Topic IDs1 (Postgres)
        └─► Map Names to DB IDs1 (reads+writes syllabus file, sets DB ids)
              └─► Passthrough to Loop Builder
                    ├─► Build Loop Items & State (Unused)  [DEAD END]
                    └─► Build Loop Items & State (reads file, emits 5 loop items, decrements counters, writes state.json)
                          └─► Flatten Loop Items ──► Loop Over Batches (SplitInBatches)
                                 ├─(done)─► Finalize Loop Output  [DEAD END]
                                 └─(loop)─► Build Gemini Prompt & Request ──► Wait Before Gemini Call ──► Generate Questions (Gemini)
```

**After Gemini (success output):**
```
Parse Gemini Response ─► Validate & Classify Questions ─► Split Into 4 Groups ─► Route by Question Group (Switch, 4 outs)
  0 valid-haveLatex  ─► Select LaTeX Questions ─► Flatten LaTeX Questions ─► Convert LaTeX to MathML ─► Attach MathML to Questions
                        ─► Merge MathML & Clean XMLNS ─► Remove LaTeX Field (LaTeX) ─► Gate: Has LaTeX Questions
                        ─► Login API (LaTeX) ─► Set Access Token (LaTeX) ─► Attach Token to LaTeX Questions
                        ─► Store LaTeX Questions ─► Verify LaTeX Store Count ─► Merge Group Results[0]
  1 valid-noLatex    ─► Set Non-LaTeX Questions ─► Remove LaTeX Field (Non-LaTeX) ─► Split Non-LaTeX Questions ─► Gate: Has Non-LaTeX Questions
                        ├─(true)─► Login API (Non-LaTeX) ─► Set Access Token (Non-LaTeX) ─► Prepare Non-LaTeX Questions ─► Store Non-LaTeX Questions ─► Verify Non-LaTeX Store Count ─► Merge[1]
                        └─(false)─► Edit Fields ─► Merge[1]
  2 invalid-haveLatex─► Skip Invalid LaTeX Questions ─► Merge[2]
  3 invalid-noLatex  ─► Skip Invalid Non-LaTeX Questions ─► Merge[3]

Merge Group Results (4 inputs) ─► Aggregate Batch Summary1 ─► Loop Over Batches   (drives next iteration)
```

**Gemini error output (poison-400 retry subsystem):**
```
Generate Questions (Gemini) [error out] ─► Route by HTTP status (Switch)
  0 retryable (408/429/500/502/503/504) ─► Strip error & count retries ─► Retries exhausted?
        ├─(true)──► Retries exhausted — stop (NoOp)
        └─(false)─► Restore clean retry body ─► Wait Before Retry (6 min) ─► Generate Questions (Gemini)
  1 fallback (nonRetryable/stop: 4xx & unknown) ─► Non-retryable error — stop (NoOp)
```

---

## 5. Node-by-Node Reference

Nodes are grouped by role. Every one of the 59 nodes appears. Depth is weighted toward the live path, the large Code nodes, and the retry subsystem; trivial nodes (sticky notes, NoOps, orphans) get a short entry as required.

### 5.1 Trigger & Setup

#### `Start (Manual Trigger)` — `manualTrigger` v1
- **Purpose:** Entry point; fires on "Execute Workflow".
- **Config/Params:** none.
- **Inputs/Outputs:** no input; single output fans out to **two** branches: `Set Generation Config` and `Fetch Category/Chapter/Topic IDs1`.
- **Execution flow:** both downstream branches run.
- **Dependencies:** none. **Error handling:** default. **Perf:** negligible.
- **Pitfall:** Because it feeds two branches, the "config" branch (5.5) executes even though its output is discarded.

#### `Fetch Category/Chapter/Topic IDs1` — `postgres` v2.6
- **Purpose:** Loads id↔name maps for categories, chapters, topics from Postgres.
- **Config:** `executeQuery`; SQL `UNION ALL` over `subject_categories`, `chapters`, `topics`, each projected to `(type, id, LOWER(name) AS name)`. `alwaysOutputData: true`.
- **Credential:** Postgres account `b1sfIH8F19ybgksK`.
- **Inputs/Outputs:** from trigger → to `Map Names to DB IDs1`. Emits one item per row.
- **Dependencies:** reachable Postgres with those tables. **Error handling:** default (would abort run on failure). **Perf:** one query; small.

#### `Map Names to DB IDs1` — `code` v2 *(live, side-effecting)*
- **Purpose:** Reads the syllabus file, matches each category/chapter/topic name to its DB id (case-insensitive, trimmed), updates the ids **in the file**, and writes it back.
- **Config:** `JSON_PATH = /home/node/.n8n-files/update-another-deepseek.json`. Builds `categoryMap/chapterMap/topicMap` from the Postgres rows (`$input.all()`), then walks `config.categories[].chapters[].topics[]`, setting `.id` (or `null` if unmatched), logging updates.
- **Inputs/Outputs:** from Postgres node → to `Passthrough to Loop Builder`. Returns `{ success, updatedIds, filePath, config, summary }`.
- **Dependencies:** filesystem write access; file must exist and be valid JSON. **Error handling:** throws if file missing/invalid. **Pitfall:** unmatched names silently become `null` id — downstream Gemini `subjectCategoryIds/chapterIds/topicIds` will contain `null`.

#### `Passthrough to Loop Builder` — `code` v2
- **Purpose:** No-op passthrough (adds `myNewField = 1` to each item) that fans out to the two loop-builder nodes.
- **Inputs/Outputs:** from `Map Names to DB IDs1` → to `Build Loop Items & State` **and** `Build Loop Items & State (Unused)`.
- **Notes:** the added field is unused; effectively a fan-out junction.

### 5.2 Loop Construction & Batching

#### `Build Loop Items & State` — `code` v2 *(live, side-effecting; core control logic)*
- **Purpose:** The scheduler. Reads the syllabus file + `state.json` cursor, selects the current category/chapter/topic, builds a **loop item** describing the batch to generate, decrements all the remaining-count bookkeeping, advances the cursor when a topic is exhausted, and persists file + state.
- **Key config:**
  - `JSON_PATH = /home/node/.n8n-files/update-another-deepseek.json`, `STATE_PATH = /home/node/.n8n-files/state.json`.
  - `batchesPerExecution = 5` — emits **5 identical loop items** per execution.
  - Loop item fields: `categoryId/Name`, `chapterId/Name`, `topicId/Name`, `subtopicsName`, `optionType` = `chapter.generationconfig.perbatchdistribution.questiontype`, `difficultyLevel` = `…perbatchdistribution.difficulty`, `perbatchdistribution`, `questionNumber` = `chapter.generationconfig.batchsize`, `responseSchema` = `data.responseschema`.
  - Decrements global `totalquestions`/`totalapicalls`, `questiondistribution.*`, category/chapter counts — each multiplied by `batchesPerExecution` (5).
  - Topic advance: `topic.questioncount -= batchSize * 5`; when `<= 0`, advance topic → chapter → category (wrapping to 0 at the end).
- **Inputs/Outputs:** from `Passthrough` → to `Flatten Loop Items`. Returns `{ loopItems: [ …5 ] }`.
- **Dependencies:** filesystem. **Error handling:** throws on invalid indices / missing arrays.
- **⚠️ Pitfalls / coupling:**
  - `batchesPerExecution = 5` is **coupled** to the `topic.questioncount -= batchSize * 5` decrement and to the `* 5` multipliers on every distribution counter. Changing one without the others corrupts the bookkeeping.
  - The counters mutate the on-disk file every run — **not idempotent**; re-running continues from wherever `state.json` left off.
  - All 5 emitted loop items are **identical** (same topic), so a single execution produces 5 batches for the same topic.

#### `Build Loop Items & State (Unused)` — `code` v2 *(DISCONNECTED — never executes)*
- **Purpose:** Earlier single-batch version of the scheduler (emits 1 loop item; `topic.questioncount -= batchSize * 5` but subtracts only `batchSize`/`1` from other counters).
- **Status:** Receives input from `Passthrough` **but has no outgoing connection** → its output is discarded; effectively dead. Retained for history. Also side-effects the same files if it ran — but it does not run.

#### `Flatten Loop Items` — `code` v2
- **Purpose:** Expands `{ loopItems: [...] }` into individual n8n items (one per batch).
- **Inputs/Outputs:** from `Build Loop Items & State` → to `Loop Over Batches`.

#### `Loop Over Batches` — `splitInBatches` v3
- **Purpose:** Iterates the flattened loop items one batch at a time; the "done" output finalizes, the "loop" output processes a batch. `Aggregate Batch Summary1` feeds back into this node to continue.
- **Config:** `options.reset = false`; `alwaysOutputData: true`.
- **Outputs:** index 0 = **done** → `Finalize Loop Output`; index 1 = **loop** → `Build Gemini Prompt & Request`.
- **Pitfall:** the feedback loop is closed by `Aggregate Batch Summary1 → Loop Over Batches`; if any branch fails to reach the Merge/Aggregate, the loop stalls.

#### `Finalize Loop Output` — `code` v2
- **Purpose:** On loop completion (`$input.context['noItemsLeft']`), emits `{ status: "Loop Complete" }`; otherwise emits nothing.
- **Inputs/Outputs:** from `Loop Over Batches` (done) → **dead end** (no downstream).

### 5.3 Gemini Request / Response

#### `Build Gemini Prompt & Request` — `code` v2 *(large prompt builder)*
- **Purpose:** Constructs the full Gemini `generateContent` request body for one batch. Assembles a very detailed **system prompt** (JEE Physics question-setter persona; exact difficulty & type distribution; strict HTML allow/forbid list; a 13-rule "math tokenization" spec requiring all math to be extracted into a top-level `latex[]` array and referenced via `{{latex[n]}}` placeholders; worked examples; a final validation checklist). Builds `systemInstruction`, multi-part `contents` (exam info, sample output schema, final distribution instruction), and a `generationConfig` with `temperature 0.8`, `topP 0.95`, `candidateCount 1`, `maxOutputTokens 20000`, `responseMimeType application/json`, and a strict `responseSchema` (ARRAY of question OBJECTs with required fields incl. `latex`, `options`, id arrays, `role`).
- **Inputs:** loop item (destructures `categoryId/Name`, `chapterId/Name`, `topicId/Name`, `subtopicsName`, `optionType`, `difficultyLevel`, `questionNumber` (default 6), `perbatchdistribution`, `responseSchema`).
- **Output:** `{ json: requestBody }` → `Wait Before Gemini Call`.
- **Notes:** `optionType`/`difficultyLevel` here are the per-batch **distribution objects** (`{single, multiple, numerical}` / `{easy, moderate, hard}`), used to instruct exact counts. Emits `console.log` diagnostics.

#### `Wait Before Gemini Call` — `wait` v1.1
- **Purpose:** Throttle before each Gemini call. `amount: 0.1 minutes` (= 6 s). Has a `webhookId` (resume webhook) — note it shares the same `webhookId` string as `Wait Before Retry`.
- **Inputs/Outputs:** → `Generate Questions (Gemini)`.

#### `Generate Questions (Gemini)` — `httpRequest` v4.4 *(external call; dual-output)*
- **Purpose:** Calls Gemini `gemini-2.5-flash:generateContent`.
- **Config:** `POST`; query param `key` = **Gemini API key #1 (masked)**; header `Content-Type: application/json`; body = raw `{{ $json }}` (the request built upstream).
- **Reliability:** `retryOnFail: true`, `waitBetweenTries: 3000`, `alwaysOutputData: true`, **`onError: continueErrorOutput`** → output 0 = success, output 1 = error.
- **Outputs:** 0 → `Parse Gemini Response`; 1 → `Route by HTTP status`.
- **Dependencies:** network + valid key. **Perf:** dominant latency + cost driver; `maxOutputTokens 20000`.
- **Pitfall:** body is the entire incoming `$json`. On the error branch, n8n appends an `error` object to the item — see the poison-payload fix (5.6).

#### `Parse Gemini Response` — `code` v2
- **Purpose:** Extracts `candidates[0].content.parts[0].text`, `JSON.parse`s it, and returns each question as an item (handles array, `{questions:[]}`, or single object).
- **Inputs/Outputs:** from Gemini success → to `Validate & Classify Questions`.
- **Error handling:** throws `JSON Parse Error` if Gemini returns non-JSON (mitigated by `responseMimeType: application/json`). **Pitfall:** defaults to `"[]"` if the path is missing → zero questions silently.

### 5.4 Validation, Classification & Routing

#### `Validate & Classify Questions` — `code` v2 *(large validator)*
- **Purpose:** Validates LaTeX placeholder integrity per question and classifies into `valid.haveLatex`, `valid.noLatex`, `invalid.haveLatex`, `invalid.noLatex`. Checks: placeholder `{{latex[n]}}` indices in range; `latex` array non-empty entries; the literal word "latex" without a valid placeholder (warning); unused latex entries (warning); duplicate latex (warning). Produces a `summary` (`totalQuestions`, `haveLatex`, `noLatex`, `validQuestions`, `invalidLatextQuestion`, `validNolatexQuestion`).
- **Inputs/Outputs:** from `Parse Gemini Response` → to `Split Into 4 Groups`.
- **Pitfall:** the `invalid.haveLatex` branch pushes `...invalidQuestion.question` (spreads the **question string**, not the object) — a latent bug in how invalid-with-latex items are shaped. Documented, not fixed.

#### `Split Into 4 Groups` — `code` v2
- **Purpose:** Emits exactly four items, one per group: `valid-haveLatex`, `valid-noLatex`, `invalid-haveLatex`, `invalid-noLatex`, each `{ group, questions }`.
- **Inputs/Outputs:** → `Route by Question Group`.

#### `Route by Question Group` — `switch` v3.4
- **Purpose:** Routes each of the four group items to its dedicated branch by `$json.group` string equality.
- **Outputs:** 0 valid-haveLatex, 1 valid-noLatex, 2 invalid-haveLatex, 3 invalid-noLatex (see flow map for targets).

### 5.5 LaTeX (MathML) Processing Branch — group 0

#### `Select LaTeX Questions` — `set` v3.4
- Assigns `haveLatex = $json.questions` (array). → `Flatten LaTeX Questions`.

#### `Flatten LaTeX Questions` — `code` v2
- Flattens `input.json.haveLatex[]` into one item per question. `alwaysOutputData: true`. → `Convert LaTeX to MathML`.

#### `Convert LaTeX to MathML` — `httpRequest` v4.4
- **Purpose:** `POST http://host.docker.internal:3000/api/mathml/batch` with `{ "expressions": <question.latex[]> }`. Returns `{ success, data:[{latex, mathml}] }`.
- **Config:** header `Content-Type: application/json`; body from `JSON.stringify($json.latex)`. → `Attach MathML to Questions`.
- **Dependencies:** local MathML service. **Pitfall:** no auth; assumes `latex` field present per item.

#### `Attach MathML to Questions` — `code` v2
- Re-shapes the stream: first item `{ haveLatex: <original questions from "Select LaTeX Questions"> }`, followed by all HTTP responses. → `Merge MathML & Clean XMLNS`.

#### `Merge MathML & Clean XMLNS` — `code` v2 *(core merge logic)*
- **Purpose:** Validates that question count == conversion count, strips the `xmlns="…/MathML"` attribute from every MathML string, recursively replaces `{{latex[n]}}` placeholders throughout each question with the converted MathML, and rewrites `question.latex` to an array of `{ latex, mathml }`.
- **Guards:** throws on count mismatch, missing `data[]`, failed conversion, per-question `latex[]` length mismatch, or latex-string mismatch between question and conversion.
- **Inputs/Outputs:** → `Remove LaTeX Field (LaTeX)`.
- **Pitfall:** strict equality checks (`latexString !== converted.latex`) make it brittle to any normalization the MathML service performs.

#### `Remove LaTeX Field (LaTeX)` — `code` v2
- Strips the `latex` field from each `input.haveLatex[]` question, returning one item per question. → `Gate: Has LaTeX Questions`.

#### `Gate: Has LaTeX Questions` — `if` v2.3
- **Condition:** `Object.keys($json).length > 0` (strict number). True → `Login API (LaTeX)`.
- **Pitfall:** tests key **count**, not semantic emptiness — an empty-ish object with keys still passes; the false branch has no downstream (silently drops).

#### `Login API (LaTeX)` — `httpRequest` v4.4
- `POST http://host.docker.internal:5000/api/auth/login` with JSON `{ email, password }` — **hardcoded creds (masked)**. → `Set Access Token (LaTeX)`.

#### `Set Access Token (LaTeX)` — `set` v3.4
- Assigns `accessToken = $json.accessToken`. → `Attach Token to LaTeX Questions`.

#### `Attach Token to LaTeX Questions` — `code` v2
- Re-emits all `Remove LaTeX Field (LaTeX)` questions as items and **writes the token into workflow static data** (`$getWorkflowStaticData('global').accessToken`). → `Store LaTeX Questions`.
- **Security note:** persists the bearer token in the n8n DB.

#### `Store LaTeX Questions` — `httpRequest` v4.4
- `POST :5000/api/questions`, header `Authorization: Bearer {{ $("Set Access Token (LaTeX)").first().json.accessToken }}`, body `JSON.stringify($json)` (one question per call). → `Verify LaTeX Store Count`.

#### `Verify LaTeX Store Count` — `code` v2
- Compares `Remove LaTeX Field (LaTeX)` count vs. validator `summary.haveLatex`; emits `{ generatedQuestions, haveLatexQuestions, success }`. → `Merge Group Results[0]`.

### 5.6 Non-LaTeX Processing Branch — group 1

#### `Set Non-LaTeX Questions` — `set` v3.4
- Assigns `noLatex = $json.questions` (array). → `Remove LaTeX Field (Non-LaTeX)`.

#### `Remove LaTeX Field (Non-LaTeX)` — `code` v2
- Maps `input.noLatex` removing each question's `latex` field. → `Split Non-LaTeX Questions`.

#### `Split Non-LaTeX Questions` — `splitOut` v1
- Splits out field `noLatex` into individual items. `alwaysOutputData: true`, `onError: continueRegularOutput`. → `Gate: Has Non-LaTeX Questions`.

#### `Gate: Has Non-LaTeX Questions` — `if` v2.3
- **Condition:** `Object.keys($json).length > 0`. True → `Login API (Non-LaTeX)`; False → `Edit Fields`.

#### `Login API (Non-LaTeX)` — `httpRequest` v4.4
- Same login as 5.5, `retryOnFail: true`. → `Set Access Token (Non-LaTeX)`.

#### `Set Access Token (Non-LaTeX)` — `set` v3.4
- Assigns `accessToken`. → `Prepare Non-LaTeX Questions`.

#### `Prepare Non-LaTeX Questions` — `code` v2
- Re-emits all `Split Non-LaTeX Questions` items as questions. → `Store Non-LaTeX Questions`.

#### `Store Non-LaTeX Questions` — `httpRequest` v4.4
- `POST :5000/api/questions`, Bearer from `Set Access Token (Non-LaTeX)`, body `JSON.stringify($json)`. `onError: continueErrorOutput`. → `Verify Non-LaTeX Store Count`.

#### `Verify Non-LaTeX Store Count` — `code` v2
- Compares `Split Non-LaTeX Questions` count vs. validator `summary.noLatex`; emits `{ generatedQuestions, noLatexQuestions, success }`. → `Merge Group Results[1]`.

#### `Edit Fields` — `set` v3.4 (raw)
- **Purpose:** When there are no non-LaTeX questions, emits a stub `{ generatedQuestions: 0, haveLatexQuestions: 0, success: true }` so the Merge input still receives data. → `Merge Group Results[1]`.

### 5.7 Invalid-Question Branches — groups 2 & 3

#### `Skip Invalid LaTeX Questions` — `code` v2
- Counts input vs. validator `valid-haveLatex` + `invalid-haveLatex` groups; emits a reconciliation summary `{ generatedQuestions, validHaveLatex, invalidHaveLatex, totalHaveLatex, success }`. → `Merge Group Results[2]`.
- **Note:** despite the name it does not "store"; it accounts for skipped invalid items.

#### `Skip Invalid Non-LaTeX Questions` — `code` v2
- Similar reconciliation for the invalid-noLatex group. → `Merge Group Results[3]`.

### 5.8 Aggregation & Loop Back

#### `Merge Group Results` — `merge` v3.2
- **Purpose:** Joins the four branch outputs (`numberInputs: 4`). Inputs: 0 = LaTeX store verify, 1 = Non-LaTeX verify / Edit Fields, 2 = invalid-latex, 3 = invalid-noLatex. → `Aggregate Batch Summary1`.
- **Pitfall:** requires **all four** inputs to arrive; a branch that silently drops (e.g. a false Gate with no downstream) can leave an input unsatisfied depending on merge mode.

#### `Aggregate Batch Summary1` — `code` v2
- **Purpose:** Rolls up per-batch counts into `{ generatedQuestions (EXPECTED from Set Generation Config.questionsperapicall || 6), validHaveLatex, validNoLatex, invalidHaveLatex, invalidNoLatex, totalValid, totalInvalid, totalProcessed, success, missingQuestions }`.
- **Inputs/Outputs:** from Merge → **back to `Loop Over Batches`** (closes the iteration loop).
- **Dependency note:** references `Set Generation Config` for `questionsperapicall` — the one live use of that otherwise-discarded node (via `$('Set Generation Config')`).

### 5.9 Gemini Error / Retry Subsystem (poison-400 fix)

> This is the subsystem referenced in project memory (`gemini-workflow-retry-loop`).

#### `Route by HTTP status` — `switch` v3.4
- **Purpose:** On Gemini's error output, routes by `$json.error.status`. Output 0 **"retryable"** matches `408/429/500/502/503/504` (combinator OR, loose typing). Fallback output 1 **"nonRetryable / stop"** catches deterministic 4xx (400/401/403/404/409/413/422) and unknown statuses.
- **Outputs:** 0 → `Strip error & count retries`; 1 → `Non-retryable error — stop`.
- **Pitfall:** depends on the error item being shaped as `$json.error.status`. If n8n's error envelope changes, routing silently falls through to "stop".

#### `Strip error & count retries` — `code` v2 *(the poison-payload fix)*
- **Purpose:** Rebuilds a **clean** Gemini request body (`{ systemInstruction, contents, generationConfig }`) — dropping the `error` field that n8n appended, which otherwise poisons the retry (Gemini → 400 "Unknown name error"). Computes a stable hash key from `contents`, increments a **per-request** counter in static data `__geminiRetry[key]` (max 5), sets `exceeded`, and clears the counter when exceeded.
- **Output:** `{ body, retryCount, maxRetries: 5, exceeded }` → `Retries exhausted?`.

#### `Retries exhausted?` — `if` v2.3
- Condition `$json.exceeded === true` (boolean). True → `Retries exhausted — stop`; False → `Restore clean retry body`.

#### `Restore clean retry body` — `set` v3.4 (raw)
- Sets the item JSON to `$json.body` (the clean rebuilt request). → `Wait Before Retry`.

#### `Wait Before Retry` — `wait` v1.1
- `amount: 6 minutes` back-off before re-hitting Gemini. Shares `webhookId` with `Wait Before Gemini Call`. → `Generate Questions (Gemini)` (re-enters the call).

#### `Retries exhausted — stop` — `noOp` v1
- Terminal. Note: "Retry cap (5) reached — give up gracefully instead of looping forever."

#### `Non-retryable error — stop` — `noOp` v1
- Terminal. Note: deterministic client errors (400/401/403/404/409/413/422) and unknown statuses terminate here — never retried.

### 5.10 Live-but-Discarded "Config" Branch

> These execute (reachable from the trigger) but their output is **not consumed** by the loop. The loop's real data source is the file read in `Build Loop Items & State`, not these nodes.

#### `Set Generation Config` — `set` v3.4 (raw)
- **Purpose:** Emits a large static JSON describing the entire JEE syllabus generation config (2130 total questions, 355 API calls, category/chapter/topic tree, difficulty & type ratios, per-batch distribution, response schema).
- **Inputs/Outputs:** from trigger → to `Merge Config & DB Lookup[0]` **and** `Fetch Category/Chapter/Topic IDs[0]`.
- **Live use:** its `questionsperapicall` (|| 6) is read by `Aggregate Batch Summary1`. The full config it emits, however, dead-ends at `Map Names to DB IDs`.

#### `Fetch Category/Chapter/Topic IDs` — `postgres` v2.6
- Same query as `…IDs1`. → `Merge Config & DB Lookup[1]`. Credential `b1sfIH8F19ybgksK`.

#### `Merge Config & DB Lookup` — `merge` v3.2
- Joins config (input 0) with DB rows (input 1). → `Map Names to DB IDs`.

#### `Map Names to DB IDs` — `code` v2
- In-memory version of the id-mapping (does **not** write a file; mutates the passed config object and returns it). **Has no outgoing connection → DEAD END.** Its result is discarded.

### 5.11 Disconnected / Orphaned Nodes (never execute)

These have **no incoming connection** (or their sole output is unwired) and therefore never run. Retained as history/scratch.

| Node | Type | Note |
|------|------|------|
| `Build Loop Items & State (Unused)` | code v2 | Older single-batch scheduler; input wired, output not → discarded. |
| `Build Gemini Prompt (Unused)` | code v2 | Alternate single-question prompt builder (default `questionNumber 8`, `maxOutputTokens 14000`). Not wired in. |
| `Generate Questions (Gemini)1` | httpRequest v4.4 | Duplicate Gemini caller with **Gemini key #2 (masked)**. Fully disconnected (position far left). |
| `Aggregate Batch Summary` | code v2 | Older aggregator (references `Loop Over Items3`, a node that doesn't exist). No incoming; output empty. |
| `Code in JavaScript` | code v2 | MathML→plain-text "searchableText" builder for embedding/RAG use. No incoming connection. |
| `Fetch Category/Chapter/Topic IDs1` | *(this one IS live — listed in 5.1)* | — |

### 5.12 Sticky Notes (annotations)

| Node | Content |
|------|---------|
| `Sticky Note` | "## get Latext using api" — labels the LaTeX/MathML region. |
| `Sticky Note1` | "## ENV" — **contains a leaked Gemini key #3 (masked)**; should be scrubbed. |

---

## 6. Data Contracts (for reproduction)

**Loop item** (produced by `Build Loop Items & State`, consumed by `Build Gemini Prompt & Request`):
```jsonc
{
  "categoryId": 1, "categoryName": "mechanics",
  "chapterId": 1, "chapterName": "units & measurements",
  "topicId": 1, "topicName": "units",
  "subtopicsName": ["…","…"],
  "optionType": { "single": 4, "multiple": 1, "numerical": 1 },      // perbatch questiontype
  "difficultyLevel": { "easy": 2, "moderate": 3, "hard": 1 },        // perbatch difficulty
  "perbatchdistribution": { "difficulty": {…}, "questiontype": {…} },
  "questionNumber": 6,                                                // = chapter batchsize
  "responseSchema": { … }
}
```

**Gemini question object** (per `responseSchema`, required fields): `question`, `explanation`, `hint`, `latex[]`, `optionType` (Single|Multiple|Numerical), `difficultyLevel` (easy|moderate|hard), `role` ("admin"), `options[]` (`{name, isCorrect}`), `images[]`, `subjectIds[]`, `subjectCategoryIds[]`, `chapterIds[]`, `topicIds[]`, `examCategoryIds[]`. Math is carried as `{{latex[n]}}` placeholders + a top-level `latex[]` array.

**MathML service response:** `{ success: true, data: [ { latex, mathml } ] }`.

**`state.json`:** `{ currentCategoryIndex, currentChapterIndex, currentTopicIndex }`.

---

## 7. Known Quirks & Troubleshooting Checklist

1. **Not idempotent / stateful:** every run mutates `update-another-deepseek.json` and `state.json`. To restart from scratch, reset `state.json` to all-zeros and restore the syllabus file's counts.
2. **`batchesPerExecution = 5` coupling:** must stay in sync with the `* 5` decrement multipliers and `topic.questioncount -= batchSize * 5`.
3. **`host.docker.internal` resolution** must work for `:3000` and `:5000`; otherwise MathML and store/login calls fail.
4. **Gemini poison-400 on retry:** handled by `Strip error & count retries`; if you see 400 "Unknown name error" on retries, verify that node still rebuilds a clean `{systemInstruction, contents, generationConfig}` body.
5. **Retry routing hinges on `$json.error.status`;** an n8n upgrade that changes the error envelope will send everything to "Non-retryable — stop".
6. **Gate IF nodes** use `Object.keys($json).length > 0` — a structural, not semantic, emptiness check.
7. **`Merge MathML & Clean XMLNS` is strict:** any normalization by the MathML service (that makes `converted.latex !== original`) throws.
8. **Unmatched syllabus names → `null` ids** propagate into stored questions.
9. **Secrets are hardcoded** (Section 2) — rotate and migrate to credentials/env before any shared use.
10. **Stale `pinData`** exists on a node named "When clicking 'Execute workflow'" that no longer exists (the trigger was renamed to "Start (Manual Trigger)"); pinned data is inert.

---

## 8. Node Inventory (all 59)

Trigger(1): Start (Manual Trigger). Postgres(2): Fetch Category/Chapter/Topic IDs, …IDs1. Code(24): Map Names to DB IDs, Map Names to DB IDs1, Passthrough to Loop Builder, Build Loop Items & State, Build Loop Items & State (Unused), Flatten Loop Items, Finalize Loop Output, Build Gemini Prompt & Request, Parse Gemini Response, Validate & Classify Questions, Split Into 4 Groups, Flatten LaTeX Questions, Attach MathML to Questions, Merge MathML & Clean XMLNS, Remove LaTeX Field (LaTeX), Attach Token to LaTeX Questions, Verify LaTeX Store Count, Remove LaTeX Field (Non-LaTeX), Prepare Non-LaTeX Questions, Verify Non-LaTeX Store Count, Skip Invalid LaTeX Questions, Skip Invalid Non-LaTeX Questions, Aggregate Batch Summary1, Aggregate Batch Summary, Strip error & count retries, Build Gemini Prompt (Unused), Code in JavaScript. HTTP Request(7): Generate Questions (Gemini), Generate Questions (Gemini)1, Convert LaTeX to MathML, Login API (LaTeX), Store LaTeX Questions, Login API (Non-LaTeX), Store Non-LaTeX Questions. Set(8): Set Access Token (LaTeX), Set Non-LaTeX Questions [set], Set Access Token (Non-LaTeX), Select LaTeX Questions, Set Generation Config, Restore clean retry body, Edit Fields. Switch(2): Route by Question Group, Route by HTTP status. IF(3): Gate: Has LaTeX Questions, Gate: Has Non-LaTeX Questions, Retries exhausted?. Merge(2): Merge Group Results, Merge Config & DB Lookup. SplitInBatches(1): Loop Over Batches. SplitOut(1): Split Non-LaTeX Questions. Wait(2): Wait Before Gemini Call, Wait Before Retry. NoOp(2): Retries exhausted — stop, Non-retryable error — stop. Sticky Note(2): Sticky Note, Sticky Note1.

*(Count note: the exact per-type tallies above are grouped for readability; the authoritative total is 59 nodes as returned by MCP.)*
