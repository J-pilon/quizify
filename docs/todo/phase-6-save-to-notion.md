# Phase 6 — Save to Notion

| | |
| --- | --- |
| Roadmap | [`ROADMAP.md` → Phase 6](../../ROADMAP.md) |
| Depends on | Phase 1 (Notion client, database IDs), Phase 4 (**Save quiz** button, autosave), Phase 5 (graded quiz) |
| Branch | `phase-6-save-to-notion` |

## Goal

**Save quiz** writes the page's Source and the quiz attempt to Notion, whether the quiz is finished or not. Saving the same quiz again updates the existing rows instead of creating duplicates.

## Context

- Notion has two databases, created in Phase 1:
  - **Sources:** one row per webpage, matched by normalized URL.
  - **Quizzes:** one row per attempt, linked to its Source. Retaking a page's quiz adds a new row.
- Each quiz's Notion page body contains readable question sections, plus a "Quizify data — do not edit" code block holding the full `Quiz` JSON. Phase 7 reads that block to resume a quiz.
- Answers on the Notion page are hidden inside collapsible toggles, so opening an unfinished quiz doesn't spoil them.
- The content hash is saved on both the Source and the Quiz.
- Use the `Notion-Version` pinned in Phase 1. Write rows to data source IDs (`sourcesDataSourceId` and `quizzesDataSourceId` in settings) if you're on version `2025-09-03` or later.
- Notion limits:
  - About 3 requests per second on average.
  - At most 100 blocks per create or append request.
  - At most 2,000 characters per rich text item.
  - At most 100 rich text items per block.

## What to implement

1. A request queue in the Notion client that serializes requests, retries on rate limits, and backs off.
2. Find-or-create for the Source, then updating it.
3. Create-or-update for the Quiz row.
4. Building the Quiz page body, including the JSON data block.
5. Status rules: *In progress* or *Completed*.
6. Wiring the **Save quiz** button, with a success notice and readable errors.

## How to implement

### Request queue (`lib/notion/client.ts`)

- Send requests one at a time, at least 350 ms apart.
- On HTTP 429, wait for the number of seconds in the `Retry-After` header (default 1 s), then retry. Give up after 3 attempts.
- On a 5xx error or network failure, retry twice with exponential backoff (500 ms, then 1500 ms).
- Never retry 400, 401, 403, or 404.

### Source (`lib/notion/sources.ts`)

`upsertSource(quiz) → sourcePageId`:

1. If `quiz.notion?.sourcePageId` is set, use it.
2. Otherwise, query Sources with a filter on the **URL** property equal to `quiz.source.normalizedUrl`. If a row matches, use it.
3. Otherwise, create a row with these properties:
   - **Name:** the page title.
   - **URL:** the normalized URL. It's both the backlink and the key used to match pages.
   - **Content hash.**
   - **Summary:** cut to 2,000 characters, ending with `…` if cut.
   - **Tags:** the quiz's tags.

   The page body is a "Summary" heading followed by the full summary as paragraphs. Split any paragraph longer than 2,000 characters across several rich text items.
4. **Updating an existing Source:** overwrite Summary and Content hash, and set Tags to the union of the existing and new tag names. Replace the body's summary: list the page's child blocks, delete them, and append the new summary blocks. This follows the roadmap's current plan; see "Decisions" below.

### Quiz row (`lib/notion/quizzes.ts`)

`upsertQuiz(quiz, sourcePageId) → { quizPageId, url }`:

| Property | Value |
| --- | --- |
| Name | `"<page title>: Quiz (<Mon D, YYYY>)"`, using `startedAt` |
| Source | relation to `[sourcePageId]` |
| Content hash | `quiz.source.contentHash` |
| Status | *Completed* if `quiz.status === "completed"`, otherwise *In progress* |
| Score | `quiz.score / 100`, because the percent format expects a fraction. Empty while in progress. |
| Started | `startedAt` as a date with time |
| Completed | `completedAt` as a date with time. Empty while in progress. |

- **Create:** if `quiz.notion?.quizPageId` isn't set, `POST /v1/pages` with the Quizzes data source as parent, the properties above, and up to the first 100 body blocks. Append the remaining blocks in batches of 100 with `PATCH /v1/blocks/{pageId}/children`.
- **Update:** if `quizPageId` is set, `PATCH /v1/pages/{id}` with the properties, then replace the whole body: list all child blocks (following pagination), delete each one, and append the new blocks in batches.
- If the stored `quizPageId` no longer exists (404 because the page was deleted), create a new page instead and replace the stored ID.

### Page body (`lib/notion/quiz-blocks.ts`)

Write this as a pure function, `buildQuizBlocks(quiz) → BlockObjectRequest[]`, so it can be tested.

1. A callout with the status and score, e.g. "Completed · 80%" or "In progress · 4 of 10 answered", plus a link to the article.
2. For each question:
   - `heading_3`: "Q3. <question>".
   - `bulleted_list_item` for each choice (multiple choice and true/false only).
   - `paragraph`: "Your answer: <value>", or "Skipped", or "Not answered yet". For graded quizzes, add the result, e.g. "— Correct".
   - `toggle` titled "Answer", containing:
     - `paragraph` "Correct answer: …"
     - `paragraph` "Feedback: …" (short answers only)
     - `paragraph` "Explanation: …"
     - `quote` with the `sourceQuote`
