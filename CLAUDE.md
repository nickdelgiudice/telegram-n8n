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

## Reusable notifier: `Send Telegram Message`

A small sub-workflow (`workflow_data/Send Telegram Message.json`, live id `1IrnXdUpfHhhT5R8`, **active** — sub-workflows must be published for other live workflows to reference them via Execute Workflow) meant to be called from any other workflow that wants to notify this Telegram chat:

1. **When Called by Another Workflow** (`n8n-nodes-base.executeWorkflowTrigger`, `inputSource: passthrough`) — receives whatever JSON the caller sends, e.g. `{ "message": "..." }`.
2. **Send Message** (`n8n-nodes-base.telegram`, `resource: message`, `operation: sendMessage`) — sends `{{ $json.message }}` to the hardcoded chat ID `1449057581` using the same Telegram API credential as `Telegram Bot`.

Call it from another workflow with an **Execute Workflow** node (`source: database`, `workflowId.value: 1IrnXdUpfHhhT5R8`), passing `workflowInputs.value.message` as the text to send.

## Cross-project integration: `Pocket - Inbound` → Telegram notify

`Pocket - Inbound` (live id `2UI2FJmTqSxx3MpH`) is a pre-existing production-ish workflow that isn't otherwise part of this repo — it verifies an HMAC-signed webhook from Pocket, dedups by `recording_id`+delivery key, fetches the completed recording/transcript, and normalizes the payload. A local copy lives at `workflow_data/Pocket - Inbound.json` **for reference only** — it omits the live `staticData` (the dedup `seenEvents` map), which is runtime state, not workflow definition, and shouldn't be tracked in git or treated as a source of truth; the live n8n instance is authoritative for that workflow.

**`Pocket - Handoff to Assistant` is intentionally detached** (disconnected from both its input and output) as of 2026-07-14 — that's a deliberate change made directly in the n8n UI, not a bug to fix.

Pocket sends **6 separate lifecycle webhooks per recording** (`recording.created`, `transcription.completed`, `speakers.labeled`, `mind_map.completed`, `summary.updated`, `summary.completed`) — each a legitimate, non-duplicate delivery, not a retry. The dedup check only catches redelivery of the *same* event, not these distinct lifecycle events, so anything wired downstream of `Normalize payload` without further gating fires once per event, not once per recording.

Chain: `Normalize payload` → **Pocket - Is Final Event?** (`n8n-nodes-base.if`, gates on `{{ $json.event }} === 'summary.completed'`, the last event in the observed sequence) → true branch only → **Pocket - Notify Telegram** (`n8n-nodes-base.executeWorkflow`, calls `Send Telegram Message` with `message: "🎙️ Pocket transcription complete: {{ $json.title || $json.recording_id }}\n\n{{ ($json.summary || '').slice(0, 100) }}"`) → `Pocket - Ack` (the only `respondToWebhook` node the Telegram branch feeds — don't also wire `Handoff to Assistant` into `Pocket - Ack` if it's ever reconnected, since a webhook can only respond once and two incoming branches to the same `respondToWebhook` node causes a "response already sent" failure on whichever fires second).

When more code is added to this repo (custom nodes, supporting services, etc.), update this file with the actual build/lint/test commands and architecture.
