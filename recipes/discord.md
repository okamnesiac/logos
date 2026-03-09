# Discord

## Library

`discord.js` — the standard Discord bot library for Node.js.

## Environment variables

- `DISCORD_BOT_TOKEN` — from the [Discord Developer Portal](https://discord.com/developers/applications)

## Setup

1. Create an application and bot in the Discord Developer Portal
2. Enable the Message Content intent (required to read message text)
3. Invite the bot to your server with appropriate permissions
4. Install `discord.js`
5. Implement the channel in `src/channels/discord.ts`

## Notes

- discord.js handles the gateway WebSocket natively — no HTTP server needed
- In servers, the bot receives all messages in channels it has access to — use mention gating to avoid responding to everything
- Supports text, embeds, attachments, threads, and reactions
