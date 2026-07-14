# n8n Workflow Builder (MCP) — Agent Reference Guide

This workspace supports an **MCP capability** that lets agents **create, edit, validate, and deploy n8n workflows** programmatically—without manually editing workflow JSON.

## What this capability is
- **Two MCP servers exist** (same interface, different environments):
  - **Dev**: `user-n8n-workflow-builder-dev`
  - **Prod**: `user-n8n-workflow-builder-prod`
- It supports both:
  - **Local/workspace workflows on disk** (create/edit nodes + connections, validate files).
  - **Live n8n instance workflows via API** (list/get/create/update live workflows).

## When an agent should use it
Use this MCP when you want to:
- Scaffold or refactor an n8n workflow quickly and safely.
- Add nodes and wire connections deterministically (idempotent-ish patterns available).
- Build AI/LangChain workflows (agent/model/memory/vector store/tools) with standardized wiring.
- Validate workflow JSON against known node schemas.
- Push a tested workflow to a live n8n instance via API.

## Core workflows (recommended patterns)

### Pattern A — Compose an AI workflow in one call (fastest for AI agent workflows)
Use `compose_ai_workflow` when you want a single operation to create/update an AI workflow layout (agent + model + memory + embeddings + vector store + tools + trigger) with basic wiring.

- **Tool**: `compose_ai_workflow`
- **Inputs**: `workflow_name`, optional `n8n_version`, and a high-level `plan` object describing which AI components to include.

Follow-up options:
- Use `add_ai_connections` if you need to wire or re-wire AI “side ports” (model/tools/memory/etc.) to an existing agent node.
- Use `connect_main_chain` if you want a standard minimal “main chain” wiring.

### Pattern B — Safer incremental editing with plan → review → apply
Use this when you’re doing non-trivial edits and want validation before writing.

- **Tool**: `plan_workflow` (non-destructive plan creation)
- **Tool**: `review_workflow_plan` (apply plan in-memory, return warnings/errors/fixes)
- **Tool**: `apply_workflow_plan` (atomic write to disk)

### Pattern C — Direct node + connection editing (surgical edits)
Use these for precise changes:
- `add_node`, `edit_node`, `delete_node`
- `add_connection`

Tip: `add_node` and `edit_node` support optional `connect_from` / `connect_to` to wire edges as part of the same operation.

### Pattern D — Validate
- **Tool**: `validate_workflow`
- Validates a workflow file against known node schemas.

### Pattern E — Deploy to live n8n via API (create/update)
These require the environment to be configured with:
- **`N8N_API_URL`**
- **`N8N_API_KEY`**

Tools:
- `list_live_workflows` (supports pagination/filtering)
- `get_live_workflow`
- `create_live_workflow`
- `update_live_workflow`

Important behavior (built-in safety):
- Live create/update **automatically strips read-only fields** (e.g., `id`, `versionId`, timestamps, metadata).
- On n8n 2.x, `active` is treated as read-only and stripped.
- Workflow `settings` keys are filtered to an API allowlist during `update_live_workflow`.

## Node discovery, parameters, and examples

### Find the right node type
- **Tool**: `list_available_nodes`
- Supports:
  - `search_term` (multi-token queries)
  - `token_logic`: `"or"` (default) or `"and"`
  - `tags`: synonym expansion (default true)
  - `n8n_version`: filter compatibility
  - pagination via `cursor`

Suggested searches:
- **AI/LangChain**: `"langchain"`, `"agent"`, `"tool"`, `"memory"`, `"lmChat"`, `"embeddings"`
- **Providers**: `"openai"`, `"anthropic"`, `"qdrant"`, `"weaviate"`, `"milvus"`

### Get minimal valid params for a node type
- **Tool**: `suggest_node_params`
- **Tool**: `list_missing_parameters` (considers visibility rules)
- **Tool**: `fix_node_params` (apply defaults for missing required fields)

### Learn by example from templates
- **Tool**: `list_template_examples`
- Returns node usage snippets extracted from templates; filter by `node_type` or `template_name`.

### Official documentation lookup (when uncertain)
- **Tool**: `query_n8n_documentation`
- Uses official docs search index so returned URLs match the live docs site.

## Workspace workflow storage model (on-disk)
- **Tool**: `create_workflow` writes a new workflow into a workspace location using a standard `workflow_data` approach.
- Most edit tools accept:
  - `workflow_name` (required)
  - optional `workflow_path` (direct path override; absolute or relative)

Practical rule for agents:
- If the repo has a known workflow path, pass `workflow_path`.
- Otherwise, rely on `workflow_name` + default workspace `workflow_data` conventions.

## Tool catalog (quick reference)

### Build / edit (disk)
- **`create_workflow(workflow_name, workspace_dir)`**: create new workflow structure on disk.
- **`get_workflow_details(workflow_name, workflow_path?)`**: inspect by name/path.
- **`add_node(...)`**: add node (supports auto casing of types; supports `connect_from`/`connect_to`).
- **`edit_node(...)`**: edit node fields (type/name/position/params; supports reconnections).
- **`delete_node(workflow_name, node_id, workflow_path?)`**
- **`add_connection(workflow_name, source_node_id, source_node_output_name, target_node_id, target_node_input_name, target_node_input_index?)`**
- **`validate_workflow(workflow_name, workflow_path?)`**

### Plan-based editing
- **`plan_workflow(workflow_name, workspace_dir?, intent?, target?)`**
- **`review_workflow_plan(workflow_name, plan, workflow_path?)`**
- **`apply_workflow_plan(workflow_name, plan, workflow_path?)`**

### AI workflow helpers
- **`compose_ai_workflow(workflow_name, n8n_version?, plan)`**
- **`add_ai_connections(workflow_name, agent_node_id, model_node_id?, tool_node_ids?, memory_node_id?, embeddings_node_id?, vector_store_node_id?, vector_insert_node_id?, vector_tool_node_id?, workflow_path?)`**
- **`connect_main_chain(workflow_name, workflow_path?, dry_run?, idempotency_key?)`**

### Live n8n API
- **`get_n8n_version_info()`**
- **`list_live_workflows(limit?, cursor?, active?)`**
- **`get_live_workflow(workflow_id)`**
- **`create_live_workflow(workflow)`**
- **`update_live_workflow(workflow_id, workflow)`**

## Suggested “agent instructions” snippet (copy/paste)

```text
This repo has access to an MCP server that can build and modify n8n workflows programmatically.

Use the n8n workflow builder MCP tools to:
- create/edit workflows on disk (create_workflow, add_node, edit_node, add_connection, validate_workflow)
- compose AI workflows quickly (compose_ai_workflow, add_ai_connections, connect_main_chain)
- plan/review/apply changes safely (plan_workflow → review_workflow_plan → apply_workflow_plan)
- interact with live n8n via API when configured (list_live_workflows, get_live_workflow, create_live_workflow, update_live_workflow)

When unsure about node types or parameters:
- list_available_nodes (search)
- suggest_node_params / list_missing_parameters / fix_node_params
- query_n8n_documentation for official docs lookup
```

## Common gotchas / rules of thumb
- **Prefer plan → review → apply** for large refactors; use direct node edits for small changes.
- **Always validate** (`validate_workflow`) after significant edits before deploying.
- **Live updates are full replacements**: `update_live_workflow` expects the complete workflow object (the tool strips read-only fields automatically).
- **Use `workflow_path`** when the workflow file isn’t in the default `workflow_data` layout.
- **Use node discovery helpers** instead of guessing node type strings.

