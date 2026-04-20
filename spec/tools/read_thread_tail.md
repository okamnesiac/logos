# read_thread_tail

Read the unconsolidated messages of a conversation thread.

## Description (shown to the model)

Return the messages in a thread that haven't been consolidated yet (everything from the cursor to the end of the JSONL), plus a `newCursor` value to pass to `advance_thread_cursor` after you've successfully processed them. Pair with `list_threads` and `advance_thread_cursor` to run a consolidation pass.

## Input

```ts
{ channelId: string, conversationId: string }
```

## Output

```ts
{
  messages: Array<{ role: "user" | "assistant", text: string, timestamp: string }>,
  newCursor: number  // index just past the last event of the last returned turn (== total at read time)
}
```

`messages` is empty if the cursor is already at the end of the thread. Always returns the discriminated shape — never `null`.

## Behavior

- Reads the sidecar cursor file (missing means `0`), reads the JSONL.
- Applies the same render filter channels use (see `architecture.md` → Cursor-based replay → Render filter): for each `turn_id`, emit the user event and the **last** assistant text; hide intermediate assistant texts, tool calls, tool results, and system events. Consolidation cares about the conversation, not the agent's internal reasoning.
- `newCursor` is the count of total **events** (lines) at read time. Pass this to `advance_thread_cursor` to mark the full span consolidated. **Don't compute the new cursor yourself** — using the value returned here avoids a race where events arriving between this call and `advance_thread_cursor` would otherwise be skipped.
- Does NOT advance the cursor. The agent is responsible for advancing only after the messages have been successfully processed (e.g. summarized into memory).

## Dependencies

Built-in, no external services. Reads `runtime/threads/`.

## Implementation notes

- Same JSONL parsing as the regular thread loader. Skip malformed lines silently (consistent with other thread readers).
- If `channelId` or `conversationId` doesn't correspond to an existing thread, return `{ messages: [], newCursor: 0 }`.
