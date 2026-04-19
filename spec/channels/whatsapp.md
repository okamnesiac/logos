# WhatsApp

## Library

`@whiskeysockets/baileys` — unofficial WhatsApp Web API. Used by both OpenClaw and NanoClaw.

## Environment variables

- None — Baileys authenticates by linking as a companion device (QR code or pairing code)

## Setup

1. Install `@whiskeysockets/baileys` and `qrcode-terminal`
2. On first run, scan the QR code with WhatsApp on your phone to link the device

## Notes

- Auth state (keys and session) should be persisted to disk so you don't need to re-scan on restart
- Baileys connects via WebSocket — no HTTP server needed
- Supports text, images, audio, video, and documents
- Unofficial library — WhatsApp could break compatibility at any time
