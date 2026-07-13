# Workflow Review — Improvement Opportunities

> **Workflow:** `for gemani ai model updated` (`RWpigWQwMT7EBzp9`) · 54 nodes · Manual trigger
> **Scope:** Review only — no logic was changed. Every finding maps to an actual node inspected via the n8n MCP.

## Priority Summary

| # | Finding | Severity | Effort |
|---|---------|----------|--------|
| 13 | Exposed secrets (Gemini key, backend creds, token) | 🔐 Critical | Low |
| 1 | Retry counter leaks in static data | 🔴 Bug | Low |
| 2 | Silent question loss on store errors | 🔴 Bug | Low |
| 3 | Count-verification asymmetry | 🔴 Bug | Low |
| 4 | `success=false` computed but ignored | 🟠 Reliability | Medium |
| 5 | Invalid questions silently dropped | 🟠 Reliability | Medium |
| 6 | Inconsistent HTTP hardening (retry/timeout) | 🟠 Reliability | Low |
| 7 | Fragile filesystem state (`state.json`) | 🟠 Reliability | Medium |
| 8 | Redundant logins per batch | 🟡 Efficiency | Low |
| 9 | No question de-duplication | 🟡 Efficiency | Medium |
| 10 | Hardcoded model/config drift | 🟡 Efficiency | Low |
| 11 | Dead / no-op nodes | 🟢 Maintainability | Low |
| 12 | ~~Always-true `If` gates~~ ✅ **Resolved** | 🟢 Maintainability | Low |

**Suggested order:** 13 → 1, 2 → 4 → 8 → 11, ~~12~~

> **Update:** finding **#12 has been implemented** (see the section below). Remaining open items are unchanged.

---

## 🔴 Correctness Bugs

### 1. Retry counter leaks in static data (unbounded growth)
In `Strip error & count retries`, the per-question retry counter is only cleaned up when retries are **exhausted**:
```js
const count = (staticData.__geminiRetry[key] || 0) + 1;
staticData.__geminiRetry[key] = count;
const exceeded = count > MAX;
if (exceeded) delete staticData.__geminiRetry[key];   // only cleaned on failure
```
On a **successful** retry the key is never deleted. Over 355 batches × repeated runs, `staticData.__geminiRetry` accumulates one stale hash key per recovered question — forever.

**Fix:** delete the key on success too. Add to `Parse Gemini Response` (the success path) or reset the object at loop start:
```js
const sd = $getWorkflowStaticData('global');
if (sd.__geminiRetry) delete sd.__geminiRetry[key];   // same hash key as the strip node
```

### 2. Silent question loss on store errors
- `Store Non-LaTeX Questions` has `onError: continueErrorOutput`, but its **error output is not wired anywhere** → failed items disappear with no trace.
- `Store LaTeX Questions` has the opposite problem: **no error handling**, so a single backend `500` fails the whole batch/loop.

**Fix:** make both consistent — route each error output to a logging / dead-letter node (or a notification), and add `Retry On Fail` + a request timeout.

### 3. Count-verification asymmetry
`Verify LaTeX Store Count` counts `$('Remove LaTeX Field (LaTeX)')`, while `Verify Non-LaTeX Store Count` counts `$('Split Non-LaTeX Questions')`. The two "generated" measures are taken at different pipeline stages, then both are compared against the validator summary — so the resulting `success` flag can be a false positive/negative.

**Fix:** align both verifiers to the same reference point (e.g. count after the same normalization step in each branch).

---

## 🟠 Reliability & Error Handling

### 4. `success === false` is computed but ignored
`Aggregate Batch Summary` produces a `success` boolean and counts, but nothing branches on it — no alert, no re-queue, no stop.

**Fix:** add an `If` on `success` → notification (Slack/email) or push the failed batch to a retry queue. Also configure a workflow-level **Error Workflow** in workflow settings.

### 5. Invalid questions are silently dropped
`Skip Invalid LaTeX Questions` and `Skip Invalid Non-LaTeX Questions` discard malformed questions, so a batch can store **fewer** than requested with no back-fill.

**Fix:** re-queue invalid questions for regeneration (feed them back into `Build Gemini Prompt & Request`) until the target count is met.

---

## 🟡 Efficiency

### 8. Redundant logins
Each batch logs in **twice** (LaTeX and Non-LaTeX branches) → ~710 logins over a full run. `Attach Token to LaTeX Questions` already caches the token in static data, but nobody reuses it.

**Fix:** log in once per run (or per batch), cache the token, and reuse it in both store calls.

### 6. Inconsistent HTTP hardening
Observed settings:

