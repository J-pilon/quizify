# Phase 4 — Quiz page UI

| | |
| --- | --- |
| Roadmap | [`ROADMAP.md` → Phase 4](../../ROADMAP.md) |
| Depends on | Phase 1 (`quizLocation` setting), Phase 3 (generated `Quiz` stored at `quiz:<id>`) |
| Branch | `phase-4-quiz-page-ui` |

## Goal

The one-question-at-a-time quiz experience from the core workflow. The quiz page shows "Question N of M", lets the user answer, **Skip**, go **Previous**, **Finish**, or **Save quiz**, opens in a tab or a window depending on the setting, and keeps progress through reloads and closing the tab.

## Context

- The quiz runs on its own extension page, `quiz.html?quizId=<id>`, never in the popup or the article.
- Users can skip questions, go back, and save for later. Unfinished quizzes are saved as *In progress* (the Notion save itself is Phase 6).
- Correct answers aren't revealed until the end (Phase 5).
- The `quizLocation` setting is `"tab"` (default) or `"window"`.
- A generated `Quiz` is already stored at `storage.local` key `quiz:<quizId>` (Phase 3). It includes `schemaVersion` and `updatedAt`.

## What to implement

1. Open the quiz page according to the `quizLocation` setting.
2. A quiz state reducer.
3. The quiz UI: header, progress bar, a view for each question type, and controls.
4. Skip, revisit, and finish behavior, including the confirmation step.
5. Autosave to `storage.local`, plus an index by normalized URL.
6. Keyboard shortcuts.
7. Light and dark appearance.

## How to implement

### Opening the quiz (`background/open-quiz.ts`)

Replace the Phase 3 `tabs.create` call with `openQuizPage(quizId, articleTab, settings.quizLocation)`:

- **Tab:** `browser.tabs.create({ url, index: articleTab.index + 1, windowId: articleTab.windowId })`, which puts the quiz right next to the article.
- **Window:** `browser.windows.get(articleTab.windowId)` to find the article window's position and size, then `browser.windows.create({ url, type: "popup", width: 460, height: articleWindow.height, left: articleWindow.left + articleWindow.width - 460, top: articleWindow.top })`.
  - Check in Safari whether `type: "popup"` gives a compact window and whether position and size are respected.
  - If Safari ignores `type: "popup"` or the position, fall back to a normal window (`type: "normal"`) with the same size.
  - Record what Safari actually does in `docs/decisions/phase-4_DECISIONS.md`.

### Reducer (`lib/quiz/reducer.ts`)

The quiz state is the `Quiz` object plus a little UI-only state (`phase: "answering" | "confirm_finish" | "finished"`). The reducer must be pure: no storage or messaging inside it.

| Action | Effect |
| --- | --- |
| `ANSWER { questionId, value }` | Sets the answer's `value`. Sets `skipped: false`. |
| `NEXT` | Moves `currentIndex` forward by one. Only allowed if the current question is answered. |
| `PREVIOUS` | Moves `currentIndex` back by one, never below 0. |
| `SKIP` | Marks the current question `skipped: true`, clears its value, and moves forward. On the last question, it marks it skipped and stays. |
| `GO_TO { index }` | Jumps to any question (used by the progress bar). |
| `REQUEST_FINISH` | If every question is answered, goes to `finished`. Otherwise goes to `confirm_finish`. |
| `CANCEL_FINISH` | Goes back to `answering`. |
| `CONFIRM_FINISH` | Goes to `finished`. Unanswered questions are recorded as `skipped: true`. |

- Every action that changes the quiz sets `updatedAt` to the current time. Pass the clock in, so tests can control it.
- Once the phase is `finished`, the reducer ignores answering and navigation actions. Phase 5 adds grading and the results view on top of `finished`.
- For short answers, an answer only counts as answered if it isn't just whitespace.

### UI (`quiz-page/`)

- **Header:** the article's title, linked to its original URL (opens in a new tab), and "Question N of M".
- **Progress bar:** one segment per question, styled as answered, skipped, unanswered, or current. Clicking a segment jumps to that question.
- **Question views:**
  - Multiple choice: radio buttons, one per choice.
  - True/false: two large buttons, **True** and **False**.
  - Short answer: a text area that grows as you type.
- **Controls:**
  - **Previous** is disabled on the first question.
  - **Skip** is always enabled.
  - **Next** is disabled until the current question is answered. On the last question it's replaced by **Finish**.
  - **Finish** is always available from the last question. It's also available from any question once every question is answered or skipped.
  - **Save quiz** is visible at all times. In this phase it calls an `onSave` handler passed in as a prop; Phase 6 connects it to Notion. Until then, it shows a "Saving to Notion arrives in Phase 6" note.
