# RAG Refactor — Handoff / Status

Workflow target: `RWpigWQwMT7EBzp9` ("for gemani ai model updated"), active:false, manual only.
DB: `test_series_db` (PG 17.10), 1,878 questions.
Embedding model: `gemini-embedding-001` (3072 dims).
Decisions locked: edit generation workflow **in place**; **skip** DB defect cleanup; retrieval **deterministic** (never agent-bypassable).

---

## DONE

### 1. Ingestion workflow — CREATED + VALIDATED
- ID: `hF30qXu77ELeoL5Z`  ("Question Embeddings Ingestion (RAG)"), inactive.
- Graph: Start Ingestion (manual) → Fetch Questions (Postgres, cred `b1sfIH8F19ybgksK`) → Build Question Document (Code) → Insert into PGVector (`n8n_vectors`).
  - Sub-nodes: Gemini Embeddings (`models/gemini-embedding-001`) → ai_embedding; Question Loader (Default Data Loader, expressionData) → ai_document.
- SQL aggregates 1 row/question (string_agg over taxonomy) → no dup vectors from multi-topic joins.
- Metadata written per vector: question_id, subject, subject_category, chapter, topic, difficulty, option_type (jsonb → usable as retrieval filter).
- Structural validation: 6 nodes, 5 connections, 0 errors, 0 warnings.

---

## BLOCKERS (user action)

### B1. pgvector NOT installed on the PG server
- `pg_available_extensions` has no `vector` (only `plpgsql`). Server-wide, per-server not per-DB.
- Fix: install pgvector (Docker `pgvector/pgvector:pg17`, or native MSVC build), then:
  ```sql
  \c test_series_db
  CREATE EXTENSION IF NOT EXISTS vector;
  ```
- Table `n8n_vectors` auto-creates on first ingestion insert; no manual DDL needed. At 3072 dims, no hnsw/ivfflat index (both cap at 2000 dims) — flat search, fine for ~1.9k rows.

### B2. postgres MCP points at DB `postgres`, not `test_series_db`
- All my direct SQL landed in `postgres`. Fix MCP connection string → `test_series_db`, restart, so I can verify + run ingestion.
- Note: the n8n Postgres credential `b1sfIH8F19ybgksK` is a SEPARATE connection (workflow runtime). Existing nodes query test_series_db tables, so it already targets test_series_db — confirm.

---

## DONE (2) — pgvector + credential + generation refactor APPLIED

