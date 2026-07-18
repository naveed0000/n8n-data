# Technical Documentation — `update-another-deepseek.json`

> **Source artifact:** `C:\Users\alion\n8n-data\update-another-deepseek.json`
> **Generated:** 2026-07-17
> **Documentation template applied:** `n8n/workflow-info.md` (n8n workflow documentation standard)

---

## 0. IMPORTANT — What this artifact actually is (read first)

This file is **not an n8n workflow**. It contains **no `nodes` array and no `connections` object**, which are the two mandatory top-level keys of every n8n workflow export. Attempting to document it as a node graph would require inventing nodes, which the template explicitly forbids ("Do not invent configuration values").

**What it is instead:** a **question-generation configuration / data contract**. It is a static input document describing *how many* physics questions to generate for **JEE Main 2026 (Physics, English)**, *how* to distribute them across categories, chapters, topics, difficulty levels and question types, *how* to batch the AI API calls, and *what shape* each generated question must have (`responseschema`).

**How it is used (inferred, not verified here):** a **separate n8n workflow** loads this JSON as its driving configuration, iterates over the categories → chapters → batches, issues batched LLM API calls (6 questions per call), and validates each returned question against `responseschema`. Project memory identifies a Gemini-based generation workflow with ID **`RWpigWQwMT7EBzp9`** ([[gemini-workflow-retry-loop]]) as the likely consumer. **That workflow has not been inspected as part of this document** — every statement below is derived strictly from the contents of `update-another-deepseek.json`.

### How this maps to the template

| Template expectation | Reality for this artifact |
|---|---|
| Node graph (trigger → nodes → connections) | **Absent.** This is config/data, not a graph. |
| Node Inventory, Detailed Node Docs, Connection Docs, Merge, HTTP, Database, Vector DB, Loops, Code Nodes | **Not applicable** — collapsed under §A with a one-line reason each. Not padded with stubs. |
| Overview, Data Flow, Expression/Schema, Performance, Execution plan, Security, Improvements | **Fully applicable** — documented from real in-file content. |

---

## 1. Overview

| Aspect | Detail (from file) |
|---|---|
| **Purpose** | Define the complete specification for generating a bank of JEE Main 2026 Physics multiple-choice / numerical questions via an AI model, in deterministic, evenly-distributed batches. |
| **Business goal** | Produce a large, curriculum-complete, difficulty-balanced question bank (target ~2064 questions) suitable for an exam-prep / admin question platform. |
| **Problem solved** | Manual authoring of thousands of syllabus-aligned questions is slow and inconsistent. This config lets an automated pipeline generate them at scale with enforced distribution ratios and a strict output schema. |
| **Expected output** | JSON question objects conforming to `responseschema` (fields such as `question`, `optiontype`, `options`, `correctanswer`, `explanation`, plus taxonomy fields `category`/`chapter`/`topic`/`subtopic` and `questionid`/`batchid`). |
| **Artifact type** | Static configuration / data schema (JSON). |
| **Automation category** | AI content generation (batched LLM generation pipeline input). |
| **Trigger type** | **N/A** — a config file has no trigger. The consuming workflow owns the trigger. |
| **Complexity** | High data complexity (7 categories, 20 chapters, 71 topics, nested per-batch distribution), low structural complexity (pure declarative JSON). |
| **Dependencies** | The consuming n8n workflow; an LLM API (Gemini/DeepSeek per filename lineage); JSON validation against `responseschema`. |
| **External services** | LLM provider API (not named in this file). |
| **Internal services** | n8n execution engine (of the consuming workflow). |
| **Advantages** | Declarative, auditable, single source of truth for distribution; strict schema enforces output consistency; batching + concurrency + retry policy defined up front. |
| **Limitations** | Contains several internal numeric inconsistencies (see §3); some chapter topic-counts do not sum to the chapter total; distribution ratios stated in prose do not match the actual numbers. |

---

## 2. High-Level Architecture (of the pipeline this config feeds)

Data-flow view only — the node graph lives in the *consuming* workflow, not here.

