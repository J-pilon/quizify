# Phase 8 — Polish and hardening

| | |
| --- | --- |
| Roadmap | [`ROADMAP.md` → Phase 8](../../ROADMAP.md) |
| Depends on | Phases 0–7 |
| Branch | `phase-8-polish-and-hardening` |

## Goal

Make Quizify dependable enough to hand to someone else who brings their own keys. A new user should be able to install, configure, and complete the full workflow using only the README and the in-app onboarding.

## Context

- Users bring their own OpenAI key and Notion integration token. There's no backend and no support channel, so error messages and setup guidance have to stand on their own.
- Distribution is a personal build for now: run from Xcode with a personal signing team. App Store distribution and notarization are out of scope.
- Safari may stop the background service worker while it's idle. Quiz generation can take tens of seconds, so this needs checking.
- API keys currently live in `browser.storage.local`, which isn't encrypted.

## What to implement

1. First-run onboarding.
2. Consistent error handling across the popup, options page, and quiz page.
3. A check of how long Safari keeps the background worker alive during long OpenAI calls, with a fix if needed.
4. A short investigation (spike) of Keychain storage for API keys, ending in a decision and implementing it only if it's simple.
5. A test coverage review and a manual test checklist for Safari.
6. Final icons and a README.
7. An estimated cost per quiz, shown in settings.

## How to implement

### Onboarding

- When `runtime.onInstalled` fires with `reason === "install"`, open `options.html?onboarding=1`.
- In onboarding mode, the options page becomes a three-step guide with progress indicators. Each step reuses the existing Phase 1 components.
  1. **OpenAI:** paste the key and pass **Test connection** before continuing.
  2. **Notion:** create the integration, share a page, paste the token, pick the page, and **Create databases**.
  3. **Defaults:** count, difficulty, types, and quiz location. Then **Finish**.
- The final screen says "Open any article and click the Quizify icon," with a small picture showing where the toolbar icon is.
- If the user leaves partway through, the popup shows **Finish setup** (`isConfigured()` from Phase 1), which reopens onboarding at the first incomplete step.

### Errors (`lib/errors.ts`)

```ts
export type ErrorCode =
  | "network" | "openai_auth" | "openai_rate_limit" | "openai_quota" | "openai_refusal"
  | "notion_auth" | "notion_not_found" | "notion_validation"
  | "not_article" | "too_short" | "restricted_page" | "generation_invalid" | "unknown";

export class QuizifyError extends Error {
  constructor(public code: ErrorCode, message: string, public retryable: boolean) { super(message); }
}
```

- Convert every adapter and client failure into a `QuizifyError`, in one place for OpenAI and one place for Notion.
- Messages sent between parts of the extension carry errors as `{ code, message, retryable }`. Never send raw `Error` objects or stack traces.
- Add one `<ErrorNotice>` component, used by the popup, options page, and quiz page. It shows the message, a **Retry** button when the error can be retried, and a **Settings** link for authentication errors.
- Tell OpenAI quota exhaustion (HTTP 429 with `insufficient_quota`) apart from rate limiting. Quota errors say to check billing, and aren't retryable.
- Make sure no error message or log line contains an API key or token. Add a test that checks this.

### Background worker lifetime

1. Measure: generate 15 hard short-answer questions from a page close to 12,000 words, five times, with Web Inspector open on the extension's background page. Record how long each takes and whether the worker is stopped mid-request.
2. If the worker is stopped:
   - First, send a heartbeat message every 10 seconds over the Phase 3 generation connection while the request is running.
   - If that's not enough, switch generation to streaming (`stream: true`, then assemble the JSON before validating).
   - As a last resort, move the OpenAI `fetch` into the quiz page itself. Extension pages can call hosts listed in `host_permissions`. Record this exception to the "all network calls go through the background" rule.
3. Record the measurements and the outcome in `docs/decisions/phase-8_DECISIONS.md`.

### Keychain investigation (time-boxed)

- Try having the extension call `browser.runtime.sendNativeMessage` to reach the wrapper app's `SafariWebExtensionHandler`, which stores and reads keys with the Security framework (`SecItemAdd` and `SecItemCopyMatching`).
- Decide based on how much complexity it adds, and whether keys can be read without delaying every API call (e.g. by caching them in memory in the background worker).
- Implement it only if it's straightforward. Otherwise, record the decision and move it to "Later" in `ROADMAP.md`. Either way, the options page keeps its plain-language note about where keys are stored.

