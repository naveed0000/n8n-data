## n8n MCP Coordination

You have access to a connected n8n MCP server. Treat it as the single source of truth.

### Rules

- Always inspect the workflow through MCP before answering.
- Never ask for exported workflow JSON if it is available through MCP.
- Never guess, infer, or fabricate workflow details.
- If information cannot be retrieved, state **"Not Present in Workflow"**.
- If the workflow has changed, refresh it from MCP before continuing.
- Query MCP again whenever additional information is required.

### Inspect

Read the complete workflow, including:

- Metadata
- Nodes
- Connections
- Parameters
- Expressions
- Variables
- Credentials (never expose secrets)
- Code nodes
- HTTP Request nodes
- AI nodes
- Database nodes
- Triggers
- IF / Switch / Merge / Loop nodes
- Execute Workflow nodes
- Error workflows
- Sub-workflows
- Environment variable references
- Static data

### Documentation Requirements

For every node explain:

- Purpose
- Configuration
- Inputs & Outputs
- Parameters & Expressions
- Execution flow
- Dependencies
- Error handling
- Performance considerations
- Security considerations
- Best practices
- Common pitfalls

Do not skip disabled nodes, default settings, or optional parameters.

If multiple workflows exist, list them first and ask which workflow to document.

The final documentation must be accurate enough for another engineer to reproduce, maintain, and troubleshoot the workflow using only the documentation.