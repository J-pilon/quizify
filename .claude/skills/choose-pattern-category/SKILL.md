---
name: choose-pattern-category
description: Determine which design pattern category (creational, structural, or behavioral) best addresses a given problem. Use when the user asks about design patterns, needs help choosing a pattern approach, or wants to solve an architecture or design problem.
---

# Choose Design Pattern Category

This skill helps identify which category of design patterns to explore based on the nature of your problem.

## Quick Decision Guide

Answer this primary question:

**What is the core challenge you're solving?**

1. **Creating objects** → **Creational Patterns**
2. **Composing objects or classes** → **Structural Patterns**
3. **Communication between objects** → **Behavioral Patterns**

## Detailed Decision Tree

### When to use CREATIONAL patterns

**Primary focus:** Object instantiation and creation mechanisms

Use when your problem involves:
- Deciding **which** objects to create
- Deciding **when** to create objects
- Deciding **how** to create objects
- Managing object creation complexity
- Controlling the number of instances
- Hiding construction complexity from clients

**Key questions that indicate creational patterns:**
- "How do I create variations of similar objects?"
- "How do I control how many instances exist?"
- "How do I simplify creating complex objects?"
- "How do I create objects without specifying exact classes?"
- "How do I defer object creation until needed?"
- "How do I create families of related objects?"

**Examples:**
- Creating different report formatters based on output type (PDF, CSV, JSON)
- Building complex query configurations with many optional parameters
- Managing API client instantiation for multiple third-party services
- Creating test fixtures with varying configurations

**→ Explore:** `docs/conventions/design-patterns/creational-patterns.md`

---

### When to use STRUCTURAL patterns

**Primary focus:** How objects and classes are composed to form larger structures

Use when your problem involves:
- Relationships between objects
- Object composition and assembly
- Simplifying complex interfaces
- Adapting incompatible interfaces
- Managing object references and memory
- Adding functionality to existing objects

**Key questions that indicate structural patterns:**
- "How do I make incompatible interfaces work together?"
- "How do I add functionality without modifying objects?"
- "How do I simplify a complex subsystem?"
- "How do I treat individual and composed objects uniformly?"
- "How do I control access to an object?"
- "How do I share data efficiently across many objects?"
- "How do I separate abstraction from implementation?"

**Examples:**
- Integrating multiple payment gateway APIs with a common interface
- Wrapping third-party data parsers with a consistent API surface
- Building hierarchies of nested items (order → line items → adjustments)
- Adding caching or logging to existing components without modifying them
- Simplifying a complex reporting subsystem behind a single entry point

**→ Explore:** `docs/conventions/design-patterns/structural-patterns.md`

---

### When to use BEHAVIORAL patterns

**Primary focus:** Communication, responsibility distribution, and algorithms

Use when your problem involves:
- How objects interact and communicate
- How responsibilities are distributed
- How algorithms are encapsulated
- Managing object state and transitions
- Implementing undo/redo functionality
- Processing chains or pipelines

**Key questions that indicate behavioral patterns:**
- "How do I pass requests through a chain of handlers?"
- "How do I encapsulate requests as objects?"
- "How do I provide different algorithms interchangeably?"
- "How do I manage object state transitions?"
- "How do I notify multiple objects of changes?"
- "How do I traverse complex data structures?"
- "How do I coordinate complex interactions between objects?"
- "How do I define a template for an algorithm?"
- "How do I add operations without modifying classes?"

**Examples:**
- Implementing processing pipelines with multiple stages (validate → enrich → persist)
- Managing workflow states (draft → submitted → approved → fulfilled)
- Switching between pricing strategies (flat rate, tiered, volume discount)
- Coordinating interactions between parser, validator, and persistence components
- Monitoring job progress across multiple worker components
- Traversing nested data hierarchies (order → line items → adjustments)
- Applying validation rules in sequence

**→ Explore:** `docs/conventions/design-patterns/behavioral-patterns.md`

---

## Diagnostic Questions

Still unsure? Answer these questions about your problem:

### Question 1: What changes?
- **Object creation logic changes** → Creational
- **Object structure or composition changes** → Structural
- **Object interaction or algorithm changes** → Behavioral

### Question 2: What's the primary pain point?
- **Too many constructors or complex initialization** → Creational
- **Incompatible interfaces or complex subsystems** → Structural
- **Complex conditionals based on state or type** → Behavioral

### Question 3: What's your goal?
- **Control object creation** → Creational
- **Simplify object relationships** → Structural
- **Coordinate object behavior** → Behavioral

## Pattern Category Comparison

| Category | Focuses On | Typical Verbs | Common Triggers |
|----------|------------|---------------|-----------------|
| **Creational** | Object creation | Create, Build, Construct, Instantiate | "new", constructors, initialization |
| **Structural** | Object relationships | Compose, Wrap, Adapt, Bridge | interfaces, hierarchies, subsystems |
| **Behavioral** | Object interaction | Communicate, Coordinate, Process, Traverse | algorithms, state, workflows |

## Multiple Categories?

Some problems benefit from combining patterns from different categories:

**Creational + Structural:**
- Factory (Creational) that creates Decorators (Structural)
- Builder (Creational) that assembles Composite (Structural) structures

**Structural + Behavioral:**
- Composite (Structural) with Visitor (Behavioral) for traversal
- Proxy (Structural) that implements Strategy (Behavioral) for access control

**All Three:**
- Factory (Creational) creates Strategies (Behavioral) that are wrapped in Decorators (Structural)

When multiple categories apply, start with the most fundamental problem (usually Creational or Structural), then layer on behavioral patterns.

## Usage Example

**Scenario:** "I need to process records through validation, enrichment, and persistence stages."

**Analysis:**
- Core challenge: Passing records through sequential handlers
- Focus: How handlers communicate and process requests
- Category: **Behavioral**
- Likely pattern: **Chain of Responsibility** or **Command**

**Next step:** Read `behavioral-patterns.md` and review Chain of Responsibility pattern.

---

**Scenario:** "I need to create report builders for different output formats with lots of configuration options."

**Analysis:**
- Core challenge: Complex object creation with many parameters
- Focus: Construction mechanism
- Category: **Creational**
- Likely patterns: **Builder** or **Factory Method**

**Next step:** Read `creational-patterns.md` and compare Builder vs Factory Method.

---

**Scenario:** "I need to make three different payment gateway APIs work with the same interface in my code."

**Analysis:**
- Core challenge: Incompatible interfaces need to work together
- Focus: Interface compatibility
- Category: **Structural**
- Likely pattern: **Adapter** or **Facade**

**Next step:** Read `structural-patterns.md` and review Adapter pattern.

## When NOT to Use Design Patterns

Patterns add complexity. Skip them when:
- The problem is simple and direct code is clearer
- You'd only use it once with no variation
- The code would be harder to understand with the pattern
- You're solving a hypothetical future problem (YAGNI)

**Remember:** Simple, clear code beats pattern-based code when the pattern doesn't pull its weight.

## Next Steps

1. Identify your pattern category using this guide
2. Read the corresponding convention document
3. Review 2-3 patterns that seem relevant
4. Compare their trade-offs against your specific needs
5. Choose the simplest pattern that solves your problem
6. Implement with tests to validate the design
