# Task

Analyze the current n8n workflow using the n8n MCP server.

## Objective

1. Read the complete workflow through the n8n MCP.
2. Understand the purpose of every node.
3. Rename each node based on what it actually does.
4. Keep the names short, clear, and descriptive.
5. After understanding the workflow, generate a `README.md`.

## Node Naming Rules

* Use action-based names.
* Avoid generic names like `Code`, `Code1`, `HTTP Request`, `Edit Fields`, `Merge2`, `If1`.
* Name nodes according to their actual responsibility.

### Examples

* Login API
* Generate Questions
* Parse LLM Response
* Extract LaTeX
* Convert LaTeX to MathML
* Validate Questions
* Remove Duplicate Questions
* Store in PostgreSQL
* Merge Generated Results
* Retry Failed Request
* Split Batch Items
* Check HTTP Status
* Format Final Output

## README Structure

The README should include:

1. Project Overview
2. Workflow Purpose
3. High-Level Architecture
4. Workflow Flow (step-by-step)
5. Node Responsibilities
6. External Services Used
7. Input Data
8. Output Data
9. Error Handling
10. Environment Variables
11. Important Notes
12. Future Improvements

## Requirements

* Do not change workflow logic.
* Only rename node names.
* Keep all node IDs and connections unchanged.
* The README should accurately reflect the current workflow after analyzing every node.
* Explain each major workflow section in simple language.
* Use Markdown formatting with headings, tables, and bullet points where appropriate.
* If a node's purpose is unclear, infer it from its connections, expressions, and configuration before naming it.
