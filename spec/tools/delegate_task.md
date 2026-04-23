# delegate_task

Spawn a focused sub-agent to handle a task in an isolated context. The sub-agent gets a task-specific prompt, an explicit list of skills (full skill bodies inlined), and an explicit allowlist of tools. Its work doesn't pollute the main agent's context — only its final response comes back.

## Description (shown to the model)

Delegate a focused task to a sub-agent. You write the prompt, choose which skills it should know, and choose which tools it can use. The sub-agent runs in isolation — it doesn't see your conversation history, identity, or memory manifest. Only its final response comes back to you.

Use this when:
- A task needs many tool calls or large intermediate outputs (web research, deep code analysis, scanning many files)
- You want a fresh focused context for a sub-task
- You want to use a specialized skill set without loading those instructions into your own prompt

The sub-agent has only what you give it. Always specify both `skills` and `tools` — there are no defaults.

## Input

```ts
{
  prompt: string,        // The task description. Be specific about what you want returned.
  skills: string[],      // Skill names to load. The full body of each gets inlined into the sub-agent's system prompt. Pass [] for no skills.
  tools: string[],       // Tool names the sub-agent may call. Pass [] for no tools (purely a thinking task).
  model?: string,        // Optional model override. Defaults to AI_MODEL.
}
```

## Output

```ts
{ ok: true, response: string, sub_agent_log: string }
| { ok: false, error: string, sub_agent_log: string }
```

- `response` — the sub-agent's final assistant message as plain text. If you need structured data, ask for JSON in the prompt.
- `sub_agent_log` — workspace-relative path to the sub-agent's full event-stream log (JSONL), for debugging or reflection. The path is nested under the caller's log (see `architecture.md` → [Sub-agents](architecture.md#sub-agents) for the exact layout).

## Behavior

The runner (in `agent/src/agents/runner.ts`):

1. **Resolve skills.** For each name in `skills`, find the skill body — search `spec/skills/{name}.md`, then `config/skills/{name}.md`, then `config/skills/{name}/SKILL.md`; config wins on collision. Read the full body. If any skill is missing, return `{ ok: false, error: "skill not found: {name}" }`.
2. **Resolve tools.** For each name in `tools`, fetch the tool definition from the loaded tools map. If any tool is missing, return `{ ok: false, error: "tool not found: {name}" }`.
3. **Build the sub-agent system prompt:**
   - A short framing line: "You are a focused sub-agent invoked by the main Protos agent. Complete the task and return your final response. You have no access to the main agent's identity, memory, or conversation history beyond what's stated below."
   - Today's date.
   - For each skill, the full body (frontmatter optional — body is what matters).
   - **Not included:** SOUL.md, memory manifest, recent journal, conversation history.
4. **Call `generateText`** with the assembled system prompt, the `prompt` as the user message, the resolved tool subset, the chosen model, and a step limit (initially the same as the main agent's, e.g. `stepCountIs(25)`).
5. **Write the event-stream log.** Every event the sub-agent produces (user prompt, assistant steps, tool calls, tool results) appends to the sub-agent's log file using the same [event schema](../architecture.md#event-schema) as threads and cron logs. The log path is determined by the caller:
   - Called from a cron run: `runtime/logs/cron/{jobname}/{ISO-timestamp}/{call_id}.jsonl`
   - Called from a thread turn: `runtime/logs/sub-agents/{channelId}/{conversationId}/{turn_id}-{call_id}.jsonl`
6. **Return** the sub-agent's final assistant text as `{ ok: true, response, sub_agent_log }` or any thrown error as `{ ok: false, error, sub_agent_log }`. The parent's `tool_result` event (written automatically by the tool-loop machinery) includes this full return value, so the sub-agent log path travels with the parent's record.

## Constraints

- **No nested sub-agents.** The runner removes `delegate_task` from the tool allowlist passed to the sub-agent (even if the caller asks for it). Single level of delegation, period. Revisit if needed.
- **Sub-agent has no awareness of being one.** The framing line is the only signal. Otherwise it's a generic LLM call with the given prompt and tools.
- **Skills are inlined, not just listed.** The point of inlining is that the sub-agent acts on the skill instructions immediately, without spending steps on `read_file`.

## Examples

Web research with a digestion skill:

```js
delegate_task({
  prompt: "Find recent (last 30 days) news about Anthropic safety research. Return a 5-bullet summary with source URLs.",
  skills: ["web-research"],
  tools: ["web_fetch"],
})
```

Pure analysis (no tools, no skills) — using the sub-agent purely for context isolation:

```js
delegate_task({
  prompt: "Given this raw text, extract a list of all dates mentioned with their context: ...",
  skills: [],
  tools: [],
})
```

Code review across many files:

```js
delegate_task({
  prompt: "Review the implementation of the cron scheduler and identify any race conditions or edge cases.",
  skills: ["code-review"],
  tools: ["read_file", "shell"],
})
```

## Dependencies

Internal — the AI SDK and its `generateText` function (already used by the main agent).

## Implementation notes

- The runner should reuse the agent's existing model client setup; just pass a different `system` and `messages`.
- Step limit: same as main agent for now. Tune later based on observed sub-agent behavior.
- Tool execution inside the sub-agent uses the same tool-loop semantics as the main agent.
- Cost is double-billed: each `delegate_task` invocation is a full LLM call (or many, if the sub-agent uses tools). Worth it when the alternative is loading 50KB of intermediate data into the main context for every subsequent turn.
