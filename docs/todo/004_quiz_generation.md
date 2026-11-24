# Ticket 004 — Quiz generation

Implements [`ROADMAP.md`](../../ROADMAP.md) Phase 3.

## Overview and Context

**Goal:** clicking **Generate quiz** produces a valid, grounded quiz, a summary, and topic tags for the current page, and shows them on the quiz page. Every question is backed by a word-for-word quote that is checked against the page's text.

**Context:**

- AI providers are pluggable through an `AIProvider` interface. OpenAI is the only adapter in v1.
- The question types are multiple choice, true/false, and short answer. The user picks count (5/10/15), difficulty, and types per quiz, prefilled from settings.
- **Grounding rule:** every question must be answerable from the page alone, and its `sourceQuote` must appear in the extracted text. A question that fails this check is never shown to the user.
- The toolbar popup closes as soon as it loses focus, so generation can't run inside it. The popup starts the flow, the background worker does the work, and the quiz page shows progress.
- All network calls go through the background worker.

## Dependencies

- `002_settings_and_connections.md` (OpenAI key, model, quiz defaults, `isConfigured`)
- `003_content_extraction_and_hashing.md` (`EXTRACT_PAGE`)

## Technical Requirements

1. The `AIProvider` interface and the quiz types in `lib/quiz/types.ts`.
2. An OpenAI adapter using Structured Outputs.
3. The generation prompt and JSON schema.
4. Output validation, including checking each quote against the page, with one retry.
5. Per-quiz options in the popup.
6. The end-to-end flow: popup → background → quiz page, with a loading state and Cancel.
7. A basic read-only display of the generated quiz on the quiz page (Ticket 005 replaces it with the real quiz UI).

## Implementation Steps

- [ ] 1. Add `lib/quiz/types.ts` (see "Types").
- [ ] 2. Add the provider interface and `getProvider` (see "Provider interface").
- [ ] 3. Add `lib/quiz/schema.ts` and `lib/quiz/prompt.ts` (see "JSON schema" and "Prompt").
- [ ] 4. Add `lib/quiz/validate.ts`, including `quoteAppearsInText` (see "Validation").
- [ ] 5. Add the OpenAI adapter, and move `TEST_OPENAI` into it (see "OpenAI adapter").
- [ ] 6. Add the generation function with the retry policy (see "Retry policy").
- [ ] 7. Add the per-quiz options to the popup (see "Popup options").
- [ ] 8. Wire the end-to-end flow (see "Flow").
- [ ] 9. Replace the Ticket 003 debug view with the basic quiz display (see "Basic quiz display").

### Types

Use the data model from `ROADMAP.md`, with two added fields:

- `schemaVersion: 1` on `Quiz`, so data saved to Notion can be checked when it's loaded back (Ticket 008).
- `updatedAt: string` (ISO timestamp) on `Quiz`, used to decide whether the local copy or the Notion copy is newer (Ticket 008).

### Provider interface

`lib/providers/types.ts`:

```ts
export interface GenerateQuizInput {
  title: string;
  text: string;                       // capped article text from EXTRACT_PAGE
  count: number;
  difficulty: "easy" | "medium" | "hard";
  types: QuestionType[];
}

export interface GeneratedQuiz {
  summary: string;
  tags: string[];
  questions: Omit<Question, "id">[];
}

export interface AIProvider {
  testConnection(signal?: AbortSignal): Promise<void>;
  generateQuiz(input: GenerateQuizInput, signal?: AbortSignal): Promise<GeneratedQuiz>;
  gradeShortAnswers(input: ShortAnswerGradeInput[], signal?: AbortSignal): Promise<ShortAnswerGrade[]>;
}
```

- In this ticket, `gradeShortAnswers` can throw "not implemented"; Ticket 006 implements it.
- `getProvider(settings)` returns the adapter for `settings.ai.provider`. Nothing outside `lib/providers/` imports `openai.ts` directly.

### OpenAI adapter

`lib/providers/openai.ts`:

- Call `fetch` directly (no SDK).
- Request Structured Outputs with a strict JSON schema.
- If the model refuses, raise an error that says so.
- Map HTTP errors to readable messages: 401 (invalid key), 429 (rate limit or quota), 5xx (OpenAI is having problems).
- Move `testConnection` here from the Ticket 002 background handler, so all OpenAI calls live in the adapter.
- Set `DEFAULT_MODEL` to the chosen default (see "Technical Decisions Required").

