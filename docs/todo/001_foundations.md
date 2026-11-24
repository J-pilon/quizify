# Ticket 001 — Foundations

Implements [`ROADMAP.md`](../../ROADMAP.md) Phase 0.

## Overview and Context

**Goal:** an empty Quizify extension that builds, loads in Safari on macOS, and can open a quiz page from the toolbar popup. The quiz page proves it can talk to the article tab by displaying the article's title.

**Context:**

- Quizify is a Manifest V3 web extension wrapped in a native macOS app, which is how Safari ships web extensions.
- Stack: TypeScript, Vite, React. Tests use Vitest.
- The quiz runs on its own extension page (`quiz.html`), never in the toolbar popup. Safari closes the popup as soon as focus leaves it.
- Nothing visible is ever injected into the article page. The content script only reads the page.
- All network calls will go through the background service worker. This ticket makes no network calls, but the message bus built here is what later tickets use.
- Content scripts can't be ES modules, so the content script must build as one self-contained IIFE file.

## Dependencies

None.

## Technical Requirements

1. An `extension/` project with Vite, React, TypeScript, ESLint, Prettier, and Vitest.
2. An MV3 `manifest.json`.
3. A build that produces the popup, options page, quiz page, background worker, and content script.
4. An Xcode wrapper app in `xcode/` that loads the build output into Safari.
5. A typed message bus shared by the popup, content script, quiz page, and background worker.
6. A "Hello" flow: a button in the popup opens `quiz.html`, which shows the title of the tab the popup was opened on.
7. The empty root `manifest.json` moves to `extension/public/manifest.json`.

## Implementation Steps

- [ ] 1. Scaffold `extension/` using the layout below, with npm scripts `build`, `dev`, `test`, `lint`, and `format` matching `AGENT.md`.
- [ ] 2. Move the root `manifest.json` to `extension/public/` and fill it in (see "Manifest").
- [ ] 3. Configure the two Vite builds (see "Build").
- [ ] 4. Add `lib/messages.ts` and the background router (see "Message bus").
- [ ] 5. Build the Hello flow across the popup, background, content script, and quiz page (see "Hello flow").
- [ ] 6. Generate the Xcode wrapper and set signing (see "Xcode wrapper").
- [ ] 7. Write the "Running locally" instructions in `extension/README.md`.

### Project layout

```
extension/
  public/
    manifest.json
    icons/                    # placeholder icons are fine for now
  src/
    background/index.ts
    content/index.ts
    popup/      index.html, main.tsx, Popup.tsx
    options/    index.html, main.tsx, Options.tsx   # placeholder page
    quiz-page/  index.html, main.tsx, QuizPage.tsx
    lib/messages.ts
  vite.config.ts              # HTML pages
  vite.scripts.config.ts      # background + content as IIFE
  package.json
  tsconfig.json
```

### Manifest

```json
{
  "manifest_version": 3,
  "name": "Quizify",
  "version": "0.1.0",
  "description": "Generate quizzes from the page you're reading and save them to Notion.",
  "action": { "default_popup": "src/popup/index.html", "default_icon": { "...": "..." } },
  "background": { "service_worker": "background.js" },
  "options_ui": { "page": "src/options/index.html", "open_in_tab": true },
  "permissions": ["storage", "activeTab", "scripting"],
  "host_permissions": ["https://api.openai.com/*", "https://api.notion.com/*"]
}
```

- Inject the content script **on demand** with `browser.scripting.executeScript({ target: { tabId }, files: ["content.js"] })` when the user acts from the popup. `activeTab` grants access to the current tab at that moment. Declaring `content_scripts` with `<all_urls>` would make Safari ask for access to every website, which isn't needed.
- Adjust the HTML paths to whatever the Vite build actually outputs.

### Build

- `vite.config.ts`: multi-page build with `build.rollupOptions.input` set to the three HTML files. Output to `extension/dist/`, with `public/` copied in.
- `vite.scripts.config.ts`: Vite library mode only supports IIFE output for a single entry, so build `background` and `content` one at a time. Pass the entry name in through an environment variable (e.g. `ENTRY=content vite build -c vite.scripts.config.ts`). Use `formats: ["iife"]`, `emptyOutDir: false`, and output names `background.js` and `content.js`.
- Background worker: build it as a single file with no shared chunks, so it doesn't depend on Safari's support for module service workers.
- Use the promise-based `browser.*` API through `webextension-polyfill` and its types, so the code is typed and consistent.