```text
update-another-deepseek.json  (THIS FILE — config/data)
        |
        v
[Consuming n8n workflow]  (e.g. RWpigWQwMT7EBzp9 — NOT inspected here)
        |
        |  reads generationconfig + categories[] + responseschema
        v
Iterate categories -> chapters -> batches (batchsize = 6)
        |
        v
For each batch: build prompt from
   chapter.generationconfig.perbatchdistribution
        |
        v
LLM API call (6 questions/call, max 10 concurrent,
   3 retries w/ exponential backoff, 30s timeout)
        |
        v
Validate each returned question against responseschema
        |
        v
Persist question bank (destination NOT defined in this file)
```

**Data movement:** the file supplies *counts and ratios* (how many easy/moderate/hard, how many single/multiple/numerical, per chapter and per batch) plus the *output contract*. The consuming workflow supplies *behaviour* (looping, prompting, calling, validating, storing).

---

## 3. Data Integrity — Known Inconsistencies (highest-value section)

These are confirmed by summing the file's own numbers. They matter because different parts of the file will lead a reader (or a workflow) to different totals.

| # | Field | Stated value | Actual value | Where it appears |
|---|---|---|---|---|
| 1 | **Total questions** | **2130** | **2064** | 2130 in `metadata.note`, `metadata.apicallsummary.*`, and bottom `apicallsummary.totalquestions`. 2064 in `generationconfig.totalquestions`, in `questiondistribution` (688+1032+344 = 2064), and as the sum of all 7 category `questioncount`s (834+150+60+480+90+210+240 = 2064). |
| 2 | **Total API calls** | **355** | **344** | 355 in `metadata.apicallsummary.totalapicalls` and bottom `apicallsummary.totalapicalls`. 344 in `generationconfig.totalapicalls` and as the sum of category `apicalls` (139+25+10+80+15+35+40 = 344). |
| 3 | **Chapter 4 (`work, energy & power`) topics** | chapter `questioncount` = 54 | topics sum to **30** | `topics[]`: `work`=30, `energy`=0, `power`=0, `collisions`=0. But chapter `questioncount`=54 and both `difficultydistribution` (18+27+9=54) and `questiontypedistribution` (36+9+9=54) say 54. 24 questions are unassigned to any topic. |
| 4 | **`difficultyratio`** | "30% easy : 50% moderate : 20% hard" | **33.3% : 50% : 16.7%** | `questiondistribution.difficulty` = 688/1032/344 of 2064 → 1/3, 1/2, 1/6. |
| 5 | **`questiontyperatio`** | "60% single : 20% multiple : 20% numerical" | **66.7% : 16.7% : 16.7%** | `questiondistribution.questiontype` = 1376/344/344 of 2064 → 2/3, 1/6, 1/6. |
| 6 | **`metadata.note` hard-coded values** | "sum exactly to that chapter's questioncount (30)" and "sum to totalquestions (2130)" | both wrong | Chapter counts are **not** all 30 (they range 54–240); total is **2064**, not 2130. |

