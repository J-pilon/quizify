---
name: Implement Ticket
description: Guide implementation of todo tickets from docs/todo/ using TDD, interactive technical decision-making, and verification against acceptance criteria. Moves completed tickets to docs/implemented/.
---

# Implement Todo Ticket Skill

## Purpose
Guide the agent through implementing a todo ticket from `docs/todo/` following TDD principles, making technical decisions interactively, and moving completed work to `docs/implemented/`.

## Instructions

### Phase 1: Select Next Ticket

1. **List available tickets:**
   ```bash
   ls -1 docs/todo/*.md | sort
   ```

2. **Check dependencies:**
   - Read each ticket's "Dependencies" section
   - Verify all dependency tickets are in `docs/implemented/`
   - If implementing ticket 001 (Project Foundation), proceed immediately

3. **Identify next eligible ticket:**
   - Find the lowest-numbered ticket with all dependencies satisfied
   - If multiple tickets are eligible, ask user which to implement

4. **Read the selected ticket completely:**
   - Overview and Context
   - Technical Requirements
   - Implementation Steps
   - Acceptance Criteria
   - Technical Decisions Required
   - Out of Scope

### Phase 2: Technical Decisions (Interactive)

1. **Review "Technical Decisions Required" section**

2. **Check for reusable decisions from prior tickets:**
   - Read all `*_DECISIONS.md` files in `docs/implemented/`
   - For each decision requirement in current ticket, check if an identical or similar decision was already made
   - If found, reuse the prior decision and note the source ticket
   - Only present decisions to user that haven't been made in prior tickets
   - Example: If "caching strategy" was chosen in Ticket 002, reuse it for Ticket 003 if same requirement exists

3. **For EACH remaining decision item:**
   - Evaluate if there is a defensible, obvious choice given:
     * The project principles in AGENT.md
     * Technical constraints from prior decisions
     * Acceptance criteria requirements
     * Alignment with project principles (prefer simple over complex, standard over custom)

   - **If the decision involves object design, architecture, or component interaction:**
     * Use the `choose-pattern-category` skill to identify relevant pattern category
     * Consider patterns from the appropriate convention document:
       - `docs/conventions/design-patterns/creational-patterns.md` (object creation)
       - `docs/conventions/design-patterns/structural-patterns.md` (object composition)
       - `docs/conventions/design-patterns/behavioral-patterns.md` (object interaction)
     * Evaluate 2-3 relevant patterns against the specific requirements
     * Include pattern rationale in decision documentation
     * Only use patterns when they reduce complexity - simple direct code beats unnecessary patterns

   - **Signs of an obvious decision:**
     * One option clearly aligns with "keep it simple" principle
     * One option is industry-standard for this use case
     * Decision was already made in similar context in prior ticket
     * Trade-offs are minimal or one-sided
     * All options would work but one is simpler/faster to implement

   - **If decision is obvious:**
     * Make the decision automatically
     * Document the choice with clear rationale
     * Note: "Decision made automatically based on [specific reason]"
     * Proceed immediately (no user approval needed)

   - **If decision has significant trade-offs or is genuinely unclear:**
     * Present the decision question to the user
     * Explain why this decision matters and why it's ambiguous
     * Present 2-3 reasonable options with trade-offs clearly explained
     * Wait for user's answer (ONLY stop here if genuinely ambiguous)

   - Document all decisions in `DECISIONS.md` file in the ticket's implementation

4. **Example interaction for obvious decision (made automatically):**
   ```
   Technical Decision 2 of 5: Data Representation Format

   ✓ Decision: JSON with schema validation

   Rationale: Start simple, expand only when requirements demonstrate need.
   Avoids over-engineering before understanding actual data shapes. Sufficient
   for initial implementation and easy to inspect during development.

   Decision made automatically based on the project principles in AGENT.md.
   ```

5. **Example interaction for non-obvious decision (ask user):**
   ```
   Technical Decision 3 of 5:

   Question: Should we use synchronous or asynchronous processing?

   Context: Processing large batches could block the main application.

   This decision has significant trade-offs:
   - Synchronous: Simpler code, easier debugging, but slower for batch operations
   - Asynchronous: Better throughput, more complex, harder to debug failures

   Given the project goals, this isn't obvious. What's your preference?

   Your decision: _____
   ```

6. **Example interaction for reused decision:**
   ```
   Technical Decision 1 of 5: Storage Backend

   ✓ Reused from Ticket 002: Local filesystem with configurable base path

   Rationale: Consistent with the approach established for document storage.
   Reusing for consistency across file-handling components.
   ```

