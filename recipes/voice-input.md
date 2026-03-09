# Voice Input (Transcription)

Enables the agent to understand voice messages received through any channel.

## Approach

When a channel receives a voice message or audio attachment, transcribe it to text before passing it to the agent. The agent sees the transcription as part of the message — it doesn't need to know the original was audio.

## Options

- **OpenAI Whisper API** — hosted, simple, high quality. Requires an OpenAI API key.
- **`whisper.cpp`** — local, no API key needed, runs on CPU or Apple Silicon. Good for privacy.
- **Deepgram** — hosted, fast, real-time streaming support.

## Environment variables

Depends on the provider:

- `OPENAI_API_KEY` — for Whisper API
- `DEEPGRAM_API_KEY` — for Deepgram

No env vars needed for local whisper.cpp.

## Implementation

1. In the channel code, detect audio/voice attachments
2. Download the audio file to a temp location
3. Send it to the transcription provider
4. Prepend or replace the message text with the transcription (e.g. `[Voice message]: <transcribed text>`)
5. Pass the message to the router as normal