**Cosmetic / structural (non-blocking) observations:**
- Category `id`s are non-sequential: **1, 9, 3, 4, 5, 6, 7** — missing 2 and 8, and 9 appears out of order. `name`s are unique, so lookups by name are safe; lookups by `id` are not contiguous.
- Inside `thermodynamics-phy`, chapter `id`s appear as **22 then 21** (out of order).
- `metadata.totalcategories` = 7 (matches array length ✓) and `metadata.totalchapters` = 20 (matches: 7+2+1+5+1+3+1 = 20 ✓).
- Two `perbatchdistribution` blocks are **identical for every chapter** (2 easy / 3 moderate / 1 hard; 4 single / 1 multiple / 1 numerical = 6/batch). This only stays internally consistent for chapters whose overall ratio is exactly 2:3:1 and 4:1:1; it does **not** reflect chapters with other ratios at the batch level (though every chapter's totals still divide evenly into 6-question batches).

> **Recommendation:** pick 2064/344 as canonical (they are corroborated by three independent in-file sources each) and correct the three "2130 / 355" summaries plus `metadata.note`. Fix chapter 4 by assigning the missing 24 questions to `energy`/`power`/`collisions`.

---

## 4. Top-Level Structure (field-by-field)

### 4.1 `generationconfig` (global)

| Field | Value | Meaning |
|---|---|---|
| `exam` | `"jee main"` | Target examination. |
| `year` | `2026` | Target exam year. |
| `subject` | `"physics"` | Single subject scope. |
| `language` | `"english"` | Output language. |
| `totalquestions` | `2064` | Canonical grand total (see §3 #1). |
| `batchsize` | `6` | Questions per generation batch. |
| `strictmode` | `true` | Enforce strict schema/distribution adherence. |
| `outputformat` | `"json"` | Output serialization. |
| `questionsperapicall` | `6` | Questions requested per LLM call (equals `batchsize`). |
| `totalapicalls` | `344` | Canonical total call count (see §3 #2). |

### 4.2 `metadata`

| Field | Value | Notes |
|---|---|---|
| `version` | `"2.0"` | Config schema version. |
| `totalcategories` | `7` | Matches `categories.length` ✓. |
| `totalchapters` | `20` | Matches sum of chapters ✓. |
| `difficultyratio` | `"30% easy : 50% moderate : 20% hard"` | **Prose only; does not match actual** (§3 #4). |
| `questiontyperatio` | `"60% single : 20% multiple : 20% numerical"` | **Prose only; does not match actual** (§3 #5). |
| `note` | see file | Contains the wrong hard-coded `(30)` and `(2130)` (§3 #6). |
| `apicallsummary.totalapicalls` | `355` | **Wrong** — should be 344 (§3 #2). |
| `apicallsummary.questionspercall` | `6` | Consistent with `generationconfig`. |
| `apicallsummary.batchesperchapter` | `"varies by chapter questioncount"` | Descriptive. |
| `apicallsummary.distributionstrategy` | `"distribute questions evenly across batches with rotation for remainders"` | Batching algorithm hint. |

### 4.3 `categories[]`

Array of 7 category objects. Each: `id`, `name`, `questioncount`, `apicalls`, `chapters[]`.

| Category `name` | `id` | `questioncount` | `apicalls` | Chapters |
|---|---|---|---|---|
| mechanics | 1 | 834 | 139 | 7 |
| thermodynamics-phy | 9 | 150 | 25 | 2 |
| oscillations & waves | 3 | 60 | 10 | 1 |
| electromagnetism | 4 | 480 | 80 | 5 |
| optics | 5 | 90 | 15 | 1 |
| modern physics | 6 | 210 | 35 | 3 |
| experimental skills | 7 | 240 | 40 | 1 |
| **Total** | — | **2064** | **344** | **20** |

Relationship holding throughout: **`apicalls × 6 = questioncount`** for every category (e.g. 139 × 6 = 834 ✓, 80 × 6 = 480 ✓). This confirms 344 (not 355) as the correct call total.

### 4.4 `chapters[]` (inside each category)

Each chapter object: `id`, `name`, `questioncount`, `apicalls`, `generationconfig`, `topics[]`.

`chapter.generationconfig` contains:
- `batchsize` (always 6),
- `difficultydistribution` `{easy, moderate, hard}` — sums to the chapter's `questioncount` (verified for all except chapter 4, which sums correctly at 54 in its distributions but whose *topics* under-sum — §3 #3),
- `questiontypedistribution` `{single, multiple, numerical}` — sums to `questioncount`,
- `perbatchdistribution.difficulty` `{easy:2, moderate:3, hard:1}` and `perbatchdistribution.questiontype` `{single:4, multiple:1, numerical:1}` — both sum to 6.

### 4.5 `topics[]` (inside each chapter)

Each topic: `name`, `subtopics[]` (array of syllabus strings), `id` (globally sequential 1–71), `questioncount` (30 for every populated topic; 0 for the three empty chapter-4 topics). `subtopics` are the granular JEE syllabus items the generator should draw from.

### 4.6 `questiondistribution` (global rollups)

```json
"difficulty":   { "easy": 688,  "moderate": 1032, "hard": 344 }   // sum 2064
"questiontype": { "single": 1376, "multiple": 344, "numerical": 344 } // sum 2064
```
Both corroborate the **2064** canonical total.

### 4.7 `apicallsummary` (bottom, global)

| Field | Value | Notes |
|---|---|---|
| `totalapicalls` | `355` | **Wrong** — 344 (§3 #2). |
| `questionspercall` | `6` | ✓ |
| `totalquestions` | `2130` | **Wrong** — 2064 (§3 #1). |
| `distributionmethod` | `"chapter-wise batching with rotation strategy"` | Algorithm. |
| `executionplan.parallelcalls` | `"Maximum 10 concurrent API calls"` | Concurrency cap (§8). |
| `executionplan.retrypolicy` | `"3 retries with exponential backoff"` | Error handling (§7). |
| `executionplan.timeout` | `"30 seconds per call"` | Per-call timeout (§7). |

### 4.8 `responseschema` (output contract) — see §5.

---

## 5. Response Schema (the output contract)

`responseschema` defines the required shape of **each generated question**. This is the most operationally important block — the consuming workflow should validate every LLM response against it.

### 5.1 `requiredfields` (15)

`role`, `question`, `optiontype`, `inputBox`, `difficultylevel`, `options`, `correctanswer`, `explanation`, `subject`, `category`, `chapter`, `topic`, `subtopic`, `questionid`, `batchid`.

### 5.2 `fieldconstraints`

| Field | Constraint |
|---|---|
| `optiontype` | one of `"Single"`, `"Multiple"`, `"Numerical"`. |
| `role` | fixed `"admin"`. |
| `inputBox` | fixed empty string `""`. |
| `difficultylevel` | one of `"easy"`, `"moderate"`, `"hard"`. |
| `subject` | fixed `"physics"`. |
| `options` | `Single`: array of exactly 4 choices (a,b,c,d). `Multiple`: array of exactly 4 choices, ≥1 correct. `Numerical`: empty string (no options). |
| `correctanswer` | `Single`: one option key, e.g. `"a"`. `Multiple`: array of keys, e.g. `["a","c"]`. `Numerical`: numeric value as a string, e.g. `"4.50"`. |

> **Note:** `optiontype` values are capitalized (`Single`/`Multiple`/`Numerical`), but `difficultylevel` and `subject` are lowercase, and `questiontypedistribution` keys are lowercase (`single`/`multiple`/`numerical`). The consuming workflow must normalize case when mapping distribution counts to `optiontype`.

### 5.3 `example`

```json
{
  "question": "What is the SI unit of force?",
  "optiontype": "Single",
  "difficultylevel": "easy",
  "options": ["Newton", "Joule", "Watt", "Pascal"],
  "correctanswer": "a",
  "explanation": "Newton is the SI unit of force, named after Sir Isaac Newton."
}
```
Note the example omits several `requiredfields` (`role`, `inputBox`, `subject`, taxonomy fields, `questionid`, `batchid`) — it is illustrative of the answer shape only, not a complete record.

---

## 6. Data Flow

- **Input data:** this JSON (static). No runtime input fields; the file *is* the input to the pipeline.
- **Intermediate data:** per-batch prompt payloads assembled from `chapter.generationconfig.perbatchdistribution` + `chapter.name` + `topics[].subtopics`.
- **Output data:** arrays of question objects validated against `responseschema`.
- **Field mappings:** distribution counts (lowercase) → `optiontype` (capitalized) and `difficultylevel`; `category.name` → question `category`; `chapter.name` → `chapter`; `topic.name` → `topic`; a chosen `subtopic` → `subtopic`; generated ids → `questionid`, `batchid`.
- **Split / batch behavior:** each chapter's `questioncount` splits into `apicalls` batches of `batchsize` (6). "Rotation for remainders" handles chapters whose distribution does not divide evenly (per `distributionstrategy`).
- **Binary data / memory / static data:** none defined in this file.

---

## 7. Error Handling & Retry (from `executionplan`)

- **Retry policy:** `"3 retries with exponential backoff"` — transient LLM/API failures are retried up to 3 times with increasing delay. Prevents hammering the API on a spike.
- **Timeout:** `"30 seconds per call"` — a call exceeding 30s is aborted (and subject to the retry policy).
- **Not defined in this file (owned by the consuming workflow):** dead-letter handling, poison-response handling, per-HTTP-status branching (401/403/404/429/5xx), and logging. Project memory notes a related **poison-400 retry bug** and a Switch-based fix in the Gemini workflow ([[gemini-workflow-retry-loop]]) — relevant to whichever workflow consumes this config, but **not present here**.

---

## 8. Performance / Execution Plan

| Parameter | Value | Implication |
|---|---|---|
| Concurrency | Max **10 concurrent** API calls | Up to 10 batches in flight; total 344 calls → ~35 sequential waves at full concurrency. |
| Batch size | **6** questions/call | Fewer, larger calls reduce overhead vs. 1/call; keeps prompt/response sizes bounded. |
| Timeout | **30 s**/call | Caps tail latency per call. |
| Retries | **3**, exponential backoff | Absorbs transient failures without unbounded retry. |
| Distribution | chapter-wise batching + rotation | Even spread; avoids clustering all "hard" questions in one batch. |

**Throughput estimate (illustrative, not in file):** at 10 concurrent calls averaging ~a few seconds each, 344 calls complete in a small number of minutes barring rate limits. Actual time depends on the LLM provider's latency and rate limits (neither specified here).

---

## 9. Security

- **No secrets present** — this file contains no API keys, tokens, or credentials. ✓ (Correct: credentials belong in n8n's credential store, not config JSON.)
- **No PII** — purely academic content specification.
- **Recommendation:** keep the LLM provider key in an n8n credential referenced by the consuming workflow; never inline it into this config or into Code nodes.

---

## A. Sections Not Applicable to This Artifact

These template sections require an n8n **node graph**, which this file does not contain. Listed explicitly (not silently dropped) with the reason:

| Template section | Why N/A here |
|---|---|
| 3. Workflow Diagram (node graph) | No `nodes`/`connections`; the pipeline diagram in §2 is the data-flow substitute. |
| 4. Node Inventory | No nodes to inventory. |
| 5. Detailed Node Documentation | No nodes. |
| 6. Connection Documentation | No `connections` object. |
| 7. Execution Flow (per-node) | No nodes to step through; §2/§6 describe data flow instead. |
| 9. Expression Documentation (`$json`, `$node`, …) | No n8n expressions exist in a static config file. |
| 10. Loop Documentation (Split In Batches, etc.) | Looping is a property of the consuming workflow, not this file. |
| 11. Conditional Logic (IF/Switch/Filter) | None present. |
| 12. Merge Documentation | No Merge node. |
| 13. Code Nodes | No Code nodes / JavaScript. |
| 14. HTTP Requests (node-level) | No HTTP Request node; only the abstract call policy in §8. |
| 15. AI Components (Agent/Model/Memory/Tools) | No AI nodes; only the output contract (`responseschema`) and call policy. |
| 16. Database Documentation | No DB nodes; persistence target undefined in this file. |
| 17. Vector Database | None. |
| 21. Logging | Not defined in this file; owned by the consuming workflow. |

---

## 10. Testing Strategy (for validating this config before a run)

- **Sum checks (unit-style):** assert Σ category `questioncount` == `generationconfig.totalquestions`; assert Σ category `apicalls` == `generationconfig.totalapicalls`; assert `apicalls × 6 == questioncount` per category and chapter.
- **Distribution checks:** assert each chapter's `difficultydistribution` and `questiontypedistribution` each sum to that chapter's `questioncount`.
- **Topic checks:** assert Σ `topics[].questioncount` == chapter `questioncount` (this currently **fails for chapter 4** — §3 #3).
- **Schema checks:** validate a sample generated question against `responseschema.requiredfields` and `fieldconstraints` (especially the per-`optiontype` `options`/`correctanswer` rules).
- **Ratio checks:** if `difficultyratio`/`questiontyperatio` prose is meant to be authoritative, assert it against `questiondistribution` (currently **fails** — §3 #4, #5).

---

## 11. Troubleshooting

| Symptom | Root cause | Diagnosis | Resolution | Prevention |
|---|---|---|---|---|
| Consumer generates 2130 or 355 calls | Reading a wrong summary field | Compare which block the workflow reads | Standardize on `generationconfig` (2064 / 344) | Fix the two wrong summaries + `metadata.note` (§3) |
| Chapter 4 short by 24 questions | Topic counts under-sum (30 vs 54) | Sum `topics[].questioncount` | Assign 24 questions to energy/power/collisions | Add the sum check in §10 |
| Wrong difficulty/type mix | Trusting prose ratios | Recompute from `questiondistribution` | Use numeric distributions, not prose | Treat prose ratios as non-authoritative |
| Lookup by category `id` misses | Non-sequential ids (no 2/8) | Inspect `categories[].id` | Look up by `name`, or renumber ids 1..7 | Keep ids contiguous |
| Schema validation fails on Numerical | `options` expected non-empty | Check `optiontype` handling | For `Numerical`, `options` = `""`, answer in `correctanswer` | Encode per-type rules in validator |

---

## 12. Maintenance Guide

- **Single source of truth:** treat `generationconfig` + `questiondistribution` + category sums as canonical; regenerate the two `apicallsummary` blocks and `metadata.note` from them rather than hand-editing.
- **Adding a chapter/topic:** update the chapter `questioncount`, both distributions, `topics[]`, the parent category totals, `metadata.totalchapters`, and the global rollups — then re-run the §10 checks.
- **Versioning:** bump `metadata.version` on structural changes.
- **Backup:** this file is under git (`update-another-deepseek.json`); commit changes with a message describing the distribution change.

---

## 13. Recommended Improvements

1. **Fix the six inconsistencies in §3** — highest priority; they will silently corrupt run totals.
2. **Derive summaries programmatically** so `totalquestions`/`totalapicalls` cannot drift again.
3. **Make prose ratios match numbers** (or delete the prose and keep only `questiondistribution`).
4. **Complete chapter 4 topic counts** (assign the missing 24 questions).
5. **Contiguous ids** for categories and chapters to make id-based lookups safe.
6. **Add a `$schema` / JSON Schema** file so this config can be machine-validated in CI before a generation run.
7. **Externalize provider + model** (currently only implied by filename lineage "deepseek"/"gemini") into an explicit field for clarity.

---

## 14. Glossary

| Term | Meaning |
|---|---|
| **JEE Main** | Joint Entrance Examination (Main) — Indian engineering entrance exam. |
| **Category / Chapter / Topic / Subtopic** | Four-level syllabus taxonomy used to organize and tag questions. |
| **Batch** | A group of `batchsize` (6) questions generated in one API call. |
| **`optiontype`** | Question format: `Single` (one correct), `Multiple` (≥1 correct), `Numerical` (numeric answer). |
| **`responseschema`** | The required output contract every generated question must satisfy. |
| **Distribution** | A count breakdown (easy/moderate/hard, or single/multiple/numerical) that must sum to a parent total. |
| **Rotation strategy** | Method for spreading remainder questions evenly across batches when counts don't divide evenly. |
| **Exponential backoff** | Retry technique where wait time grows after each failed attempt. |
| **n8n workflow** | A node-graph automation (has `nodes` + `connections`); the *consumer* of this config, not this file. |

---

## 15. Assumptions & Missing Information (explicit)

Per the template's rule to state what is missing rather than invent it:

- **The consuming n8n workflow is not included** in this file and was **not inspected** for this document. Its trigger, nodes, prompt text, LLM credential, storage/DB target, and error branches are therefore **undocumented here**. Memory suggests workflow `RWpigWQwMT7EBzp9` ([[gemini-workflow-retry-loop]]).
- **LLM provider/model is not stated** in the file (only implied by the filename lineage "deepseek"/"gemini").
- **Persistence destination** (where generated questions are stored) is **not defined** in this file.
- **No configuration values were invented.** Every number, field, and ratio above is quoted from `update-another-deepseek.json`; discrepancies are reported, not resolved silently.
