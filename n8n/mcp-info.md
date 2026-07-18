# Step-by-Step Architecture: Codex + n8n MCP

This document explains how Codex should work with an n8n instance through the n8n MCP server. It is written as an implementation guide for creating, inspecting, validating, testing, and maintaining n8n workflows from Codex.

Source used: n8n MCP tool documentation and a live `n8n_health_check` attempt on 2026-07-16.

Current health-check result:

```json
{
  "success": false,
  "error": "SSRF protection: Localhost access is blocked in strict mode",
  "code": "REQUEST_ERROR",
  "details": {
    "apiUrl": "http://localhost:5678",
    "hint": "Check if n8n is running and API is enabled"
  }
}
```

This means the MCP server is available, but the configured n8n API URL points to localhost and strict SSRF protection blocks that request. Use a reachable n8n URL or adjust the MCP/n8n deployment configuration before relying on API operations.

---

## 1. High-Level Architecture

```text
User
  |
  v
Codex
  |
  | MCP tool calls
  v
n8n MCP Server
  |
  | n8n Public API
  v
n8n Instance
  |
  +--> Workflows
  +--> Nodes
  +--> Credentials
  +--> Executions
  +--> Templates
  +--> Audit Data
```

The MCP server acts as the control layer between Codex and n8n. Codex does not edit the n8n database directly. It asks the MCP server to call the supported n8n API operations.

---

## 2. Main Components

| Component | Responsibility |
|---|---|
| User | Provides workflow goals, IDs, templates, credentials, and review decisions. |
| Codex | Reads requirements, designs workflow changes, calls MCP tools, and writes local documentation. |
| n8n MCP Server | Exposes safe tool wrappers around n8n workflow, credential, execution, audit, and template APIs. |
| n8n Public API | Receives authenticated workflow-management requests. |
| n8n Instance | Stores and runs workflows, credentials, executions, and settings. |
| Local Markdown Docs | Store implementation notes, architecture, prompts, and workflow documentation. |

---

## 3. Required Configuration

The n8n MCP API tools require these values to be configured in the MCP server environment:

```text
N8N_API_URL=https://your-n8n-host.example.com
N8N_API_KEY=your-n8n-api-key
```

Recommended configuration rules:

- Use an externally reachable n8n URL when strict SSRF protection blocks localhost.
- Confirm the n8n Public API is enabled.
- Use an API key with only the permissions required for the task.
- Keep credentials out of markdown files, workflow notes, and Code node source.
- Run `n8n_health_check` before performing workflow changes.

---

## 4. Standard Operating Flow

```text
Discover tools
  |
  v
Check n8n connectivity
  |
  v
Inspect existing workflow or template
  |
  v
Discover node schemas
  |
  v
Design workflow graph
  |
  v
Validate nodes and workflow
  |
  v
Create or update workflow
  |
  v
Configure credentials
  |
  v
Test execution
  |
  v
Review executions and errors
  |
  v
Document final architecture
```

---

## 5. Step 1: Discover MCP Capabilities

Start with the MCP documentation tool:

```json
{
  "tool": "tools_documentation",
  "arguments": {
    "topic": "overview",
    "depth": "full"
  }
}
```

Use this to confirm which tools are available in the current Codex session. The n8n MCP docs currently describe these major groups:

- System and health checks.
- Node discovery and node configuration.
- Node and workflow validation.
- Template search and template deployment.
- Workflow create, read, update, delete, and activation operations.
- Credential schema discovery and credential management.
- Workflow testing and execution inspection.
- Audit, version history, rollback, and cleanup operations.

---

## 6. Step 2: Check API Connectivity

Run a health check before making workflow changes:

```json
{
  "tool": "n8n_health_check",
  "arguments": {
    "mode": "status"
  }
}
```

Expected healthy result:

```json
{
  "success": true,
  "status": "ok"
}
```

Current observed result in this workspace:

```text
SSRF protection blocked access to http://localhost:5678.
```

Fix options:

- Change `N8N_API_URL` to a reachable host URL instead of `http://localhost:5678`.
- Confirm n8n is running.
- Confirm the Public API is enabled.
- Confirm the API key is valid.
- If this is an intentional local-only setup, adjust the MCP server security settings only if that is acceptable for the environment.

---

## 7. Step 3: Inspect Existing Workflows

List workflows when the API is reachable:

```json
{
  "tool": "n8n_list_workflows",
  "arguments": {
    "limit": 100,
    "excludePinnedData": true
  }
}
```

Inspect a workflow by ID:

```json
{
  "tool": "n8n_get_workflow",
  "arguments": {
    "id": "WORKFLOW_ID",
    "mode": "structure"
  }
}
```

