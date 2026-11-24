# Ticket 008 — Resume from Notion

Implements [`ROADMAP.md`](../../ROADMAP.md) Phase 7.

## Overview and Context

**Goal:** quizzes saved for later can be picked back up from the popup, either on the page they belong to or from a **Saved quizzes** list. When a page's content has changed since it was saved, Quizify says so.

**Context:**

- Unfinished quizzes are saved to Notion with Status *In progress* (Ticket 007).
- Each Quiz page in Notion ends with a "Quizify data — do not edit" code block containing the full `Quiz` JSON, including `schemaVersion`, `updatedAt`, and the `notion` IDs. It may be split across several consecutive code blocks.
- The same quiz may also exist in local storage at `quiz:<id>`, indexed by `quizIndex:<normalizedUrl>` (Ticket 005).
- The content hash, a SHA-256 of the page's cleaned text, is stored on both the Source and the Quiz. A different current hash means the page's material has changed.
- Completing a resumed quiz must update the same Notion row. Ticket 007 does this automatically when `quiz.notion.quizPageId` is set.
- Full page versioning is a "Later" feature. In v1, a changed page is only flagged.

## Dependencies

- `003_content_extraction_and_hashing.md` (extraction and hash)
- `005_quiz_page_ui.md` (quiz page, local autosave and index)
- `007_save_to_notion.md` (Notion rows and data block)

## Technical Requirements

1. A page lookup when the popup opens: the current page's Source, its in-progress quizzes, and whether its content has changed.
2. **Resume quiz** next to **Generate new quiz** in the popup.
3. A **Saved quizzes** list in the popup.
4. Loading a quiz from Notion's data block, with validation.
5. Choosing the newer copy between local storage and Notion.
6. A "page changed" notice in the popup and on resumed quizzes.

## Implementation Steps

- [ ] 1. Add `parseStoredQuiz` (see "Loading a quiz").
- [ ] 2. Add `lib/notion/load-quiz.ts` and the `LOAD_QUIZ` background message (see "Loading a quiz").
- [ ] 3. Change the quiz page to call `LOAD_QUIZ` when it opens (see "Loading a quiz").
- [ ] 4. Add `background/lookup-page.ts` and the `LOOKUP_PAGE` message (see "Page lookup").
- [ ] 5. Add **Resume quiz** and the changed-page notice to the popup (see "Popup").
- [ ] 6. Add the **Saved quizzes** list view to the popup (see "Saved quizzes list").
- [ ] 7. Add the changed-page notice to the quiz page (see "Changed-page notice on the quiz page").
- [ ] 8. Check completing a resumed quiz end to end (see "Completing a resumed quiz").

### Page lookup

`LOOKUP_PAGE { tabId }`, sent when the popup opens:

1. Run `EXTRACT_PAGE` (Ticket 003) to get `normalizedUrl` and `contentHash`. If extraction fails, still show the saved quizzes list, but hide the page-specific parts.
2. **Local check:** read `quizIndex:<normalizedUrl>` and load those quizzes. Keep the ones with `status: "in_progress"`.
3. **Notion check:** query Sources where URL equals `normalizedUrl`. If a row is found, query Quizzes where Source contains that page ID and Status is *In progress*, sorted by Started, newest first.
4. Merge the two lists by quiz ID. A quiz that exists only locally (never saved) still appears.
5. Return:

```ts
{
  extracted: boolean;
  source?: { pageId: string; savedHash: string };
  contentChanged: boolean;        // source exists && savedHash !== current contentHash
  inProgress: { quizId: string; quizPageId?: string; title: string; startedAt: string;
                answeredCount?: number; savedToNotion: boolean }[];
}
```

Show the popup immediately and fill in Notion results when they arrive, with a small loading indicator. A slow Notion response must not block **Generate quiz**.

### Popup

- With in-progress quizzes for this page: a **Resume quiz** button for the newest one, a "+N more" control when there are several, and **Generate new quiz**.
- With no in-progress quizzes: **Generate quiz**, as before.
- When `contentChanged` is true: a notice saying "This page has changed since you last saved it. Quizzes may not match the current content."
- A **Saved quizzes** link opens the list view inside the popup.

### Saved quizzes list