### Tests and manual checklist

- Review the unit tests for: the reducer, grading, URL normalization, hashing, quote matching, building Notion blocks, and parsing stored quizzes. Fill any gaps against each module's public functions.
- Add `docs/dev-processes/manual-test-checklist.md`, covering:
  - A fresh install and onboarding.
  - Generating each question type.
  - Tab and window modes.
  - Skip, Previous, and Finish confirmation.
  - Reload recovery.
  - Grading with short answers.
  - Saving partway through and saving completed.
  - Resuming from the popup and from **Saved quizzes**.
  - The changed-page notice.
  - Every error path (bad keys, offline, unshared Notion page, a page that isn't an article).
  - Light and dark mode.

### Icons and README

- Toolbar icon: a simple monochrome shape that's readable at 16 px, exported as PNGs at 16, 32, 48, 96, and 128 px for the manifest.
- App icon: added to the Xcode asset catalog at the sizes macOS requires.
- `README.md` at the repository root:
  - What Quizify does, with one screenshot.
  - Requirements: macOS, Safari 17.2 or later, Xcode, a free Apple ID.
  - Build and run steps (from Phase 0).
  - Enabling the extension in Safari.
  - Setting up OpenAI and Notion (pointing to onboarding).
  - Privacy: page text is sent to OpenAI, quiz data is sent to your Notion, and keys stay local.
  - Troubleshooting.

### Cost estimate

- Add `lib/providers/pricing.ts`: a small table of per-million-token input and output prices for the models you expect to use, with a `PRICES_UPDATED` date.
- Estimate a quiz's cost: input tokens ≈ (characters in prompt + 12,000 words × about 6 characters) ÷ 4, and output tokens ≈ count × 120. Add a rough estimate for a grading call.
- In settings, under the model field, show "About $0.004 per 10-question quiz (prices as of <date>)". For a model missing from the table, show "Cost estimate unavailable for this model."

## Acceptance criteria

- [ ] On a fresh install, the onboarding guide opens automatically, and a user can complete OpenAI setup, Notion setup, and defaults without leaving it.
- [ ] Onboarding won't move past a step until that step's connection test or database creation succeeds.
- [ ] Leaving onboarding partway through makes the popup show **Finish setup**, which reopens onboarding at the first incomplete step.
- [ ] Each of these shows a specific, readable message with the right action (Retry or Settings): an invalid OpenAI key, OpenAI rate limiting, exhausted OpenAI quota, going offline, an invalid Notion token, unshared databases, a page that isn't an article, and a restricted page.
- [ ] No API key or token appears in any error message, notice, or console log, as shown by a test.
- [ ] Background worker lifetime has been measured on a 15-question generation from a long page. Generation finishes reliably in all five runs, and the findings and any fix are recorded in the decisions file.
- [ ] The Keychain investigation has a recorded decision. If it's implemented, keys are stored in the Keychain and the options page says so. If not, it's listed under "Later" in `ROADMAP.md`.
- [ ] `npm test`, `npm run lint`, and `npm run build` pass, and the modules listed above all have unit tests.
- [ ] `docs/dev-processes/manual-test-checklist.md` exists, and every item passes in Safari.
- [ ] The toolbar icon is legible in both light and dark Safari toolbars, and the app has a proper icon.
- [ ] Someone who hasn't seen the project can follow the README and onboarding to install Quizify, configure it, and save a completed quiz to Notion, without help.
- [ ] Settings show an estimated cost per quiz for known models and a clear "unavailable" message for others.

## Test plan

- `errors.test.ts`
  - Each OpenAI and Notion failure maps to the expected `ErrorCode`, message, and retryable flag.
  - `insufficient_quota` is told apart from rate limiting.
  - Messages never contain the key or token passed in.
- `pricing.test.ts`: estimates for a known model, and the unavailable result for an unknown one.
- `onboarding.test.tsx`: each step blocks moving on until it succeeds, and reopening starts at the first incomplete step.
- Everything else is checked with the manual checklist.

## Out of scope

- Other AI providers, iOS, PDFs, and Chrome or Firefox (all in "Later").
- App Store distribution and notarization.
- Page versioning and quote highlighting (in "Later").
