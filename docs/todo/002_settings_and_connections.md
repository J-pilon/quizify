# Ticket 002 — Settings and connections

Implements [`ROADMAP.md`](../../ROADMAP.md) Phase 1.

## Overview and Context

**Goal:** a user can bring their own OpenAI key and Notion integration token, test both, set quiz defaults, and have Quizify create its two Notion databases, all from the options page without touching code.

**Context:**

- There's no backend. Keys are stored locally in `browser.storage.local` and sent only to their own API.
- All network calls go through the background service worker. The Notion API doesn't send CORS headers, so the options page must ask the background to make calls instead of calling `fetch` itself.
- OpenAI is the only AI provider in v1, but the settings shape should allow more later.
- Notion layout: two related databases, **Sources** (one row per webpage) and **Quizzes** (one row per quiz attempt), joined by a two-way relation.
- Notion API versions from `2025-09-03` onward split each database into one or more **data sources**. Creating rows, querying, and relations target the data source ID, not the database ID.

## Dependencies

- `001_foundations.md` (build, message bus, options page shell)

## Technical Requirements

1. A settings module with a typed schema, defaults, and load/save helpers.
2. An options page with sections for AI provider, quiz defaults, quiz location, and Notion.
3. **Test connection** for OpenAI and for Notion.
4. A Notion setup flow: instructions, token entry, parent page picker, and **Create databases**.
5. A minimal Notion client in the background worker with a pinned `Notion-Version`.

## Implementation Steps

- [ ] 1. Add `lib/settings.ts` (see "Settings").
- [ ] 2. Add `lib/notion/client.ts` with `notionFetch` and the pinned version (see "Notion client").
- [ ] 3. Add the background messages `TEST_OPENAI`, `TEST_NOTION`, `SEARCH_NOTION_PAGES`, and `CREATE_NOTION_DATABASES` (see "Background messages").
- [ ] 4. Add `lib/notion/schema.ts` to build the two database requests (see "Creating the databases").
- [ ] 5. Build the options page sections (see "Options page").
- [ ] 6. Build the Notion setup flow UI (see "Setup flow UI").

### Settings

```ts
export interface Settings {
  ai: { provider: "openai"; apiKey: string; model: string };
  quizDefaults: {
    count: 5 | 10 | 15;
    difficulty: "easy" | "medium" | "hard";
    types: ("multiple_choice" | "true_false" | "short_answer")[];   // at least one
  };
  quizLocation: "tab" | "window";                                     // default "tab"
  notion: {
    token: string;
    parentPageId?: string;
    sourcesDatabaseId?: string;
    sourcesDataSourceId?: string;
    quizzesDatabaseId?: string;
    quizzesDataSourceId?: string;
  };
}
```

- Store everything under one `settings` key. `loadSettings()` merges stored values over `DEFAULT_SETTINGS`, so adding fields later doesn't break older installs.
- Define `DEFAULT_MODEL` once, in `lib/providers/openai.ts`. Use a reasonable placeholder for now; Ticket 004 chooses the real default.
- `isConfigured(settings)` returns true only when the OpenAI key and every Notion ID are present. Later tickets use it to decide whether the popup shows **Generate quiz** or **Finish setup**.

### Notion client

- `notionFetch(path, init)` sets `Authorization: Bearer <token>`, `Notion-Version: <pinned>`, and `Content-Type: application/json`, then parses errors into `{ status, code, message }`.
- Pin the API version in one constant. Check the request shapes below against Notion's current docs before coding.

### Background messages

| Message | What it does |
| --- | --- |
| `TEST_OPENAI { apiKey }` | `GET https://api.openai.com/v1/models` with `Authorization: Bearer <key>`. Returns ok, or an error message. |
| `TEST_NOTION { token }` | `GET https://api.notion.com/v1/users/me`. Returns ok plus the integration's name. |
| `SEARCH_NOTION_PAGES { token, query }` | `POST /v1/search` filtered to pages. Returns `{ id, title, icon }[]`. |
| `CREATE_NOTION_DATABASES { token, parentPageId }` | Creates both databases (below) and returns their IDs. |

### Creating the databases

Create **Sources** first, because Quizzes' relation needs Sources' data source ID.

1. `POST /v1/databases`, with parent `{ type: "page_id", page_id: parentPageId }`, title "Quizify Sources", and these initial data source properties:

   | Property | Type |
   | --- | --- |
   | Name | `title` |
   | URL | `url` |
   | Content hash | `rich_text` |
   | Summary | `rich_text` |
   | Tags | `multi_select` |
   | Captured | `created_time` |

