# Testing Conventions

## Purpose

This document defines testing conventions for this project. Tests should be focused on **what is under test**, not setup logic. Factory functions and test helpers enable clean, maintainable tests.

## Core Principles

### 1. Tests Focus on Assertions, Not Setup

The body of a test should be almost entirely assertions. Setup noise belongs in factory functions.

**Bad:**
```
test_order_totaling:
    # 15 lines creating line items, tax tables, currency objects…
    item1 = create raw item with price 10.00, qty 2, tax 0.08
    item2 = create raw item with price 5.00, qty 1, tax 0.08
    order = create raw order, add item1, add item2, finalize

    # Finally, the actual test
    assert order.total == 25.00
```

**Good:**
```
test_order_totaling:
    item1 = make_line_item(price=10.00, qty=2)
    item2 = make_line_item(price=5.00, qty=1)
    order = make_order(items=[item1, item2])

    assert order.total == 25.00
```

### 2. Lightweight Factory Functions Over Heavy Dependencies

Use **simple factory functions** rather than external factory frameworks when possible:
- Zero or few extra dependencies
- Easy to understand and modify
- Composable: call one factory from another

## Test Organization Structure

### Directory Layout

```
<module>/
  tests/
    <entity>_factories.py      # Factory functions for creating instances
    test_helpers.py             # Reusable helper functions and assertions
    test_models.py              # Unit tests for data models
    test_integration.py         # Integration tests
    test_<feature>.py           # Feature-specific tests
```

### Naming Conventions

1. **Factory files:** `<entity>_factories.py` (e.g., `order_factories.py`, `invoice_factories.py`)
   - **Not** a generic `factories.py` — be specific about what's inside

2. **Factory functions:** `make_<entity>(...)` or `create_<entity>(...)`, applied consistently within a project

## Factory Functions

### Purpose

Factory functions abstract away the boilerplate of creating test instances, letting tests focus on the behavior being verified.

### Guidelines

#### 1. Simple, Focused Functions

Each factory does one thing well:

```
# Pseudocode
function make_line_item(sku="ITEM-001", price=9.99, qty=1):
    """Create a LineItem with sensible defaults."""
    return LineItem(sku=sku, unit_price=price, quantity=qty)
```

#### 2. Sensible Defaults with Flexibility

Provide defaults but allow customization:

```
# Pseudocode
function make_order(customer=None, items=None, status="draft"):
    """Create an Order instance.

    Args:
        customer: Optional Customer. Creates a default one if omitted.
        items: List of LineItems. Creates one default item if omitted.
        status: Initial order status.
    """
    if customer is None:
        customer = make_customer()
    if items is None:
        items = [make_line_item()]
    return Order(customer=customer, line_items=items, status=status)
```

**Usage:**
```
# Minimal — all defaults
order = make_order()

# Customized
order = make_order(status="fulfilled", items=[make_line_item(qty=5)])
```

#### 3. Compose Factories for Complex Scenarios

Build specialized factories from simpler ones:

```
# Pseudocode
function make_fulfilled_order(item_count=2):
    """Create an order that has already been fulfilled."""
    items = [make_line_item() for _ in range(item_count)]
    order = make_order(items=items, status="fulfilled")
    order.save()
    return order

function make_invoice_for_order(order=None):
    """Create an Invoice linked to a fulfilled order."""
    if order is None:
        order = make_fulfilled_order()
    return Invoice(order=order)
```

#### 4. Document Factory Purpose

Include docstrings explaining what the factory does and when to use it:

```
# Pseudocode
function make_versioned_invoice(version=1, line_count=3):
    """Create an Invoice with a specific version number and line items.

    Args:
        version: Invoice version number
        line_count: Number of line items to attach

    Returns:
        (invoice, line_items) tuple

    Example:
        invoice, lines = make_versioned_invoice(version=2, line_count=5)
    """
```

## Test Helper Functions

### Purpose

Helpers extract reusable logic that doesn't fit into factories:
- Boolean checks (`is_<state>()`)
- Assertion helpers
- Common test patterns
- Utility functions used across multiple tests

### Location

Create `test_helpers.py` in the module's test directory.

### Examples

