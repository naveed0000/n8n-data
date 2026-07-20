# Master Prompt — Replace Gemini HTTP Request with AI Agent + PostgreSQL Vector Store (RAG)

## Objective

Refactor the existing n8n workflow by **removing the `Generate Questions (Gemini)` HTTP Request node** and replacing it with an **AI Agent** powered by Gemini and a **PostgreSQL Vector Store**.

The goal is **not simply to change the node**, but to redesign the generation pipeline into a Retrieval-Augmented Generation (RAG) architecture that prevents duplicate questions, improves semantic awareness, reduces token consumption, and enables long-term scalability.

---

# Architecture

Replace

```
Build Gemini Prompt
        │
        ▼
Generate Questions (Gemini)
        │
        ▼
Parse Response
```

with

```
Build Prompt
        │
        ▼
AI Agent
    │
    ├── Chat Model (Gemini)
    ├── Embedding Model
    └── PostgreSQL Vector Store
            │
            ▼
Semantic Similarity Search
            │
            ▼
Retrieved Existing Questions
            │
            ▼
Generate New Unique Questions
            │
            ▼
Existing Validation Pipeline
```

The downstream workflow (validation, MathML conversion, API storage, retry logic, etc.) must remain unchanged.

---

# Design Principles

The AI Agent must never generate questions in isolation.

Before generating a batch, it must retrieve semantically similar questions from PostgreSQL Vector Store and use them as generation context.

The objective is to avoid:

* duplicate questions
* paraphrased duplicates
* repeated concepts
* repeated numerical values
* repeated solution approaches
* repeated distractors

The agent should always maximize conceptual diversity.

---

# Vector Store Responsibilities

Use PostgreSQL with pgvector as the single vector database.

The vector store should contain one vector per question.

Each document should include:

* Question
* Explanation
* Hint
* Chapter
* Topic
* Subtopic
* Difficulty
* Question Type
* Metadata
* Question ID

Do **not** store entire batches as one document.

Each question must be indexed independently.

---

# Embedding Pipeline

Create a dedicated ingestion workflow.

```
Question Database
        │
        ▼
Split Out
        │
        ▼
One Question
        │
        ▼
Embedding Model
        │
        ▼
Postgres Vector Store
```

Responsibilities:

* split every question individually
* generate embeddings
* insert into Vector Store
* update existing vectors
* prevent duplicate records
* support incremental indexing

---

# AI Agent Responsibilities

The AI Agent must perform the following sequence for every generation request:

1. Receive generation request.
2. Convert request into embeddings.
3. Perform semantic similarity search.
4. Retrieve the most relevant existing questions.
5. Inject retrieved context into the LLM.
6. Generate only new, unique questions.
7. Return JSON compatible with the existing workflow.

The AI Agent should not bypass retrieval.

Retrieval is mandatory before every generation.

---

# Retrieval Strategy

Retrieve only the most relevant questions.

Recommended:

* Top K = 3–5
* Semantic similarity search only
* Metadata filtering before similarity search

Possible filters:

* Subject
* Category
* Chapter
* Topic
* Subtopic
* Difficulty
* Question Type

Do not search the entire database unnecessarily.

---

# Duplicate Prevention

The retrieved context represents previously generated knowledge.

The AI Agent must ensure:

* no exact duplicates
* no semantic duplicates
* no paraphrased duplicates
* no repeated formulas unless intentionally required
* no repeated numerical scenarios
* no repeated conceptual testing

If similarity is high, generate an entirely different question instead.

---

# Token Optimization

Never send thousands of previous questions to the LLM.

Instead:

1. Perform vector search.
2. Retrieve only the most relevant questions.
3. Inject only those results into the prompt.

This keeps prompt size small while preserving high-quality context.

---

# Existing Workflow Compatibility

The following workflow sections must remain unchanged:

* Prompt Builder
* Response Parser
* Question Validation
* LaTeX Validation
* MathML Conversion
* API Storage
* Retry Logic
* Batch Processing
* Loop Logic
* State Management
* Error Handling

Only replace the question-generation component.

---

# PostgreSQL Requirements

Use PostgreSQL as the unified backend for:

* Vector Store
* Question Database
* Chat Memory (future)
* Metadata
* AI Memory

Do not introduce additional vector databases unless absolutely necessary.

---

# Implementation Guidelines

* Prefer native n8n AI nodes.
* Use PostgreSQL Vector Store node.
* Use an Embedding Model compatible with the selected Gemini model.
* Keep the implementation modular.
* Separate ingestion workflow from generation workflow.
* Ensure the Vector Store can be rebuilt without affecting production.
* Make retrieval configurable (Top-K, similarity threshold, metadata filters).
* Support future expansion to millions of indexed questions.

---

# Success Criteria

The implementation is considered successful when:

* `Generate Questions (Gemini)` is completely replaced by an AI Agent.
* Every generation request performs vector retrieval before generation.
* Duplicate and semantically similar questions are significantly reduced.
* Existing downstream nodes continue working without modification.
* Token usage decreases by retrieving only relevant context.
* The workflow remains scalable, maintainable, and production-ready.