3. `divider`.
4. `heading_3`: "Quizify data — do not edit".
5. `code` block with `language: "json"`, containing `JSON.stringify(quiz)` split into rich text items of at most 2,000 characters each. Include the `notion` IDs in this JSON. If the JSON would need more than 100 rich text items (about 200,000 characters), split it across several consecutive code blocks. Phase 7 joins them back together.

Every rich text value longer than 2,000 characters must be split into multiple items. Put that in one helper, `toRichText(text)`.

### Saving (`background/save-quiz.ts`)

`SAVE_QUIZ { quizId }`:

1. Load `quiz:<quizId>` from storage.
2. `upsertSource`, then `upsertQuiz`.
3. Write `quiz.notion = { sourcePageId, quizPageId }` and a new `updatedAt` back to local storage. Then update the data block one more time so it includes those IDs. Alternatively, include the IDs before writing the body when you already know them; either way works, as long as the saved data block contains the IDs.
4. Return `{ ok: true, url }`, or `{ ok: false, error }`.

Ignore a second **Save quiz** click while a save is already running, and show the button as "Saving…".

### Errors

| Notion response | Message shown |
| --- | --- |
| 401 `unauthorized` | "Your Notion token isn't valid. Update it in Quizify settings." |
| 404 `object_not_found` on a database | "Quizify can't find its Notion databases. Make sure they're still shared with your integration, or run setup again." |
| 400 `validation_error` | "Notion rejected the data: <message>" (and log the details to the console, without the token) |
| Network failure or 5xx after retries | "Couldn't reach Notion. Your quiz is saved locally; try again." |

### UI

- On success, show a notice "Saved to Notion" with an **Open in Notion** link to the Quiz page, using the `url` field from Notion's response.
- Show the quiz's save status next to the button: "Not saved", "Saved 2 minutes ago", or "Unsaved changes" (when the local `updatedAt` is newer than the last save).
- After a successful save, Phase 4's warning on closing the page no longer appears until the quiz changes again.

## Decisions

- The roadmap lists "summary on retake" as an open question. This ticket follows the roadmap's current plan: overwrite the summary and hash, and merge tags. If that changes before implementation, update this ticket and Phase 7's changed-page note together.

## Acceptance criteria

- [ ] Clicking **Save quiz** on a quiz for a page that has never been saved creates one Sources row and one Quizzes row, linked to each other.
- [ ] The Sources row has the page title, the normalized URL (which opens the article), the content hash, a summary of at most 2,000 characters, and the tags. Its page body contains the full summary.
- [ ] Saving an unfinished quiz creates a Quizzes row with Status *In progress*, a Started date and time, no Completed date, and no Score.
- [ ] Saving from the results screen creates or updates the row with Status *Completed*, Started and Completed dates and times, and a Score that displays as the same percentage shown on the results screen.
- [ ] The Quizzes row has the content hash of the page the quiz was generated from.
- [ ] The quiz page body shows each question, its choices, and your answer, with the correct answer, explanation, and quote inside a collapsed **Answer** toggle.
- [ ] The quiz page body ends with a "Quizify data — do not edit" code block whose combined text parses as JSON equal to the local quiz, including the `notion` IDs.
- [ ] Saving the same quiz twice, or saving it partway through and again after finishing, updates the same Quizzes row. It never creates a second one.
- [ ] Generating and saving a new quiz for a page that already has a Source reuses that Source and adds a second Quizzes row linked to it.
- [ ] Saving a second quiz for the same page merges tags on the Source instead of replacing them.
- [ ] A quiz with 15 questions (more than 100 blocks) saves completely, with all questions and the data block present.
- [ ] Text longer than 2,000 characters (e.g. a long summary or a long short answer) saves without a Notion validation error.
- [ ] Bursts of requests never fail with HTTP 429. Rate-limited requests are retried after the `Retry-After` delay.
- [ ] An invalid token, unshared or deleted databases, and network failures each show the specific message listed above, and the quiz stays saved locally.
- [ ] After saving, an **Open in Notion** link opens the Quizzes row.
- [ ] Double-clicking **Save quiz** results in only one save.

## Test plan

- `quiz-blocks.test.ts`
  - Body structure for an in-progress quiz and for a completed quiz.
  - Toggles contain the answer key.
  - Skipped and unanswered wording.
  - The data block round-trips: joining its rich text and parsing it gives back the original quiz.
  - Quizzes over 200,000 characters of JSON are split across several code blocks.
- `rich-text.test.ts`: `toRichText` splits at 2,000 characters and leaves shorter strings alone.
- `sources.test.ts` (mocked client)
  - Uses the stored `sourcePageId` without querying.
  - Matches an existing Source by normalized URL.
  - Creates a Source when none exists.
  - Merges tags.
- `quizzes.test.ts` (mocked client)
  - Creates on the first save and updates on later saves.
  - Batches appends in groups of 100.
  - Writes Score as a fraction.
  - Maps status.
  - Recreates the page when the stored ID returns 404.
- `client-queue.test.ts` (fake timers)
  - Requests are spaced at least 350 ms apart.
  - A 429 waits for `Retry-After`.
  - 5xx errors back off, and 401 errors aren't retried.
- Add `makeSavedQuiz()` (a quiz with `notion` IDs) to the factories, and `makeNotionPage()` and `makeBlockList()` to `notion-factories.ts`.

## Out of scope

- Loading quizzes back from Notion to resume them (Phase 7).
- Flagging changed pages (Phase 7).
- Page versioning, where a changed hash creates a new Source version (in "Later").
- Optional extra fields such as time spent and model used (in "Later").