```
# Pseudocode — test_helpers.py

function is_locked(invoice):
    """Check if invoice is in a locked (non-editable) state."""
    return invoice.status == "locked"

function assert_line_items_equal(items_a, items_b, ignore_fields=None):
    """Assert two lists of line items match, optionally ignoring fields."""
    assert len(items_a) == len(items_b)
    ignore = ignore_fields or []
    for a, b in zip(items_a, items_b):
        for field in a.fields:
            if field not in ignore:
                assert a[field] == b[field], f"Field '{field}' doesn't match"
```

### When to Use Helpers vs. Factories

- **Factory:** Creates instances for testing
- **Helper:** Performs checks, assertions, or transformations on existing instances

## Testing Patterns

### Pattern 1: Arrange-Act-Assert with Factories

```
test_fulfilled_order_prevents_modification:
    # Arrange: use factory to reach the desired starting state
    order = make_fulfilled_order()
    new_item = make_line_item()

    # Act & Assert: focus on the behavior
    assert raises(ImmutableOrderError):
        order.add_item(new_item)
```

### Pattern 2: Shared Setup

For tests sharing common preconditions, initialize shared data in a setup hook:

```
class TestInvoiceVersioning:
    setup:
        self.order = make_fulfilled_order()
        self.invoice = make_invoice_for_order(self.order)

    test_initial_version_is_one:
        assert self.invoice.version == 1

    test_amendment_increments_version:
        self.invoice.amend(reason="price correction")
        assert self.invoice.version == 2
```

### Pattern 3: Parameterized Tests for Multiple Scenarios

```
@parameterize([
    ("standard",  10.00, 0.10, 11.00),
    ("exempt",     5.00, 0.00,  5.00),
    ("reduced",   20.00, 0.05, 21.00),
])
test_tax_calculation(order_type, subtotal, rate, expected_total):
    order = make_order(type=order_type, subtotal=subtotal, tax_rate=rate)
    assert order.total == expected_total
```

## Import Conventions

### Organize Imports Clearly

```
# tests/test_invoices.py  (example for Python — adapt import style to your stack)

# Standard library / test framework
import <your_test_framework>   # e.g. pytest, jest, rspec

# Module under test
from invoices.models import Invoice

# Factories and helpers
from invoices.tests.invoice_factories import make_invoice_for_order, make_line_item
from invoices.tests.test_helpers import is_locked
```

### Cross-Module Factory Usage

Factories can be imported across modules:

```
# In invoice tests, use order factories
from orders.tests.order_factories import make_fulfilled_order

function make_invoice_with_order(version=1):
    order = make_fulfilled_order()
    return Invoice(order=order, version=version)
```

## Common Factory Examples

### User / Auth Factories

```
# Pseudocode
function make_admin_user(username="admin"):
    return User.create(username=username, role="admin")

function make_regular_user(username="user"):
    return User.create(username=username, role="user")
```

### File / Attachment Factories

```
# Pseudocode
function make_temp_file(content="test content", suffix=".txt"):
    """Create a temporary file on disk. Caller responsible for cleanup."""
    path = tempfile.mktemp(suffix=suffix)
    write(path, content)
    return path
```

## Testing Anti-Patterns to Avoid

### ❌ Inline Test Data Creation

```
# DON'T: Repeat setup logic in every test
test_something:
    item = LineItem()
    item.sku = "ITEM-001"
    item.unit_price = 9.99
    item.quantity = 1
    item.save()
    order = Order()
    order.add_item(item)
    order.save()
    # ...now finally test something
```

### ❌ Generic "Utils" Without Clear Purpose

```
# DON'T: Vague utility functions
function make_thing(**kwargs):   # What thing?
    pass

function do_setup():             # Setup for what?
    pass
```

### ❌ Over-Engineering Factories

```
# DON'T: Complex inheritance hierarchies for test factories
class BaseFactory(metaclass=FactoryMeta):
    class Meta:
        abstract = True
    # 50 lines of abstraction when a function would do
```

### ❌ Testing Implementation Details

```
# DON'T: Test private methods or internal state directly
test_internal_calculation:
    order = Order()
    result = order._recalculate_totals()   # private method
```

## Benefits

Following these conventions provides:

✅ **Readability** — Tests clearly show what's being tested
✅ **Maintainability** — Changes to setup happen in one place
✅ **Reusability** — Factories and helpers work across all tests
✅ **Speed** — Writing new tests is faster with existing factories
✅ **Focus** — Tests focus on behavior, not mechanics
✅ **Simplicity** — Lightweight approach that scales with the project

## Summary

