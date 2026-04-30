# Slack

## Library

`@slack/bolt` — official Slack app framework.

## Configuration

`config/channels.yaml` entry:

```yaml
slack:
  bot_token: $SLACK_BOT_TOKEN
  app_token: $SLACK_APP_TOKEN
  owner_id: U123ABC456
```

- **`bot_token`** — Bot User OAuth Token (starts with `xoxb-`)
- **`app_token`** — App-Level Token for Socket Mode (starts with `xapp-`)
- **`owner_id`** — your Slack member ID. Open your profile in Slack, click the "…" menu, and "Copy member ID".

## Setup

1. Create a Slack app at [api.slack.com](https://api.slack.com/apps)
2. Enable Socket Mode (avoids needing a public URL)
3. Add bot token scopes: `chat:write`, `channels:history`, `groups:history`, `im:history`, `app_mentions:read`
4. Subscribe to events: `message.channels`, `message.groups`, `message.im`, `app_mention`
5. Install the app to your workspace
6. Install `@slack/bolt`

## Non-text content

Slack messages can carry uploaded files (`event.files[]`).

- **Image files** — for each file whose `mimetype` starts with `image/`, fetch the bytes from `file.url_private` with `Authorization: Bearer ${bot_token}` (Slack file URLs require auth even for the bot's own workspace), hash (sha256), write to `runtime/blobs/{sha256}.{ext}`, and attach as an `image` on the dispatched message. The message `text` is `event.text` if present; if blank, fall back to `file.title` or `file.initial_comment` (Slack often puts captions there for file-only posts).
- **Non-image files** (pdf, docx, audio, …) — normalize to a placeholder string (e.g. `[document: report.pdf]`, `[voice message]`). Transcription and document parsing are out of scope for v1.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Receipt ack

Slack has no bot-visible typing indicator, so the channel posts a short placeholder message as soon as it accepts an inbound event, before dispatching to the router. This gives the user immediate feedback that the message was received and is being worked on; the agent's actual reply lands as a follow-up message (the ack stays in the channel — there's no edit-in-place or deletion).

- The placeholder is a randomly-selected playful gerund — "Thinking…", "Pondering…", "Noodling…", "Mulling…", "Brewing…", etc. — followed by `...`. The variation is deliberate (mirrors the Claude Code thinking indicator) and the verb is picked fresh on every inbound event. Keep the list playful but tasteful; one of the entries should always be the literal `Thinking`.
- Post in the same channel, on the same thread (`thread_ts` from the inbound event for DM messages; the thread root — `event.thread_ts ?? event.ts` — for `app_mention`).
- If the post fails (e.g. transient API error), log `[slack] ack failed: <msg>` and continue — the dispatch still runs. Failing the receipt must never prevent the message from reaching the router.

## Markdown conversion

The assistant writes standard markdown; Slack uses "mrkdwn", a similar but incompatible format. Convert messages before posting with a library like `slackify-markdown`. Key differences:

- `**bold**` → `*bold*`
- `[text](url)` → `<url|text>`
- Headings (`#`, `##`, …) → bold line (Slack has no heading syntax)

Code spans, fenced code blocks, blockquotes, and bullet lists are already compatible.

## Notes

- Socket Mode connects via WebSocket — no HTTP server or public URL needed
- In channels, respond only to mentions or DMs to avoid noise
