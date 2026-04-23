# Terminal

A client-server chat channel. The daemon binds a Unix socket; `agent/protos chat` connects a client. Multiple clients can connect simultaneously, each tracking its own cursor into the conversation's JSONL file.

## Library

Node standard library only — `node:net`, `node:fs/promises`, `node:fs`, `node:readline`. No external dependencies.

## Environment variables

- `PROTOS_TERMINAL` — `true` (default) or `false`. When false, the channel skips registering and the socket isn't created. No other env vars required.
- `PROTOS_TERMINAL_SOCKET` — optional override for the socket path. Defaults to `runtime/agent.sock`.

## Channel identity

- `channelId`: `terminal`
- `conversationId`: `cli` (single persistent thread). All client sessions share this conversation by default.

## Setup

Zero setup. The channel is always generated during build. Start the daemon with `agent/protos start`, then connect with `agent/protos chat` from another terminal (or the same one, after backgrounding).

## Client-server protocol

Unix socket at `runtime/agent.sock`. Messages are newline-delimited JSON.

**Client → server:**

```
{"from": <int>}                               # first line: where to start replay
{"type": "message", "text": "..."}            # user message
```

**Server → client:**

```
{"type": "replay", "role": "user|assistant", "index": N, "text": "..."}
{"type": "live-start"}                         # caught up; now streaming live
{"type": "thinking"}                           # protos is processing — render an indicator
{"type": "message", "role": "user|assistant", "index": N, "text": "..."}
```

The `thinking` event is broadcast to all connected clients when a user-originated message is dispatched to the router. It's **cleared implicitly when the next live `message` event arrives** — no explicit "thinking-stop" event needed. Cron-originated messages (heartbeat, etc.) do NOT trigger `thinking`, since no one is waiting on them.

## Cursor-based replay

