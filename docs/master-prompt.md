Yes. For your master prompt, it should be much more explicit than just describing the workflow. It should become a **non-negotiable architectural contract** that Claude Code must preserve.

You can use the following section in your master prompt.

````md
# HTTP REQUEST ERROR HANDLING ARCHITECTURE (MANDATORY)

This workflow contains a production-grade HTTP Request retry architecture.

Before modifying anything, you MUST understand the existing execution flow.

```
                  ┌──────────────► Success Path
                  │               (Extract AI Response to JSON)
                  │
HTTP Request2 ────┤
(Gemini API)      │
                  │
                  └──────────────► Error Output
                                   │
                                   ▼
                              Status Router
                          (Current: If Node)
                          (Target: Switch Node)
                                   │
        ┌──────────────┬──────────────┬──────────────┐
        │              │              │              │
      Retryable    Non-Retryable    Logging      Default
        │
        ▼
      Wait
        │
        ▼
Retry HTTP Request2
```

## Existing Behaviour

The HTTP Request node is configured with:

- Retry On Fail = Enabled
- Maximum Retries = 3
- Wait Between Retries = 3000 ms
- On Error = Continue (using Error Output)

This means the Error Output is reached ONLY after the built-in HTTP retries have been exhausted.

Do NOT duplicate or replace this internal retry mechanism.

Instead, preserve it exactly as implemented.

---

## Architectural Contract

The Success path and the Error path are independent.

### Success Path

```
HTTP Request
      │
      ▼
Extract AI Response to JSON
```

This path is production-tested.

Do NOT modify it unless explicitly instructed.

---

### Error Path

Only the Error Output may be refactored.

The existing IF node is an implementation detail—not the architecture.

When requested, replace ONLY the IF node with a Switch node while preserving the overall execution flow.

The resulting architecture must remain:

```
HTTP Request Error Output
            │
            ▼
      Switch Node
            │
   Status-based Branches
            │
     Wait (retry only)
            │
            ▼
    HTTP Request (same node)
```

Never redesign the retry loop.

Never reconnect the Success branch.

Never change upstream or downstream workflow logic.

---

## Retryable Status Codes

Only these status codes may return to the HTTP Request node:

- 408
- 429
- 500
- 502
- 503
- 504

These branches may include:

Switch
→ Wait
→ Retry HTTP Request2

---

## Non-Retryable Status Codes

The following errors must NEVER be routed back into the retry loop:

- 400
- 401
- 403
- 404
- 409
- 413
- 422

These branches must terminate gracefully after logging or handling the error.

Never create a retry loop for deterministic client errors.

---

## Loop Safety

Every retry loop MUST include:

- Retry counter
- Maximum retry limit
- Exit condition

Infinite retry loops are strictly prohibited.

---

## Preservation Rules

When implementing this refactor:

- Preserve the HTTP Request node.
- Preserve the Success branch.
- Preserve the Wait node unless explicitly instructed otherwise.
- Preserve all existing expressions.
- Preserve node IDs where possible.
- Preserve existing connections outside the Error branch.
- Modify only the Error routing logic.

This is an architectural refactor, not a workflow redesign.
````

This wording makes it clear that the **architecture** is fixed, only the **error classification mechanism** (IF → Switch) is allowed to change, and the rest of the workflow—including the success path, retry loop, and HTTP Request node—must remain intact. It also aligns the retry behavior with the HTTP status handling rules from your reference document.
Yes. For your master prompt, it should be much more explicit than just describing the workflow. It should become a **non-negotiable architectural contract** that Claude Code must preserve.

You can use the following section in your master prompt.

````md
# HTTP REQUEST ERROR HANDLING ARCHITECTURE (MANDATORY)

This workflow contains a production-grade HTTP Request retry architecture.

Before modifying anything, you MUST understand the existing execution flow.

```
                  ┌──────────────► Success Path
                  │               (Extract AI Response to JSON)
                  │
HTTP Request2 ────┤
(Gemini API)      │
                  │
                  └──────────────► Error Output
                                   │
                                   ▼
                              Status Router
                          (Current: If Node)
                          (Target: Switch Node)
                                   │
        ┌──────────────┬──────────────┬──────────────┐
        │              │              │              │
      Retryable    Non-Retryable    Logging      Default
        │
        ▼
      Wait
        │
        ▼
Retry HTTP Request2
```

## Existing Behaviour

The HTTP Request node is configured with:

- Retry On Fail = Enabled
- Maximum Retries = 3
- Wait Between Retries = 3000 ms
- On Error = Continue (using Error Output)

This means the Error Output is reached ONLY after the built-in HTTP retries have been exhausted.

Do NOT duplicate or replace this internal retry mechanism.

Instead, preserve it exactly as implemented.

---

## Architectural Contract

The Success path and the Error path are independent.

### Success Path

```
HTTP Request
      │
      ▼
Extract AI Response to JSON
```

This path is production-tested.

Do NOT modify it unless explicitly instructed.

---

### Error Path

Only the Error Output may be refactored.

The existing IF node is an implementation detail—not the architecture.

When requested, replace ONLY the IF node with a Switch node while preserving the overall execution flow.

The resulting architecture must remain:

```
HTTP Request Error Output
            │
            ▼
      Switch Node
            │
   Status-based Branches
            │
     Wait (retry only)
            │
            ▼
    HTTP Request (same node)
```

Never redesign the retry loop.

Never reconnect the Success branch.

Never change upstream or downstream workflow logic.

---

## Retryable Status Codes

Only these status codes may return to the HTTP Request node:

- 408
- 429
- 500
- 502
- 503
- 504

These branches may include:

Switch
→ Wait
→ Retry HTTP Request2

---

## Non-Retryable Status Codes

The following errors must NEVER be routed back into the retry loop:

- 400
- 401
- 403
- 404
- 409
- 413
- 422

These branches must terminate gracefully after logging or handling the error.

Never create a retry loop for deterministic client errors.

---

## Loop Safety

Every retry loop MUST include:

- Retry counter
- Maximum retry limit
- Exit condition

Infinite retry loops are strictly prohibited.

---

## Preservation Rules

When implementing this refactor:

- Preserve the HTTP Request node.
- Preserve the Success branch.
- Preserve the Wait node unless explicitly instructed otherwise.
- Preserve all existing expressions.
- Preserve node IDs where possible.
- Preserve existing connections outside the Error branch.
- Modify only the Error routing logic.

This is an architectural refactor, not a workflow redesign.
````

This wording makes it clear that the **architecture** is fixed, only the **error classification mechanism** (IF → Switch) is allowed to change, and the rest of the workflow—including the success path, retry loop, and HTTP Request node—must remain intact. It also aligns the retry behavior with the HTTP status handling rules from your reference document.
