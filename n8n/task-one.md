Yes. I would go one step further.

Don't append it as "another section."

Integrate it as a **new mandatory architecture rule** so Claude Code treats duplicate prevention as part of the workflow architecture rather than an optional enhancement.

The new section should come **after AI NODE STANDARDS** and **before DATABASE STANDARDS**, because it sits between LLM generation and persistence.

It should read something like this:

---

# ============================================================================

# AI QUESTION GENERATION ARCHITECTURE (MANDATORY)

# ============================================================================

This project generates JEE Main Physics questions using Gemini.

The primary engineering objective is **to guarantee that every stored question is unique** while minimizing API usage, token consumption, execution time, and infrastructure cost.

Duplicate prevention is a mandatory part of the architecture.

It is NOT optional.

Never generate questions and immediately store them.

Every generated question MUST pass through a uniqueness verification pipeline before entering the validation, MathML conversion, or storage stages.

---

## REQUIRED WORKFLOW ARCHITECTURE

The workflow architecture MUST remain incremental.

Never redesign the existing production workflow unless explicitly instructed.

Always extend the existing workflow.

The required architecture is:

```text
Loop Over Batches
        │
        ▼
Build Gemini Prompt
        │
        ▼
Generate Questions (Gemini)
        │
        ▼
Parse Gemini Response
        │
        ▼
Normalize Questions
        │
        ▼
Generate Embeddings
        │
        ▼
PostgreSQL pgvector Similarity Search
        │
        ▼
Similarity Decision
        │
 ┌──────┴────────┐
 │               │
Unique        Duplicate
 │               │
 ▼               ▼
Validate     Retry Generation
 │               │
 ▼               │
Latex          Gemini
 │               │
 ▼               │
MathML───────────┘
 │
 ▼
Store Questions
 │
 ▼
Store Embeddings
```

This architecture is mandatory.

Do not bypass any stage.

---

## NEVER MODIFY THE EXISTING PIPELINE

The existing production workflow already contains:

* Loop Over Batches
* Build Gemini Prompt
* Generate Questions
* Parse Gemini Response
* Validation
* LaTeX Processing
* MathML Conversion
* Database Storage

Do NOT replace these nodes.

Do NOT redesign this pipeline.

Instead, insert new functionality before the validation stage.

Only extend the workflow.

Preserve the current architecture whenever possible.

---

## INSERTION POINT

The duplicate detection pipeline MUST be inserted immediately after

```text
Parse Gemini Response
```

before

```text
Validate & Classify Questions
```

Never perform LaTeX conversion, MathML conversion, or storage until uniqueness has been verified.

---

# QUESTION NORMALIZATION

Before embeddings are generated, every question MUST be normalized.

Never generate embeddings directly from raw HTML.

Normalization MUST remove

• HTML

• MathML

• LaTeX placeholders

• Multiple spaces

• Punctuation

• Formatting

Example

Input

```html
<p>A particle moves with velocity {{latex[0]}}</p>
```

Output

```text
particle moves with velocity
```

Store the normalized text separately.

Never overwrite the original question.

---

# EMBEDDING STRATEGY

Embeddings MUST be generated from

```text
normalizedQuestion
```

NOT

```text
question
```

This significantly improves semantic similarity accuracy.

Never embed HTML.

Never embed MathML.

Never embed LaTeX.

Never embed formatting.

---

# VECTOR SEARCH

Every generated question MUST be searched against PostgreSQL pgvector before validation.

The similarity search MUST retrieve the nearest existing questions.

Retrieve at least the Top 10 nearest neighbours.

The search result should include

• questionId

• similarity score

• concept

• formula

• normalizedQuestion

Never perform duplicate detection using string comparison alone.

Always use vector similarity.

---

# DUPLICATE DECISION

Determine whether the generated question is acceptable using similarity thresholds.

Recommended thresholds

Similarity ≥ 0.95

Reject immediately.

