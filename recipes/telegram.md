# Telegram

## Library

`grammy` — modern Telegram bot framework for Node.js.

## Environment variables

- `TELEGRAM_BOT_TOKEN` — message [@BotFather](https://t.me/BotFather) on Telegram, send `/newbot`, follow the prompts, and copy the token it gives you. To retrieve an existing token, send `/mybots`, select the bot, and tap "API Token."
- `TELEGRAM_OWNER_ID` — your Telegram user ID (a number). Message [@userinfobot](https://t.me/userinfobot) on Telegram and it will reply with your ID.

## Setup

1. Create a bot via BotFather and copy the token
2. Install `grammy`
3. Implement the channel in `src/channels/telegram.ts`

## Owner filtering

Only forward messages where the chat ID matches `TELEGRAM_OWNER_ID`. Silently ignore all other messages.

## Non-text content

Supports text, photos, documents, voice messages, and stickers. For non-text content, normalize to a placeholder string (e.g. `[photo]`, `[voice message]`) since the agent only handles text for now.

## Gotchas

- **`bot.start()` blocks forever.** It runs long polling and does not resolve until the bot stops. Don't await it during registration or it will block the entire process. Fire and forget it.
- **Message length limit.** Telegram caps messages at 4096 characters. The agent can easily exceed this. Split long replies into multiple messages.
- **Typing indicator expires.** Telegram's typing indicator lasts about 5 seconds. If the agent takes longer, re-send the typing action periodically until the response is ready.
- **Markdown formatting.** The agent responds with Markdown, but grammY sends plain text by default. Set `parse_mode` to `"Markdown"` when sending messages so formatting renders correctly.
- grammY handles long polling natively — no webhook or HTTP server needed.