### Message bus

`lib/messages.ts` defines a discriminated union of message types, plus small typed helpers:

```ts
export type Message =
  | { type: "OPEN_QUIZ_PAGE"; tabId: number }
  | { type: "GET_PAGE_TITLE" };

export type MessageResponse<M extends Message> =
  M extends { type: "OPEN_QUIZ_PAGE" } ? { ok: true } :
  M extends { type: "GET_PAGE_TITLE" } ? { title: string; url: string } :
  never;

export function sendToBackground<M extends Message>(msg: M): Promise<MessageResponse<M>>;
export function sendToTab<M extends Message>(tabId: number, msg: M): Promise<MessageResponse<M>>;
```

- The background worker owns a single `runtime.onMessage` router that switches on `type`. Later tickets add cases to it.
- The content script registers its own `runtime.onMessage` handler. Guard against registering twice if it's injected again (e.g. check a `window.__quizifyLoaded` flag).

### Hello flow

1. The popup shows a **Generate quiz** button (the real behavior comes in Ticket 004). Clicking it gets the active tab and sends `OPEN_QUIZ_PAGE { tabId }` to the background.
2. The background calls `browser.tabs.create({ url: "quiz.html?articleTabId=<tabId>" })` with the built path.
3. The quiz page reads `articleTabId` from the query string and asks the background to inject the content script into that tab and send it `GET_PAGE_TITLE`.
4. The quiz page shows "Hello from Quizify — article: <title>".

### Xcode wrapper

```bash
xcrun safari-web-extension-converter extension/dist \
  --project-location xcode \
  --app-name Quizify \
  --bundle-identifier <your.reverse.domain>.quizify \
  --macos-only --swift --no-open
```

- Don't pass `--copy-resources`. The Xcode project should reference `extension/dist`, so rebuilding with `npm run build` and re-running in Xcode picks up changes.
- Set signing to your personal team (a free Apple ID is enough) so Safari keeps the extension enabled across relaunches. As a fallback, unsigned builds work after enabling *Allow Unsigned Extensions* in Safari's developer settings, but that setting resets every time Safari restarts.

## Technical Decisions Required

1. **Bundle identifier** for the wrapper app (e.g. `com.<you>.quizify`). This is the user's choice.
2. **Content script injection:** on demand with `scripting.executeScript` (recommended above, because Safari won't ask for access to every site), or declared in the manifest's `content_scripts`.
3. **Browser API typing:** `webextension-polyfill` and its types (recommended above), or the global `chrome`/`browser` objects with `@types/chrome`.
4. **Script build setup:** one parameterized IIFE config run once per entry (recommended above), or a separate config file for each script.

Record all decisions in `docs/implemented/001_DECISIONS.md`.

## Acceptance Criteria

- [ ] `cd extension && npm install && npm run build` succeeds and produces `dist/` containing the manifest, the three HTML pages, `background.js`, and `content.js`.
- [ ] `content.js` and `background.js` are each a single file with no `import` statements.
- [ ] `npm test`, `npm run lint`, and `npm run format` all run successfully. One trivial test is enough to prove Vitest is wired up.
- [ ] Running the Xcode project installs the Quizify app, and Quizify appears in Safari > Settings > Extensions and can be enabled.
- [ ] The Quizify icon appears in the Safari toolbar, and clicking it opens the popup.
- [ ] Clicking the popup's button on an ordinary web page opens a new tab showing `quiz.html` with that page's title.
- [ ] No visible element is added to the article page. Inspecting the page's DOM after the flow shows nothing new.
- [ ] Clicking the button on a page Safari won't let extensions access (e.g. a Safari settings page) shows a readable error in the popup instead of failing silently.
- [ ] The options page opens in a tab from Safari's extension settings, even though it's only a placeholder.
- [ ] The root `manifest.json` has moved to `extension/public/manifest.json`.

## Test Plan

- `lib/messages.test.ts`: the background router sends each message type to the right handler, and unknown types are rejected.
- Everything else in this ticket (the build and Safari loading) is checked manually against the acceptance criteria.

## Out of Scope

- Real extraction, quiz generation, settings, or Notion calls.
- Polished icons (placeholders are fine).
- The tab-or-window setting (Ticket 002 adds the setting, Ticket 005 honors it).
