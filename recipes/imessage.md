# iMessage

## Library

No npm library needed. Uses the [BlueBubbles](https://bluebubbles.app) server's REST API.

## Prerequisites

- A Mac with Messages.app signed into your Apple ID
- BlueBubbles server installed and running on that Mac (can be the same machine running Logos)

## Environment variables

- `BLUEBUBBLES_URL` — BlueBubbles server URL (e.g. `http://localhost:1234`)
- `BLUEBUBBLES_PASSWORD` — server password for API authentication

## Setup

1. Install and configure BlueBubbles server on a Mac
2. Implement the channel in `src/channels/imessage.ts`
3. Use the REST API to send messages and webhooks to receive them

## Notes

- BlueBubbles exposes iMessage via REST API and webhooks
- Receiving messages requires a webhook endpoint — this channel needs a local HTTP server
- Supports text, attachments, reactions, and read receipts
- The Private API option in BlueBubbles enables typing indicators and richer features
