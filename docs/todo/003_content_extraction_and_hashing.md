# Ticket 003 — Content extraction and hashing

Implements [`ROADMAP.md`](../../ROADMAP.md) Phase 2.

## Overview and Context

**Goal:** given an article tab, Quizify returns its clean text, title, normalized URL, and a stable content hash. Pages that aren't articles fail with a clear reason.

**Context:**

- Only article-style HTML pages are supported in v1 (docs, blog posts, HTML papers).
- The extracted text is what quizzes are generated from, and what every `sourceQuote` is checked against in Ticket 004.
- The **content hash** is a SHA-256 of the cleaned, normalized article text. It's saved to Notion and later used to tell whether a page's material has changed. It must not change when only ads, timestamps, or markup change.
- The **normalized URL** is how a page is matched to its Notion Source row.
- The content script reads the page and adds no visible UI.

## Dependencies

- `001_foundations.md` (content script injection, message bus)

## Technical Requirements

1. Readability extraction in the content script.
2. Conversion of the extracted HTML into structured plain text (keeping heading and list markers).
3. A length cap with truncation reporting.
4. Detection of pages that aren't articles.
5. URL normalization.
6. The content hash.
7. One background message, `EXTRACT_PAGE { tabId }`, that ties these together.
8. HTML fixtures and unit tests.

## Implementation Steps

- [ ] 1. Save the HTML fixtures (see "Fixtures").
- [ ] 2. Add `lib/extract/structured-text.ts` (see "Structured text").
- [ ] 3. Add `lib/extract/readability.ts` and call it from `content/index.ts` (see "Extraction").
- [ ] 4. Add the length cap and the checks for pages that aren't articles (see "Length cap" and "Detecting pages that aren't articles").
- [ ] 5. Add `lib/url.ts` (see "URL normalization").
- [ ] 6. Add `lib/extract/hash.ts` (see "Content hash").
- [ ] 7. Add the `EXTRACT_PAGE` background message (see "Background message").
- [ ] 8. Add a temporary debug view to the quiz page (see "Debug view").

### Extraction

- Add `@mozilla/readability`.
- Check `isProbablyReaderable(document)` first. If it returns false, still try extraction, but fail if the result is too short (below).
- Run `new Readability(document.cloneNode(true) as Document).parse()`. Always pass a clone, because Readability modifies the document it's given.
- Use `article.title`, falling back to `document.title`. Use `article.content` (HTML) for the structured text conversion below.

### Structured text

Convert the Readability HTML into plain text that keeps the article's structure:

- Headings `h1`–`h6` become lines starting with `#` (one `#` per level).
- List items become `- item`. Ordered lists become `1. item`.
- Paragraphs, blockquotes, and `pre` blocks are separated by blank lines.
- Within a block, whitespace collapses to single spaces.
- Images, figures without captions, scripts, and styles are dropped. Captions are kept as their own paragraphs.

Parse with `DOMParser` and walk the nodes. Keep this function pure (HTML string in, text out) so it can be tested without a browser.

### Length cap

- `MAX_WORDS = 12_000`.
- If the text is longer, cut it at the last paragraph boundary before the limit and set `truncated: true`.
- The quiz is generated from the capped text. The hash is computed from the full text (below).

### Detecting pages that aren't articles

Fail with a reason code the UI can show:

| Reason | When |
| --- | --- |
| `not_article` | Readability returns `null` |
| `too_short` | Fewer than `MIN_WORDS` words extracted |
| `restricted_page` | The content script can't be injected (Safari pages, extension pages, etc.) |

### URL normalization

`normalizeUrl(raw: string): string`:

- Lowercase the scheme and host.
- Drop default ports (`:80`, `:443`) and the fragment (`#...`).
- Remove tracking parameters: `utm_*`, `fbclid`, `gclid`, `dclid`, `msclkid`, `mc_cid`, `mc_eid`, `ref`, `ref_src`. Keep the list in a constant.
- Sort the remaining query parameters, so parameter order doesn't affect matching.
- Remove a trailing slash from the path, except for the root `/`.

### Content hash

```ts
export function normalizeForHash(text: string): string {
  return text.normalize("NFC").replace(/\s+/g, " ").trim();
}

export async function contentHash(text: string): Promise<string> {
  const bytes = new TextEncoder().encode(normalizeForHash(text));
  const digest = await crypto.subtle.digest("SHA-256", bytes);
  return [...new Uint8Array(digest)].map((b) => b.toString(16).padStart(2, "0")).join("");
}
```

