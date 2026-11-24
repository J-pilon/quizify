# Commit Process Guide

## Purpose
Define when and how to commit changes to maintain a clear, useful project history that helps future developers understand what was done and why.

## When to Commit

### ✅ Commit When:

1. **Tests are written and passing**
   - You've added new tests
   - All tests pass
   - Message: `test: add tests for [feature]`

2. **Application code is implemented**
   - Feature works as intended
   - Tests pass
   - No broken functionality
   - Message: `feat: implement [feature description]`

3. **A bug is fixed**
   - The bug is resolved
   - All tests pass
   - Message: `fix: resolve [bug description]`

4. **Refactoring is complete**
   - Code is cleaner/better organized
   - All tests still pass
   - No behavior changes
   - Message: `refactor: simplify [what was refactored]`

5. **Documentation is updated**
   - README, docstrings, or comments added/updated
   - Message: `docs: update [what was documented]`

6. **A ticket milestone is reached**
   - Part of a ticket is complete and stable
   - All tests for that part pass
   - Message: `feat: implement [milestone description]`

7. **A ticket is fully complete**
   - All acceptance criteria met
   - All tests passing
   - Ticket moved to docs/implemented/
   - Message: `feat: complete [ticket name]`

8. **Before switching context**
   - About to work on something completely different
   - Current work is in a stable state (even if incomplete)
   - Message: `wip: [describe current state]`

### ❌ Don't Commit When:

