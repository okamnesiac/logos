# Telegram

## Library

`grammy` — modern Telegram bot framework for Node.js.

## Environment variables

- `TELEGRAM_BOT_TOKEN` — get this from [@BotFather](https://t.me/BotFather)
- `TELEGRAM_OWNER_CHAT_ID` — the owner's Telegram chat ID. Send a message to the bot, then check the logs or use [@userinfobot](https://t.me/userinfobot) to find it.

## Setup

1. Create a bot via BotFather and copy the token
2. Install `grammy`
3. Implement the channel in `src/channels/telegram.ts`

## Owner filtering

Only forward messages where the chat ID matches `TELEGRAM_OWNER_CHAT_ID`. Silently ignore all other messages.

## Notes

- grammY handles long polling natively — no webhook or HTTP server needed
- Supports text, photos, documents, voice messages, and stickers
