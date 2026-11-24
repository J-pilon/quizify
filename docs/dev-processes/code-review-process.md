# Code Review Process

## Purpose

Review the changes made for a ticket and report only problems that are real. Every finding in the report must be confirmed against the actual code and come with a concrete way it fails. A short list of verified findings beats a long list of guesses.

This process produces a **report, not code changes**. Nothing is fixed until the user picks which findings to address.

## When to Use

- Phase 7 of the `implement-ticket` skill, after a ticket's acceptance criteria are met.
- Before opening a pull request, if the changes weren't reviewed as part of a ticket.

## Step 1: Define the Scope

Review exactly the changes made for the ticket, nothing more.

1. **On a feature branch:** `git diff main...HEAD`.
2. **On `main`:** find the last commit before the ticket's first commit (`git log --oneline`), then run `git diff <that-commit>..HEAD`.
3. **Uncommitted changes:** if `git status` shows any, include `git diff HEAD` as well.

Write down the exact range you reviewed (e.g. "`38df301..HEAD` plus working tree"). It goes in the report.

Also read the ticket itself: its **Acceptance Criteria**, **Out of Scope**, and **Technical Decisions Required** sections, and its `DECISIONS.md` in `docs/implemented/`.

## Step 2: Run the Automated Review

Run one dedicated review pass over the diff from Step 1:

- **Claude Code:** `/code-review`.
- **Cursor:** the Bugbot review subagent, using the same diff.
- **Neither available:** review the diff by hand, file by file, using the checklist below.

Then go through the **Quizify checklist** yourself, whichever tool ran. Automated reviewers don't know this project's rules.

### Quizify checklist

**Architecture rules (from `AGENT.md`)**

- [ ] Network calls to OpenAI and Notion happen only in the background worker. No `fetch` to an external API in the popup, options page, quiz page, or content script, unless a recorded decision allows it.
- [ ] Nothing visible is added to the article page. The content script only reads.
- [ ] The quiz never runs in the toolbar popup.
- [ ] API keys and tokens never appear in logs, error messages, thrown errors, or messages sent between parts of the extension.
- [ ] The content script and background worker still build as single files with no `import` statements.

**Correctness and grounding**

- [ ] Every question shown to the user has passed quote validation against the extracted page text.
- [ ] Grading relies on the page (the `sourceQuote`), not the model's general knowledge.
- [ ] Changes to the `Quiz` shape keep `schemaVersion` and `updatedAt` correct, so data saved to Notion can still be loaded.

**Notion and storage**

- [ ] Rich text values are split at 2,000 characters, block appends are batched at 100, and requests go through the rate-limited queue.
- [ ] Storage keys match the existing ones (`settings`, `quiz:<id>`, `quizIndex:<normalizedUrl>`, `pending:<id>`).
- [ ] Saving the same quiz again updates the existing Notion row and never creates a duplicate.

**Scope and tests**

- [ ] Every acceptance criterion in the ticket is covered by a test or was checked by hand.
- [ ] Nothing from the ticket's **Out of Scope** section or `AGENT.md`'s "Explicitly Out of Scope" list was built.
- [ ] Tests follow `docs/conventions/testing-conventions.md`: factories for setup, test bodies focused on assertions.

## Step 3: Verify Every Finding Against the Code

Treat each candidate finding as a suspicion until it's confirmed.

For each one:

1. Open the file at the reported line and read the surrounding code, not just the diff hunk.
2. Trace the inputs: where does the data come from, and can it really reach this line in the problematic state?
3. Check whether something elsewhere already handles it, such as a guard, a validation step, a caller that never passes the bad value, or a test that covers it.
4. Write the **concrete failure scenario**: the specific state or input, what happens, and why it's wrong. For example: "A quiz with 15 short-answer questions → `buildQuizBlocks` returns 112 blocks → the create request sends all of them → Notion rejects it with `validation_error`."

**Drop the finding** if you can't write a concrete failure scenario, or if step 3 shows it's already handled. Keep count of what you dropped.

## Step 4: Check for Existing Precedent

Before keeping a finding, check whether the code is following an established choice:

- A decision recorded in `docs/implemented/*_DECISIONS.md`.
- A rule in `AGENT.md` or `ROADMAP.md`.
- The same pattern used deliberately elsewhere in the codebase.

If the code follows a recorded decision, drop the finding, or keep it only if the decision itself now looks wrong. In that case, say so and point to the decision.

## Step 5: Rank by Severity

| Severity | Meaning |
| --- | --- |
| **Critical** | Leaks a key or token, loses user data (quiz progress or Notion content), or breaks the core workflow for everyone. |
| **High** | An acceptance criterion isn't actually met, or a common path fails (e.g. saving a typical quiz errors out). |
| **Medium** | An edge case fails, an error is handled badly, or a project rule is broken without visible harm yet. |
| **Low** | Maintainability problems that are likely to cause bugs later. Style and formatting are excluded, because lint and Prettier handle them. |

Order the report from most to least severe.

## Step 6: Report

Use this format:

```markdown
## Code review: Ticket XXX — <name>

Reviewed: `<commit>..HEAD` (+ working tree)
Automated pass: <tool used, or "manual">
Findings: <N> verified, <M> dropped after verification

### 1. [High] `extension/src/lib/notion/quizzes.ts:84`
**Concern:** The page is created with every block in a single request.
**Failure scenario:** A 15-question quiz produces 112 blocks → `POST /v1/pages` → Notion rejects it with `validation_error` → the quiz can't be saved.
**Suggested fix:** Send the first 100 blocks with the create request and append the rest in batches.

### 2. [Medium] ...
```

- If no findings survive verification, say so explicitly: "No verified findings. <M> candidates were dropped after checking the code."
- Don't include dropped findings in the list. Only give their count.
- End by asking the user which findings, if any, they want fixed.

## Rules

- **Don't fix anything during the review.** Fix only the findings the user asks for, then re-run the test suite.
- **Never present a suspicion as a confirmed bug.** If something looks risky but you can't confirm it, either drop it or say clearly that it's unconfirmed and why.
- **Stay in scope.** Problems in code the ticket didn't touch go in a separate "Noticed outside this ticket" note, not in the findings.
- **If a finding means an acceptance criterion can't be met,** stop and follow the "Error Handling" section of the `implement-ticket` skill.
