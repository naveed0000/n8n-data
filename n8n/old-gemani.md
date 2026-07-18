# Technical Documentation — Workflow "for gemani ai model updated"

> **Source:** `C:\Users\alion\n8n-data\for gemani ai model updated.json`
> **Workflow ID:** `RWpigWQwMT7EBzp9`  ·  **Active:** `false`  ·  **Node count:** 152 (151 functional + 1 sticky note)
> **Engine settings:** `executionOrder: v1`, `binaryMode: separate`
> **Generated:** 2026-07-17  ·  **Template applied:** `n8n/workflow-info.md`

---

## 0. Read this first — state of the workflow

This **is** a real n8n workflow (it has `nodes` + `connections`). Two facts dominate everything below and you must internalize them before reading section by section:

1. **It is a fragmented "parts-bin" / work-in-progress.** Of 151 functional nodes, **51 have no inbound connection** and many chains dead-end. Only **~23 nodes are reachable from the single trigger** (`When clicking 'Execute workflow'`). The remaining ~128 nodes form **disconnected islands** — self-contained sub-pipelines (Gemini call, LaTeX→MathML, auth+submit, validation) that were built and tested independently and are **not currently wired into the live path**. `active: false` is consistent with this being under active development.

2. **It contains hardcoded live secrets committed to git** (see §20 — this is the highest-priority issue): a Google Gemini API key in a query parameter and a real login email/password in the `Login API*` request bodies.

Because of (1), this document distinguishes clearly between:
- **the Live Path** — what actually runs when you press Execute (§7A), documented in full; and
- **the Islands** — the disconnected sub-pipelines (§7B), documented per functional cluster with their real endpoints/logic, but flagged as **not reachable from the trigger** so no execution order is invented.

Per the template rule "do not invent configuration values": where a node's downstream wiring does not exist, that is stated, not fabricated.

**Related project memory:** this workflow is the one described in `[[gemini-workflow-retry-loop]]` (the poison-400 retry bug). The bug is still visibly present here — node **`If9`** routes both HTTP **429 and 400** into the retry `Wait3` loop (§11, §18). If a Switch-based fix was applied, it is in a different version than this export.

---

## 1. Overview

| Aspect | Detail |
|---|---|
| **Purpose** | Generate JEE-style physics exam questions with an LLM (Google **Gemini 2.5 Flash**), convert their LaTeX to MathML, validate them, and submit the valid ones to a backend question API. |
| **Business goal** | Automate bulk creation of a syllabus-aligned question bank (see the companion config `update-another-deepseek.json`), replacing manual authoring. |
| **Problem solved** | Manually authoring thousands of correctly-formatted, math-rendered, schema-valid questions is slow. This workflow batches generation + validation + persistence. |
| **Expected output** | Validated question objects POSTed to `http://host.docker.internal:5000/api/questions`; intermediate state persisted to `state.json` on disk and to `$getWorkflowStaticData('global')`. |
| **Workflow type** | Long-running batch generation pipeline with an internal state machine and Split-In-Batches loops. |
| **Automation category** | AI content generation + ETL (extract taxonomy from Postgres, transform via LLM/MathML, load to API). |
| **Trigger type** | **Manual** (`manualTrigger`) only. No webhook/schedule. |
| **Complexity** | Very high — 152 nodes, 7 batch loops, 13 merges, 12 IF, a Switch, 61 Code nodes, workflow-static-data state machine, external LLM + 3 backend services. |
| **Dependencies** | Google Generative Language API; a backend at `:5000` (`/api/auth/login`, `/api/questions`); a MathML service at `:3000` (`/api/mathml/batch`); a Postgres DB (taxonomy tables); local disk files under `/home/node/.n8n-files/`. |
| **External services** | Google Gemini API. |
| **Internal services** | Backend API (:5000), MathML service (:3000), Postgres, n8n filesystem. |
| **Advantages** | End-to-end automation; retry/wait handling for rate limits; taxonomy-name→DB-id mapping; state persistence for resumability. |
| **Limitations** | Fragmented graph (most nodes unwired); hardcoded secrets; hardcoded hostnames incl. a LAN IP `192.168.1.17`; the poison-400 retry bug; 40+ near-duplicate/placeholder Code nodes; `active:false`. |

---

## 2. High-Level Architecture

**Intended** end-to-end pipeline (assembled from the islands — parts exist but are not all wired together):

