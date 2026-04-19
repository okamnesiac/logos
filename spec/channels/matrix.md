# Matrix

## Library

`matrix-js-sdk` — official Matrix client SDK for JavaScript.

## Environment variables

- `MATRIX_HOMESERVER` — homeserver URL (e.g. `https://matrix.org`)
- `MATRIX_USER_ID` — bot's user ID (e.g. `@logos:matrix.org`)
- `MATRIX_ACCESS_TOKEN` — access token for the bot account

## Setup

1. Create a Matrix account for the bot on your homeserver
2. Generate an access token
3. Install `matrix-js-sdk`

## Notes

- matrix-js-sdk handles sync via long polling — no HTTP server needed
- Supports text, formatted messages (HTML), files, and end-to-end encryption (with additional setup)
- Self-hosted homeservers give you full control over data