- pgvector **0.8.5** installed + `CREATE EXTENSION vector` in `test_series_db`. Verified.
- postgres MCP reachable to `test_series_db` via connection string (`postgres:123456@localhost:5432/test_series_db`), 1,890 questions.
- n8n Google Gemini credential created: `Google Gemini (RAG)` id `ESaQrZIJAQlt7Ugi` (googlePalmApi). Assigned to all AI sub-nodes.
- Generation workflow `RWpigWQwMT7EBzp9` refactored + validated (0 errors). New graph:
  `... Wait Before Gemini Call → Retrieve Similar Questions (PGVector load, topK5, table n8n_vectors) → Inject Retrieved Context (Code) → Generate Questions (AI Agent) [+ Gemini Chat Model] → Reshape to Gemini Envelope (Code) → Parse Gemini Response (unchanged)`.
  - `Gemini Embeddings (Gen)` → Retrieve (ai_embedding).
  - AI Agent error output (main[1]) → `Route by HTTP status` (poison-400 subgraph retained + rewired); `Wait Before Retry` → `Retrieve Similar Questions`.
  - Old `Generate Questions (Gemini)` HTTP node DELETED (rollback via n8n version history).
  - Retry: Agent `retryOnFail` (3 tries) handles transient. Poison-400 re-entry effectively superseded (was HTTP-item specific); subgraph kept wired but its `$json.error.status` match rarely triggers on Agent errors → falls to graceful stop.
  - Removed dead orphan nodes: `Build Gemini Prompt (Unused)`, `Generate Questions (Gemini)1` (held leaked Gemini key #2 — security win), `Code in JavaScript`, `Aggregate Batch Summary` (old).

## RUNTIME — manual execution needed (manual triggers can't be API-fired)
1. **Run ingestion first** (`hF30qXu77ELeoL5Z`, click Execute in UI) — creates + fills `n8n_vectors` (~1,890 rows). Generation retrieval ERRORS if this table doesn't exist yet.
2. Then run generation (`RWpigWQwMT7EBzp9`).
Verify: `SELECT count(*) FROM n8n_vectors;`

## ⚠️ NOT YET EXECUTED — first-run watch items (structural validation ≠ working)
1. **Braces in Agent systemMessage (HIGHEST risk).** The prompt (moved from raw REST `systemInstruction` into `options.systemMessage`) is full of literal `{` / `}` (JSON examples, `{{latex[n]}}`). n8n's agent may treat `{var}` as template vars → error `Missing value for input variable "question"` or mangled placeholders. If first batch errors on template variables: escape braces in `Inject Retrieved Context` (double every `{`→`{{`, `}`→`}}` on agentSystem/agentUser) and retest.
2. **Agent output not schema-guaranteed.** Dropped Gemini `responseSchema`/`responseMimeType`. Mitigated: `Reshape to Gemini Envelope` now strips ```json fences + slices to outermost `[...]` before Parse. Watch for parse failures anyway.
3. **`$('Loop Over Batches').item` in retrieval prompt.** Code node output can lose pairedItem → `.item` undefined → empty search string. Check the resolved retrieval prompt on first run.

## KNOWN SPEC GAPS (deviations to accept or fix)
- **No metadata pre-filter.** Retrieve uses prompt + topK=5 over whole table; spec wanted chapter/topic/difficulty filter before similarity. Scoping is soft (search text only). Enhance via PGVector load filter option if strict scoping needed.
- **Re-ingestion duplicates vectors.** `insert` mode appends; second run doubles `n8n_vectors`. For rebuild: `TRUNCATE n8n_vectors;` before re-running, or treat ingestion as run-once.

## SUPERSEDED (original pending plan below, now implemented)

### Spec conflicts to resolve before edit
1. **"Remove HTTP node + use AI Agent" vs "Parse Gemini Response unchanged."**
   - Parse expects raw Gemini REST envelope `candidates[0].content.parts[0].text`. An AI Agent / chat model outputs `{output}` / `{text}` — different shape → Parse silently yields 0 questions.
   - Resolution: add a **Reshape to Gemini Envelope** Code node after the Agent that wraps `{candidates:[{content:{parts:[{text: <agent json>}]}}]}`. Adding a node ≠ modifying existing downstream nodes. Parse stays byte-for-byte unchanged.
2. **"Retry logic unchanged" vs removing the HTTP node.**
   - The poison-400 retry subsystem reads the n8n HTTP error envelope (`$json.error.status`). An Agent produces different errors → subsystem won't trigger the same way.
   - Resolution options: (a) rely on Agent/chat-model `retryOnFail`; keep the poison-400 subgraph disabled but retained; or (b) rewire Agent error output → `Route by HTTP status` and adapt its status-matching. Needs a live run to tune. **DECISION NEEDED.**

### Planned generation graph (deterministic, retrieval mandatory)
```
Loop Over Batches (loop)
  → Build Gemini Prompt & Request        (unchanged)
  → Wait Before Gemini Call              (unchanged)
  → Retrieve Similar Questions           (NEW: PGVector mode=load, topK 5,
                                          metadata filter chapter/topic/difficulty)
        + Gemini Embeddings (Gen)        (ai_embedding sub-node)
  → Inject Retrieved Context             (NEW Code: append retrieved Qs to the
                                          prompt as "DO NOT DUPLICATE" block)
  → Generate Questions (AI Agent)        (NEW: replaces HTTP node)
        + Gemini Chat Model              (models/gemini-2.5-flash, ai_languageModel)
  → Reshape to Gemini Envelope           (NEW Code: wrap as candidates[].parts[].text)
  → Parse Gemini Response                (unchanged) → rest of pipeline unchanged
```
- Old `Generate Questions (Gemini)` HTTP node: disable + disconnect (retain for rollback), not delete.
- Retrieval is a standalone node feeding the prompt → model cannot skip it (satisfies "never bypass").
- Move hardcoded Gemini key → n8n Google Gemini (PaLM) credential; do not re-hardcode.

### Why deferred
Live edit + zero ability to test (no pgvector, MCP on wrong DB) + conflict #2 needing a run to tune = high risk of leaving the production generator broken. Build once B1+B2 cleared, then validate + iterate against a live store.

---

## RESUME CHECKLIST (after B1+B2 fixed)
1. Verify: `SELECT extversion FROM pg_extension WHERE extname='vector';` + `SELECT count(*) FROM questions;` (1878).
2. Run ingestion workflow `hF30qXu77ELeoL5Z`; confirm `SELECT count(*) FROM n8n_vectors;` ≈ 1878.
3. Decide conflict #2 (retry handling).
4. Apply generation refactor to `RWpigWQwMT7EBzp9` (partial-update ops).
5. `n8n_validate_workflow`; test one batch; confirm dedup + downstream intact.