Use modes carefully:

| Mode | Use Case |
|---|---|
| `minimal` | Read ID, name, active state, and tags only. |
| `structure` | Read nodes and connections without full heavy configuration. |
| `full` | Read the draft workflow body and metadata. |
| `details` | Read full workflow plus execution statistics. |
| `active` | Read the published graph that is actually running. |
| `filtered` | Read selected heavy nodes, such as Code nodes, without loading the entire workflow. |

Important: n8n has a draft/publish model. The draft can differ from the active workflow that is actually running.

---

## 8. Step 4: Discover Nodes Before Building

Search for a node:

```json
{
  "tool": "search_nodes",
  "arguments": {
    "query": "webhook"
  }
}
```

Then read the node schema with standard detail first:

```json
{
  "tool": "get_node",
  "arguments": {
    "nodeType": "nodes-base.webhook",
    "detail": "standard"
  }
}
```

Only request full node detail if the standard schema is insufficient. Full schemas can be large.

For Code nodes, read the relevant guide first:

```json
{
  "tool": "tools_documentation",
  "arguments": {
    "topic": "javascript_code_node_guide",
    "depth": "full"
  }
}
```

or:

```json
{
  "tool": "tools_documentation",
  "arguments": {
    "topic": "python_code_node_guide",
    "depth": "full"
  }
}
```

---

## 9. Step 5: Design the Workflow Graph

An n8n workflow graph has two primary parts:

```json
{
  "nodes": [],
  "connections": {}
}
```

Each node should include:

```json
{
  "id": "unique-node-id",
  "name": "Readable Node Name",
  "type": "nodes-base.httpRequest",
  "typeVersion": 1,
  "position": [600, 300],
  "parameters": {}
}
```

Connections are keyed by source node name:

```json
{
  "Webhook": {
    "main": [
      [
        {
          "node": "HTTP Request",
          "type": "main",
          "index": 0
        }
      ]
    ]
  }
}
```

Design rules:

- Use stable, readable node names.
- Keep node IDs unique.
- Put trigger nodes at the start of the graph.
- Keep connections explicit.
- Avoid hardcoded secrets in parameters, expressions, and Code nodes.
- Use credentials for authentication.
- Add error handling for external APIs.

---

## 10. Step 6: Validate Nodes

Validate each node configuration before creating or updating a workflow:

```json
{
  "tool": "validate_node",
  "arguments": {
    "nodeType": "nodes-base.httpRequest",
    "config": {
      "method": "GET",
      "url": "https://example.com/api"
    },
    "mode": "full",
    "profile": "ai-friendly"
  }
}
```

Use validation profiles this way:

| Profile | When To Use |
|---|---|
| `minimal` | Fast checks during early design. |
| `runtime` | Default runtime readiness checks. |
| `ai-friendly` | Helpful suggestions while Codex is designing nodes. |
| `strict` | Pre-deployment checks. |

---

## 11. Step 7: Validate the Full Workflow

Before deployment, validate the complete graph:

```json
{
  "tool": "validate_workflow",
  "arguments": {
    "workflow": {
      "nodes": [],
      "connections": {}
    },
    "options": {
      "profile": "strict",
      "validateNodes": true,
      "validateConnections": true,
      "validateExpressions": true
    }
  }
}
```

Validation should catch:

- Missing required node parameters.
- Broken node connections.
- Invalid expression syntax.
- References to missing nodes.
- Incorrect AI-tool connections.
- Common n8n type-version issues.

---

## 12. Step 8: Create a Workflow

Create workflows inactive by default:

```json
{
  "tool": "n8n_create_workflow",
  "arguments": {
    "name": "Example Workflow",
    "nodes": [],
    "connections": {},
    "settings": {
      "executionOrder": "v1",
      "saveDataSuccessExecution": "all",
      "saveDataErrorExecution": "all"
    }
  }
}
```

Recommended creation behavior:

- Create inactive.
- Validate immediately after creation.
- Configure credentials before activation.
- Test with representative input.
- Activate only after successful test execution.

---

## 13. Step 9: Update Existing Workflows

Use full updates only when replacing the entire workflow graph:

```json
{
  "tool": "n8n_update_full_workflow",
  "arguments": {
    "id": "WORKFLOW_ID",
    "nodes": [],
    "connections": {},
    "settings": {}
  }
}
```

Use partial updates when the MCP server exposes `n8n_update_partial_workflow` and the task is incremental. Incremental changes are safer for adding nodes, patching fields, moving nodes, rewiring connections, and activating or deactivating workflows.

Recommended update sequence:

