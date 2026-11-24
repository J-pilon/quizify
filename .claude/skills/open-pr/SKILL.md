---
name: open-pr
description: Push the current branch and open a GitHub pull request with a standardized description (Summary/Changes/Concerns/Resources). Use when the user asks to open, create, or submit a PR. Strictly limited to gh pr create/view/edit — never runs gh pr merge or any other PR-mutating command.
---

# Open Pull Request

## Purpose

Publish the current branch and open a pull request using this project's standard PR description format. This skill is intentionally narrow: it creates and inspects PRs, and can touch up their metadata afterward — it never merges, closes, or otherwise finalizes one.

## Guardrail: Allowed Commands Only

This skill may run **only** the commands listed below. No other command — `gh` or otherwise — is permitted while operating under this skill, even if the user explicitly asks for it mid-workflow. If a request would require a command outside this list, stop and tell the user to run it themselves.

**Allowed:**
- `git status`, `git log`, `git diff` — read-only, to understand what's going into the PR
- `git push -u origin <branch>` — publish the current branch. Never pass `--force`/`-f`; if the push is rejected because the branch diverged, stop and ask the user rather than forcing it
- `gh pr create ...` — open the PR
- `gh pr view ...` — check a PR's status, URL, or metadata (without `--web` unless the user asks to open a browser)
- `gh pr edit ...` — update an already-opened PR's title, body, labels, or reviewers

**Forbidden — never run these under this skill:**
- `gh pr merge` — under no circumstances, no matter how the user phrases it ("merge it," "land it," "ship it," "go ahead"). Merging changes the base branch for everyone and isn't easily undone. If asked, tell the user this skill won't do it and that they should run it themselves.
- `gh pr close`, `gh pr reopen`, `gh pr ready`, `gh pr review`, or any `gh pr` subcommand not explicitly listed above as allowed.
- Any destructive git command (`push --force`, `reset --hard`, etc.) used to force a push through.

## Workflow

1. **Survey what's going into the PR**
   - `git status` — confirm nothing uncommitted is missing from the branch
   - `git log <base-branch>..HEAD --oneline` — list the commits the PR will contain
   - `git diff <base-branch>...HEAD --stat` — see the full file-level diff

2. **Draft the PR description**
   Invoke the `pr-description` skill to generate the description — it owns the project's standard Summary/Changes/Concerns/Resources format and the guidelines for each section. Do not redefine or duplicate that format here; this skill only consumes its output as the `--body` for `gh pr create` in step 4.

3. **Push the branch**
   ```bash
   git push -u origin <branch-name>
   ```
   If this fails because the remote diverged, stop and ask the user how to proceed. Do not force push.

4. **Create the PR**
   ```bash
   gh pr create --title "<concise title, under 70 chars>" --body "$(cat <<'EOF'
   <PR description from step 2>
   EOF
   )"
   ```
   Always pass the body through a heredoc so formatting survives exactly as drafted.

5. **Confirm and report**
   - Run `gh pr view` (no `--web`) to confirm creation and capture the PR URL.
   - Report the URL back to the user.
   - Take no further action on the PR — no merging, no auto-assigning reviewers — unless the user explicitly asks, and even then only through `gh pr edit`.

6. **Editing an existing PR**
   If the user later asks to change title, description, labels, or reviewers on a PR this skill opened:
   ```bash
   gh pr edit <number-or-url> --title "..." --body "..."
   ```
   Still never use this (or anything else) to merge, close, or mark the PR ready in a way that triggers merge-adjacent automation.

## Why these guardrails

Opening a PR, viewing one, or editing its metadata are all reversible and low-stakes — easy to undo or correct. Merging is not: it rewrites the base branch's history for everyone, can trigger CI/CD side effects, and isn't something that can be silently walked back. This skill is scoped to the reversible half of PR management. Merging stays a deliberate, separate action the user takes themselves, outside this skill.
