# I think I installed Telegram Air on my laptop and we started at some point to setup the bot but I do not recall it now. What is the fastest path to hook Telegram to n8n?

Fastest path: **create a bot with BotFather → add Telegram credentials in n8n → drop a Telegram Trigger node → make sure n8n has a public HTTPS URL (WEBHOOK_URL) → test by messaging the bot.**[^1][^2][^3]

Here’s the concrete, minimal flow you can follow today.

## 1. Create the Telegram bot (BotFather)

In any Telegram client (desktop/mobile):

1. Search for `@BotFather` and start a chat.[^2][^3]
2. Send `/newbot` and follow the prompts:
    - Give it a name (e.g. `Echogrid Assistant`).
    - Give it a username ending in `bot` (e.g. `echogrid_assistant_bot`).[^3][^2]
3. BotFather will reply with an **HTTP API token** like `123456789:ABC-DEF...`.
    - That token is what n8n needs; keep it handy.[^4][^2][^3]

You don’t need Telegram Air specifically here; any client will do as long as you can talk to BotFather.

## 2. Add Telegram credentials in n8n

In your n8n (dev or prod):

1. Open any workflow and add a **Telegram Trigger** node (search “Telegram”).[^1][^3]
2. Under *Credentials* click “Create New” and choose **Telegram API**.[^2][^1]
3. Paste the BotFather token into the *Access Token* field and save.[^3][^2]

That’s it for credentials; n8n now knows how to talk to your bot.

## 3. Configure the Telegram Trigger node

In the same Telegram Trigger node:

1. Set **Trigger On** to `Message` (or `All Updates` if you want everything, which can help while debugging).[^5][^6][^1]
2. Save the workflow, then click **Execute Node** (or activate the workflow in “Production” mode).
3. Behind the scenes n8n will register a **Telegram webhook** pointing from your bot to your n8n instance.[^6][^5]

If you want to confirm the webhook, you can call:

```text
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo
```

and check that the URL is your n8n HTTPS endpoint.[^5][^6]

## 4. Make sure n8n is reachable via HTTPS

This is usually the only “pain point” when self‑hosting:

- Telegram requires the webhook URL to be **HTTPS, public, and resolvable**.[^7][^6]
- On self‑hosted n8n you typically set environment variables:
    - `WEBHOOK_URL=https://your-domain-or-ngrok-url/`
    - (Sometimes also `WEBHOOK_TUNNEL_URL` depending on setup.)[^8][^6]
- If you’re running locally, the fastest hack is to use **ngrok** to expose your n8n dev port and set `WEBHOOK_URL` to that HTTPS tunnel.[^8]

After changing env vars, restart n8n so the Telegram Trigger can register the correct webhook.[^6][^7]

## 5. Test with a message from Telegram

Once the Trigger is executing:

1. Open Telegram, find your newly created bot, and send it a test message (e.g. `test` or `/start`).[^2][^3]
2. In n8n, the Telegram Trigger node should immediately receive an execution with JSON that includes:
    - `message.text`
    - `chat.id` (your chat ID)
    - `from.id`, etc.[^4][^1]
3. Use that chat ID to restrict the bot to you:
    - Add an **IF** node after the Trigger: `chat.id == <your id>`.[^4][^2]
    - This ensures nobody else can drive your personal workflows even if they discover the bot.[^2][^4]

At that point, Telegram is “hooked” to n8n: you can wire the Trigger into your Pocket → assistant workflows, agent nodes, or any HTTP / Retell / CRM logic you want.[^9][^10][^11]

If you tell me how your n8n is deployed (Docker with a domain, local dev with ngrok, Cloudflare tunnel, etc.), I can spell out the exact `WEBHOOK_URL` and env settings for your current instance so you can go from “nothing” to “Telegram assistant listening” in one pass.
<span style="display:none">[^12][^13][^14][^15][^16]</span>

<div align="center">⁂</div>

[^1]: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram

[^2]: https://www.youtube.com/watch?v=QZ93nQGwnPg

[^3]: https://www.youtube.com/watch?v=QpnH5R5FbKU

[^4]: https://community.n8n.io/t/getting-started-with-the-telegram-node/371

[^5]: https://community.n8n.io/t/solved-telegram-trigger-configuration/80224

[^6]: https://community.n8n.io/t/telegram-trigger-webhook-issue/100237

[^7]: https://community.n8n.io/t/telegram-trigger-bad-request-bad-webhook-an-https-url-must-be-provided-for-webhook/212165

[^8]: https://www.reddit.com/r/n8n/comments/1m300mn/guide_connecting_telegram_to_n8n_a_stepbystep/

[^9]: https://www.youtube.com/watch?v=ODdRXozldPw

[^10]: https://www.youtube.com/watch?v=NMwCESJTvgE

[^11]: https://n8n.io/workflows/2462-angie-personal-ai-assistant-with-telegram-voice-and-text/

[^12]: https://n8nautomation.cloud/blog/n8n-telegram-bot-integration-guide

[^13]: https://www.youtube.com/watch?v=-BpN4OZZDwE

[^14]: https://community.n8n.io/t/using-n8n-cloud-can-i-trigger-webhook-using-telegram/157627

[^15]: https://community.n8n.io/t/telegram-bot-webhook-responses-ai-agent/117183

[^16]: https://www.youtube.com/watch?v=xoW8laTHUVw