2. `POST /v1/databases`, with title "Quizify Quizzes" and these properties:

   | Property | Type |
   | --- | --- |
   | Name | `title` |
   | Source | `relation` to the Sources data source, `type: "dual_property"` |
   | Content hash | `rich_text` |
   | Status | `select` with options *In progress* and *Completed* |
   | Score | `number`, format `percent` |
   | Started | `date` |
   | Completed | `date` |

3. A two-way relation automatically adds a back-relation property to Sources. If Notion doesn't name it **Quizzes**, rename it by updating the Sources data source's properties.
4. Save all four IDs (both database IDs and both data source IDs) to settings.

Notes:

- **Status** is a `select`, because the Notion API can't create `status` properties.
- Notion's `percent` number format expects a fraction (0.8 displays as 80%). Ticket 007 writes `score / 100`.
- If setup fails partway through (e.g. Sources was created but Quizzes failed), show the error and let the user retry. On retry, reuse the Sources database already recorded in settings instead of creating a second one.

### Options page

- **AI provider:** a provider dropdown (only OpenAI for now), a password input for the API key, a model text input defaulting to `DEFAULT_MODEL`, and **Test connection**.
- **Quiz defaults:** question count (5/10/15), difficulty (easy/medium/hard), and question-type checkboxes. At least one type must stay checked.
- **Quiz location:** a radio group for **New tab** (default) or **Separate window**.
- **Notion:** setup steps (below), connection status, and the created database names with links.
- Show a note near the key fields: "Your keys are stored only in this browser and are sent only to OpenAI and Notion."
- Save each field on change, and show a small "Saved" confirmation.

### Setup flow UI

1. Instructions, with a link to Notion's integrations page:
   - Create an internal integration.
   - Copy its token.
   - Open the Notion page where Quizify should live and connect the integration to it from the page's connections menu.
2. A token input with **Test connection**.
3. A searchable picker listing pages the integration can access (from `SEARCH_NOTION_PAGES`). If the list is empty, explain that the page must be shared with the integration first.
4. **Create databases**, followed by a success state that links to both databases.
5. Once databases exist, replace the create button with **Reset Notion setup**, which clears the stored IDs. It never deletes anything in Notion.

## Technical Decisions Required

1. **Notion API version to pin.** Recommended: `2025-09-03`, so new code doesn't depend on behavior Notion is deprecating. The alternative is `2022-06-28`, which uses database IDs everywhere and doesn't have data sources. This resolves the roadmap's "Notion API version" open question.
2. **Placeholder default model** until Ticket 004 chooses the real one.

Record all decisions in `docs/implemented/002_DECISIONS.md`.

## Acceptance Criteria

- [ ] On a fresh install, the options page loads with default values: 10 questions, medium difficulty, multiple choice selected, and quiz location set to new tab.
- [ ] Every setting persists after closing and reopening the options page and after restarting Safari.
- [ ] The question-type checkboxes never allow all types to be unchecked.
- [ ] **Test connection** for OpenAI shows success with a valid key, and a clear message for an invalid key (401) or a network failure.
- [ ] **Test connection** for Notion shows success and the integration's name with a valid token, and a clear message for an invalid token.
- [ ] The page picker lists only pages shared with the integration, and explains what to do when there are none.
- [ ] **Create databases** creates "Quizify Sources" and "Quizify Quizzes" under the chosen page, with exactly the properties and types listed above, including a two-way relation that shows as **Source** on Quizzes and **Quizzes** on Sources.
- [ ] After setup, all four Notion IDs are stored and the options page links to both databases.
- [ ] If database creation fails partway through, retrying doesn't create a duplicate Sources database.
- [ ] No network request is made from the options page itself. Every API call goes through the background worker.
- [ ] API keys are never written to the console or included in error messages.
- [ ] `isConfigured()` returns false until the OpenAI key and Notion setup are both complete.
- [ ] The chosen `Notion-Version` is recorded in `docs/implemented/002_DECISIONS.md`.

## Test Plan

- `lib/settings.test.ts`
  - `loadSettings` returns defaults when nothing is stored.
  - Stored partial settings merge over the defaults.
  - `isConfigured` is false when any required value is missing, and true when all are present.
- `lib/notion/schema.test.ts`
  - The Sources request body has the expected properties and types.
  - The Quizzes request body builds its relation from the given Sources data source ID.
- `lib/notion/client.test.ts` (with a mocked `fetch`)
  - Sets the auth and version headers.
  - Turns a Notion error response into `{ status, code, message }`.
- Add a `notion-factories.ts` with `makeNotionErrorResponse()` and `makeCreateDatabaseResponse()`, following the testing conventions.

## Out of Scope

- Generating quizzes or using the OpenAI key beyond **Test connection** (Ticket 004).
- Writing rows to Notion (Ticket 007).
- Opening the quiz in a window (Ticket 005 honors the setting saved here).
- First-run onboarding (Ticket 009).
- Storing keys in the macOS Keychain (Ticket 009).