| Node | retryOnFail | onError | timeout |
|------|-------------|---------|---------|
| Generate Questions (Gemini) | ✅ | continueErrorOutput | ❌ |
| Login API (LaTeX) | ❌ | — | ❌ |
| Login API (Non-LaTeX) | ✅ | — | ❌ |
| Convert LaTeX to MathML | ❌ | — | ❌ |
| Store LaTeX Questions | ❌ | — | ❌ |
| Store Non-LaTeX Questions | ❌ | continueErrorOutput | ❌ |

**Fix:** standardize `Retry On Fail` (with back-off) and a request **timeout** on every HTTP node.

### 7. Fragile filesystem state
`Build Loop Items & State` reads/writes `/home/node/.n8n-files/state.json` via `fs`. Concurrent runs race on this file, and it makes runs non-idempotent and hard to reason about.

**Fix:** move the cursor/state into Postgres (or workflow static data) with a proper transactional update.

### 9. No question de-duplication
Nothing prevents storing duplicates when a batch is retried/reprocessed.

**Fix:** add a dedup step, or send an idempotency key to `POST /api/questions` and let the backend upsert.

### 10. Hardcoded model / config drift
`gemini-2.5-flash` is baked into the request URL, and the two prompt builders default `questionNumber` differently (`6` in the live builder vs `8` in the disconnected copy).

**Fix:** parameterize model name and per-batch question count via `Set Generation Config`.

---

## 🟢 Maintainability

### 11. Dead / no-op nodes to remove
- `Build Gemini Prompt (Unused)` — disconnected alternate prompt builder.
- `Build Loop Items & State (Unused)` — disconnected duplicate loop builder.
- `Finalize Loop Output`, `Passthrough to Loop Builder`, `Skip Invalid LaTeX Questions`, `Skip Invalid Non-LaTeX Questions` — pass-throughs still holding default `myNewField = 1` boilerplate.

### 12. Always-true `If` gates — ✅ Resolved
**Original problem:**
- `Gate: Run DB Lookup` — condition was `true == true`; the DB lookup always ran.
- `Proceed to Store (LaTeX)` — condition was `1 == 1`; it never filtered.
- The LaTeX branch **lacked** the "has questions" guard that the non-LaTeX branch has (`Gate: Has Non-LaTeX Questions`).

**What was implemented:**
- **Removed** `Gate: Run DB Lookup`. `Set Generation Config` now connects directly to `Fetch Category/Chapter/Topic IDs` (and still to `Merge Config & DB Lookup`), preserving the merge inputs.
- **Converted** `Proceed to Store (LaTeX)` into a real guard, renamed **`Gate: Has LaTeX Questions`**, with condition `{{ Object.keys($json).length }} > 0` — identical to the non-LaTeX branch's `Gate: Has Non-LaTeX Questions`. Empty items are now skipped instead of hitting login + store; valid questions (which always have fields) pass through unchanged.

**Verification after change:** 53 nodes · 55 connections · zero stale `$('…')` references. Flow intact:
`Set Generation Config → Fetch Category/Chapter/Topic IDs → Merge Config & DB Lookup` and
`Remove LaTeX Field (LaTeX) → Gate: Has LaTeX Questions → Login API (LaTeX)`.

---

## 🔐 Security (highest-value overall)

### 13. Secrets are exposed
- **Gemini API key** is passed as a **URL query parameter** (`?key=...`) and is duplicated in the `ENV` sticky note → it lands in execution logs and any exported workflow JSON.
- **Backend login** is **hardcoded** (`naveed@gmail.com` / same value as password) in both `Login API` nodes.
- **Access token** is stored in a `Set` field (n8n's `SET_CREDENTIAL_FIELD` warning).

**Fix:**
1. Move all secrets into n8n **credentials** (HTTP Header Auth / Google credential / Postgres credential).
2. Delete the API key from the `ENV` sticky note.
3. **Rotate** the currently-exposed Gemini key and backend password.

---

## Notes on Implementation Safety
Any change that renames or removes a node must also update `$('node name')` references inside Code nodes and HTTP headers — n8n's rename updates connections but **not** these string references. Current reference map:

| Referenced node | Used by |
|-----------------|---------|
| `Validate & Classify Questions` | Verify LaTeX Store Count, Verify Non-LaTeX Store Count |
| `Remove LaTeX Field (LaTeX)` | Verify LaTeX Store Count, Attach Token to LaTeX Questions |
| `Split Non-LaTeX Questions` | Verify Non-LaTeX Store Count, Prepare Non-LaTeX Questions |
| `Select LaTeX Questions` | Attach MathML to Questions |
| `Set Access Token (LaTeX)` | Attach Token to LaTeX Questions, Store LaTeX Questions (header) |
| `Set Access Token (Non-LaTeX)` | Store Non-LaTeX Questions (header) |

Verify zero stale references after any structural edit.