```text
Get workflow structure
  |
  v
Get selected full nodes if needed
  |
  v
Apply focused change
  |
  v
Validate workflow
  |
  v
Test workflow
  |
  v
Check execution result
```

---

## 14. Step 10: Manage Credentials

Always discover the credential schema before creating credentials:

```json
{
  "tool": "n8n_manage_credentials",
  "arguments": {
    "action": "getSchema",
    "type": "httpHeaderAuth"
  }
}
```

Then create or update credentials using the required fields:

```json
{
  "tool": "n8n_manage_credentials",
  "arguments": {
    "action": "create",
    "name": "Example API Header",
    "type": "httpHeaderAuth",
    "data": {
      "name": "Authorization",
      "value": "Bearer REDACTED"
    }
  }
}
```

Security rules:

- Do not write live credential values into markdown.
- Do not store API keys in workflow JSON files.
- Prefer n8n credential objects over raw headers in node parameters.
- Rotate credentials when access changes.
- Use `includeUsage` only when you need to audit where credentials are referenced.

---

## 15. Step 11: Test Workflows

Test workflows with supported trigger types:

```json
{
  "tool": "n8n_test_workflow",
  "arguments": {
    "workflowId": "WORKFLOW_ID",
    "triggerType": "webhook",
    "httpMethod": "POST",
    "data": {
      "example": true
    },
    "waitForResponse": true,
    "timeout": 120000
  }
}
```

Supported external trigger styles:

- Webhook trigger.
- Form trigger.
- Chat trigger.

Testing checklist:

- Test a happy path.
- Test missing optional fields.
- Test missing required fields.
- Test malformed input.
- Test authentication failures.
- Test external API timeout behavior.
- Test rate-limit behavior if the workflow calls third-party APIs.

---

## 16. Step 12: Inspect Executions

List executions:

```json
{
  "tool": "n8n_executions",
  "arguments": {
    "action": "list",
    "workflowId": "WORKFLOW_ID",
    "limit": 20
  }
}
```

Inspect a failed execution:

```json
{
  "tool": "n8n_executions",
  "arguments": {
    "action": "get",
    "id": "EXECUTION_ID",
    "mode": "error",
    "includeExecutionPath": true,
    "includeStackTrace": false,
    "fetchWorkflow": true
  }
}
```

Use execution inspection to identify:

- The failed node.
- The path that led to failure.
- Upstream input items.
- Node output shape.
- Expression failures.
- API response errors.

---

## 17. Step 13: Audit Security

Run an instance audit when API access is working:

```json
{
  "tool": "n8n_audit_instance",
  "arguments": {
    "includeCustomScan": true,
    "customChecks": [
      "hardcoded_secrets",
      "unauthenticated_webhooks",
      "error_handling",
      "data_retention"
    ]
  }
}
```

Audit focus areas:

- Hardcoded secrets in workflow nodes.
- Unauthenticated public webhooks.
- Missing error handling.
- Excessive execution-data retention.
- Credentials with broad permissions.
- Abandoned workflows.

---

## 18. Step 14: Manage Workflow Versions

List workflow versions:

```json
{
  "tool": "n8n_workflow_versions",
  "arguments": {
    "mode": "list",
    "workflowId": "WORKFLOW_ID",
    "limit": 10
  }
}
```

Rollback when needed:

```json
{
  "tool": "n8n_workflow_versions",
  "arguments": {
    "mode": "rollback",
    "workflowId": "WORKFLOW_ID",
    "versionId": 123,
    "validateBefore": true
  }
}
```

Versioning rules:

- Validate before rollback.
- Keep a record of why a rollback happened.
- Test after rollback.
- Prune old versions only after confirming retention requirements.

---

## 19. Template-Based Workflow Deployment

Search for templates:

```json
{
  "tool": "search_templates",
  "arguments": {
    "searchMode": "keyword",
    "query": "slack alert",
    "limit": 10
  }
}
```

Deploy a template:

```json
{
  "tool": "n8n_deploy_template",
  "arguments": {
    "templateId": 1234,
    "name": "Slack Alert Workflow",
    "stripCredentials": true,
    "autoFix": true,
    "autoUpgradeVersions": true
  }
}
```

Template deployment rules:

- Strip credentials during deployment unless there is a strong reason not to.
- Review all nodes after deployment.
- Validate the deployed workflow.
- Configure credentials in the n8n UI or through credential-management tools.
- Test before activation.

---

## 20. Error Handling Architecture

Recommended workflow error path:

```text
Main Trigger
  |
  v
Validate Input
  |
  +--> Invalid Input Response
  |
  v
Main Processing
  |
  +--> External API Error
  |       |
  |       v
  |     Retry or Backoff
  |       |
  |       v
  |     Error Log
  |
  v
Success Response
```

