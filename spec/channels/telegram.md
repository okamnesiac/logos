# Telegram

## Library

`grammy` — modern Telegram bot framework for Node.js.

## Configuration

`config/channels.yaml` entry:

```yaml
telegram:
  bot_token: $TELEGRAM_BOT_TOKEN
  owner_id: 456789
```

- **`bot_token`** — message [@BotFather](https://t.me/BotFather) on Telegram, send `/newbot`, follow the prompts, and copy the token. To retrieve an existing token, send `/mybots`, select the bot, and tap "API Token."
- **`owner_id`** — your Telegram user ID (a number). Message [@userinfobot](https://t.me/userinfobot) on Telegram and it will reply with your ID.

## Setup

1. Create a bot via BotFather and copy the token
2. Install `grammy`

## Owner filtering

Only forward messages where the chat ID matches the entry's `owner_id`. Silently ignore all other messages.

## Non-text content

Telegram delivers text, photos, documents, voice messages, and stickers.

- **Photos** — fetch the highest-resolution variant via `ctx.getFile()` / `bot.api.getFile()`, hash the bytes (sha256), write to `runtime/blobs/{sha256}.{ext}` (extension derived from the file's `mime_type`), and attach as an `image` on the dispatched message. The message `text` is the photo's caption if present, empty string otherwise.
- **Documents, voice messages, stickers** — normalize to a placeholder string (e.g. `[document: filename.pdf]`, `[voice message]`, `[sticker: 👋]`). Transcription and document parsing are out of scope for v1.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Markdown conversion

The assistant writes standard markdown; Telegram expects MarkdownV2 (with required escaping of reserved characters `_ * [ ] ( ) ~ \` > # + - = | { } . !`). Convert messages with a library like `telegramify-markdown` and send with `parse_mode: "MarkdownV2"`. Standard `**bold**`, headings, and lists all come through correctly after conversion.

## Gotchas

- **`bot.start()` blocks forever.** It runs long polling and does not resolve until the bot stops. Don't await it during registration or it will block the entire process. Fire and forget it.
- **Message length limit.** Telegram caps messages at 4096 characters. The agent can easily exceed this. Split long replies into multiple messages.
- **Typing indicator expires.** Telegram's typing indicator lasts about 5 seconds. If the agent takes longer, re-send the typing action periodically until the response is ready.
- grammY handles long polling natively — no webhook or HTTP server needed.