1. **Use lightweight factory functions**, not heavy frameworks
2. **Name factory files specifically** (`order_factories.py` not `factories.py`)
3. **Extract common logic into test helpers**
4. **Tests should focus on assertions** — setup should be minimal
5. **Compose factories** for complex scenarios
6. **Document factory usage** with clear docstrings
7. **Keep it simple** — factories are tools, not architecture

## Examples

Stack: TypeScript + Vitest. Types come from `extension/src/lib/quiz/types.ts`. Files are kebab-case, so `<entity>_factories.py` becomes `<entity>-factories.ts`.

### Factory function

```ts
// extension/src/lib/quiz/tests/quiz-factories.ts
import type { Answer, Question, Quiz } from "../types";

let nextId = 0;
const uid = (prefix: string) => `${prefix}-${++nextId}`;

/** Create a multiple-choice Question with sensible defaults. */
export function makeQuestion(overrides: Partial<Question> = {}): Question {
  return {
    id: uid("q"),
    type: "multiple_choice",
    prompt: "What does the harness do?",
    choices: ["Routes messages", "Renders UI", "Stores files", "Signs builds"],
    correctAnswer: "Routes messages",
    explanation: "The harness routes messages between components.",
    sourceQuote: "The harness routes messages between components.",
    ...overrides,
  };
}

/** Create an Answer for a question; skipped answers carry no value. */
export function makeAnswer(
  question: Question,
  overrides: Partial<Answer> = {},
): Answer {
  return { questionId: question.id, value: question.correctAnswer, skipped: false, ...overrides };
}

/** Create an in-progress Quiz. Builds three default questions if none are given. */
export function makeQuiz(overrides: Partial<Quiz> = {}): Quiz {
  const questions = overrides.questions ?? [makeQuestion(), makeQuestion(), makeQuestion()];
  return {
    id: uid("quiz"),
    source: {
      url: "https://example.com/post",
      normalizedUrl: "https://example.com/post",
      title: "Example Post",
      contentHash: "0".repeat(64),
    },
    summary: "A short summary.",
    tags: ["agents"],
    settings: { count: questions.length, difficulty: "medium", types: ["multiple_choice"] },
    questions,
    answers: [],
    currentIndex: 0,
    status: "in_progress",
    startedAt: "2026-11-23T12:00:00.000Z",
    ...overrides,
  };
}

/** Compose: a Quiz where every question has been answered correctly. */
export function makeCompletedQuiz(questionCount = 3): Quiz {
  const questions = Array.from({ length: questionCount }, () => makeQuestion());
  return makeQuiz({
    questions,
    answers: questions.map((q) => makeAnswer(q)),
    currentIndex: questionCount - 1,
    status: "completed",
    completedAt: "2026-11-23T12:10:00.000Z",
  });
}
```

### Test helper

```ts
// extension/src/lib/quiz/tests/test-helpers.ts
import { expect } from "vitest";
import type { Quiz } from "../types";

/** True when the quiz has an answer (skipped or not) for every question. */
export function isFullyAnswered(quiz: Quiz): boolean {
  return quiz.questions.every((q) => quiz.answers.some((a) => a.questionId === q.id));
}

/** Assert two quizzes match, optionally ignoring top-level fields (e.g. ids, timestamps). */
export function expectQuizzesEqual(a: Quiz, b: Quiz, ignore: (keyof Quiz)[] = []): void {
  const strip = (quiz: Quiz) => {
    const copy: Partial<Quiz> = { ...quiz };
    ignore.forEach((field) => delete copy[field]);
    return copy;
  };
  expect(strip(a)).toEqual(strip(b));
}
```

### Using them together

```ts
// extension/src/lib/quiz/tests/grading.test.ts
import { describe, expect, it } from "vitest";
import { gradeLocally } from "../grading";
import { makeAnswer, makeQuestion, makeQuiz } from "./quiz-factories";
import { isFullyAnswered } from "./test-helpers";

describe("gradeLocally", () => {
  it("scores skipped questions as incorrect", () => {
    const [q1, q2] = [makeQuestion(), makeQuestion()];
    const quiz = makeQuiz({
      questions: [q1, q2],
      answers: [makeAnswer(q1), makeAnswer(q2, { value: undefined, skipped: true })],
    });

    expect(isFullyAnswered(quiz)).toBe(true);
    expect(gradeLocally(quiz).score).toBe(50);
  });
});
```