Similarity between 0.90 and 0.95

Treat as probable duplicate.

Regenerate.

Similarity between 0.80 and 0.90

Review concept overlap.

Use engineering judgement.

Similarity below 0.80

Accept.

Thresholds should be configurable.

Never hardcode them inside JavaScript.

---

# DUPLICATE REGENERATION

If a duplicate is detected

DO NOT discard the entire batch.

Instead

Regenerate only the rejected questions.

Never regenerate questions that already passed similarity validation.

This minimizes API cost.

---

# BATCH COMPLETION STRATEGY

The workflow currently generates multiple questions in a single Gemini request.

If

6 questions requested

and

4 are unique

2 are duplicates

Never regenerate all six.

Instead

Keep the four accepted questions.

Generate only the missing two.

Continue until

Accepted Questions

equals

Requested Questions.

This significantly reduces

API calls

LLM cost

Execution time

Token usage

---

# RETRY STRATEGY

Maintain a retry counter for every rejected question.

Retry only the rejected questions.

Maximum retries must be configurable.

Recommended maximum

5

After the maximum retry limit

Stop regeneration.

Return an engineering error explaining

Similarity remained above threshold after maximum retry attempts.

---

# RETRY PROMPT

When regenerating a duplicate

Never ask the LLM to "try again."

Provide the reason.

Example

Similarity Score

0.96

Existing Concept

Relative Velocity

Existing Formula

v = u + at

Instruction

Generate a completely different question.

Do NOT reuse

• same derivation

• same formula

• same physical scenario

• same numerical values

• same reasoning

• same conceptual approach

Always explain WHY regeneration is required.

---

# PROMPT ENHANCEMENT

The generation prompt MUST include recently generated concepts.

Never send hundreds of previous questions.

Instead send only summarized concepts.

Example

Previously Generated Concepts

Relative Velocity

Escape Velocity

Rolling Motion

Boat Crossing River

Projectile Maximum Height

SHM Energy

Do NOT generate questions using these concepts.

This dramatically reduces token usage.

---

# DATABASE DESIGN

Do not store embeddings inside the questions table.

Maintain a dedicated table

question_embeddings

Recommended fields

questionId

chapterId

topicId

difficulty

questionType

normalizedQuestion

concept

formula

embedding

createdAt

Maintain a foreign-key relationship with the questions table.

---

# STORAGE ORDER

Never store embeddings before the question exists.

Correct order

Store Question

↓

Generate Embedding

↓

Store Embedding

This guarantees referential integrity.

---

# REQUIRED NODE INSERTIONS

When extending the existing workflow

Insert only these additional stages

Normalize Question

↓

Generate Embedding

↓

Vector Search

↓

Similarity Decision

↓

Retry Branch

↓

Store Embedding

Do not modify downstream LaTeX processing unless explicitly requested.

---

# IMPLEMENTATION RULES

When implementing this architecture using Claude Code

You MUST use **n8n-mcp**.

Never manually edit workflow JSON without first reading the workflow through the MCP server.

Always inspect the existing workflow before making modifications.

Determine the safest insertion point.

Reuse existing nodes whenever possible.

Never recreate nodes that already exist.

Preserve node IDs, execution order, connections, retry logic, and error handling wherever possible.

After implementation, read the workflow again through **n8n-mcp** and verify:

* The new nodes are correctly connected.
* Existing execution order is preserved.
* Validation, LaTeX, MathML, and storage stages remain unchanged except for the new uniqueness gate.
* Duplicate questions are routed to the retry branch.
* Unique questions continue through the existing pipeline.
* Embeddings are stored only after successful question storage.

Only consider the implementation complete after this verification succeeds.

---

This version doesn't just describe the feature—it **defines it as a mandatory architectural constraint**. That makes Claude Code much more likely to preserve your existing workflow, insert the new nodes in the correct location, and use **n8n-mcp** to inspect and validate the live workflow before and after making changes.
