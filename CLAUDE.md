# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

`workflow_data/Telegram Bot.json` holds the first scaffolded workflow (see below). There is still no package manifest or test suite, so there are no build/lint/test commands to run — the only "build" step is editing/validating/deploying this JSON via the n8n-workflow-builder MCP tools.

## Purpose

Per `docs/Telegram and n8n getting started.md`, the goal is to connect a Telegram bot to an n8n workflow instance:

- A Telegram bot is created via `@BotFather`, which issues an HTTP API token.
- That token is added as a **Telegram API** credential in n8n, consumed by a **Telegram Trigger** node.
- n8n must be reachable via a public HTTPS URL (`WEBHOOK_URL` env var) for Telegram to deliver webhook updates — for local dev this typically means an ngrok tunnel or similar.
- Incoming messages carry `message.text`, `chat.id`, and `from.id`; workflows commonly gate on `chat.id` (via an IF node) to restrict the bot to a specific user before continuing into downstream automation.

## Current workflow: `Telegram Bot`

Scaffolded via the `n8n-workflow-builder` MCP skill (dev server) and pushed to the local dev n8n instance (`http://localhost:5678`, live workflow id `LGDFRDKFfvyTkhPN`). The on-disk copy at `workflow_data/Telegram Bot.json` is the source of truth for further edits via the MCP tools; re-push with `update_live_workflow` after changing it.

Nodes:

1. **Telegram Trigger** (`n8n-nodes-base.telegramTrigger`) — fires on `message` updates. Needs a **Telegram API** credential (BotFather token) added manually in the n8n UI; the MCP tools can't set credentials.
2. **Chat ID Gate** (`n8n-nodes-base.if`) — compares `{{ $json.message.chat.id }}` against a placeholder `YOUR_TELEGRAM_CHAT_ID`. Replace the placeholder with your real chat ID (send the bot a message once the trigger is active, read `message.chat.id` off the execution) before relying on this gate.
3. **Downstream Placeholder** (`n8n-nodes-base.noOp`) — stub wired to the IF node's true branch; replace with real automation.

Known quirk: the `n8n-workflow-builder-dev` MCP container's local schema validator currently reports `unknown_node_type` for every node (its static catalog is empty — `get_n8n_version_info` shows `supportedNodesCount: 0`), even for core nodes like `if`/`noOp`. This is cosmetic; the live n8n instance accepts the workflow without issue, so treat `validate_workflow` output as unreliable until that catalog is repopulated.

When more code is added to this repo (custom nodes, supporting services, etc.), update this file with the actual build/lint/test commands and architecture.
