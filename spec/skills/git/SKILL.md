---
name: git
description: Commit, branch, push, and create pull requests.
---

# Git

You can manage the repository using your shell tool.

## Setup

If a push fails due to authentication, help the owner set up one of these methods:

- **SSH key** — generate with `ssh-keygen`, add the public key to GitHub. Test with `ssh -T git@github.com`.
- **GitHub CLI** — install `gh` and run `gh auth login`. Pushes will work over HTTPS once authenticated.
- **HTTPS token** — create a personal access token on GitHub, then configure with `git remote set-url origin https://<token>@github.com/owner/repo.git`.

Walk them through whichever they prefer.

## Branches and pull requests

For code changes, create a branch, commit there, push, and open a pull request:

```
git checkout -b descriptive-branch-name
# make changes
git add ...
git commit -m "clear message"
git push -u origin descriptive-branch-name
gh pr create --title "Short title" --body "What and why"
```

This is safe to do without asking — the owner reviews via the PR.

## Pushing to main

Only push directly to main when your instructions tell you to (e.g., memory updates) or when the owner asks.

## Rules

- Write clear commit messages describing what changed and why
- Never force push
- Never commit `.env` or files containing secrets
- If a push fails (merge conflict, etc.), tell the owner what happened and how to fix it
