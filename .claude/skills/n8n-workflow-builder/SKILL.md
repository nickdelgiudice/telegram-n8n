---
name: n8n-workflow-builder
description: Build, edit, validate, and deploy n8n workflows programmatically via the n8n-workflow-builder MCP server (dev/prod), instead of hand-editing workflow JSON. Use whenever the task involves creating an n8n workflow, adding/wiring nodes, building an AI/LangChain agent workflow, validating workflow JSON, or pushing a workflow to a live n8n instance.
user-invocable: false
---

# n8n Workflow Builder (MCP) Skill

Full reference: `docs/n8n-workflow-builder-mcp-guide.md` in this repo. This skill is the condensed, decision-oriented version — read the doc if a tool's exact parameters aren't covered here.

## Before doing anything

The MCP tools live on two Docker-backed servers: `n8n-workflow-builder-dev` (registered locally for this project, targets the dev n8n instance) and `n8n-workflow-builder-prod` (not yet registered here — ask before adding it). Both run as persistent containers (`docker exec -i n8n-workflow-builder-<env> node dist/index.js`); see `/home/nicol/dev/n8n-workflow-builder-mcp/docs/docker-local-testing.md` for the container setup if a tool call fails with a connection error (check `docker ps` for the container first).

Their schemas are deferred — they will not appear as callable tools until fetched. Run `ToolSearch` (e.g. query `"n8n workflow"` or `"select:<tool_name>"`) before attempting to call any tool named below. A server added mid-session won't show up in ToolSearch until the session restarts — if it's missing, check `claude mcp list` before assuming it's broken.

Default to the **dev** server unless the user explicitly asks to touch the live/prod n8n instance — prod is not configured for this project yet, so using it means adding it first (with confirmation).

## Decision guide — which pattern to use

- **Building an AI/LangChain workflow (agent + model + memory + tools + vector store) from scratch** → Pattern A: `compose_ai_workflow` in one call, then `add_ai_connections` or `connect_main_chain` for any follow-up wiring.
- **Non-trivial edit to an existing workflow (multi-node refactor)** → Pattern B: `plan_workflow` → `review_workflow_plan` → `apply_workflow_plan`. Don't skip straight to writing; review the plan's warnings/errors first.
- **Small, precise change (one node, one connection)** → Pattern C: `add_node` / `edit_node` / `delete_node` / `add_connection` directly. `add_node`/`edit_node` accept `connect_from`/`connect_to` to wire in the same call — use that instead of a separate `add_connection` call when possible.
- **Not sure the workflow is well-formed** → Pattern D: `validate_workflow`. Always run this after any non-trivial edit, before deploying.
- **Pushing to a live n8n instance** → Pattern E, and only if `N8N_API_URL` / `N8N_API_KEY` are configured for that environment. `list_live_workflows` / `get_live_workflow` to inspect, `create_live_workflow` / `update_live_workflow` to write. `update_live_workflow` is a **full replacement** — send the complete workflow object; the tool strips read-only fields (`id`, `versionId`, timestamps, `active` on n8n 2.x) automatically, so don't hand-strip them yourself.

## Node discovery — never guess a node type string

1. `list_available_nodes` with `search_term` (try `"langchain"`, `"agent"`, `"tool"`, `"memory"`, `"lmChat"`, `"embeddings"` for AI nodes; provider names like `"openai"`, `"anthropic"`, `"qdrant"` otherwise). `token_logic: "and"` narrows a multi-word search.
2. `suggest_node_params` / `list_missing_parameters` to get minimal valid params for that node.
3. `fix_node_params` to auto-fill defaults for anything still missing.
4. `list_template_examples` (filter by `node_type`) if you want to see real usage before wiring params by hand.
5. `query_n8n_documentation` when still uncertain — it hits the official docs search index, so URLs it returns are safe to cite.

## Workflow file location

Most edit tools take `workflow_name` (required) plus an optional `workflow_path` override. If this repo has a known on-disk location for the workflow, pass `workflow_path` explicitly rather than relying on the default `workflow_data` convention.

## Rules of thumb

- Plan → review → apply for anything touching more than one or two nodes; direct edits only for small, obviously-safe changes.
- Validate before deploy, every time — not just on request.
- Live `update_live_workflow` replaces the whole workflow object; don't assume partial/patch semantics.
- Use the discovery tools (`list_available_nodes`, `suggest_node_params`, `query_n8n_documentation`) instead of guessing node type strings or param shapes — n8n node schemas are too easy to get subtly wrong by hand.
