# Voice Output (Text-to-Speech)

Enables the agent to reply with voice messages.

## Approach

After the agent generates a text response, convert it to audio and send it back through the channel as a voice message. This can be the default behavior for a conversation, or triggered by the agent when it decides a voice reply is appropriate.

## Options

- **OpenAI TTS API** — hosted, multiple voices, simple.
- **ElevenLabs** — hosted, high-quality voices, voice cloning.
- **`piper`** — local, fast, no API key needed. Good for privacy.

## Environment variables

Depends on the provider:

- `OPENAI_API_KEY` — for OpenAI TTS
- `ELEVENLABS_API_KEY` — for ElevenLabs

No env vars needed for local piper.

## Implementation

1. Generate audio from the agent's response text
2. Save to a temp file
3. Send via the channel's attachment/voice message API
4. Clean up the temp file

## Notes

- Not all channels support voice messages — fall back to text where unsupported
- Consider making this per-conversation configurable (some chats voice, some text)
