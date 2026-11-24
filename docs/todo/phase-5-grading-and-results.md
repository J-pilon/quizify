# Phase 5 — Grading and results

| | |
| --- | --- |
| Roadmap | [`ROADMAP.md` → Phase 5](../../ROADMAP.md) |
| Depends on | Phase 3 (`AIProvider`, OpenAI adapter), Phase 4 (reducer `finished` state, quiz page) |
| Branch | `phase-5-grading-and-results` |

## Goal

When a quiz is finished, Quizify grades it and shows an end-of-quiz review: the score, then every question with your answer, the correct answer, an explanation, and the supporting quote from the page.

## Context

- Feedback appears only at the end, never after each question.
- Multiple-choice and true/false questions can be graded locally. Short answers need AI grading.
- Grading must be based on what the page says, not the model's general knowledge. Every question has a `sourceQuote` copied from the page, and short-answer grading uses it as the reference.
- Skipped and unanswered questions count as incorrect.
- Highlighting quotes on the article page is a "Later" feature. In v1, quotes are shown as text on the results screen.

## What to implement

1. Local grading for multiple-choice and true/false questions.
2. `gradeShortAnswers` in the OpenAI adapter, as one batched call.
3. Score calculation, with partial credit.
4. A grading step in the quiz flow after **Finish**, with a loading state and error handling.
5. The results screen.
6. Recording `completedAt`, `score`, and `status: "completed"`.

## How to implement

### Local grading (`lib/quiz/grading.ts`)

```ts
export function gradeLocally(question: Question, answer?: Answer): Answer["grade"] | "needs_ai";
```

| Question | Result |
| --- | --- |
| Skipped, or no answer | `incorrect` |
| Multiple choice or true/false | `correct` if `answer.value === question.correctAnswer`, otherwise `incorrect` |
| Short answer with text | `needs_ai` |

The comparison is exact because the value always comes from the question's own choices.

### AI grading (`lib/providers/openai.ts`)

```ts
interface ShortAnswerGradeInput {
  questionId: string;
  prompt: string;
  referenceAnswer: string;   // question.correctAnswer
  sourceQuote: string;
  userAnswer: string;
}

interface ShortAnswerGrade {
  questionId: string;
  grade: "correct" | "partial" | "incorrect";
  feedback: string;          // one sentence
}
```

- Send **one** Chat Completions request for all short answers in the quiz, using Structured Outputs with a schema for `{ grades: ShortAnswerGrade[] }`.
- Prompt rules:
  - Judge the user's answer only against the reference answer and the source quote, never outside knowledge.
  - Accept paraphrases and synonyms.
  - `partial` means the answer is on the right track but incomplete or partly wrong.
  - Ignore spelling and grammar unless they change the meaning.
  - Feedback is one sentence that tells the user what was missing or wrong. Don't repeat the whole reference answer.
- Check the response: there must be exactly one grade for each input `questionId`, with no extras. If a grade is missing or malformed, retry once. If it's still invalid, treat it as an error (below).

### Scoring

```ts
score = Math.round(100 * (correctCount + 0.5 * partialCount) / questions.length);
```

The score is an integer from 0 to 100, stored in `Quiz.score`. Notion conversion happens in Phase 6.

### Flow after Finish

1. The reducer reaches `finished` (Phase 4).
2. The quiz page enters a `grading` state ("Checking your answers…"). It grades locally and collects every `needs_ai` question.
3. If any questions need AI, the quiz page sends `GRADE_SHORT_ANSWERS { quizId, inputs }` to the background, which calls `provider.gradeShortAnswers`.
4. Add a reducer action, `APPLY_GRADES { grades, completedAt }`. It writes each answer's `grade` (and `feedback` for short answers), sets `score`, `status: "completed"`, `completedAt`, and `updatedAt`, then moves to the `results` phase.
5. **If AI grading fails:** show the error with **Retry grading**. The quiz stays `in_progress` with its answers locked. It's never marked completed with missing grades. **Save quiz** still works and saves it as *In progress*.
6. If there are no short answers, skip the network call and go straight to the results.
7. The quiz autosaves (Phase 4) after every step, so a reload during or after grading returns to the right state. A reload during grading restarts grading.

### Results screen (`quiz-page/Results.tsx`)

- **Score header:** "8 of 10 correct · 80%". Partial credit appears as "7 correct, 2 partial".
- **One card per question, in order:**
  - Question number, a grade badge (Correct, Partial, Incorrect, or Skipped), and the question.
  - **Your answer:** the answer text, or "Skipped".
  - **Correct answer:** the correct choice, or the reference answer for short answers.
  - **Feedback:** short answers only, from AI grading.
  - **Explanation.**
  - **From the page:** the `sourceQuote` in a blockquote. The card header links to the article.
- An **Incorrect only** filter toggle.
- **Save quiz** stays visible, as in Phase 4.

## Acceptance criteria

- [ ] After **Finish**, multiple-choice and true/false questions are graded without any network request.
- [ ] A selected answer that matches the correct answer is graded Correct; any other selected answer is graded Incorrect.
- [ ] Skipped and unanswered questions are graded Incorrect and shown as "Skipped" on the results screen.
- [ ] All short answers in a quiz are graded in a single API request.
- [ ] A paraphrased short answer that's correct according to the source quote is graded Correct.
- [ ] A short answer that's correct by general knowledge but contradicts the page is graded Incorrect or Partial, and its feedback points to what the page says.
- [ ] Each short answer shows one sentence of feedback.
- [ ] The score equals `round(100 × (correct + 0.5 × partial) / total)`. For example, 7 correct and 2 partial out of 10 scores 80.
- [ ] The results screen lists every question with the user's answer, the correct answer, the explanation, and the supporting quote from the page.
- [ ] The **Incorrect only** toggle hides questions graded Correct.
- [ ] After grading, the quiz has `status: "completed"`, a `completedAt` timestamp, and a `score`, and these survive a page reload.
- [ ] If AI grading fails, the quiz stays in progress, an error appears with **Retry grading**, and retrying can finish grading successfully.
- [ ] A quiz with no short-answer questions reaches the results screen without calling the API.

## Test plan

- `grading.test.ts`
  - Correct and incorrect choices.
  - Skipped and unanswered questions are incorrect.
  - Short answers return `needs_ai`.
  - Scoring works for all-correct, all-incorrect, mixed with partial credit, and rounding.
- `reducer.test.ts`
  - `APPLY_GRADES` sets grades, feedback, score, `completedAt`, and status, and moves to `results`.
  - Answers can't be changed after grading.
- `openai-grading.test.ts` (mocked `fetch`)
  - One request carries all inputs.
  - A response with a missing grade triggers exactly one retry, then an error.
  - Grades are matched to answers by `questionId`, not by position in the list.
- Component tests: the results screen shows the right badge and "Skipped" for each grade, and the filter toggle works.
- Add `makeShortAnswerQuestion()` and `makeGradedQuiz({ correct, partial, incorrect })` to `quiz-factories.ts`.

## Out of scope

- Saving the results to Notion (Phase 6).
- Highlighting quotes on the article page (in "Later").
- "Retake missed questions" and spaced repetition (in "Later").
