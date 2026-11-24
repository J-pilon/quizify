# Quizify

## Project Purpose
Quizify is a Safari extension for macOS that turns passive reading into active recall. It uses AI to generate a quiz about the webpage you're reading, then saves a page summary (with backlink) and your quiz attempt to Notion. Built for personal use first, but designed so others can use it by bringing their own API keys.

**Key distinction**: This is a prototype/MVP — favor speed of iteration, and document assumptions rather than over-engineering for distribution.

## Core Functionality

1. **Extraction**: Content script pulls clean article text via Readability and computes a SHA-256 content hash
2. **Quiz Generation**: One click produces a grounded quiz, summary, and tags via a pluggable `AIProvider` (OpenAI in v1)
3. **Quiz UI**: Dedicated extension page presents one question at a time with Skip/Previous/Save
4. **Grading & Review**: Local grading for choice questions, AI grading for short answers, results with source quotes
5. **Notion Sync**: Saves to Sources and Quizzes databases; in-progress quizzes can be resumed from Notion

## High-Level Architecture

- **Language**: TypeScript
- **Framework**: Vite + React, Manifest V3 web extension wrapped in a native Xcode Safari app
- **Database**: None; `browser.storage.local` for settings/autosave, Notion as the system of record
- **Runtime mode**: Local only, no backend; background worker calls OpenAI and Notion APIs directly
- **Key dependencies**: Mozilla Readability, Vitest; content script must build as a single IIFE bundle

## Commands

- **Install**: `cd extension && npm install`
- **Run**: `cd extension && npm run build` (then run the Xcode wrapper to load in Safari)
- **Test**: `cd extension && npm test`
- **Lint**: `cd extension && npm run lint`
- **Format**: `cd extension && npm run format`

## Critical Context / Principles

- Speed of iteration over completeness; spike freely and document assumptions
- All network calls go through the background worker (Notion has no CORS headers)
- Quiz runs on its own extension page, never in the popup (Safari closes it on focus loss)
- Every answer must be grounded in a verbatim `sourceQuote` validated against the extracted text
- API keys stay local and are sent only to their own provider; nothing is injected into article pages

## Development Process

### Commit Guidelines
All commits must follow the project's commit process. See [`docs/dev-processes/commit-process.md`](docs/dev-processes/commit-process.md) for complete guidelines.

**Key principles:**
- Use present tense in commit messages
- Separate test commits from code commits
- Follow format: `<type>: <description>`
- Never include ticket numbers in messages
- Commit after each logical unit is complete
- **NEVER create merge commits with new content** (merge commits must only combine existing parent branches without introducing new files or changes)

### Design Before Implementation
Before creating features, building components, adding functionality, or modifying behavior, use the `superpowers` skill to refine the idea, present a design, and get approval.

### Ticket Implementation
Implementation follows a structured TDD workflow defined in the `implement-ticket` skill. Tickets in `docs/todo/` are implemented sequentially based on dependencies and moved to `docs/implemented/` when complete.

### Pull Requests
Create a pull request when one or more related tickets are complete and ready for review. Use the `pr-description` skill to generate clear, structured PR descriptions.

**When to create a PR:**
- One or more tickets are fully implemented with all tests passing
- Related bug fixes or improvements that form a logical unit
- Documentation updates that benefit from review

**PR should include:**
- Summary of what was accomplished
- List of changes made
- Any concerns or trade-offs
- Links to relevant tickets in `docs/implemented/`

## Testing Conventions
Tests must focus on **what is under test**, not setup logic. Use lightweight factory functions and test helpers to keep tests clean and maintainable.

**Key principles:**
- Use factory functions for creating test instances
- Extract reusable logic into test helpers
- Tests should focus on assertions, not mechanics
- Keep factories simple and composable
- Name factory files specifically (e.g., `order_factories.py` not `factories.py`)

See [`docs/conventions/testing-conventions.md`](docs/conventions/testing-conventions.md) for complete guidelines, patterns, and examples.

## Explicitly Out of Scope

- ❌ Browsers other than macOS Safari (no Chrome/Firefox, iOS/iPadOS in v1)
- ❌ A backend or hosted service; users bring their own API keys
- ❌ AI providers other than OpenAI in v1 (interface exists, adapters come later)
- ❌ Non-article content: PDFs, YouTube transcripts, highlighted-text quizzes
- ❌ UI injected into the article page
- ❌ App Store distribution or notarization in v1
- ❌ Page versioning and spaced repetition (see ROADMAP "Later")

## Domain Context
Quizify sits between web reading, LLM-generated assessment, and a Notion knowledge base. Each page maps to one Source; each quiz attempt maps to a Quiz row, and retaking adds a new row.

### Glossary
- **Source**: Notion row for one webpage, matched by normalized URL
- **Quiz**: One attempt at a quiz; status is *In progress* or *Completed*
- **Content hash**: SHA-256 of cleaned, normalized Readability text, used to detect changed pages
- **sourceQuote**: Verbatim page excerpt supporting a question's answer
- **Normalized URL**: URL minus fragment, tracking params, and trailing slash
- **Data block**: The "Quizify data" JSON code block on a Notion quiz page, used to resume

## Success Criteria

- A new user can install, configure keys and Notion, and complete the full quiz-to-Notion workflow using only the README
- A quiz saved partway through can be resumed later and ends as a single completed Notion entry
- Generated quizzes are always grounded: every question's source quote appears in the page text
