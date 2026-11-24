---
name: Commit Changes
description: Enforce project commit conventions by analyzing changes, planning commit structure, and executing proper commits following docs/dev-processes/commit-process.md
---

# Commit Changes Skill

## Purpose
Ensure all commits follow project conventions by forcing a structured review process before any commit is made.

**Key principle:** Show planning steps for transparency, but proceed automatically unless there's a genuine problem that requires user input.

## Instructions

### Phase 1: Read Commit Conventions

**FIRST:** Read the commit conventions document:

```bash
Read: docs/dev-processes/commit-process.md
```

Review especially:
- When to commit (lines 10-58)
- Staging strategy (lines 86-100)
- Message format (lines 105-165)
- Domain-based commit breakdown (lines 224-330)
- Pre-commit checklist (lines 564-578)
- Golden rules (lines 606-610)

### Phase 2: Analyze What Changed

Run these commands to understand the changes:

```bash
git status
git diff --stat
git diff
```

**List out:**
1. What files changed
2. What was added/modified/deleted
3. Are tests mixed with code?
4. Are multiple unrelated features mixed?

### Phase 3: Run Tests

**CRITICAL:** Never commit if tests are failing.

Run the test command listed in the **Commands** section of `AGENT.md`.

**If tests fail:** STOP. Fix failures before proceeding.

**If tests pass:** Continue to Phase 4.

### Phase 4: Plan Commit Structure

Based on what changed, plan how to break this into proper commits.

**Apply these rules (from commit-process.md):**

1. **ALWAYS separate tests from code** (lines 96-99, 219)
   - Tests go in their own commit with message: `test: add tests for [feature]`
   - Code goes in separate commit(s): `feat: implement [feature]`

2. **Separate by domain boundaries** (lines 308-330)
   - One commit per logical domain/feature
   - Ask: "Could this be reverted independently?" → Separate commit
   - Ask: "Does this serve a distinct purpose?" → Separate commit

3. **Documentation separate** (line 555-559)
   - Documentation updates: `docs: [what was documented]`

**Create a commit plan:**
```
Commit 1: test: [description]
  Files: [list test files]

Commit 2: feat: [description]
  Files: [list implementation files]

Commit 3: feat: [description]
  Files: [list other domain files]

Commit 4: docs: [description]
  Files: [list doc files]
```

**Show this plan** for transparency, then proceed to Phase 5.

### Phase 5: Pre-Commit Checklist

For EACH planned commit, verify:

- [ ] Tests pass
- [ ] No debug code (print statements, console.log, breakpoints)
- [ ] No commented-out code
- [ ] No sensitive data (passwords, keys, tokens)
- [ ] Changes belong to one logical domain
- [ ] Test commits only contain test files
- [ ] Code commits only contain application files
- [ ] Message is one line, present tense
- [ ] No ticket numbers in message
- [ ] Could explain in one sentence to teammate

**If tests fail:** Stop and report the failure - this needs user attention.

**If other items fail:** Fix automatically (remove debug code, separate files properly, etc.) and continue.

### Phase 6: Execute Commits

For each commit in the plan:

**Step 1: Stage the specific files**
```bash
git add [specific files for this commit]
```

**Step 2: Verify what's staged**
```bash
git diff --cached --stat
```

**Step 3: Write the commit message**

**Format:**
```
<type>: <short description in present tense>
```

**Types:**
- `test`: Tests only (NEVER mixed with code)
- `feat`: New feature or application code
- `fix`: Bug fix
- `refactor`: Code improvement (no new features)
- `docs`: Documentation only
- `chore`: Dependencies, config

**Rules:**
1. Present tense: "add feature" not "added feature"
2. One line only (NO multi-line)
3. No ticket numbers
4. Be specific: "implement order line-item totaling" not "add stuff"

**Step 4: Execute the commit**
```bash
git commit -m "[type]: [description]"
```

**Step 5: Verify the commit**
```bash
git log -1
git show --stat
```

**Check:**
- Message is one line
- Only intended files included
- No mixed test/code files
- Makes sense as standalone commit

**Step 6: Repeat for next commit**