```text
Manual Trigger
   -> Load generation config (disk JSON / Postgres taxonomy)
   -> Map category/chapter/topic NAMES -> DB IDs
   -> Build loop items + loop state (workflow static data "global")
   -> [BATCH LOOP] for each batch of questions:
        -> Build Gemini prompt (config + per-batch distribution)
        -> POST Gemini 2.5 Flash generateContent
             -> on 429/400 -> Wait -> retry (see poison-400 caveat)
        -> Parse candidates[0].content.parts[0].text -> JSON questions
        -> Validate questions (LaTeX + schema) -> valid / invalid
        -> Split valid into haveLatex / noLatex (Switch on "group")
        -> haveLatex: POST /api/mathml/batch to render LaTeX -> MathML
        -> Login (/api/auth/login) -> Bearer token -> POST /api/questions
        -> Update loop state; persist state.json
   -> Continue until all batches complete
```

**Actual current wiring (Live Path)** is only the front of this: trigger → load file/DB taxonomy → replace names with DB ids → build loop items → (stops). The generation/validation/submit machinery exists as islands. See §7.

Data moves as n8n items (`$json`). Cross-node state is carried three ways: (a) item passing, (b) `$getWorkflowStaticData('global')` for loop/token state, (c) `state.json` on disk for durable resume.

---

## 3. Workflow Diagram

### 3A. Live Path (reachable from trigger)

```text
When clicking 'Execute workflow'
 ├─> If ──true──> Execute a SQL query ──> Merge1(in1)
 ├─> Merge1 (in0)                         [DEAD-END: no outbound]
 ├─> Read/Write Files from Disk ─> Extract from File ─> remove data key from json
 │        ├─> If10 ──true──> Execute a SQL query1 ─> Merge10(in1)
 │        └─> Merge10(in0)
 │                 └─> replace id with DB id
 │                        ├─> create loop Items and loop info1 ─> Code in JavaScript17  [DEAD-END]
 │                        └─> create loop Items and loop info3 ─> Code in JavaScript40  [DEAD-END]
 ├─> Code in JavaScript41  (writes /home/node/.n8n-files/state.json)  [DEAD-END]
 └─> Edit Fields4
          ├─> If11 ──true──> Execute a SQL query2 ─> Merge11(in1)
          └─> Merge11(in0)
                   └─> replace id with DB id1 ─> Code in JavaScript42 ─> create loop Items and loop info4  [DEAD-END]
```

### 3B. Islands (NOT reachable from trigger — independent sub-pipelines)

```text
[Gemini request builder]  Code in JavaScript ─> Code in JavaScript1 ─> HTTP Request (Gemini GET)

[Retry-aware Gemini call]  Code in JavaScript11 ─> Wait1 ─> HTTP Request2 (Gemini POST)
        ├─main0─> extract ai response to json ─> Question Validator ─> Code in JavaScript21 ─> Switch
        │             ├ valid-haveLatex ─> pick only have Latex ─> extract to different object ─> HTTP Request3 (MathML) ─> Code in JavaScript29 ─> Code in JavaScript13 ─> remove latex key1 ─> If6 ...
        │             ├ valid-noLatex  ─> Edit Fields8 ─> remove latex key2 ─> Split Out ─> If7 ...
        │             └ (2,3) ─> Merge8 ─> Code in JavaScript27 ─> Loop Over Items2
        └─main1─> If9 (status 429 OR 400) ─> Wait3 ─> HTTP Request2   [RETRY LOOP; poison-400]

[Auth + submit]  If3/6/7/8 ─> Login APIn ─> Edit Fieldsn ─> Mergen ─> HTTP Requestn (POST /api/questions)

[Batch state machine]  create loop Items and loop info ─> Split Out loop items / extract forInfo
        ─> merge forLoopInfo and ai model response ─> validation check batch is completed

[File/state loaders]  create loop Items and loop info4 (reads update-initial-json.json)

[Misc]  Loop Over Items <-> Replace Me (placeholder self-loop); DebugHelper; Sticky Note
```

---

## 4. Node Inventory (all 152)

Legend — **R** = reachable from trigger (Live Path); **I** = island (no path from trigger). Full per-node parameters for the important nodes are in §5; the 40+ boilerplate/placeholder Code nodes are grouped in §13.

