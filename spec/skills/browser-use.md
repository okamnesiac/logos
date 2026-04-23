---
name: browser-use
description: Drive a real browser via browser-harness when web_fetch isn't enough — auth, clicks, forms, screenshots, signed-in pages.
---

# Browser use

Two ways to read the web. Pick the cheap one first.

1. **`web_fetch` tool** — anonymous reads of public-ish pages. Articles, GitHub HTML, docs, blog posts. No setup. JS-rendered sites work via the `jina` or `playwright` backends. **Try this first.**
2. **`browser-harness` CLI via `shell`** — a real, attached Chrome session with the owner's logins. Use only when `web_fetch` can't do the job: signed-in pages, multi-step forms, clicks, file uploads, screenshots, anything CDP-only.

If you're not sure which you need, try `web_fetch` first. If it returns junk, an empty body, a login wall, or a captcha, escalate.

## Privacy

`browser-harness` drives the owner's **real, logged-in Chrome**. Anything you do happens inside their authenticated sessions — banking, email, work tools, all of it. Treat its output the way you'd treat the owner's own browsing: don't paste page contents into third-party services, don't log full screenshots, and prefer `delegate_task` for multi-step workflows so the raw context stays out of the main thread.

For sub-agent or stealth work where the owner's primary Chrome is the wrong tool, browser-harness supports the cloud browsers from `cloud.browser-use.com`. See the upstream README before reaching for that.

## Setup (one-time)

Check whether browser-harness is already installed before doing anything:

```bash
which browser-harness && browser-harness --doctor
```

If both succeed, you're done — skip to **Usage**.

If `browser-harness` is missing, walk the owner through installation. Confirm before you start:

```bash
git clone https://github.com/browser-use/browser-harness vendor/browser-harness
uv tool install -e ./vendor/browser-harness
```

If `uv` isn't installed, point the owner at https://docs.astral.sh/uv/ — don't try to install it for them, package managers are the owner's call.

Then walk them through enabling Chrome remote debugging:

> Open `chrome://inspect/#remote-debugging` in Chrome and tick the "Discover network targets" checkbox so browser-harness can attach. Leave Chrome running.

Verify with `browser-harness --doctor`. If it can't reach Chrome, tell the owner what the error said rather than retrying blindly.

## Usage

Pipe Python to the CLI via heredoc. Helpers (`new_tab`, `wait_for_load`, `page_info`, `screenshot`, `click`, `type_text`, …) are preloaded — call them directly, don't import.

```bash
browser-harness <<'PY'
new_tab("https://news.ycombinator.com")
wait_for_load()
print(page_info())
PY
```

The full helper surface lives in `vendor/browser-harness/helpers.py`. Read it when you need to know what's available.

### Heavy or multi-step work → delegate

Page contents and screenshots blow up context fast. For research, scraping, or any multi-step browser task, use `delegate_task` with `tools: ["shell"]` and a skill list that includes `browser-use`. The raw output stays in the sub-agent's context; only the summary comes back.

## Extending helpers.py

If a needed helper is missing (the upstream design assumes this happens often — that's the point of browser-harness), follow the **`coding` skill** with the working directory set to `vendor/browser-harness/` and a prompt like:

> Add a function to `helpers.py` that does X. Keep it small and consistent with the existing helpers. Commit on a local `protos-local` branch so upstream pulls rebase cleanly.

Then re-run your task. Because browser-harness was installed with `uv tool install -e`, the new helper is live immediately — no reinstall.

Don't hand-edit `helpers.py` from your shell tool. Delegate to the coding agent so the change is reviewed, committed, and cleanly mergeable against upstream.

## Site-specific patterns

`vendor/browser-harness/domain-skills/` contains community-contributed patterns for specific sites (github, linkedin, amazon, …). When working on one of those sites, read the relevant `domain-skills/{site}/` first — it'll save selector hunting.

When you discover a non-obvious pattern yourself (a private API, a stable selector, a needed wait), file it under `vendor/browser-harness/domain-skills/{site}/` via the `coding` skill. Per upstream convention, don't hand-author these — let the coding agent generate them from what actually worked in the browser.

## When to refuse

- The owner asked for something that would require their auth on a service they haven't used in this thread or in memory. Confirm first — "I'd be using your logged-in {service} session for this; OK?"
- Any action that spends money, sends a message, or makes a public post. Confirm.
- Anything resembling scraping at scale, automated account creation, or evading rate limits / bot detection. Refuse and explain.

## Disabling

There's no env-var kill switch. To disable: don't install browser-harness (or `uv tool uninstall browser-harness`). Once it's not on `$PATH`, the `shell` invocation will fail with a clear error and you'll fall back to `web_fetch`.