- Query Quizzes where Status is *In progress*, sorted by Started, newest first, 25 at a time with **Load more** (use Notion's pagination cursor). Also include local-only in-progress quizzes from storage.
- Each item shows the quiz name (which contains the page title) and its start date.
- Selecting an item:
  1. Loads the quiz (below) to get `source.url`.
  2. Opens the article in a new tab. If the active tab already shows the same normalized URL, uses it instead.
  3. Opens the quiz page with `openQuizPage` (Ticket 005, which respects the tab or window setting), using `quiz.html?quizId=<id>`.

### Loading a quiz

`LOAD_QUIZ { quizId, quizPageId? }`, used when the quiz page opens and when choosing from the list:

1. Read the local copy from `quiz:<quizId>`, if it exists.
2. If `quizPageId` is known (from the local copy's `notion` field or from the list), fetch the Notion copy:
   - List the page's child blocks with `GET /v1/blocks/{id}/children`, following `next_cursor` until there are no more pages.
   - Find the "Quizify data — do not edit" heading and join the `plain_text` of every consecutive `code` block after it.
   - Parse it as JSON.
3. **Check the parsed data** with `parseStoredQuiz(json): Quiz | Error`:
   - `schemaVersion` must be `1`.
   - Required fields must exist with the right types.
   - The question and answer IDs must match up.
   - On failure, show "This quiz's saved data in Notion is damaged or from a newer version of Quizify," and offer to open the Notion page.
4. **Choose a copy:** use whichever has the later `updatedAt`. If only one exists, use that one. Save the chosen copy to local storage (and add it to `quizIndex`) so the quiz page can work from local storage as usual.
5. If Notion can't be reached but a local copy exists, use the local copy and show a small "Using your local copy; Notion couldn't be reached" notice.

The Ticket 005 quiz page currently loads only from local storage. Change it to call `LOAD_QUIZ` first, so opening `quiz.html?quizId=<id>` works on any machine that has the Notion page.

### Changed-page notice on the quiz page

When a resumed quiz opens, compare `quiz.source.contentHash` with the article's current hash:

- If the article tab is open, get the current hash from `EXTRACT_PAGE` on that tab.
- Otherwise, skip the check (don't open the page just to hash it).

If the hashes differ, show a dismissible notice: "This page has changed since this quiz was created. Some questions may no longer match." The quiz keeps working normally.

### Completing a resumed quiz

No new Notion code is needed. The loaded quiz has `notion.quizPageId`, so Ticket 007's `upsertQuiz` updates the same row and sets it to *Completed*. The work in this step is to check this end to end.

## Technical Decisions Required

1. **Show answered counts for Notion-only quizzes in the popup?** Suggested: no. Getting the count means downloading every quiz's data block, which slows the popup. Show the start date only.
2. **Opening the article when resuming from the list.** Suggested: reuse the active tab if it already shows the same normalized URL, otherwise open a new tab. Finding the article in any other open tab would need the `tabs` permission, which Quizify doesn't otherwise need.

Record all decisions in `docs/implemented/008_DECISIONS.md`.

## Acceptance Criteria

- [ ] Opening the popup on a page with an in-progress quiz shows **Resume quiz** next to **Generate new quiz**.
- [ ] Opening the popup on a page with no in-progress quizzes shows only **Generate quiz**.
- [ ] Clicking **Resume quiz** opens the quiz page (as a tab or window, depending on the setting) at the question where the user stopped, with every earlier answer and skipped mark restored.
- [ ] A quiz saved to Notion can be resumed after the local copy is deleted (e.g. after reinstalling the extension or clearing its storage). Its answers come back from Notion's data block.
- [ ] When both copies exist, the one with the later `updatedAt` is used.
- [ ] A quiz that was never saved to Notion but is in progress locally appears in the popup and can be resumed.
- [ ] **Saved quizzes** lists every in-progress quiz from Notion and local storage, newest first. **Load more** fetches the next 25.
- [ ] Selecting a saved quiz opens its article and its quiz page.
- [ ] Finishing a resumed quiz and clicking **Save quiz** changes the same Notion row to *Completed* with a score and completion time. No second row is created.
- [ ] If the current page's hash differs from the saved Source's hash, the popup shows the "This page has changed" notice.
- [ ] If a resumed quiz's hash differs from the open article's current hash, the quiz page shows the "page has changed" notice and can still be completed.
- [ ] A missing, damaged, or newer-version data block shows a clear error with a link to the Notion page. It never shows a broken quiz.
- [ ] If Notion is unreachable, quizzes stored locally can still be resumed, and the popup still offers **Generate quiz**.
- [ ] A slow Notion lookup doesn't delay showing **Generate quiz** in the popup.

## Test Plan

- `parse-stored-quiz.test.ts`
  - Accepts a valid quiz.
  - Rejects a wrong `schemaVersion`, missing fields, and answers whose IDs don't match any question.
- `load-quiz.test.ts` (mocked client and storage)
  - Joins split code blocks.
  - Follows pagination across several block pages.
  - Picks the copy with the later `updatedAt`.
  - Falls back to the local copy when Notion fails.
  - Errors when there's no data block.
- `lookup-page.test.ts`
  - Merges local and Notion in-progress quizzes without duplicates.
  - `contentChanged` is true only when the Source exists and its hash differs.
  - Still returns saved quizzes when extraction fails.
- An end-to-end test with fakes: save partway through, clear local storage, load from Notion, finish, save again. The fake client must record exactly one Quizzes row, with Status *Completed*.
- Reuse `makeSavedQuiz()` and `makeBlockList()`. Add `makeDataBlocks(quiz, { split })`.

## Out of Scope

- Page versioning, where a changed hash creates a new Source version and keeps history (in the roadmap's "Later").
- Showing completed quizzes, history, or statistics in the extension.
- Highlighting quotes on the article (in the roadmap's "Later").