| # | Node Name | Type | R/I | Purpose |
|---|---|---|---|---|
| 1 | When clicking 'Execute workflow' | manualTrigger | R | Sole entry point. |
| 2 | If | if | R | Gate (`$json === true`) → run taxonomy SQL. |
| 3 | Execute a SQL query | postgres | R | Load category/chapter/topic id+name (UNION). |
| 4 | Merge1 | merge | R | Combine trigger + SQL; **dead-end**. |
| 5 | Read/Write Files from Disk | readWriteFile | R | Read a JSON file from `/home/node/.n8n-files/`. |
| 6 | Extract from File | extractFromFile | R | Parse the read file into JSON. |
| 7 | remove data key from json | code | R | Spread `item.json.data[]` to top level. |
| 8 | If10 | if | R | Gate (`true`) → taxonomy SQL. |
| 9 | Execute a SQL query1 | postgres | R | Same taxonomy query as #3. |
| 10 | Merge10 | merge | R | Merge parsed file + taxonomy. |
| 11 | replace id with DB id | code | R | Map taxonomy names→DB ids on the config. |
| 12 | create loop Items and loop info1 | code | R | Build batch loop items + loop info. |
| 13 | Code in JavaScript17 | code | R | Flatten loopItems; **dead-end**. |
| 14 | create loop Items and loop info3 | code | R | Variant loop-item builder. |
| 15 | Code in JavaScript40 | code | R | Flatten loopItems; **dead-end**. |
| 16 | Code in JavaScript41 | code | R | Write `state.json` to disk; **dead-end**. |
| 17 | Edit Fields4 | set | R | Seed a boolean/flag → If11/Merge11. |
| 18 | If11 | if | R | Gate (`true`) → taxonomy SQL2. |
| 19 | Execute a SQL query2 | postgres | R | Same taxonomy query as #3. |
| 20 | Merge11 | merge | R | Merge Edit Fields4 + taxonomy. |
| 21 | replace id with DB id1 | code | R | Map names→ids (variant). |
| 22 | Code in JavaScript42 | code | R | Boilerplate passthrough. |
| 23 | create loop Items and loop info4 | code | R | Read `update-initial-json.json`; build loop items; **dead-end**. |
| 24 | Code in JavaScript | code | I | Build Gemini request body (prompt). |
| 25 | Code in JavaScript1 | code | I | Assemble Gemini `config`+`executionContext` payload. |
| 26 | HTTP Request | httpRequest | I | **Gemini** `generateContent` (GET). |
| 27 | Code in JavaScript2 | code | I | Add null ids to categories tree. |
| 28 | Merge | merge | I | Combine branches (island). |
| 29 | If1 | if | I | String-exists gate. |
| 30 | Loop Over Items1 | splitInBatches | I | Batch loop (islands). |
| 31 | logic for looping | code | I | Initialize loop tracking vars. |
| 32 | Code in JavaScript3/4/5 | code | I | Loop helpers / placeholders. |
| 35 | Loop Over Items | splitInBatches | I | Self-loops with "Replace Me". |
| 36 | Replace Me | noOp | I | Placeholder in Loop Over Items. |
| 37 | Loop Over Items2 | splitInBatches | I | Batch loop after Merge8. |
| 38 | create loop Items and loop info | code | I | **Core** state/loop-item builder. |
| 39 | Split Out loop items | splitOut | I | Explode loopItems array. |
| 40 | Aggregate / Aggregate1 / Aggregate2 | aggregate | I | Re-collect items into arrays. |
| 43 | HTTP Request1 | httpRequest | I | **Gemini** generateContent (POST). |
| 44 | Wait / Wait1 / Wait2 / Wait3 | wait | I | Rate-limit / retry delays. |
| 48 | Code in JavaScript8 / 10 | code | I | Batch state machine (6.3 KB each). |
| 50 | extract forInfo object for maintaing state | set | I | Extract `forInfo` for state. |
| 51 | merge forLoopInfo and ai model response … | merge | I | Merge loop info + AI response. |
| 52 | validation check batch is completed | code | I | Decide if batch loop is done. |
| 53 | Code in JavaScript7 | code | I | Placeholder. |
| 54 | Code in JavaScript11 | code | I | **Largest** (14 KB) prompt/state builder. |
| 55 | HTTP Request2 | httpRequest | I | **Gemini** generateContent (POST) — retry target. |
| 56 | extract ai response to json | code | I | Parse `candidates[0].content.parts[0].text`→JSON. |
| 57 | If9 | if | I | **429 OR 400** → retry (poison-400). |
| 58 | Wait3 | wait | I | Retry backoff before HTTP Request2. |
| 59 | Question Validator(Valid \| inValid) | code | I | Validate LaTeX + schema; produce valid/invalid + summary. |
| 60 | Code in JavaScript21 | code | I | Wrap validated output for Switch. |
| 61 | Switch | switch | I | Route by `group`: valid-haveLatex / valid-noLatex / others. |
| 62 | pick only have Latex | set | I | Keep only `haveLatex`. |
| 63 | extract to different object of question | code | I | Flatten `haveLatex[]`. |
| 64 | HTTP Request3 | httpRequest | I | **MathML** `/api/mathml/batch` (host.docker.internal:3000). |
| 65 | HTTP Request10 / 13 | httpRequest | I | **MathML** batch at `192.168.1.17:3000` (LAN IP). |
| 66 | HTTP Request11 | httpRequest | I | **MathML** batch at host.docker.internal:3000. |
| 67 | HTTP Request12 / 14 | httpRequest | I | **Gemini** generateContent (POST). |
| 68 | Edit Fields8 | set | I | Prep noLatex branch. |
| 69 | remove latex key / …1 / …2 | code | I | Strip `latex` field from questions. |
| 70 | Split Out / Split Out every object… | splitOut | I | Explode question arrays. |
| 71 | Login API … Login API7 (8 nodes) | httpRequest | I | **Auth** POST `/api/auth/login` (hardcoded creds). |
| 79 | Initialize Token | code | I | Decode JWT, store token+expiry in global state. |
| 80 | Edit Fields1/2/3/5/6/7/9/10 | set | I | Carry token / shape payloads. |
| 88 | If2/3/4/5/6/7/8 | if | I | Token-valid / latex-present gates. |
| 95 | Filter | filter | I | Empty-configured filter (no-op). |
| 96 | HTTP Request4/5/7/8/9 | httpRequest | I | **Submit** POST `/api/questions` (Bearer). |
| 101 | Merge3/4/5/6/7/8/9 | merge | I | Join token + question payloads. |
| 108 | Code in JavaScript16/18/19/20/22/23/25/26/27/29/30/31/32/33/34/35/36/37/38/39 | code | I | Payload shaping, counters, error mapping, state. |
| 128 | global config | code | I | Read `create loop Items and loop info` output (loopInfo). |
| 129 | Aggregate2 | aggregate | I | Recombine. |
| 130 | create loop Items and loop info / 1/2/3/4 | code | I | Family of loop-item/state builders. |
| 135 | Code in JavaScript24 | code | I | Timestamp/log record ("Question Creation"). |
| 136 | Execute Workflow | executeWorkflow | I | Call a sub-workflow (island). |
| 137 | Merge2 | merge | I | Island merge. |
| 138 | Loop Over Items3/4/5 | splitInBatches | I | Batch loops (submit stages). |
| 141 | No Operation, do nothing1 | noOp | I | Placeholder. |
| 142 | DebugHelper | debugHelper | I | Generate debug/sample data. |
| 143 | Sticky Note | stickyNote | I | Doc note: "get Latext using api". |
| 144 | Read/Write Files / Extract from File | — | R | (see #5/#6). |
| 145–152 | HTTP Request6, Merge, misc | mixed | I | Auxiliary/older variants. |

> The table groups the ~40 boilerplate `Code in JavaScriptN` nodes; §13 documents the boilerplate pattern and the distinctive ones individually. No node is omitted — every name from the file appears above or in §13.

---

## 5. Detailed Node Documentation (distinctive nodes)

### 5.1 `When clicking 'Execute workflow'` (manualTrigger)
- **Purpose/why:** the only entry point; produces one empty item. **Has pinned data** (see `pinData`) so test runs start from a fixed sample.
- **Output:** single item. Fans out to `If`, `Merge1`, `Read/Write Files from Disk`, `Code in JavaScript41`, `Edit Fields4`.
- **Failure/recovery:** cannot fail; if nothing happens on Execute, the wiring downstream is the issue, not the trigger.

### 5.2 `Execute a SQL query` / `…1` / `…2` (postgres, v2.6)
- **Operation:** `executeQuery`. **Identical query** in all three:
  ```sql
  SELECT 'category' AS type, id, LOWER(name) AS name FROM subject_categories
  UNION ALL SELECT 'chapter' AS type, id, LOWER(name) AS name FROM chapters
  UNION ALL SELECT 'topic'  AS type, id, LOWER(name) AS name FROM topics;
  ```
- **Purpose:** produce a flat lookup of taxonomy `{type, id, name}` so the Code nodes can replace human-readable names in the generation config with database IDs before submission.
- **Credentials:** a Postgres credential (referenced by n8n credential store — not inspected here; ensure it exists).
- **Performance:** three full-table scans of three tables; small taxonomy, negligible. **Optimization:** the three copies are redundant — one query cached in static data would suffice.
- **Failure:** DB down / bad creds → node errors and the branch stops. No retry configured.

### 5.3 Gemini calls — `HTTP Request`, `HTTP Request1/2/12/14`, `HTTP Request1` (httpRequest v4.4)
- **Endpoint:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- **Method:** POST (GET on the first `HTTP Request`).
- **Auth:** **API key in query param** `key=AQ.Ab8RN6Ky…` (⚠ hardcoded — §20). `authentication: none` at node level because the key is passed manually.
- **Headers:** `Content-Type: application/json`.
- **Body:** `={{ $json }}` — the upstream Code node builds the full Gemini request (contents/prompt/generationConfig).
- **Response:** Gemini JSON; consumers read `candidates[0].content.parts[0].text` (see `extract ai response to json`).
- **Error handling:** `HTTP Request2` has a **second output** (main#1) wired to `If9` for error routing (429/400). Others have no error branch.
- **Rate limits/backoff:** `Wait`/`Wait1`/`Wait3` nodes insert delays; retry loop via `If9 → Wait3 → HTTP Request2`.

### 5.4 `If9` (if v2.3) — the retry gate ⚠
- **Conditions (OR/`combinator: and` on two number-equals):** `$json.error.status == 429` and `== 400`.
- **Behavior:** routes Gemini errors to `Wait3 → HTTP Request2` (retry).
- **BUG (matches `[[gemini-workflow-retry-loop]]`):** HTTP **400 is a permanent** "bad request" (malformed prompt / safety block), not transient. Lumping it with 429 makes the workflow **retry a poison request indefinitely**. **Fix:** retry only 429 (and 5xx); send 400 to a dead-letter/skip branch — the memory notes a Switch-based fix. Not present in this export.

### 5.5 `Login API` … `Login API7` (httpRequest v4.4) ⚠
- **Endpoint:** `POST http://host.docker.internal:5000/api/auth/login`.
- **Body (hardcoded):** `{"email":"naveed@gmail.com","password":"naveed@2003"}` — ⚠ **plaintext real credentials in git** (§20).
- **Downstream:** `Initialize Token` decodes the returned JWT and stores `{accessToken, expiresAt}` in `$getWorkflowStaticData('global')`; `Edit Fields*` carry the token; `If3/4/6/7/8` gate on token validity/expiry.
- **Why 8 copies:** each submit sub-pipeline re-logs-in independently — duplication that should be a shared sub-workflow.

### 5.6 Submit calls — `HTTP Request4/5/7/8/9` (httpRequest v4.4)
- **Endpoint:** `POST http://host.docker.internal:5000/api/questions`.
- **Auth:** header `Authorization: Bearer {{ $json.accessToken }}`.
- **Body:** JSON question payload (built by preceding Code/Merge nodes; some bodies are still empty `{}` templates — incomplete).

### 5.7 MathML calls — `HTTP Request3/10/11/13` (httpRequest v4.4)
- **Endpoints:** `POST …/api/mathml/batch` on `host.docker.internal:3000` (#3, #11) **and** on hardcoded LAN IP `192.168.1.17:3000` (#10, #13) — ⚠ inconsistent host targets.
- **Body:** `{"expressions": {{ JSON.stringify($json.latex) }}}` — sends the question's LaTeX array to be rendered to MathML.

### 5.8 `Switch` (switch v3.4)
- **Routing key:** `{{ $json.group }}`.
- **Cases:** `valid-haveLatex` (→ `pick only have Latex`), `valid-noLatex` (→ `Edit Fields8`), plus outputs 2 & 3 → `Merge8`.
- **Purpose:** split validated questions into those needing LaTeX→MathML rendering vs. plain-text ones, before submission.

### 5.9 `Question Validator(Valid | inValid)` (code v2, 3.9 KB)
- **Purpose:** `validateLatex(questionObj)` collects `errors`/`warnings`, classifies each question, and emits a `summary` with counts (`haveLatex`, `noLatex`, valid/invalid). Downstream counters (`Code in JavaScript25/26/31`) read `…Validator…').last().json.summary`.

### 5.10 `create loop Items and loop info` family (code v2, 3.3–5.8 KB)
- **Purpose:** the batch **state machine**. Reads the generation config, computes how many API calls/batches per chapter, builds `loopItems[]` (one per batch) and a `loopInfo`/`forInfo` control object, and reads/writes `$getWorkflowStaticData('global')` for resumable progress. Variants `1/2/3/4` are iterations of the same idea; `…4` reads `/home/node/.n8n-files/update-initial-json.json` from disk.

### 5.11 `Code in JavaScript41` (code v2) & `Read/Write Files from Disk` / `Extract from File`
- **`Code in JavaScript41`:** `const fs=require('fs'); fs.writeFileSync('/home/node/.n8n-files/state.json', …)` — durable state snapshot. **Dead-end** (nothing consumes it downstream).
- **`Read/Write Files from Disk` → `Extract from File` → `remove data key from json`:** load a JSON config file from disk, parse it, and spread its `data[]` array to top-level items for the id-replacement step.

### 5.12 `extract ai response to json` (code v2)
```js
const item = $input.first().json;
const rawText = item.candidates?.[0]?.content?.parts?.[0]?.text ?? "[]";
// JSON.parse(rawText) -> array of question objects
```
- **Edge cases:** if Gemini wraps JSON in markdown fences or returns prose, `JSON.parse` throws — needs a fence-stripping guard.

---

## 6. Connection Documentation

All 112 connections are enumerated in the working file `n8n/_conn_dump.txt`. Key semantics:

- **Trigger fan-out (5 edges):** `→ If`, `→ Merge1(in0)`, `→ Read/Write Files from Disk`, `→ Code in JavaScript41`, `→ Edit Fields4`. These start five independent front-ends; only the file-read and Edit-Fields4 branches proceed meaningfully.
- **`Merge*(in1)` from SQL:** taxonomy results always enter merges on **input index 1**, config data on **index 0**, so id-replacement sees both.
- **Retry edge:** `HTTP Request2 (main#1) → If9 → Wait3 → HTTP Request2` — a cycle (retry loop).
- **Self-loop placeholder:** `Loop Over Items (main#1) → Replace Me → Loop Over Items` — an unfinished loop body.
- **Dead-ends (no outbound):** `Merge1`, `Code in JavaScript17`, `Code in JavaScript40`, `Code in JavaScript41`, `create loop Items and loop info4`, and every island terminating in an Aggregate/HTTP submit without a consumer.

---

## 7. Execution Flow

### 7A. Live Path (what actually runs on Execute)
1. **Trigger** emits one item (or pinned data).
2. **`If`** (`true`) → **`Execute a SQL query`** loads taxonomy → **`Merge1`** (then stops).
3. In parallel, **`Read/Write Files from Disk`** reads a JSON config → **`Extract from File`** parses it → **`remove data key from json`** flattens `data[]`.
4. → **`If10`** (`true`) → **`Execute a SQL query1`** taxonomy → **`Merge10`** joins parsed config + taxonomy.
5. → **`replace id with DB id`** maps names→ids → **`create loop Items and loop info1`** and **`…info3`** build batch items → **`Code in JavaScript17`/`40`** flatten and **stop** (no generation downstream).
6. In parallel, **`Code in JavaScript41`** writes `state.json` and stops.
7. In parallel, **`Edit Fields4`** → **`If11`** → **`Execute a SQL query2`** → **`Merge11`** → **`replace id with DB id1`** → **`Code in JavaScript42`** → **`create loop Items and loop info4`** (reads `update-initial-json.json`) and **stops**.

**Net effect today:** the workflow loads a config + taxonomy, maps IDs, prepares batch loop items, snapshots state — and ends. **No Gemini calls, validation, or submission happen on the live path** because those islands are not connected to it.

### 7B. Islands (independent; run only if manually started from that node)
Each island is a coherent sub-pipeline (Gemini call → parse → validate → Switch → MathML → login → submit). They were evidently built/tested by executing individual nodes. Because none is wired to the trigger, no global execution order exists — documented structurally in §3B, not as a single run.

---

## 8. Data Flow
- **Input:** disk JSON config (`update-initial-json.json` / similar) + Postgres taxonomy.
- **Intermediate:** id-mapped config; `loopItems[]` + `loopInfo`; Gemini request bodies; raw Gemini responses; parsed question arrays; validator `summary`; MathML batches; JWT token in global state.
- **Output:** questions POSTed to `/api/questions`; `state.json` on disk; `$getWorkflowStaticData('global')`.
- **Splits:** `Split Out*` explode arrays into items; `splitInBatches` chunk loops.
- **Merges:** taxonomy on index 1, payloads on index 0 (see §12).
- **Binary:** `binaryMode: separate`; the file-read/extract path handles file binary→JSON.

## 9. Expression Documentation
- `{{ $json }}` — pass whole item as Gemini body.
- `{{ $json.accessToken }}` — Bearer token from login.
- `{{ JSON.stringify($json.latex) }}` — serialize LaTeX array for MathML.
- `{{ $('Initialize Token').item.json.expiresAt }}` — token expiry gate (`If4`).
- `{{ $json.group }}` — Switch routing key.
- `{{ $json.error.status }}` — HTTP error code for `If9`.
- `$getWorkflowStaticData('global')` — cross-execution state (loop progress, token). `$('NodeName').last()/.all()` — pull data from a named upstream node (used heavily by counters/validators).

## 10. Loop Documentation
Seven `splitInBatches` (v3) loops: `Loop Over Items`, `…1`, `…2`, `…3`, `…4`, `…5`, `itration over latex equation`. Pattern: process a batch, do work (HTTP/Code), return to the loop for the next batch (output#0 = "done", output#1 = "loop body"). **Common bug present:** `Loop Over Items ↔ Replace Me` is an empty placeholder body (infinite/no-op). State that governs "batch complete" lives in `validation check batch is completed` + global static data.

## 11. Conditional Logic
- 12 **IF** nodes: most are trivial `1 == 1` gates (`If3/5/6/7/8`) used as manual on/off switches during development, plus real ones: `If2` (`$json.latex.length > 0`), `If4` (token expiry), `If9` (**429/400 retry — buggy**), `If`/`If10`/`If11` (`true` boolean gates).
- 1 **Switch** (`group` routing, §5.8). 1 **Filter** (empty conditions → passes all; effectively a no-op).

## 12. Merge Documentation
13 Merge nodes (v3.2). Standard use: input 0 = primary payload (config/questions), input 1 = secondary (taxonomy or token). Modes not individually shown here but follow n8n "combine/append" defaults; verify each `mode` before relying on positional join. **Race note:** merges that wait for both inputs will stall if one upstream island never runs (a real risk given the disconnected graph).

## 13. Code Nodes
61 Code nodes (v2, JavaScript). Three tiers:
- **Boilerplate/placeholder (~15):** `Code in JavaScript4/5/7/12/39/42`, etc. contain the n8n default `item.json.myNewField = 1` stub or trivial passthroughs — **safe to delete** once wiring is finalized.
- **Payload shapers (~25):** strip `latex`, split arrays, attach `accessToken`, map ids. Small and pure.
- **Heavy logic (a few):** `Code in JavaScript11` (14 KB), `Code in JavaScript36` (11 KB), `Code in JavaScript8/10` (6.3 KB), `create loop Items and loop info*` (3–6 KB), `Question Validator` (3.9 KB). These implement prompt building, the batch state machine, and validation. **Before modifying, read the full body** — they use `$getWorkflowStaticData('global')` and cross-node `$('...')` references that are order-sensitive.

## 14. HTTP Requests
23 HTTP nodes. Three targets: **Gemini** (public, key in query), **backend :5000** (`/api/auth/login`, `/api/questions`, Bearer), **MathML :3000** (`/api/mathml/batch`). Hosts mix `host.docker.internal` and hardcoded `192.168.1.17`. No node-level retry/backoff config except the manual `Wait`+`If9` loop. Timeouts are default (no explicit `options.timeout`).

## 15. AI Components
- **Model:** Google **Gemini 2.5 Flash** via `generateContent`.
- **Prompting:** built in Code nodes (`Code in JavaScript`, `…1`, `…11`) from the generation config + per-batch distribution. **Structured output** is expected as JSON in `parts[0].text`, then `JSON.parse`d — no JSON-mode/`responseMimeType` enforcement visible, so parsing is fragile.
- **Token usage/cost:** not tracked. **Recommendation:** set `generationConfig.responseMimeType: "application/json"` and a response schema; log `usageMetadata` for cost.

## 16. Database
Postgres, read-only taxonomy `SELECT … UNION ALL` (categories/chapters/topics). No insert/update/delete/transactions here — persistence of questions is via the backend API, not direct SQL. Optimization: run the taxonomy query once, cache in static data.

## 17. Vector Database
**Not applicable** — no embeddings, vector store, or similarity search in this workflow.

## 18. Error Handling
- **Gemini:** only `HTTP Request2` has an error output → `If9` (429/400 → retry). **400 must not be retried** (§5.4). Other Gemini/backend/MathML calls have **no error branch** — a failure aborts that island.
- **HTTP 401/403 (expired/invalid token):** `If4` checks `expiresAt`, but most submit calls don't re-check before posting → risk of 401.
- **Timeouts/5xx:** unhandled.
- **Dead-letter:** none. **Logging:** `console.log` in Code nodes + `Code in JavaScript24` timestamp record; no structured audit.

## 19. Performance Optimization
- **Batching:** `questionsperapicall = 6` (from config) via splitInBatches — good.
- **Parallelism:** config targets ≤10 concurrent Gemini calls; not enforced by node config here.
- **Waits:** fixed `Wait` delays for rate limits — replace with 429-driven backoff.
- **Redundancy:** 3× identical taxonomy queries, 8× login — cache/consolidate.

## 20. Security ⚠ (highest priority)
| Issue | Location | Risk | Action |
|---|---|---|---|
| **Gemini API key hardcoded** | `HTTP Request2` etc., query param `key=AQ.Ab8RN6Ky…` (redacted) | Key is in git history; anyone with repo access can bill your Google account | **Rotate the key now**; move to an n8n **credential** / env var; scrub git history |
| **Login email + plaintext password** | `Login API*` body `naveed@gmail.com` / `naveed@2003` (redacted) | Account takeover of the backend | **Change the password now**; use an n8n credential, never inline |
| **Hardcoded internal hosts** | `host.docker.internal`, `192.168.1.17` | Portability + info leak | Parameterize via env vars |
| **No auth on services** | :3000/:5000 over plain HTTP | Interceptable on shared networks | Use HTTPS + auth |

> These values are already present in the committed file. This doc **redacts** them; treat both as compromised and rotate.

## 21. Logging
Ad-hoc `console.log` inside Code nodes; `Code in JavaScript24` writes an execution timestamp record ("Question Creation"). No centralized monitoring, no error workflow. n8n execution logs are the only audit trail; with `active:false` and manual runs, retention is whatever the instance default is.

## 22. Testing Strategy
- **Pinned data** exists on 8 nodes (`pinData`) — enables node-level testing without live calls. Keep for unit tests of Code nodes.
- **Mock APIs:** point Gemini/backend/MathML at local mocks (there are `mock-latex*.json` files in `n8n-related-files/`).
- **Edge cases:** Gemini returns non-JSON / markdown fences; 400 vs 429; expired token; empty `latex`; DB unavailable.
- **Integration:** wire ONE island end-to-end behind the trigger and run a single batch before scaling.

## 23. Troubleshooting
| Symptom | Root cause | Diagnosis | Resolution | Prevention |
|---|---|---|---|---|
| Execute "does nothing" useful | Generation islands not wired to trigger | Trace from trigger (§7A) — stops at loop-item builders | Connect the intended pipeline | Finish wiring; delete dead nodes |
| Infinite ret/looping on a bad prompt | `If9` retries 400 | Check `error.status` in execution | Retry only 429/5xx; DLQ 400 | Split retry logic (Switch) — see `[[gemini-workflow-retry-loop]]` |
| 401 on `/api/questions` | Token expired/not refreshed | Inspect `Initialize Token` expiry vs. now | Re-login before submit; check `If4` | Central token refresh sub-workflow |
| Gemini JSON parse error | Model wrapped JSON in prose/fences | Log `parts[0].text` | Strip fences; enforce `responseMimeType: application/json` | Structured output + schema |
| MathML calls fail intermittently | Two different hosts (`.internal` vs `192.168.1.17`) | Compare `HTTP Request3/10/11/13` | Standardize one host via env var | Parameterize hosts |
| Merge stalls | One input island never ran | Check both merge inputs received items | Ensure both branches execute | Avoid cross-island merges |

## 24. Maintenance Guide
- **Updating nodes:** several nodes are old default stubs — prune before upgrading typeVersions.
- **Replacing APIs:** externalize the three base URLs; swap Gemini model id in one place (currently repeated across 5+ nodes).
- **Credential rotation:** move Gemini key + backend login into n8n credentials; rotate on the schedule.
- **Versioning:** `versionId` in file; keep the Switch-based retry fix in a tracked version.
- **Backup/restore:** workflow is under git; `state.json` on disk is the runtime resume point.

## 25. Improvements
1. **Rotate secrets and move to credentials** (blocking, §20).
2. **Fix the poison-400 retry** (`If9` → Switch: 429/5xx retry, 400 dead-letter).
3. **Wire ONE clean end-to-end pipeline** to the trigger; delete the ~50 orphan/placeholder nodes.
4. **Deduplicate** the 8 Login flows and 3 taxonomy queries into reusable sub-workflows.
5. **Enforce Gemini JSON output** (`responseMimeType` + schema) to kill parse failures.
6. **Parameterize hosts** via env vars; standardize on one MathML host.
7. **Add a central error workflow** + structured logging + token-usage tracking.
8. **Add explicit timeouts/backoff** on all HTTP nodes.

## 26. Best Practices (for this workflow)
- Never inline API keys or passwords — always n8n credentials.
- One retry policy, driven by status class (retry 429/5xx, fail-fast 4xx except 429).
- Idempotent submits (dedupe on question id) to survive retries.
- Keep the graph connected and acyclic except intentional loop backs; remove placeholders.
- Enforce structured LLM output; validate before persistence (the Validator is good — keep it on the live path).
- Activate only after a full happy-path + failure-path test.

## 27. Glossary
| Term | Meaning |
|---|---|
| **Island / orphan node** | A node with no path from the trigger; won't run in a normal execution. |
| **Live Path** | The subgraph actually reachable from the trigger. |
| **`generateContent`** | Gemini API endpoint that returns model output. |
| **Poison-400 retry** | Endlessly retrying a permanent HTTP 400 as if it were transient. |
| **Split In Batches** | n8n loop node that processes items in chunks. |
| **Workflow static data** | `$getWorkflowStaticData()` — persistent key/value store across executions. |
| **MathML** | XML math markup rendered from LaTeX for display. |
| **JWT / Bearer token** | Auth token from `/api/auth/login`, sent as `Authorization: Bearer …`. |
| **Merge (by index)** | Combining two inputs; here taxonomy on input 1, payload on input 0. |
| **Dead-letter queue** | A sink for messages that permanently fail, so the pipeline isn't blocked. |

---

## 28. Assumptions & Missing Information (explicit)
- **Downstream wiring of the islands is not defined** in the file — I documented each island's internal chain but **did not invent** a global order joining them (§7B).
- **Credentials** (Postgres, and whether any HTTP node references a stored credential) are referenced by id in n8n's store and were **not inspected**; verify they exist before running.
- **Merge `mode`s** were not expanded per-node; confirm combine-vs-append before relying on positional joins (§12).
- **Exact prompt text** inside the large Code nodes (`Code in JavaScript11/36`) was summarized by purpose, not reproduced line-by-line; read the node bodies before editing.
- **No configuration values were invented.** Endpoints, SQL, conditions, and the secrets' existence are quoted from the file; secret values are **redacted** here and must be rotated.
