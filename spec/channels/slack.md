# Slack

## Library

`@slack/bolt` — official Slack app framework.

## Environment variables

- `SLACK_BOT_TOKEN` — Bot User OAuth Token (starts with `xoxb-`)
- `SLACK_APP_TOKEN` — App-Level Token for Socket Mode (starts with `xapp-`)

## Setup

1. Create a Slack app at [api.slack.com](https://api.slack.com/apps)
2. Enable Socket Mode (avoids needing a public URL)
3. Add bot token scopes: `chat:write`, `channels:history`, `groups:history`, `im:history`, `app_mentions:read`
4. Subscribe to events: `message.channels`, `message.groups`, `message.im`, `app_mention`
5. Install the app to your workspace
6. Install `@slack/bolt`

## Notes

- Socket Mode connects via WebSocket — no HTTP server or public URL needed
- In channels, respond only to mentions or DMs to avoid noise