1. Tests are failing (unless it's intentional WIP)
2. Code doesn't run
3. Work is incomplete AND unstable
4. Multiple unrelated changes are mixed together

---

## How to Commit

### Step 1: Review Your Changes

```bash
# See what you've changed
git status

# Review the actual changes
git diff

# Check for unintended changes (debug code, temp files, etc.)
```

**Ask yourself:**
- Did I accidentally include debug code?
- Are there any print statements or debug logs to remove?
- Are all changes related to the same logical unit?

### Step 2: Stage Your Changes

```bash
# Stage specific files
git add path/to/file1 path/to/file2

# Or stage all changes (be careful!)
git add .

# Remove accidentally staged files
git reset path/to/file
```

**Staging Strategy:**
- If multiple unrelated changes exist, stage and commit separately
- Group related changes by domain (one domain = one commit)
- Don't mix feature code with test code in the same commit
- Don't mix data models with generated schema/migration files in the same commit
- Don't mix framework setup with specific implementations
- Don't mix feature code with formatting changes
- When in doubt: smaller, more focused commits are better

### Step 3: Write a Clear Commit Message

#### Message Format:

```
<type>: <short description>
```

**That's it. Keep it simple.**

#### Types:

- `feat`: New feature or application code
- `fix`: Bug fix
- `test`: Adding or updating tests (ONLY test files)
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `docs`: Documentation changes
- `chore`: Maintenance tasks (dependencies, config, etc.)
- `wip`: Work in progress (use sparingly)

#### Critical Rules:

1. **Use present tense**: "add feature" not "added feature"
2. **Separate test commits from code commits**: Never mix them
3. **No ticket numbers in the message**: Just describe what you did
4. **One line only**: No multi-line messages, no footers, no trailers, no 'Co-authored-by: Cursor <cursoragent@cursor.com>', no 'Co-authored-by: <anyone>'
5. **Be specific**: "implement order line-item totaling" not "add stuff"
6. **NEVER create merge commits with new content**: Never use `git merge --no-commit` to add new work—merge commits must only combine existing branches without introducing new files or changes

#### Examples:

**Good commit messages:**
```bash
feat: add user authentication

feat: implement order line-item totaling

test: add tests for invoice generation

test: verify order immutability after fulfillment

fix: correct precision calculation for edge case

refactor: extract pricing logic into separate module

docs: update README with local setup instructions

chore: add linting configuration
```

**Bad commit messages:**
```bash
"feat(ticket-001): added feature"     # No ticket numbers, wrong tense
"update stuff"                        # Too vague
"fix bug"                             # Which bug?
"WIP"                                 # What work?
"test: add tests and implement code"  # Mixed test and code
"feat: adds new parser
Implemented the new parser with
these features:
- feature 1
- feature 2"                          # No multi-line, no trailers
```

### Step 4: Make the Commit

```bash
# Commit with message (one line)
git commit -m "feat: add user authentication"
```

**Never use multi-line commits.** If you can't describe it in one line, break it into smaller commits.

### Step 5: Verify the Commit

```bash
# View your last commit
git log -1

# Or with changes
git show

# Check you're on the right branch
git branch
```

---

## Commit Workflow Pattern

### The Typical TDD Cycle:

1. **Write tests** → Commit:
   ```bash
   git add tests/
   git commit -m "test: add tests for order processing"
   ```

2. **Implement code** → Commit by domain:
   ```bash
   # If implementing one focused component:
   git add app/order_processor.py   # or equivalent for this project's stack
   git commit -m "feat: implement order processing"

   # If multiple files form one cohesive domain concept:
   git add app/models/order.py app/serializers/order.py
   git commit -m "feat: implement order data model"
   ```

3. **Refactor if needed** → Commit:
   ```bash
   git add app/order_processor.py
   git commit -m "refactor: simplify order processing logic"
   ```

**Notice:** Tests and code are NEVER in the same commit.

---

## Breaking Down Large Features by Domain

When implementing a new module or large feature, break commits into logical domains rather than committing everything at once.

### Key Separations:

**1. Data models before generated files (ALWAYS separate):**
```bash
# First: commit the model/schema
git add app/models/invoice.py
git commit -m "feat: implement invoice data model"

# Second: commit generated files separately (migrations, schemas, etc.)
git add app/migrations/
git commit -m "feat: add database migrations for invoice model"
```

**Why separate?** Models are hand-written logic; generated files are artifacts. They serve different purposes in code review and can be rolled back independently.

**2. Configuration separate from feature code:**
```bash
# Configuration often goes with the core framework commit
git add app/config.py app/settings.py
git commit -m "feat: implement notification framework core"
```

**3. Core infrastructure before specific implementations:**

Commit in dependency order:
- Base classes → Concrete implementations
- Utilities/helpers → Code that uses them
- Foundation → Features built on it

### Domain-Based Commit Examples:

**Example: Adding a new module with multiple components**

```bash
# Commit 1: Tests first (always)
git add app/invoices/tests/
git commit -m "test: add invoice generation tests"

# Commit 2: Core data models
git add app/invoices/models.py
git commit -m "feat: implement invoice data models"

# Commit 3: Generated files (if applicable)
git add app/invoices/migrations/
git commit -m "feat: add database migrations for invoice models"

# Commit 4: Base architecture
git add app/invoices/base.py app/invoices/registry.py
git commit -m "feat: implement invoice generator base classes and registry"

# Commit 5: Shared utilities
git add app/invoices/formatters.py
git commit -m "feat: implement currency and date formatting utilities"

# Commit 6-N: Domain-specific implementations (one domain per commit)
git add app/invoices/generators/standard.py
git commit -m "feat: implement standard invoice generator"

git add app/invoices/generators/proforma.py
git commit -m "feat: implement proforma invoice generator"
```

### What Belongs Together vs. Apart:

**DO group together in one commit:**
- Files implementing one cohesive domain concept (e.g., all pricing-related logic)
- A base class + its `__init__.py` that exports it
- Tightly coupled files that don't make sense separately
- Configuration changes needed for the feature to work

**DON'T group together (separate commits):**
- Generated files (migrations, schemas) + the models they're based on
- Multiple unrelated processors/validators/formatters
- Test files + application code
- Framework setup + domain-specific logic
- Core infrastructure + features built on it

### Signs You Need Multiple Commits:

Split into multiple commits if you're:
- Adding both hand-written models AND generated artifacts (always separate these)
- Implementing multiple unrelated processors/validators/handlers
- Adding framework setup AND specific feature implementations
- Creating 10+ files (usually means multiple logical domains)
- Struggling to write a one-line commit message that covers everything

### Finding Domain Boundaries:

Ask yourself:
- **"Could this be reverted independently?"** → Separate commit
- **"Does this serve a distinct purpose?"** → Separate commit
- **"Would a reviewer understand this grouping?"** → Good commit boundary
- **"Is this generated code + hand-written code?"** → Separate commits

---

## Commit Frequency Guidelines

### Good Frequency:
- **After each complete TDD cycle** (separate test and code commits)
- **At natural breakpoints** (end of function, end of class, etc.)
- **When switching tasks** (if current work is stable)

### Signs You're Not Committing Enough:
- Hours pass without commits
- Commit messages say "implement entire feature"
- Can't remember everything you changed
- Afraid to commit because too much changed

### Signs You're Committing Too Much in One Commit:
- Commit includes both hand-written models and generated artifacts
- Commit mixes tests and application code
- Commit includes 10+ unrelated files
- Commit message needs "and" multiple times ("implement X and Y and Z")
- Commit includes both framework setup and specific features
- Reviewing the commit requires understanding multiple domains

---

## Special Cases

### Work in Progress (WIP)

Sometimes you need to commit unfinished work:

```bash
git commit -m "wip: partial implementation of invoice generator"
```

**When to use WIP:**
- End of day but work isn't done
- Need to switch branches urgently
- Want to save current state before trying something risky

**WIP Rules:**
- Keep message short and descriptive
- Never push WIP to main branch
- Clean up WIP commits before merging (squash them)

### Fixing a Commit Message

```bash
# Oops, typo in last commit message
git commit --amend -m "feat: add user authentication"

# Add forgotten file to last commit
git add forgotten_file.py
git commit --amend --no-edit
```

**Warning:** Only amend commits that haven't been pushed!

### Splitting Large Changes

If you forgot to commit and have many changes:

```bash
# Stage changes interactively
git add -p

# This lets you stage specific parts of files
# Stage related parts together, commit, repeat
```

---

## ⛔ FORBIDDEN PATTERN: Merge Commits with New Content

### What This Means

**NEVER use `git merge --no-commit <branch>` to add new work while merging.**

A merge commit should **ONLY** combine changes that already exist in its parent branches. It must **NEVER** introduce new files or changes that don't exist in either parent.

### Why This Is Forbidden

This pattern creates three serious problems:

1. **Impossible to understand history**: Looking at the merge commit, you can't tell what came from which parent vs. what was added during the merge
2. **Breaks git tools**: Rebase, cherry-pick, and bisect all fail because the merge introduces work that has no linear history
3. **Review nightmare**: Reviewers can't distinguish between "merged existing work" and "added new work during merge"

### Correct Patterns

#### Pattern 1: Commit New Work First, Then Merge

```bash
# ✅ CORRECT: Add new work as regular commits first
git checkout feature-branch
git add app/processor.py
git commit -m "feat: implement processing pipeline"

# Now merge if needed
git merge main
```

#### Pattern 2: Stay on Your Branch, No Merge Needed

```bash
# ✅ CORRECT: Just continue working on your feature branch
git checkout feature-branch
git add new_file.py
git commit -m "feat: implement new feature"

# Merge happens later via PR or explicit merge
```

#### Pattern 3: Rebase Instead of Merge

```bash
# ✅ CORRECT: Rebase to incorporate main's changes
git checkout feature-branch
git rebase main

# Then add your new work as regular commits
git add new_file.py
git commit -m "feat: implement new feature"
```

### The Golden Rule

**Merge commits are for merging. Regular commits are for adding work. Never mix them.**

---

## Ticket-Based Commit Strategy

### During Ticket Implementation:

A ticket often results in multiple commits as you build up functionality layer by layer.

**Pattern: Tests → Core → Specifics → Documentation**

1. **Tests first (all at once or incrementally):**
   ```bash
   test: add tests for invoice management
   ```

2. **Core models and schemas:**
   ```bash
   feat: implement invoice data models
   ```

3. **Generated artifacts (separate from models):**
   ```bash
   feat: add database migrations for invoice models
   ```

4. **Base functionality:**
   ```bash
   feat: implement line-item total calculation
   ```

5. **Additional features by domain:**
   ```bash
   feat: implement invoice versioning
   ```

6. **More domain-specific features:**
   ```bash
   feat: implement version immutability enforcement
   ```

7. **Completing ticket (documentation):**
   ```bash
   docs: complete invoice management ticket
   ```

**Key principle:** Each commit should represent one complete, logical step. Tests are separate. Models and generated artifacts are separate. Domain-specific features are separate.

---

## Pre-Commit Checklist

Before every commit, verify:

- [ ] Tests pass (run the test command from `AGENT.md`)
- [ ] No debug code (print statements, breakpoints, etc.)
- [ ] No commented-out code (remove it)
- [ ] No sensitive data (passwords, keys, tokens)
- [ ] Changes belong to one logical domain/concept
- [ ] Test commits only contain test files
- [ ] Hand-written models and generated artifacts are in separate commits
- [ ] Code commits only contain application files
- [ ] Commit message is one line, present tense
- [ ] No ticket numbers in commit message
- [ ] Could explain this commit in one sentence to a teammate

---

## Summary: The Simple Version

**When?** After tests pass and a logical unit is complete. Always separate test commits from code commits.

**How?**
1. `git status` - see what changed
2. `git add <files>` - stage related changes
3. `git commit -m "type: description"` - commit with clear message
4. `git log -1` - verify it looks right

**Message Format:**
```
<type>: <what you do in present tense>
```

**Examples:**
- `test: add tests for order processing`
- `feat: implement order line-item totaling`
- `fix: correct tax calculation for edge case`
- `refactor: simplify pricing logic`
- `docs: update setup instructions`

**Golden Rules:**
1. Present tense always
2. One line only
3. Tests and code never in same commit
4. Hand-written code and generated artifacts never in same commit
5. No ticket numbers in message
6. Break large features into domain-based commits
7. Commit when you'd be happy explaining what you did to a teammate

**Common Pattern:**
```bash
# First: tests only
git add tests/
git commit -m "test: add tests for feature X"

# Second: code by domain (not all at once)
git add app/models.py
git commit -m "feat: implement X data model"

git add app/feature_x.py
git commit -m "feat: implement X feature"
```