### JSON schema

Strict mode requires every property to be listed in `required` and `additionalProperties: false`. Represent "no choices" as an empty array instead of leaving the field out:

```ts
{
  type: "object", additionalProperties: false,
  required: ["summary", "tags", "questions"],
  properties: {
    summary: { type: "string" },
    tags: { type: "array", items: { type: "string" } },
    questions: {
      type: "array",
      items: {
        type: "object", additionalProperties: false,
        required: ["type", "prompt", "choices", "correctAnswer", "explanation", "sourceQuote"],
        properties: {
          type: { type: "string", enum: ["multiple_choice", "true_false", "short_answer"] },
          prompt: { type: "string" },
          choices: { type: "array", items: { type: "string" } },
          correctAnswer: { type: "string" },
          explanation: { type: "string" },
          sourceQuote: { type: "string" }
        }
      }
    }
  }
}
```

### Prompt

System message rules:

- Use only facts stated in the article. Never use outside knowledge.
- Generate exactly `count` questions, using only the allowed `types` and spreading them across the types.
- Every `sourceQuote` must be copied word for word from the article: one or two consecutive sentences that support the correct answer.
- Multiple choice: four choices, exactly one correct; `correctAnswer` must equal one choice exactly; distractors must be plausible.
- True/false: `choices` must be exactly `["True", "False"]`, and `correctAnswer` must be one of them.
- Short answer: `choices` is `[]`, and `correctAnswer` is a concise model answer.
- Difficulty: *easy* covers key facts and definitions; *medium* covers relationships and reasons; *hard* covers applying, comparing, or reasoning about the material.
- No questions about the website itself, the author, or the publication date unless they're central to the article.
- `summary`: three to five sentences. `tags`: three to six short lowercase topic tags.

The user message contains the article title and text, wrapped in clear delimiters.

### Validation

`validateGeneratedQuiz(generated, input, articleText)` returns `{ valid: Question[]; errors: string[] }`. A question fails validation if any of these are true:

- Its `type` isn't one of the requested types.
- It's multiple choice and doesn't have four unique choices, or `correctAnswer` isn't one of them.
- It's true/false and `choices` isn't `["True", "False"]`.
- It's short answer and `choices` isn't empty.
- Its prompt, explanation, or quote is empty.
- `quoteAppearsInText(sourceQuote, articleText)` is false.

`quoteAppearsInText` should be forgiving about formatting but strict about content:

- Normalize both strings: NFC, lowercase, turn curly quotes into straight quotes and dashes into `-`, remove the `#` and `-` markers added by the structured text, and collapse whitespace.
- Pass if the normalized quote is a substring of the normalized text.
- Otherwise, pass if some window of the text with the same number of words contains at least `QUOTE_MATCH_THRESHOLD` of the quote's words in order.

The whole quiz also fails if the number of valid questions differs from `count`.

### Retry policy

1. If validation fails, make one more call. Add a message listing the specific errors and asking the model for a corrected full response.
2. Validate the second response. If at least one question is valid, use the valid ones and tell the user, e.g. "Generated 8 of 10 questions. 2 couldn't be verified against the page."
3. If no questions are valid, fail with a clear error and a **Try again** button.
4. Never show a question that failed validation.

Assign question IDs (`crypto.randomUUID()`) after validation.

### Popup options

- Above **Generate quiz**, show the question count, difficulty, and type checkboxes, prefilled from `settings.quizDefaults`.
- Changing them here applies only to this quiz and doesn't change the saved defaults.
- If `isConfigured()` is false, replace the button with **Finish setup**, which opens the options page.

### Flow

1. The popup sends `START_QUIZ { tabId, settings }` to the background.
2. The background runs `EXTRACT_PAGE`. On failure, it returns the error and the popup shows it (e.g. "This page doesn't look like an article").
3. On success, the background creates a quiz ID, stores a pending record in `storage.local` under `pending:<quizId>` (the extraction result plus the chosen settings), opens `quiz.html?quizId=<id>` in a new tab, and returns. The popup can then close.
4. The quiz page opens a long-lived connection with `browser.runtime.connect({ name: "generate:<quizId>" })`.
5. The background reads the pending record, calls `provider.generateQuiz` with an `AbortController`, validates (retrying if needed), builds the `Quiz` (status `in_progress`, `startedAt` and `updatedAt` set to now, `answers: []`, `currentIndex: 0`), saves it to `storage.local` under `quiz:<quizId>`, deletes the pending record, and posts the result over the connection.
6. **Cancel** on the quiz page, or closing the tab, disconnects the connection. The background calls `abort()` when the connection disconnects.
7. When the quiz page opens, it checks `quiz:<quizId>` first. If a quiz already exists, it shows it without generating again, so reloading the tab doesn't cost a second API call.

