# Test

For the `test` command — generate or refresh tests in `agent/test/` that verify the implementation matches the spec, then run them.

## Hard rule: only `test` touches `agent/test/`

`bootstrap` and `update` never read, write, run, or even mention tests — even if tests already exist. The user explicitly invokes `test` after `update` (or whenever else) to refresh the suite. This keeps the cost of tests visible: someone who tried `test` once and moved on doesn't pay for test maintenance on every spec sync.

## Framework

Use [vitest](https://vitest.dev). Add as a devDependency to `agent/package.json` and a `test` script (`vitest run`). If the user prefers a different framework, ask before installing.

## Test layout

- Tests live in `agent/test/`, organized by spec topic (e.g. `agent/test/skills.test.ts`, `agent/test/render-filter.test.ts`, `agent/test/tools.test.ts`).
- Each test or describe block leads with a one-line comment linking to the spec section it covers, so a failure traces back to intent (e.g. `// architecture.md → Tool return shapes`).
- Use temporary directories for filesystem fixtures — never write into the live `spec/`, `config/`, `memory/`, or `runtime/`.

## What to test

Spec-defined invariants and contracts — things the spec explicitly promises about behavior. Examples:

- **Tool return shapes** — succeed-or-fail vs. succeed-or-miss conventions; never bare `null` from a lookup tool (architecture.md → Tool return shapes).
- **Tool call / result pairing** — every assistant turn that produces a `tool_call` appends a corresponding `tool_result` event, even when the tool throws (architecture.md → Tools).
- **Skill loader** — spec scans `*.md` only; config scans both `*.md` and `*/SKILL.md`; flat wins within config; merge across roots (config frontmatter wins per field, body appended).
- **Memory link resolution** — shortest-path tiebreak; aliases; `add_memory` / `rename_memory` preserve resolutions when changes would otherwise shadow or un-shadow links (architecture.md → Memory format).
- **Render filter** — last non-empty assistant text per `turn_id`; intermediate tool events skipped; `NO_REPLY` skipped (architecture.md → Cursor-based replay).
- **Channel `send()` contract** — `text.trim() === "NO_REPLY"` treated as a lifecycle marker, not a message; substring matches don't count (architecture.md → Channel send() contract).
- **Cron layering** — frontmatter override + body append; same merge rules as skills.
- **Sub-agent runner constraints** — `delegate_task` always stripped from sub-agent's tool allowlist; missing skill returns `{ ok: false }`, doesn't throw.

## What NOT to test

- **Channel platform behavior** (real Telegram/Slack/Discord interactions) — coupled to external services; manual verification only.
- **LLM output** — non-deterministic.
- **Implementation details the spec doesn't constrain** — private function signatures, internal data shapes, refactor-sensitive structure.

## Running

`npm test` (which runs `vitest run`). The `test` invocation runs the suite after writing or updating tests, and reports the result back to the user. A failing test means either the implementation drifted from the spec or the spec changed without the test catching up — flag both possibilities to the user; don't auto-fix.

## Run logs

Tee the full test runner output to `runtime/tests/{ISO-timestamp}.log` (e.g. `runtime/tests/2026-04-23T18:02:49Z.log`) so the user can dig into a failure after the fact without re-running. Same shape as cron logs under `runtime/logs/cron/` — append-only, never trimmed, useful for the agent itself to read back via `read_file` if asked to investigate a past failure. Show the last few lines of the log path in the chat reply so the user knows where to look.