The conversation lives at `runtime/threads/terminal/cli.jsonl` like any other channel. Each line is an event in the [shared schema](../architecture.md#event-schema).

When a client connects:

1. Client sends `{"from": N}` — the last index it has seen locally.
2. Server reads lines `N..end` from the JSONL file, applies the [render filter](#render-filter) to collapse multi-step turns into a single displayable message, and streams the results as `replay` messages.
3. Server emits `{"type": "live-start"}` to signal the client is caught up.
4. Server subscribes to the JSONL file via `fs.watch` and streams new appends (filtered the same way) as `message` events.

### Render filter

The JSONL stores the full agent event stream (user messages, assistant steps with tool calls, tool results). The client only wants to see the user-facing conversation. The server collapses events per `turn_id` before emitting to clients:

- `user` events → emitted as-is.
- `assistant` events → only the **last** assistant text per `turn_id` is emitted (intermediate "Let me check…" texts from multi-step turns are skipped). Empty-text assistant events and the literal `NO_REPLY` lifecycle marker are also skipped — `NO_REPLY` is not a message.
- An assistant event's `tool_calls` field, and `role: "tool"` events → **not emitted** to the client in the default view.
- `audit` events (`cron_start`, `cron_end`) → **not emitted** to the client.

The `index` carried on each emitted message is the JSONL line index of the event that sourced it — the user event for a user message, the chosen assistant-text event for a collapsed assistant reply. The filter is idempotent on any prefix of the stream: re-running it as new events append produces the same messages with the same indices, so the client's `lastEmittedIndex` check dedupes re-emits when a turn grows.

The client stores its cursor at `runtime/clients/{session}.cursor` (default session: `chat`). The default replay window on connect is computed as `from = min(stored_cursor, total - 20)` — show **at least the last 20 messages** for context, plus everything queued since the last disconnect if that span is longer. This means:

- A fresh `agent/protos chat` in an existing conversation always shows recent context (not an empty screen).
- Messages queued while disconnected (e.g., cron firing overnight) are never skipped.
- Whichever span is longer — "last 20" or "since cursor" — is the one you see.

The flags `--from-start`, `--last N`, and `--new` override this default.

## Owner filtering

None needed. The socket lives in the workspace filesystem; only processes with filesystem access to `runtime/agent.sock` can connect. Document that the socket should not be exposed over the network.

## Implementation notes

- The channel's `register()` is called once at daemon startup. It creates the socket server, registers the accept handler, and returns `{ channelId, ownerConversationId: "cli", send }`.
- The `send` function is called by the router with agent replies. No direct socket write is needed: the router has already appended events to the JSONL via `threads.appendEvent` as the agent produced them, and connected clients pick up new lines via their `fs.watch` subscription.
- The channel maintains one `fs.watch` per connected client (or one watch shared across clients — implementation choice). On each new line, apply the [render filter](#render-filter) and emit to clients.
- On daemon shutdown, close all client sockets and remove the socket file.

### Thinking indicator

When the channel receives a user message from a connected client and dispatches it to the router, broadcast `{"type": "thinking"}` to all connected clients (including other clients watching the same conversation). Sequence per user turn:

1. Client sends `{"type": "message", "text": "..."}`.
2. Channel forwards to the router (which appends to the JSONL file → fs.watch broadcasts the user echo to all clients).
3. Channel broadcasts `{"type": "thinking"}` to all clients.
4. Agent processes; channel's `send` is eventually called with the reply (which appends to the JSONL → fs.watch broadcasts the assistant message).
5. Clients render the assistant message and clear the thinking indicator implicitly.

Don't broadcast `thinking` for messages that arrive via channels other than this one (e.g., the user typing in Telegram while a terminal is connected) — only for messages this channel itself dispatched. Cron-originated messages also skip the indicator (no one is waiting).


## Client (`agent/protos chat`)

Lives at `agent/src/cli/chat.ts` — isolated from the rest of the codebase. Only imports `node:` standard library and the shared protocol types from `agent/src/channels/terminal-protocol.ts`. Does not import the router, agent, memory, or any other engine code. May additionally import thin npm packages for rendering (see below).

Flags:

- `agent/protos chat` — show at least the last 20 messages on connect (see [Cursor-based replay](#cursor-based-replay))
- `agent/protos chat --from-start` — replay the entire conversation
- `agent/protos chat --last N` — replay last N messages
- `agent/protos chat --new` — advance cursor to end without replay (show only new messages from this point on)

Exit with `Ctrl+D`, `/quit`, or `/exit`. Client disconnect doesn't affect the daemon.

If the daemon isn't running, `chat` fails fast with: `error: daemon not running. Start it with "agent/protos start".` Don't auto-start.

### Rendering

The client formats assistant replies for terminal readability:

- **Word-boundary wrapping.** Wrap output at the terminal width (`process.stdout.columns`, defaulting to 80 if unavailable). Preserve leading whitespace so indented content (list items, code blocks) lines up correctly on continuation lines. ANSI escape codes must not count toward visible width. Use a small well-maintained package such as [`wrap-ansi`](https://www.npmjs.com/package/wrap-ansi) — don't hand-roll the wrapping logic.
- **Inline markdown → ANSI.** The agent replies with markdown; render the common inline styles:
  - `**bold**` / `__bold__` → ANSI bold (`\x1b[1m` … `\x1b[22m`)
  - `*italic*` / `_italic_` → ANSI italic (`\x1b[3m` … `\x1b[23m`)
  - `` `code` `` → ANSI inverse or a dim color
  - Fenced code blocks (```` ``` ````) → indented, colored if you like
  - Headings (`# Title`) → bold, optionally with a blank line before
  - Links `[text](url)` → show as `text (url)` or with underline on `text`
- **User input and control messages** (your own lines, `❯ ` prompt, thinking indicator) are plain — no markdown parsing applied.
- Keep it simple: small regex-based renderer, not a full markdown parser. Block-level features beyond code fences and headings can be passed through as-is.

Re-render the *entire* assistant message after the wrapping pass — otherwise partial ANSI codes from one line can leak into the next.

### Chat flow

The loop should feel like a normal chat: the user types a line, sees it rendered in the transcript, the assistant replies, type the next line. All messages (user and assistant, live and replay) go through the **same render path** — the transcript is a canonical record of the conversation, regardless of which client sent what.

- **Prompt:** `❯ ` (just the prompt character). This is what the user sees before typing.
- **Typing echo is readline's job, not ours.** While the user is typing, `node:readline` handles character echo and line wrapping natively. Don't intercept keystrokes, don't `rl.prompt(true)` on input events, and don't trigger redraws in response to any client-side or server-side signal while typing is in progress. Extra redraws during typing reprint the whole prompt + in-progress input each time — on a wrapped line, that produces a new copy of the input on every character past the terminal width, stacking into a wall of duplicates.
- **Pastes appear as a collapsed placeholder.** Enable **bracketed paste mode** at client startup (`\x1b[?2004h`) and restore on exit (`\x1b[?2004l`). Terminals that support it wrap pasted content in `\x1b[200~ … \x1b[201~` markers; the client treats everything between those markers as a single input chunk (not individual keystrokes or Enter events).

  **Long or multi-line pastes are inserted into the input buffer as an opaque placeholder token**, not as raw text. The token format is `[Pasted text #N +M lines]` where `N` is a per-session counter (incremented on each paste) and `M` is the number of lines in the paste. Hold the real content in an in-memory map keyed on the token. The user can compose around the token — cursor past it, backspace/Ctrl-W to delete it, re-paste to replace it — but cannot edit inside the paste (readline is single-line). On Enter, before sending, scan the line for placeholder tokens and expand them to their stashed content.

  Render the placeholder token in a dimmed ANSI color in the input so it's visually distinct from typed text.

  A paste is "long" if it contains any `\n` OR exceeds 200 characters. Shorter single-line pastes insert verbatim into readline's buffer (no placeholder needed).

  **Newline normalization in raw mode.** stdin is in raw mode (`ICRNL` disabled), so pasted newlines arrive as bare `\r`, not `\n`. Normalize pasted content: replace every `\r\n` and bare `\r` with `\n` before stashing. When a placeholder is expanded back on send, the expansion keeps the `\n`s — they ride the wire as `\n`.
- **Enter key behavior:**
  - **Plain Enter with non-empty input:** intercept the Enter keypress *before* readline processes it (same mechanism as the Alt+Enter interception below). At that moment `rl.line` still holds the typed input and the cursor is still on the echoed row(s) — use the same clear path as async interruptions (compute row count from `rl.line` + prompt width, step up through the wrapped rows, clear each). Then expand placeholder tokens in the captured line, send to server as `{type: "message", text: "..."}`, reset `rl.line` to empty, and redraw the prompt. The displayed message comes from the server's broadcast, which reaches every connected client uniformly; the local echo exists only in the client the user typed into, so clearing it at the keypress — on that client only — is the right scope.
  - **Plain Enter with empty or whitespace-only input:** no-op. Don't send, don't clear the prompt.
  - **Alt+Enter:** insert a literal `\n` into readline's **single input buffer** at the cursor position. The multi-line message lives in one editable buffer — the user can arrow back to earlier lines and edit them, same as any other text in the buffer. On plain Enter, the whole buffer (newlines and all) is sent in one `{type: "message"}` payload. Implementation: intercept the raw byte sequence `ESC\r` (`0x1B 0x0D`) before readline's submit path, splice `\n` into `rl.line` at `rl.cursor`, advance `rl.cursor` past it, then call `rl._refreshLine()` to redraw. These are private readline APIs — the surface is small (three touchpoints) and has been stable in Node for years; leave a comment at the call site noting the dependency. **Terminal config required:** the Option/Alt key must be set to emit `ESC+` for the sequence to reach the process (iTerm2: Profiles → Keys → "Left Option key" = `Esc+`; macOS Terminal.app: Profiles → Keyboard → "Use Option as Meta key"; similar on other emulators).
- **Server echoes via `fs.watch`:** the JSONL append triggers a broadcast; the client receives a `{type: "message", role: "user", ...}` event and renders it normally. Same render path as replay. This is also how other connected clients see the message — cross-client visibility is free.
- **Assistant reply arrives:** render the body wrapped with markdown applied.
- **Spacing:** every rendered message (user or assistant) is followed by exactly one blank line. Adjacent messages are therefore separated by a single blank line, and the prompt sits one blank line below the most recent message. Don't add a leading blank line — the trailing blank from the previous message is the separator.
- **Floating prompt (always at the bottom).** The `❯ ` prompt — and any in-progress input the user is typing — must always sit at the bottom of the screen. When async output (assistant reply, other client's echo, etc.) arrives while the user is mid-typing, the message is printed *above* the prompt and the prompt is redrawn *below* with the user's in-progress input intact. To achieve this: clear the prompt line(s) (account for input that wraps across multiple terminal rows), print the message (with its trailing blank line per the spacing rule), then redraw the prompt with the preserved in-progress input. Never print over the user's input or the prompt itself.
- **Prompt draws are idempotent.** Any code path that draws the prompt (initial startup, after printing async output, on `live-start`, etc.) must first clear the row(s) the prompt will occupy before writing the prefix. Drawing without clearing stacks prompts back-to-back (`❯ ❯`) when two draws happen in sequence — e.g. initial startup draw followed by a `live-start` draw with no replay events between them.
- **Distinguishing speakers:** user and assistant messages are differentiated visually, not with text labels. Past user messages render with the `❯ ` prefix on their **first line only**; continuation lines of a multi-line user message indent two spaces to align under the first line's text (no additional `❯` per line — that makes one message look like many). Every line of the user message carries the highlighted background (a subtle ANSI background color spanning the visible width of the line) and bright white foreground. Assistant replies render plain — default foreground, no background, no prefix. The in-progress input the user is currently typing at the prompt renders in the terminal's default color (no bright white, no background) — the highlight is applied only once the message is committed and rendered as part of the transcript.
- **Input history (↑ / ↓).** Up-arrow scrolls back through prior messages the user has sent in this conversation, down-arrow scrolls forward toward the empty prompt. There is no separate history file — the conversation JSONL already records every user event. The client seeds `node:readline`'s `history` buffer from the user events it sees during replay, and appends each new sent message to the buffer. Skip blank lines and `/quit` / `/exit` commands. No separate cap — `node:readline` handles a reasonable in-memory history size on its own. For multi-line input (e.g. pasted content spanning several lines), ↑ / ↓ navigate between lines of the current input when the cursor is not on the first/last line; only cross into history when the cursor is at the first/last line.
- **Line editing.** Use `node:readline` for input handling rather than raw keypress + manual rendering. readline gives arrow-key navigation, Ctrl-A / Ctrl-E / Ctrl-K / Ctrl-U / Ctrl-W, word navigation, delete/backspace, and terminal-native line wrapping for free. Rolling your own quickly accumulates bugs (redraw-on-every-keystroke, broken wrap handling, missing shortcuts) and reimplements a solved problem.


### Thinking indicator (client)

The indicator is just a **prompt swap**. No separate line, no "protos is thinking" text.

- Default prompt: `❯ `
- While thinking: a single animated character followed by a space (animation style is up to the client — keep it subtle)

On `{"type": "thinking"}`: swap the prompt to the animated thinking indicator and redraw the prompt line (clear + redraw with the in-progress input preserved).

On `{"type": "message", "role": "assistant"}` OR client-side ~60 s timeout: restore the default prompt and redraw. The timeout is a safety net for daemon crashes; under normal operation the indicator clears as soon as the assistant reply arrives. User-echo messages do NOT clear the indicator — otherwise it would vanish before the agent has had a chance to think.

Only one indicator at a time per client; if a second `thinking` arrives while the indicator is already showing, just reset the timeout.
