# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository is in its initial state: it contains no application code yet, only a `docs/` folder with research notes. There are no build, lint, or test commands to run because no source tree, package manifest, or test suite exists.

## Purpose

Per `docs/Telegram and n8n getting started.md`, the goal is to connect a Telegram bot to an n8n workflow instance:

- A Telegram bot is created via `@BotFather`, which issues an HTTP API token.
- That token is added as a **Telegram API** credential in n8n, consumed by a **Telegram Trigger** node.
- n8n must be reachable via a public HTTPS URL (`WEBHOOK_URL` env var) for Telegram to deliver webhook updates — for local dev this typically means an ngrok tunnel or similar.
- Incoming messages carry `message.text`, `chat.id`, and `from.id`; workflows commonly gate on `chat.id` (via an IF node) to restrict the bot to a specific user before continuing into downstream automation.

When code is added to this repo (e.g. n8n workflow exports, custom nodes, or supporting services), update this file with the actual build/lint/test commands and architecture.