Recommended error handling:

- Validate input as early as possible.
- Use explicit error branches for expected failures.
- Use retry settings for transient external API errors.
- Avoid infinite retries.
- Save failed execution data when debugging production issues.
- Use a central error workflow for production workflows.

---

## 21. Security Architecture

```text
Secrets
  |
  v
n8n Credentials Store
  |
  v
Workflow Node Credential Reference
  |
  v
Runtime API Call
```

Security rules:

- Never hardcode secrets in Markdown, Code nodes, Set nodes, or HTTP headers.
- Prefer credential references over raw token values.
- Use least-privilege API keys.
- Rotate credentials regularly.
- Disable or protect public webhooks.
- Avoid saving unnecessary sensitive execution data.
- Use audits to detect hardcoded secrets and unauthenticated webhooks.

---

## 22. Draft vs Active Workflow Model

n8n can store a draft workflow that differs from the published active workflow.

```text
Draft Workflow
  |
  | publish / activate
  v
Active Workflow
  |
  v
Runtime Executions
```

Operational rule:

- Use `mode: "full"` to inspect the draft.
- Use `mode: "active"` to inspect what is actually running.
- Compare draft and active versions before debugging production behavior.

---

## 23. Local Folder Structure

Current local documentation target:

```text
n8n-data/
  n8n/
    step-by-step-arhictecture.md
```

Recommended local documentation expansion:

```text
n8n-data/
  n8n/
    step-by-step-arhictecture.md
    workflows/
      WORKFLOW_ID.readme.md
      WORKFLOW_ID.execution-notes.md
    credentials/
      credential-inventory.md
    audits/
      YYYY-MM-DD-security-audit.md
```

Do not store live secrets in this folder.

---

## 24. Common Failure Cases

| Failure | Cause | Fix |
|---|---|---|
| SSRF protection blocks localhost | MCP server cannot call `http://localhost:5678` in strict mode | Use a reachable n8n host URL or adjust approved security settings. |
| `N8N_API_URL` missing | MCP API tools are not configured | Set the API URL in the MCP environment. |
| `N8N_API_KEY` invalid | API authentication fails | Create or rotate the n8n API key. |
| Workflow test cannot trigger | Workflow has no webhook, form, or chat trigger | Trigger manually in n8n or add a supported trigger. |
| Draft differs from active workflow | Debugging the wrong graph | Inspect both `full` and `active` modes. |
| Credential read unsupported | n8n API/version/settings restrict credential reads | Use schema/create/update operations or configure API permissions. |
| Expression validation fails | Bad syntax or missing node reference | Validate expressions and inspect referenced node names. |
| Node type version mismatch | Imported or old node version | Run autofix or upgrade node versions when supported. |

---

## 25. Production Checklist

Before activating a workflow:

- MCP health check passes.
- Workflow structure is inspected.
- Node schemas are checked with `get_node`.
- Nodes are validated.
- Full workflow validation passes.
- Credentials are configured through n8n credentials.
- No live secrets are stored in Markdown or workflow code.
- Webhooks are authenticated or intentionally public.
- Happy-path test passes.
- Failure-path test passes.
- Execution logs are reviewed.
- Error handling is documented.
- Rollback/version plan is available.

---

## 26. Quick Reference

| Task | MCP Tool |
|---|---|
| Read MCP docs | `tools_documentation` |
| Check n8n connectivity | `n8n_health_check` |
| Search nodes | `search_nodes` |
| Read node schema | `get_node` |
| Validate one node | `validate_node` |
| Validate workflow JSON | `validate_workflow` |
| List workflows | `n8n_list_workflows` |
| Get workflow | `n8n_get_workflow` |
| Create workflow | `n8n_create_workflow` |
| Update workflow | `n8n_update_full_workflow` or `n8n_update_partial_workflow` when available |
| Test workflow | `n8n_test_workflow` |
| Inspect executions | `n8n_executions` |
| Manage credentials | `n8n_manage_credentials` |
| Search templates | `search_templates` |
| Deploy template | `n8n_deploy_template` |
| Audit instance | `n8n_audit_instance` |
| Manage versions | `n8n_workflow_versions` |

---

## 27. Next Action

The next required infrastructure step is to fix MCP-to-n8n connectivity. The current configured API URL is:

```text
http://localhost:5678
```

That URL is blocked by strict SSRF protection in the current MCP environment. After the API URL is changed to a reachable host and `n8n_health_check` succeeds, Codex can safely use the n8n MCP tools to inspect, create, validate, test, audit, and document real workflows.