- **Finish confirmation:** a dialog that says "You have 2 skipped and 1 unanswered question. They'll count as incorrect." with **Go back** and **Finish anyway**.
- **Finished state:** a placeholder ("Quiz finished — results coming soon"). Phase 5 replaces it with the results screen.
- **Unsaved-changes warning:** when the window is about to close, warn only if the quiz has changes that haven't been saved to Notion. Browsers decide the wording of that warning.

### Autosave (`lib/quiz/storage.ts`)

- After every state change, save the quiz to `quiz:<quizId>` in `storage.local`, at most once every 300 ms (debounced), plus a final save when the page closes (`pagehide`).
- Keep an index at `quizIndex:<normalizedUrl>`: an array of quiz IDs for that page, newest first. Add the quiz ID when the quiz is first saved. Phase 7 uses this index to offer **Resume** from local data.
- When the page loads, read `quiz:<quizId>`. If it doesn't exist, show "This quiz wasn't found" with a link back to the article.

### Keyboard shortcuts

- `1`–`4`: pick a multiple-choice answer. `1`/`2`: pick True or False.
- `←`: Previous. `→`: Next (only when answered). `S`: Skip. `Enter`: Next, or Finish on the last question.
- Ignore all shortcuts while focus is in the short-answer text area (except `Escape`, which removes focus from it).

### Appearance

- Use `color-scheme: light dark`, and CSS custom properties with `@media (prefers-color-scheme: dark)` overrides.
- The layout must work in both a full tab and a 460 px-wide window.
- Everything should be reachable by keyboard: visible focus outlines, labels on radio buttons, and live announcements for screen readers when the question changes.

## Acceptance criteria

- [ ] With quiz location set to **New tab**, the quiz opens in a tab directly to the right of the article tab.
- [ ] With quiz location set to **Separate window**, the quiz opens in its own window placed beside the article window, or in a normal window if Safari doesn't support the compact popup-style one. Safari's actual behavior is recorded in the decisions file.
- [ ] The header shows the article title (linking to the article) and "Question N of M", with M equal to the number of questions.
- [ ] Only one question is visible at a time.
- [ ] Each question type renders with the right controls: radio buttons for multiple choice, True/False buttons, and a text area for short answer.
- [ ] **Next** stays disabled until the current question is answered. A short answer containing only spaces doesn't count.
- [ ] **Skip** marks the question as skipped in the progress bar and moves to the next question.
- [ ] Going back to a skipped question and answering it clears its skipped mark.
- [ ] **Previous** is disabled on the first question, and returns to the previous question with its answer still filled in.
- [ ] Clicking a progress bar segment jumps to that question.
- [ ] Finishing with skipped or unanswered questions opens a confirmation dialog that states how many there are. **Go back** returns to the quiz; **Finish anyway** finishes it.
- [ ] Finishing with every question answered doesn't show the confirmation.
- [ ] After finishing, answers can't be changed.
- [ ] **Save quiz** is visible on every question and after finishing.
- [ ] Reloading the quiz tab restores the same question, answers, and skipped marks.
- [ ] Closing the quiz tab and reopening `quiz.html?quizId=<id>` restores the same progress.
- [ ] The keyboard shortcuts work as listed, and don't fire while typing a short answer.
- [ ] The quiz page follows the system's light or dark mode and stays usable at 460 px wide.
- [ ] Opening the page with an unknown quiz ID shows "This quiz wasn't found".

## Test plan

- `reducer.test.ts` (with a fixed clock)
  - `ANSWER` sets the value and clears `skipped`.
  - `NEXT` is blocked when the current question is unanswered or whitespace-only.
  - `PREVIOUS` never goes below 0.
  - `SKIP` marks the question and moves forward, and stays put on the last question.
  - `GO_TO` jumps to the given question.
  - `REQUEST_FINISH` goes straight to `finished` when everything is answered, and to `confirm_finish` otherwise.
  - `CONFIRM_FINISH` records unanswered questions as skipped.
  - Actions are ignored after finishing.
  - `updatedAt` changes on every action that modifies the quiz.
- `storage.test.ts` (with an in-memory `storage.local` fake)
  - Saving writes `quiz:<id>` and adds the ID to `quizIndex:<normalizedUrl>` exactly once.
  - Loading returns null for unknown IDs.
- `open-quiz.test.ts` (with a mocked `browser.tabs` and `browser.windows`)
  - The tab setting creates a tab at `index + 1`.
  - The window setting computes the size and position from the article window.
- Component tests (React Testing Library) for the controls' enabled and disabled states and the confirmation dialog.
- Use `makeQuiz` and `makeAnswer`, and add `makeQuizState({ phase })`.

## Out of scope

- Grading and the results screen (Phase 5).
- Saving to Notion (Phase 6).
- Resuming from Notion (Phase 7).
- Highlighting quotes on the article (in "Later").