- Hash the **full structured text before the length cap**, so a page near the limit doesn't change its hash when the cap changes.
- **Compute the hash in the background worker, not the content script.** `crypto.subtle` only exists in secure contexts, so it's unavailable to a content script on a plain-HTTP page. The hash function stays in `lib/extract/`.

### Background message

`EXTRACT_PAGE { tabId }`:

1. Inject `content.js` into the tab (reuse the Ticket 001 helper).
2. Send `EXTRACT_ARTICLE` to the content script. It returns `{ title, url, fullText }`, or `{ error: reason }`.
3. In the background: `normalizeUrl(url)`, `contentHash(fullText)`, then apply the cap.
4. Return one of:

```ts
type ExtractResult =
  | { ok: true; title: string; url: string; normalizedUrl: string; text: string;
      wordCount: number; truncated: boolean; contentHash: string }
  | { ok: false; reason: "not_article" | "too_short" | "restricted_page"; message: string };
```

### Debug view

This ticket has no user-facing UI of its own. Add a temporary debug view to the Ticket 001 quiz page (title, word count, truncated flag, hash, and the first 500 characters) so extraction can be checked in Safari. Ticket 004 replaces it.

### Fixtures

Save three real pages as static HTML in `extension/src/lib/extract/tests/fixtures/`: a documentation page, a blog post, and a news article. Add one page that isn't an article (e.g. a search results or login page). Run the tests in Vitest's `jsdom` environment.

## Technical Decisions Required

1. **Remove a leading `www.` during URL normalization?** Removing it matches `www.example.com` and `example.com` as the same page, but a few sites serve different content on each.
2. **`MIN_WORDS` threshold** for `too_short`. Suggested: 250.
3. **`MAX_WORDS` cap.** Suggested: 12,000. Larger caps cost more per quiz.
4. **Where to compute the hash.** Recommended: the background worker, for the plain-HTTP reason above. Note that `ROADMAP.md` and `AGENT.md` currently say the content script does it; update both if this is confirmed.

Record all decisions in `docs/implemented/003_DECISIONS.md`.

## Acceptance Criteria

- [ ] On each article fixture, extraction returns the article's title and body text without navigation, footer, ads, or comments.
- [ ] Headings in the result start with `#` markers and list items start with `-` or `1.`, matching the source page's structure.
- [ ] The non-article fixture fails with `not_article` or `too_short`. It never returns a garbage "article".
- [ ] Running extraction on a restricted page returns `restricted_page` with a readable message, and nothing throws.
- [ ] A page longer than `MAX_WORDS` returns `truncated: true`, and its text ends at a paragraph boundary at or below the limit.
- [ ] Extracting the same page twice produces the same hash.
- [ ] Changing only whitespace, line breaks, or Unicode composition (e.g. `é` written as one character versus `e` plus a combining accent) doesn't change the hash.
- [ ] Changing one word in one paragraph changes the hash.
- [ ] The hash is the same whether or not the text was truncated, because it's computed from the full text.
- [ ] The hash is a 64-character lowercase hexadecimal string.
- [ ] Extraction works on a plain-HTTP article page, because hashing runs in the background.
- [ ] `normalizeUrl` returns the same value for `https://Example.com/post/?utm_source=x&b=2&a=1#intro` and `https://example.com/post?a=1&b=2`.
- [ ] `normalizeUrl` keeps query parameters that aren't tracking parameters (e.g. `?id=123`).
- [ ] The article page's DOM isn't changed by extraction.

## Test Plan

- `structured-text.test.ts`: headings, nested lists, ordered lists, blockquotes, `pre` blocks, dropped images and scripts, and whitespace collapsing.
- `readability.test.ts`: each fixture returns the expected title and starts with the expected first sentence; the non-article fixture fails with the right reason.
- `hash.test.ts`: stable for identical input; unchanged by whitespace and NFC differences; changed by a one-word edit; correct output format.
- `url.test.ts`: a table-driven test (`it.each`) covering every normalization rule.
- Add `extract-factories.ts` with `makeArticleHtml({ paragraphs, headings })` to build synthetic pages for edge cases (e.g. exactly `MIN_WORDS` words, or `MAX_WORDS + 1` words).

## Out of Scope

- PDFs, YouTube transcripts, highlighted-text quizzes, and chunking very long pages (all in the roadmap's "Later").
- Using the hash to flag changed pages (Ticket 008).
- Saving the hash to Notion (Ticket 007).