The long-lived connection also keeps the background worker active while it waits for OpenAI. Ticket 009 checks how this holds up on long requests in Safari.

### Basic quiz display

The quiz page shows a loading state ("Reading the page and writing your quiz…") with **Cancel**. When the quiz arrives, it shows the summary, the tags, and a numbered list of questions with their choices, without answers. Ticket 005 replaces this list with the one-question-at-a-time UI.

## Technical Decisions Required

1. **Default OpenAI model.** Pick a cost-efficient model that produces well-grounded questions on the Ticket 003 fixtures. This resolves the roadmap's "Default model" open question.
2. **Which OpenAI endpoint:** Chat Completions (`POST /v1/chat/completions` with `response_format: { type: "json_schema", json_schema: { name, strict: true, schema } }`; read `choices[0].message.content`, and check `message.refusal`), or the Responses API (`POST /v1/responses` with `text.format`). Chat Completions is suggested, because its response shape is simple to parse without the SDK.
3. **`QUOTE_MATCH_THRESHOLD`** for quote matching. Suggested: 0.9.
4. **What to do when only some questions are valid after the retry.** Suggested: show the valid ones with a notice, as described in "Retry policy". The alternative is to fail the whole quiz.

Record all decisions in `docs/implemented/004_DECISIONS.md`.

## Acceptance Criteria

- [ ] On a configured install, clicking **Generate quiz** on an article opens a quiz tab that shows a loading state, then a summary, three to six tags, and the questions.
- [ ] The quiz has exactly the requested number of questions, or a visible message explains why there are fewer.
- [ ] Every question is one of the selected types. With only true/false selected, every question is true/false.
- [ ] Every multiple-choice question has four unique choices, and its correct answer is one of them.
- [ ] Every true/false question has the choices True and False.
- [ ] Every question's `sourceQuote` passes `quoteAppearsInText` against the page's extracted text. No question that fails the check is ever shown.
- [ ] Changing count, difficulty, or types in the popup changes the generated quiz, but doesn't change the saved defaults in settings.
- [ ] Clicking **Cancel**, or closing the quiz tab while generating, stops the OpenAI request.
- [ ] Reloading the quiz tab after generation shows the same quiz without making another API call.
- [ ] On a page that isn't an article, the popup shows the extraction error and no quiz tab opens.
- [ ] An invalid OpenAI key, a rate limit, or a network failure shows a readable error on the quiz page with **Try again**.
- [ ] If Quizify isn't configured, the popup shows **Finish setup** instead of **Generate quiz**.
- [ ] No module outside `lib/providers/` imports the OpenAI adapter directly.
- [ ] The default model is chosen and recorded in `docs/implemented/004_DECISIONS.md`.

## Test Plan

- `validate.test.ts`
  - Accepts a fully valid quiz.
  - Rejects each rule violation on its own: wrong type, `correctAnswer` not among the choices, wrong number of choices, true/false with other choices, short answer with choices, empty fields, a quote not found in the text.
  - Reports the count mismatch.
- `quote-match.test.ts`
  - Exact match passes.
  - Curly-versus-straight quotes, different whitespace, and different letter case pass.
  - One changed word in a long quote passes under the threshold.
  - A made-up sentence fails.
  - A quote that crosses a heading marker in the structured text passes.
- `openai.test.ts` (mocked `fetch`)
  - Sends the schema and the chosen model.
  - Parses the response content.
  - Raises on a refusal.
  - Maps 401, 429, and 500 to the right messages.
- `generate.test.ts`
  - With a fake `AIProvider`, a failed first response triggers exactly one retry.
  - A partially valid second response keeps only the valid questions.
  - A fully invalid second response raises an error.
- Use `makeQuestion`, `makeQuiz`, and `makeGeneratedQuiz` from `quiz-factories.ts`, as described in the testing conventions.

## Out of Scope

- The real one-question-at-a-time UI, answering, and opening the quiz in a window (Ticket 005).
- Grading and short-answer grading (Ticket 006).
- Saving to Notion (Ticket 007).
- Providers other than OpenAI (in the roadmap's "Later").
