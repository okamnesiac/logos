# Telegram

## Library

`grammy` — modern Telegram bot framework for Node.js.

## Environment variables

- `TELEGRAM_BOT_TOKEN` — get this from [@BotFather](https://t.me/BotFather)

## Setup

1. Create a bot via BotFather and copy the token
2. Install `grammy`
3. Implement the channel in `src/channels/telegram.ts`

## Notes

- grammY handles long polling natively — no webhook or HTTP server needed
- Supports text, photos, documents, voice messages, and stickers
- Group chats: the bot only receives messages when mentioned by name or replied to, unless group privacy is disabled via BotFather