Continue until all planned commits are done.

### Phase 7: Final Verification

After all commits:

```bash
# Review commit history
git log --oneline -5
```

Run the test command from `AGENT.md` to verify tests still pass.

**Check:**
- All commits follow conventions
- Each commit is focused and reviewable
- Tests pass
- Nothing left uncommitted (unless intentional)

## Common Patterns

### Pattern 1: Test-Driven Development Cycle

```bash
# Commit 1: Tests first
git add <module>/tests/test_*.py   # or equivalent test path for this project
git commit -m "test: add tests for [feature] implementation"

# Commit 2: Implementation
git add <module>/<files>
git commit -m "feat: implement [feature description]"
```

### Pattern 2: New Module with Data Models

```bash
# Commit 1: Tests
git add <module>/tests/
git commit -m "test: add tests for [module] models"

# Commit 2: Models / core data structures
git add <module>/models.py   # or equivalent for this project's stack
git commit -m "feat: implement [module] data models"

# Commit 3: Any schema or migration files (separate from models)
git add <module>/migrations/   # if applicable
git commit -m "feat: add database migrations for [module]"
```

### Pattern 3: Multiple Features Added

```bash
# Commit 1: Tests for feature A
git add tests/test_feature_a.py
git commit -m "test: add tests for feature A"

# Commit 2: Feature A implementation
git add app/feature_a.py
git commit -m "feat: implement feature A"

# Commit 3: Tests for feature B
git add tests/test_feature_b.py
git commit -m "test: add tests for feature B"

# Commit 4: Feature B implementation
git add app/feature_b.py
git commit -m "feat: implement feature B"
```

## Red Flags

**STOP and ask for help if:**

❌ Tests are failing and you can't determine the cause
❌ Sensitive data detected (passwords, keys, tokens)
❌ Uncertainty about which domain a change belongs to
❌ Can't determine how to split a complex change properly

**FIX automatically if:**

⚠️ Test files and code files staged together → Re-plan commits to separate them
⚠️ Multiple unrelated features staged together → Re-plan commits to separate them
⚠️ Commit message has "and" multiple times → Split into multiple commits
⚠️ Commit message spans multiple lines → Rewrite as single line
⚠️ Commit message includes ticket numbers → Remove ticket numbers
⚠️ Debug code still present → Remove it

## Example: Good vs Bad

### ❌ BAD (violates conventions):
```bash
git add app/processor.py \
        tests/test_processor.py \
        tests/test_validator.py \
        docs/todo/008_IMPLEMENTATION_STATUS.md

git commit -m "docs: update Ticket 008 status to 60% complete with 27 passing tests"
```

**Problems:**
- Mixed tests and code
- Multiple features in one commit
- Message focuses on documentation not implementation
- Vague about what was actually built

### ✅ GOOD (follows conventions):
```bash
# Commit 1
git add tests/test_processor.py
git commit -m "test: add processing pipeline tests"

# Commit 2
git add app/processor.py
git commit -m "feat: implement order processing pipeline"

# Commit 3
git add tests/test_validator.py
git commit -m "test: add input validation tests"

# Commit 4
git add app/validator.py
git commit -m "feat: implement order input validation"

# Commit 5
git add docs/todo/008_IMPLEMENTATION_STATUS.md
git commit -m "docs: update Ticket 008 implementation progress"
```

**Why good:**
- Tests separated from code
- Each commit is one focused feature
- Messages describe what was built
- Each commit is independently reviewable

## Usage

This skill should be used:
- At the end of every TDD cycle
- When work reaches a stable checkpoint
- Before switching tasks
- At the end of implementing a ticket milestone

**Autonomy principle:** This skill plans, shows the plan for transparency, then executes automatically. Only stop if tests fail or there's genuine ambiguity about how to structure commits.

**Never skip this skill.** Proper commits are part of code quality, not an afterthought.

## Integration with Implement Ticket Skill

The `implement-ticket` skill should invoke this skill:
- After each TDD cycle completes
- After each implementation milestone
- At the end of ticket implementation

This ensures commits happen incrementally with proper structure, not all at once at the end.