7. **After all decisions are made:**
   - Create `DECISIONS.md` summarizing all choices (reused, automatic, and user-provided)
   - For reused decisions, clearly note the source ticket
   - For automatic decisions, note they were made based on project principles
   - For design pattern choices, include:
     * Which pattern was selected (if any)
     * Which category it came from (creational/structural/behavioral)
     * Why it was chosen over alternatives
     * Reference to convention document
   - Show decision summary and proceed with implementation

### Phase 3: TDD Implementation

1. **Start with acceptance criteria:**
   - Convert each acceptance criterion into one or more test cases
   - Write test names/descriptions first (don't write test code yet)
   - Show the test plan: "Test cases planned: [list test names]"
   - Proceed with implementation (user can review test plan in output)

2. **Write first failing test:**
   - Pick the simplest test from the list
   - Write the test code
   - Run the test to confirm it fails (use the test command from `AGENT.md`)
   - Output: "✗ Test fails as expected (nothing implemented yet)"

3. **Implement minimum code to pass the test:**
   - Write only enough code to make this ONE test pass
   - No extra features
   - No premature optimization
   - If implementation becomes complex or has many conditionals, consider:
     * Is there a design pattern that would simplify this?
     * Use the `choose-pattern-category` skill
     * Refactor using appropriate pattern only if it reduces complexity
     * Remember: simple direct code beats unnecessary pattern abstraction

4. **Run the test:**
   - Run the test command from `AGENT.md`
   - Confirm it passes
   - Output: "✓ Test passes"

5. **Refactor if needed:**
   - Clean up code while keeping tests green
   - Ask: "Does this need refactoring?"

6. **Repeat steps 2-5 for each test:**
   - One test at a time
   - Red → Green → Refactor cycle
   - Show progress: "Test 3 of 12 complete"

7. **Commit after TDD cycle:**
   - After completing a logical group of tests and implementation (e.g., 3-5 related tests)
   - Use the `commit` skill
   - This ensures incremental commits with proper structure
   - Tests and code will be committed separately per conventions

8. **After core tests pass:**
   - Write integration tests if specified
   - Write edge case tests
   - Run entire test suite

### Phase 4: Complete Implementation Steps

1. **Review "Implementation Steps" section**

2. **For each step not covered by tests:**
   - Implement the step
   - Verify it works (manual check or automated)
   - Check off the step

3. **Examples of non-test steps:**
   - Creating data models or schema files
   - Adding CLI commands or management utilities
   - Creating fixtures or seed data
   - Writing documentation

### Phase 5: Verification

1. **Run complete test suite** using the test command from `AGENT.md`.

2. **Verify all acceptance criteria:**
   - Go through checklist one by one
   - Manually verify each criterion is met
   - Ask user to spot-check critical functionality

3. **Check for regressions:**
   - Run tests from previous tickets
   - Ensure nothing broke

4. **Review "Out of Scope":**
   - Confirm nothing out-of-scope was implemented
   - Keep the implementation minimal

### Phase 6: Move to Implemented

1. **Create implementation summary:**
   ```markdown
   # Ticket XXX: [Name] - Implementation Summary

   ## Completed: [Date]

   ## Technical Decisions Made:
   - Decision 1: [choice and rationale]
   - Decision 2: [choice and rationale]
   ...

   ## Tests Added:
   - [count] unit tests
   - [count] integration tests
   - Test coverage: [%]

   ## Files Changed:
   - [list of new/modified files]

   ## Notes:
   - Any important implementation details
   - Deviations from original plan
   - Known limitations
   ```

2. **Append summary to original ticket file**

3. **Move completed ticket:**
   ```bash
   mv docs/todo/XXX_ticket_name.md docs/implemented/
   ```

4. **Final commit:**
   - Use the `commit` skill
   - Commit any remaining documentation updates
   - Commit the ticket move to docs/implemented/
   - Verify all work is committed with proper structure

5. **Announce completion:**
   ```
   ✓ Ticket XXX complete!

   Summary:
   - [X] acceptance criteria met
   - [X] tests passing
   - [X] all work committed following conventions
   - [X] moved to docs/implemented/
   - [X] diagrams generated (if changes warranted them)

   Next step: Phase 7 - Review the Implementation
   ```

### Phase 7: Review the Implementation

1. **Follow `docs/dev-processes/code-review-process.md` in full** — it owns
   the complete process (run the automated review, verify every finding
   against the actual code, check for existing precedent, rank by severity,
   report only what survives verification). Do not re-derive or duplicate
   that process here.

2. **Report verified findings to the user and stop — do not fix anything yet:**
   - Present each surviving finding per the process doc's format (file:line,
     the concern, a concrete failure scenario)
   - If no findings survive verification, say so explicitly
   - Ask the user which findings, if any, they want fixed
   - This phase produces a report, not a code change; wait for the user's
     answer before touching any code (same as Phase 2's decision gate)

3. **Act only on what the user confirms:**
   - Fix only the findings the user explicitly asks to be fixed
   - Leave everything else as-is — do not unilaterally decide a finding is
     "real" enough to warrant a fix
   - Re-run the test suite if any fixes were made, to confirm nothing broke
   - If a finding shows the acceptance criteria can't actually be met as
     implemented, stop and follow this skill's "Error Handling" section

4. **Announce completion:**
   ```
   ✓ Code review complete for Ticket XXX

   - [X] findings verified against actual code
   - [X] findings reported to user
   - [X] user-confirmed fixes applied (or: no findings / no fixes requested)

   Next step: Phase 8 - Diagram Generation
   ```

### Phase 8: Generate Diagrams (If Warranted)

1. **Analyze change complexity:**

   Evaluate if the implemented changes warrant diagrams based on:
   - **Model changes:** New models or relationships between 3+ models
   - **Data flow complexity:** Multi-step processing pipelines or cross-component flows
   - **Architecture changes:** New modules, significant refactoring, or integration points
   - **Complex relationships:** Non-obvious interactions between components
   - **Complex systems:** Algorithms with multiple phases, engines with interacting components (create separate architecture/algorithm/workflow files per Pattern 4)

   **Skip diagrams if:**
   - Simple CRUD operations only
   - Single model changes with no relationships
   - Minor bug fixes or refactoring
   - Pure test additions without implementation changes

2. **Generate diagrams following conventions:**

   Read and follow `docs/conventions/diagram-conventions.md` to:
   - Determine which diagram types are appropriate (ERD, sequence, component)
   - Create diagrams using proper Mermaid syntax
   - Apply formatting and styling conventions
   - Keep diagrams high-level and architectural

3. **Create diagram files in `docs/implemented/`:**

   - Separate files for each diagram type: `XXX_DIAGRAMS_ERD.md`, `XXX_DIAGRAMS_SEQUENCE.md`, `XXX_DIAGRAMS_ARCHITECTURE.md`
   - Single file if only one diagram: `XXX_DIAGRAMS.md`
   - Use same ticket number as the implemented ticket

4. **Commit diagrams:**

   - Use the `commit` skill
   - Commit message: `docs: add diagrams for ticket XXX`
   - Ensure diagrams are committed separately from implementation

5. **Report diagram creation:**

   ```
   ✓ Diagrams generated for Ticket XXX

   Created:
   - XXX_DIAGRAMS_ERD.md (entity relationships)
   - XXX_DIAGRAMS_SEQUENCE.md (processing flow)

   Diagrams committed and ready for PR.
   ```

   **If no diagrams were warranted:**
   ```
   ℹ Changes do not warrant architectural diagrams (simple/isolated changes).
   Proceeding to next ticket.
   ```

6. **Final announcement:**

   ```
   ✓✓ Ticket XXX fully complete!

   All phases complete:
   - [X] implementation and tests
   - [X] moved to docs/implemented/
   - [X] diagrams assessed and created (if needed)

   Next eligible ticket: [number]
   ```

## Critical Principles

1. **Make obvious decisions automatically** - Use the project principles in AGENT.md to make defensible choices; only ask user when trade-offs are significant or unclear
2. **TDD strictly** - Test first, then minimal implementation
3. **Follow acceptance criteria** - These define "done"
4. **Keep it simple** - No gold-plating, no extra features
5. **Document all decisions** - Both automatic and user-provided choices need clear rationale
6. **Commit incrementally** - Use the `commit` skill after logical milestones; never save all commits for the end
7. **Verify before moving** - All tests must pass
8. **Show planning, proceed automatically** - Print test plans and progress for transparency, but only stop for genuinely ambiguous decisions
9. **Generate diagrams for complexity** - After ticket completion, automatically assess if changes warrant architectural diagrams; create separate diagram files only when complexity justifies them

## Error Handling

**If tests fail after implementation:**
1. Debug the failing test
2. Fix the implementation
3. Re-run tests
4. If still failing after reasonable attempts, stop and report the issue
5. Do NOT move ticket to implemented until all tests pass

**If acceptance criteria can't be met:**
1. Stop and discuss with user
2. Determine if criteria needs adjustment
3. Update ticket if user approves
4. Document the change

**When to stop and ask for help:**
- Tests repeatedly fail and you can't determine root cause
- Acceptance criteria appears impossible to meet with current approach
- Technical decision has significant trade-offs with no clear winner
- Unexpected requirement conflicts discovered
- Implementation reveals fundamental design issue

## Usage

```
Agent: I'll implement the next todo ticket using the implementation skill.

[Agent follows phases 1-6 interactively with user]
```
