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

## Markdown conversion

The assistant writes standard markdown; Slack uses "mrkdwn", a similar but incompatible format. Convert messages before posting with a library like `slackify-markdown`. Key differences:

- `**bold**` → `*bold*`
- `[text](url)` → `<url|text>`
- Headings (`#`, `##`, …) → bold line (Slack has no heading syntax)

Code spans, fenced code blocks, blockquotes, and bullet lists are already compatible.

## Diagnostics & reconnection

Bolt's Socket Mode client auto-reconnects on transient WebSocket disruption (ping timeouts, brief network drops). What turns a transient blip into a stuck-disconnected daemon is almost always an unhandled error in the receiver chain — Bolt logs it but the connection never re-establishes. Before designing any self-healing on top, the channel must surface what's actually happening.

The channel implementation MUST:

1. **Register a global error handler.** Catches errors that bubble out of listeners; without this they go to Bolt's logger only and can be drowned out by other output.

   ```ts
   app.error(async (err) => {
     console.error(`[slack] unhandled: ${err.message}`);
   });
   ```

2. **Log Socket Mode lifecycle events.** Reach the underlying client through `app.receiver` (a `SocketModeReceiver` when `socketMode: true`) and listen for the six state events the client emits: `connecting`, `connected`, `authenticated`, `reconnecting`, `disconnecting`, `disconnected`. Log each with a timestamp and the elapsed time since the last received event. (The client uses `eventemitter3` and does not emit `error` events — error surfacing is the `app.error()` handler's job.) Without this timeline, "the bot stopped responding" is indistinguishable from "the daemon crashed silently" or "Slack rate-limited us."

3. **Track `lastEventAt` in the channel module.** Update it whenever the client emits a `slack_event` (covers every inbound Slack event including messages and app_mentions). Not consumed for recovery yet — it's the signal a future watchdog or `/health`-style probe will read.

The `log_level` field in `channels.yaml` defaults to `warn`. Bump to `info` while diagnosing — Bolt emits reconnect attempts at INFO, and their presence or absence tells us whether the client is even trying.

**Recovery is out of scope for this section.** The agent process cannot self-restart via the `agent/protos` wrapper — the wrapper runs as a child of the daemon, so killing the daemon tears down the wrapper subprocess before it reaches the start step. Recovery designs go through an external supervisor (launchd, systemd, etc.) or in-process re-registration, addressed separately once the diagnostics here have identified the actual failure mode.

## Notes

- Socket Mode connects via WebSocket — no HTTP server or public URL needed
- In channels, respond only to mentions or DMs to avoid noise
