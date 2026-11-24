# Behavioral Design Patterns

Behavioral patterns are concerned with algorithms and the assignment of responsibilities between objects. They describe not just patterns of objects or classes but also the patterns of communication between them.

---

## Chain of Responsibility

### Intent
Pass requests along a chain of handlers. Each handler decides either to process the request or to pass it to the next handler in the chain.

### Problem Space
- Multiple objects can handle a request, but the handler isn't known in advance
- You want to issue a request without knowing which object will handle it
- The set of handlers should be specified dynamically
- You need to execute several handlers in a specific order

### Solution
Link handler objects into a chain. Each handler has a reference to the next handler. When a request comes in, each handler either processes it or passes it along the chain.

**Structure:**
- Handler interface declares method for handling requests and optionally storing next handler
- Base Handler implements default chaining behavior
- Concrete Handlers contain actual request-processing code
- Client composes chains and initiates requests

### Trade-offs

**Benefits:**
- Control the order of request handling
- Single Responsibility Principle: decouple senders from receivers
- Open/Closed Principle: introduce new handlers without breaking existing code
- Handlers can be added or removed dynamically

**Drawbacks:**
- Some requests may go unhandled
- Can be hard to observe or debug the chain
- Performance concerns if chain is long

### Implementation Information

**When to use in this project:**
- Processing document validation rules in sequence
- Applying multiple text preprocessing steps
- Implementing evaluation pipelines with multiple stages
- Building middleware-style processing chains

**Example pattern:**
```python
from abc import ABC, abstractmethod
from typing import Optional

class DocumentHandler(ABC):
    """Handler interface."""
    
    def __init__(self):
        self._next_handler: Optional[DocumentHandler] = None
    
    def set_next(self, handler: 'DocumentHandler') -> 'DocumentHandler':
        """Set the next handler in the chain."""
        self._next_handler = handler
        return handler  # Allow chaining: h1.set_next(h2).set_next(h3)
    
    @abstractmethod
    def handle(self, document: dict) -> dict:
        """Process document or pass to next handler."""
        pass

class ValidationHandler(DocumentHandler):
    """Validate document structure."""
    
    def handle(self, document: dict) -> dict:
        if "content" not in document:
            raise ValueError("Missing content")
        
        # Pass to next handler
        if self._next_handler:
            return self._next_handler.handle(document)
        return document

class SanitizationHandler(DocumentHandler):
    """Sanitize document content."""
    
    def handle(self, document: dict) -> dict:
        document["content"] = document["content"].strip()
        
        if self._next_handler:
            return self._next_handler.handle(document)
        return document

class EnrichmentHandler(DocumentHandler):
    """Add metadata to document."""
    
    def handle(self, document: dict) -> dict:
        document["word_count"] = len(document["content"].split())
        
        if self._next_handler:
            return self._next_handler.handle(document)
        return document

# Usage: Build processing chain
validation = ValidationHandler()
sanitization = SanitizationHandler()
enrichment = EnrichmentHandler()

validation.set_next(sanitization).set_next(enrichment)

# Process through chain
result = validation.handle({"content": "  Sample input text  "})
# Result: validated, sanitized, enriched
```

**Project-specific guidance:**
- Useful for document processing pipelines
- Consider for validation chains with multiple rules
- Can implement evaluation preprocessing stages
- Each handler should be independently testable
- Consider using middleware patterns common to your framework

---

## Command

### Intent
Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

### Problem Space
- You want to parameterize objects with operations
- You need to queue operations, schedule their execution, or execute remotely
- You want to implement undo/redo functionality
- You need to log operations or implement transactions

### Solution
Turn requests into standalone objects containing all information about the request. This transformation lets you pass requests as method arguments, delay or queue execution, and support undoable operations.

**Structure:**
- Command interface declares execution method
- Concrete Commands implement various requests
- Receiver performs actual work
- Invoker triggers commands
- Client creates and configures concrete commands

### Trade-offs

**Benefits:**
- Single Responsibility Principle: decouple operation invokers from performers
- Open/Closed Principle: introduce new commands without changing existing code
- Implement undo/redo
- Implement deferred execution
- Assemble complex commands from simple ones

**Drawbacks:**
- Code complexity increases (layer between senders and receivers)
- Can lead to many small command classes

### Implementation Information

**When to use in this project:**
- Implementing evaluation workflows that can be scheduled or queued
- Creating undoable operations on test data
- Building command-line management commands with complex logic
- Implementing experiment macros (sequences of operations)

**Example pattern:**
```python
from abc import ABC, abstractmethod
from typing import List

# Command interface
class Command(ABC):
    @abstractmethod
    def execute(self) -> dict:
        pass
    
    @abstractmethod
    def undo(self) -> None:
        pass

# Receiver: Object that performs actual work
class DocumentStore:
    def __init__(self):
        self.documents = {}
    
    def add_document(self, doc_id: str, content: str):
        self.documents[doc_id] = content
        return {"status": "added", "doc_id": doc_id}
    
    def remove_document(self, doc_id: str):
        if doc_id in self.documents:
            return self.documents.pop(doc_id)
        return None

# Concrete Commands
class AddDocumentCommand(Command):
    """Command to add a document."""
    
    def __init__(self, store: DocumentStore, doc_id: str, content: str):
        self._store = store
        self._doc_id = doc_id
        self._content = content
    
    def execute(self) -> dict:
        return self._store.add_document(self._doc_id, self._content)
    
    def undo(self) -> None:
        self._store.remove_document(self._doc_id)

class RemoveDocumentCommand(Command):
    """Command to remove a document."""
    
    def __init__(self, store: DocumentStore, doc_id: str):
        self._store = store
        self._doc_id = doc_id
        self._backup = None
    
    def execute(self) -> dict:
        self._backup = self._store.remove_document(self._doc_id)
        return {"status": "removed", "doc_id": self._doc_id}
    
    def undo(self) -> None:
        if self._backup:
            self._store.add_document(self._doc_id, self._backup)

# Invoker: Manages command execution and history
class CommandInvoker:
    """Execute commands and maintain history."""
    
    def __init__(self):
        self._history: List[Command] = []
    
    def execute_command(self, command: Command) -> dict:
        result = command.execute()
        self._history.append(command)
        return result
    
    def undo_last(self) -> None:
        if self._history:
            command = self._history.pop()
            command.undo()

# Usage
store = DocumentStore()
invoker = CommandInvoker()

# Execute commands
cmd1 = AddDocumentCommand(store, "doc1", "Content 1")
invoker.execute_command(cmd1)

cmd2 = AddDocumentCommand(store, "doc2", "Content 2")
invoker.execute_command(cmd2)

# Undo last command
invoker.undo_last()  # Removes doc2
```

**Project-specific guidance:**
- Use for complex evaluation workflows
- Implement experiment scripts as command objects
- Useful for test data setup/teardown
- Can implement CLI commands or scheduled tasks as Command objects
- Consider for batch operations that need to be queued

---

## Iterator

### Intent
Provide a way to access elements of a collection sequentially without exposing its underlying representation.

### Problem Space
- You need to traverse a complex data structure without exposing its internals
- You want to provide multiple traversal algorithms for the same collection
- You need a uniform interface for traversing different collection types
- You want to traverse a collection while modifying it during iteration

### Solution
Extract the traversal behavior into a separate iterator object. The iterator implements a standard interface with methods like `next()` and `has_next()`. The collection returns an iterator instance.

**Structure:**
- Iterator interface declares operations for traversing a collection
- Concrete Iterators implement specific traversal algorithms
- Collection interface declares method for creating iterators
- Concrete Collections return appropriate iterator instances

### Trade-offs

**Benefits:**
- Single Responsibility Principle: separate traversal logic from collection
- Open/Closed Principle: add new iterators without changing collections
- Can iterate over same collection in parallel
- Can delay iteration and resume it

**Drawbacks:**
- Overkill for simple collections
- Less efficient than direct traversal for some specialized collections

### Implementation Information

**When to use in this project:**
- Traversing complex document structures (clauses, annotations)
- Iterating through evaluation results with different orderings
- Providing multiple ways to traverse ground-truth datasets
- Lazy-loading large datasets during iteration

**Example pattern:**
```python
from abc import ABC, abstractmethod
from typing import Iterator, List

# Python's iterator protocol
class DocumentCollection:
    """Collection that can be iterated."""
    
    def __init__(self):
        self._documents: List[dict] = []
    
    def add(self, document: dict):
        self._documents.append(document)
    
    def __iter__(self) -> Iterator[dict]:
        """Return iterator for sequential traversal."""
        return iter(self._documents)
    
    def reverse_iterator(self) -> Iterator[dict]:
        """Return iterator for reverse traversal."""
        return reversed(self._documents)
    
    def filtered_iterator(self, predicate) -> Iterator[dict]:
        """Return iterator with filtering."""
        return (doc for doc in self._documents if predicate(doc))

# Custom iterator for complex traversal
class ClauseTreeIterator:
    """Depth-first iterator for clause hierarchy."""
    
    def __init__(self, root_clause):
        self._stack = [root_clause]
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if not self._stack:
            raise StopIteration
        
        current = self._stack.pop()
        
        # Add children to stack (reverse order for correct DFS)
        if hasattr(current, 'children'):
            self._stack.extend(reversed(current.children))
        
        return current

class ClauseTree:
    """Collection with custom iterator."""
    
    def __init__(self, root):
        self.root = root
    
    def __iter__(self):
        """Return depth-first iterator."""
        return ClauseTreeIterator(self.root)
    
    def breadth_first(self):
        """Alternative traversal strategy."""
        queue = [self.root]
        while queue:
            current = queue.pop(0)
            yield current
            if hasattr(current, 'children'):
                queue.extend(current.children)

# Usage
collection = DocumentCollection()
collection.add({"id": 1, "type": "order"})
collection.add({"id": 2, "type": "permit"})

# Default iteration
for doc in collection:
    print(doc)

# Reverse iteration
for doc in collection.reverse_iterator():
    print(doc)

# Filtered iteration
items = collection.filtered_iterator(lambda d: d["type"] == "order")
for item in items:
    print(item)
```

**Python-specific guidance:**
- Python has built-in iterator protocol (`__iter__`, `__next__`)
- Use generators for lazy iteration:
  ```python
  def document_generator(file_path):
      """Generator for lazy loading."""
      with open(file_path) as f:
          for line in f:
              yield parse_document(line)
  ```
- Consider `itertools` module for complex iteration patterns
- ORM querysets are iterators

**Project-specific guidance:**
- Use generators for lazy-loading large document sets
- Implement custom iterators for clause hierarchies
- Useful for streaming evaluation results
- Consider for pagination of ground-truth data

---

## Mediator

### Intent
Reduce chaotic dependencies between objects. Restrict direct communications between objects and force them to collaborate through a mediator object.

### Problem Space
- Objects communicate in complex ways with many interdependencies
- It's hard to reuse classes because they depend on many other classes
- You want to customize behavior distributed across several classes without subclassing
- A set of objects communicate in well-defined but complex ways

### Solution
Extract all relationships between classes into a separate mediator class. Components communicate only with the mediator and don't know about other components.

**Structure:**
- Mediator interface declares communication methods
- Concrete Mediator encapsulates relations between components
- Components store reference to mediator and communicate through it
- Components don't know about other components

### Trade-offs

**Benefits:**
- Single Responsibility Principle: extract communications into one place
- Open/Closed Principle: introduce new mediators without changing components
- Reduce coupling between components
- Reuse individual components more easily

**Drawbacks:**
- Mediator can evolve into a God Object
- Can become complex if it manages too much

### Implementation Information

**When to use in this project:**
- Coordinating complex evaluation workflows with multiple components
- Managing interactions between document parser, extractor, and validator
- Orchestrating experiment setup involving multiple services
- Coordinating UI components (if UI is added)

**Example pattern:**
```python
from abc import ABC, abstractmethod
from typing import List

# Mediator interface
class EvaluationMediator(ABC):
    @abstractmethod
    def notify(self, sender: object, event: str, data: dict = None):
        pass

# Concrete Mediator
class EvaluationCoordinator(EvaluationMediator):
    """Coordinates evaluation components."""
    
    def __init__(self):
        self._parser = None
        self._extractor = None
        self._evaluator = None
        self._reporter = None
    
    def set_components(self, parser, extractor, evaluator, reporter):
        self._parser = parser
        self._parser.set_mediator(self)
        
        self._extractor = extractor
        self._extractor.set_mediator(self)
        
        self._evaluator = evaluator
        self._evaluator.set_mediator(self)
        
        self._reporter = reporter
        self._reporter.set_mediator(self)
    
    def notify(self, sender: object, event: str, data: dict = None):
        """Coordinate component interactions."""
        if event == "document_parsed":
            # Parser finished, trigger validation
            self._extractor.extract(data["document"])
        
        elif event == "validation_complete":
            # Validation done, run enrichment
            self._evaluator.evaluate(data["extracted_data"])
        
        elif event == "evaluation_complete":
            # Evaluation done, generate report
            self._reporter.generate_report(data["results"])
        
        elif event == "error":
            # Handle errors from any component
            print(f"Error from {sender.__class__.__name__}: {data['message']}")

# Components
class Component:
    """Base component with mediator reference."""
    
    def __init__(self):
        self._mediator: EvaluationMediator = None
    
    def set_mediator(self, mediator: EvaluationMediator):
        self._mediator = mediator

class DocumentParser(Component):
    def parse(self, file_path: str):
        try:
            document = self._parse_file(file_path)
            self._mediator.notify(self, "document_parsed", {"document": document})
        except Exception as e:
            self._mediator.notify(self, "error", {"message": str(e)})
    
    def _parse_file(self, file_path):
        return {"content": "..."}

class DataExtractor(Component):
    def extract(self, document: dict):
        extracted = {"parties": [...], "terms": [...]}
        self._mediator.notify(self, "validation_complete", {"validated_data": validated})

class Evaluator(Component):
    def evaluate(self, data: dict):
        results = {"accuracy": 0.95, "precision": 0.92}
        self._mediator.notify(self, "evaluation_complete", {"results": results})

class Reporter(Component):
    def generate_report(self, results: dict):
        print(f"Report: {results}")

# Usage
mediator = EvaluationCoordinator()
parser = DocumentParser()
extractor = DataExtractor()
evaluator = Evaluator()
reporter = Reporter()

mediator.set_components(parser, extractor, evaluator, reporter)

# Trigger chain through mediator
processor.run("order-123")
# Mediator coordinates: parse → extract → evaluate → report
```

**Project-specific guidance:**
- Use for complex experiment orchestration
- Keep mediator focused on coordination, not business logic
- Consider framework event/signal systems as built-in mediator patterns
- Useful for managing evaluation pipeline stages
- Can help coordinate test setup across multiple fixtures

---

## Memento

### Intent
Capture and externalize an object's internal state without violating encapsulation, so the object can be restored to this state later.

### Problem Space
- You need to save/restore previous states of an object
- You want to implement undo/redo functionality
- Direct access to object's fields violates encapsulation
- You need checkpointing or transactional behavior

### Solution
Delegate creating state snapshots to the owner object (originator). The originator creates a memento containing a snapshot of its current state. Only the originator can access the memento's content.

**Structure:**
- Originator produces snapshots of its state and can restore from them
- Memento is a value object acting as snapshot
- Caretaker knows when to capture state and when to restore
- Caretaker stores mementos but never examines their contents

### Trade-offs

**Benefits:**
- Preserve encapsulation boundaries
- Simplify originator code by delegating state management
- Support undo/redo operations

**Drawbacks:**
- Can consume lots of memory if state is large or created frequently
- Caretakers must track originator's lifecycle
- Dynamic languages make it hard to guarantee memento immutability

### Implementation Information

**When to use in this project:**
- Implementing undo for document annotations
- Checkpointing during long-running evaluations
- Saving experiment state for resumption
- Creating snapshots of test data states

**Example pattern:**
```python
from copy import deepcopy
from typing import List
from datetime import datetime

# Memento
class ExperimentMemento:
    """Snapshot of experiment state."""
    
    def __init__(self, state: dict, timestamp: datetime):
        self._state = deepcopy(state)  # Immutable snapshot
        self._timestamp = timestamp
    
    def get_state(self) -> dict:
        return deepcopy(self._state)
    
    def get_timestamp(self) -> datetime:
        return self._timestamp

# Originator
class Experiment:
    """Object whose state we want to save/restore."""
    
    def __init__(self, name: str):
        self._name = name
        self._config = {}
        self._results = []
        self._status = "initialized"
    
    def configure(self, config: dict):
        self._config = config
        self._status = "configured"
    
    def run_step(self, result: dict):
        self._results.append(result)
        self._status = "running"
    
    def complete(self):
        self._status = "completed"
    
    def save_state(self) -> ExperimentMemento:
        """Create memento with current state."""
        state = {
            "name": self._name,
            "config": self._config,
            "results": self._results,
            "status": self._status
        }
        return ExperimentMemento(state, datetime.now())
    
    def restore_state(self, memento: ExperimentMemento):
        """Restore from memento."""
        state = memento.get_state()
        self._name = state["name"]
        self._config = state["config"]
        self._results = state["results"]
        self._status = state["status"]
    
    def get_status(self) -> str:
        return f"{self._name}: {self._status}, {len(self._results)} results"

# Caretaker
class ExperimentHistory:
    """Manages experiment mementos."""
    
    def __init__(self, experiment: Experiment):
        self._experiment = experiment
        self._history: List[ExperimentMemento] = []
    
    def backup(self):
        """Save current state."""
        print(f"Saving: {self._experiment.get_status()}")
        self._history.append(self._experiment.save_state())
    
    def undo(self):
        """Restore previous state."""
        if not self._history:
            return
        
        memento = self._history.pop()
        self._experiment.restore_state(memento)
        print(f"Restored to: {self._experiment.get_status()}")

# Usage
experiment = Experiment("Model Evaluation")
history = ExperimentHistory(experiment)

experiment.configure({"model": "gpt-4", "threshold": 0.8})
history.backup()  # Checkpoint 1

experiment.run_step({"accuracy": 0.85})
experiment.run_step({"accuracy": 0.87})
history.backup()  # Checkpoint 2

experiment.complete()

# Oops, need to undo completion
history.undo()  # Back to checkpoint 2
```

**Project-specific guidance:**
- Use for experiment checkpointing
- Implement undo for ground-truth annotations
- Consider for test data snapshots
- Be careful with memory usage for large states
- Python's `pickle` can serialize state, but be careful with security

---

## Observer

### Intent
Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

### Problem Space
- Changes to one object require changing others, and you don't know how many
- An object should notify others without making assumptions about who they are
- You want loose coupling between interacting objects
- Some objects need to observe others temporarily

### Solution
Add a subscription mechanism to the subject object. Observers register themselves with the subject. When the subject's state changes, it notifies all observers.

**Structure:**
- Subject maintains list of observers and notifies them of changes
- Observer interface declares notification method
- Concrete Observers implement notification method to react to updates
- Client creates subjects and observers and establishes subscriptions

### Trade-offs

**Benefits:**
- Open/Closed Principle: introduce new observers without changing subject
- Establish relationships between objects at runtime
- Loose coupling between subjects and observers

**Drawbacks:**
- Observers are notified in random order
- Can lead to unexpected updates and hard-to-track bugs
- Memory leaks if observers aren't properly unsubscribed

### Implementation Information

**When to use in this project:**
- Monitoring evaluation progress
- Tracking document processing events
- Implementing event logging across components
- Notifying multiple systems when experiments complete

**Example pattern:**
```python
from abc import ABC, abstractmethod
from typing import List

# Observer interface
class EvaluationObserver(ABC):
    @abstractmethod
    def update(self, event: str, data: dict):
        """React to events from subject."""
        pass

# Concrete Observers
class ProgressLogger(EvaluationObserver):
    """Logs progress to console."""
    
    def update(self, event: str, data: dict):
        if event == "progress":
            print(f"Progress: {data['percent']}% complete")

class MetricsCollector(EvaluationObserver):
    """Collects metrics for analysis."""
    
    def __init__(self):
        self.metrics = []
    
    def update(self, event: str, data: dict):
        if event == "metric":
            self.metrics.append(data)

class AlertSystem(EvaluationObserver):
    """Sends alerts on errors."""
    
    def update(self, event: str, data: dict):
        if event == "error":
            print(f"ALERT: {data['message']}")

# Subject
class EvaluationRunner:
    """Subject that observers watch."""
    
    def __init__(self):
        self._observers: List[EvaluationObserver] = []
        self._progress = 0
    
    def attach(self, observer: EvaluationObserver):
        """Subscribe an observer."""
        self._observers.append(observer)
    
    def detach(self, observer: EvaluationObserver):
        """Unsubscribe an observer."""
        self._observers.remove(observer)
    
    def notify(self, event: str, data: dict):
        """Notify all observers."""
        for observer in self._observers:
            observer.update(event, data)
    
    def run_evaluation(self, documents: List[dict]):
        """Run evaluation and notify observers of progress."""
        total = len(documents)
        
        for i, doc in enumerate(documents):
            try:
                # Process document
                result = self._evaluate_document(doc)
                
                # Notify metric collected
                self.notify("metric", {
                    "doc_id": doc["id"],
                    "accuracy": result["accuracy"]
                })
                
                # Notify progress
                self._progress = int((i + 1) / total * 100)
                self.notify("progress", {"percent": self._progress})
                
            except Exception as e:
                # Notify error
                self.notify("error", {
                    "doc_id": doc["id"],
                    "message": str(e)
                })
    
    def _evaluate_document(self, doc: dict) -> dict:
        return {"accuracy": 0.9}

# Usage
runner = EvaluationRunner()

# Attach observers
logger = ProgressLogger()
metrics = MetricsCollector()
alerts = AlertSystem()

runner.attach(logger)
runner.attach(metrics)
runner.attach(alerts)

# Run - all observers are automatically notified
documents = [{"id": 1}, {"id": 2}, {"id": 3}]
runner.run_evaluation(documents)
```

**Project-specific guidance:**
- Framework event/signal systems often implement the Observer pattern
- Use for evaluation progress monitoring
- Useful for experiment event logging
- Consider for test result notifications
- Be careful with memory leaks—always detach observers
- Python's `property` with setters can trigger notifications

---

## State

### Intent
Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

### Problem Space
- An object's behavior depends on its state, and it must change behavior at runtime
- Operations have large conditional statements that depend on object state
- State-specific behavior is scattered across multiple operations
- You want to avoid massive conditionals (if/elif/else or switch)

### Solution
Create separate classes for each possible state. Extract state-specific behaviors into these classes. The context object stores a reference to a state object and delegates state-specific work to it.

**Structure:**
- Context stores reference to current state and delegates state-specific operations
- State interface declares state-specific methods
- Concrete States implement behavior specific to a particular state
- States may trigger state transitions in context

### Trade-offs

**Benefits:**
- Single Responsibility Principle: organize state-specific code
- Open/Closed Principle: introduce new states without changing existing ones
- Simplify context code by eliminating bulky conditionals
- State transitions become explicit

**Drawbacks:**
- Overkill if there are few states or they rarely change
- Increases number of classes

### Implementation Information

**When to use in this project:**
- Managing evaluation workflow states (initialized, running, paused, completed, failed)
- Document processing states (raw, parsed, annotated, validated)
- Experiment lifecycle management
- Import/export state machines

**Example pattern:**
```python
from abc import ABC, abstractmethod

# Context
class EvaluationRun:
    """Context that changes behavior based on state."""
    
    def __init__(self, name: str):
        self._name = name
        self._state = InitializedState(self)
        self._results = []
    
    def set_state(self, state: 'EvaluationState'):
        """Transition to new state."""
        print(f"Transitioning to {state.__class__.__name__}")
        self._state = state
    
    def start(self):
        self._state.start()
    
    def pause(self):
        self._state.pause()
    
    def resume(self):
        self._state.resume()
    
    def complete(self):
        self._state.complete()
    
    def fail(self, error: str):
        self._state.fail(error)
    
    def add_result(self, result: dict):
        self._results.append(result)

# State interface
class EvaluationState(ABC):
    """Base state class."""
    
    def __init__(self, context: EvaluationRun):
        self._context = context
    
    @abstractmethod
    def start(self):
        pass
    
    @abstractmethod
    def pause(self):
        pass
    
    @abstractmethod
    def resume(self):
        pass
    
    @abstractmethod
    def complete(self):
        pass
    
    @abstractmethod
    def fail(self, error: str):
        pass

# Concrete States
class InitializedState(EvaluationState):
    """Can only start from this state."""
    
    def start(self):
        print("Starting evaluation...")
        self._context.set_state(RunningState(self._context))
    
    def pause(self):
        print("Cannot pause: not started")
    
    def resume(self):
        print("Cannot resume: not started")
    
    def complete(self):
        print("Cannot complete: not started")
    
    def fail(self, error: str):
        print(f"Initialization failed: {error}")
        self._context.set_state(FailedState(self._context))

class RunningState(EvaluationState):
    """Can pause or complete from this state."""
    
    def start(self):
        print("Already running")
    
    def pause(self):
        print("Pausing evaluation...")
        self._context.set_state(PausedState(self._context))
    
    def resume(self):
        print("Already running")
    
    def complete(self):
        print("Completing evaluation...")
        self._context.set_state(CompletedState(self._context))
    
    def fail(self, error: str):
        print(f"Evaluation failed: {error}")
        self._context.set_state(FailedState(self._context))

class PausedState(EvaluationState):
    """Can resume or fail from this state."""
    
    def start(self):
        print("Cannot start: already initialized")
    
    def pause(self):
        print("Already paused")
    
    def resume(self):
        print("Resuming evaluation...")
        self._context.set_state(RunningState(self._context))
    
    def complete(self):
        print("Cannot complete while paused")
    
    def fail(self, error: str):
        print(f"Evaluation failed while paused: {error}")
        self._context.set_state(FailedState(self._context))

class CompletedState(EvaluationState):
    """Terminal state."""
    
    def start(self):
        print("Cannot start: already completed")
    
    def pause(self):
        print("Cannot pause: already completed")
    
    def resume(self):
        print("Cannot resume: already completed")
    
    def complete(self):
        print("Already completed")
    
    def fail(self, error: str):
        print("Cannot fail: already completed")

class FailedState(EvaluationState):
    """Terminal state."""
    
    def start(self):
        print("Cannot start: failed")
    
    def pause(self):
        print("Cannot pause: failed")
    
    def resume(self):
        print("Cannot resume: failed")
    
    def complete(self):
        print("Cannot complete: failed")
    
    def fail(self, error: str):
        print(f"Already failed: {error}")

# Usage
evaluation = EvaluationRun("Model Test")
evaluation.start()    # Transitions to RunningState
evaluation.pause()    # Transitions to PausedState
evaluation.resume()   # Transitions to RunningState
evaluation.complete() # Transitions to CompletedState
evaluation.start()    # "Cannot start: already completed"
```

**Project-specific guidance:**
- Use for evaluation workflow management
- Consider for document processing pipelines
- Can model experiment lifecycle
- Keep state transitions clear and documented
- Each state should validate operations it allows

---

## Strategy

### Intent
Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

### Problem Space
- You have many similar classes that differ only in behavior
- You need different variants of an algorithm
- An algorithm uses data clients shouldn't know about
- A class has massive conditional operators that switch between algorithm variants

### Solution
Extract algorithms into separate classes called strategies. The context class has a reference to a strategy object and delegates algorithm execution to it. Strategies can be switched at runtime.

**Structure:**
- Context maintains reference to a strategy and delegates work to it
- Strategy interface is common to all algorithms
- Concrete Strategies implement different algorithm variants
- Client creates specific strategy and passes it to context

### Trade-offs

**Benefits:**
- Swap algorithms at runtime
- Isolate implementation details from code that uses them
- Replace inheritance with composition
- Open/Closed Principle: introduce new strategies without changing context

**Drawbacks:**
- Increases number of objects
- Clients must be aware of differences between strategies
- Overkill if you have just a few algorithms that rarely change

### Implementation Information

**When to use in this project:**
- Different pricing algorithms (flat rate, tiered, volume discount)
- Multiple evaluation metrics or scoring strategies
- Various document parsing approaches
- Different matching algorithms for provenance

**Example pattern:**
```python
from abc import ABC, abstractmethod

# Strategy interface
class PricingStrategy(ABC):
    @abstractmethod
    def extract(self, document: str) -> dict:
        pass

# Concrete Strategies
class FlatRatePricing(PricingStrategy):
    """Extract using regular expressions."""
    
    def extract(self, document: str) -> dict:
        # Flat rate pricing logic
        import re
        parties = re.findall(r'Party [A-Z]', document)
        return {"parties": parties, "method": "regex"}

class TieredPricing(PricingStrategy):
    """Extract using AI model."""
    
    def __init__(self, model_name: str):
        self._model_name = model_name
    
    def extract(self, document: str) -> dict:
        # Tiered pricing logic
        # result = call_ai_model(self._model_name, document)
        return {"parties": [...], "method": f"ai:{self._model_name}"}

class VolumeDiscountPricing(PricingStrategy):
    """Extract using hybrid approach."""
    
    def extract(self, document: str) -> dict:
        # First pass with regex, refine with AI
        base_price = FlatRatePricing().calculate(order)
        # Validate/enhance with AI...
        return {"parties": regex_result["parties"], "method": "hybrid"}

# Context
class DocumentExtractor:
    """Context that uses a pricing strategy."""
    
    def __init__(self, strategy: PricingStrategy):
        self._strategy = strategy
    
    def set_strategy(self, strategy: PricingStrategy):
        """Switch strategy at runtime."""
        self._strategy = strategy
    
    def process_document(self, document: str) -> dict:
        """Delegate to current strategy."""
        return self._strategy.extract(document)

# Usage
document = "Party A and Party B agree..."

# Use regex strategy
calculator = OrderPricing(FlatRatePricing())
result1 = extractor.process_document(document)
print(result1)  # {"parties": ["Party A", "Party B"], "method": "regex"}

# Switch to tiered pricing at runtime
calculator.set_strategy(TieredPricing(tiers=[100, 500, 1000]))
result2 = calculator.calculate_price(order)
print(result2)  # {"total": 450.00, "method": "tiered"}

# Switch to volume discount pricing
calculator.set_strategy(VolumeDiscountPricing())
result3 = calculator.calculate_price(order)
```

**Project-specific guidance:**
- Essential for comparing different algorithm approaches
- Use when experimenting with different algorithms
- Allows runtime selection based on configuration
- Each strategy should be independently testable
- Consider using Strategy with Factory for strategy selection

---

## Template Method

### Intent
Define the skeleton of an algorithm in a base class but let subclasses override specific steps without changing the algorithm's structure.

### Problem Space
- You have several classes with similar algorithms that differ in details
- You want to let clients extend only particular steps of an algorithm
- Common behavior should be centralized to avoid code duplication
- You need to control extension points in an algorithm

### Solution
Break the algorithm into steps and turn them into methods. Place these methods into a template method that defines the algorithm structure. Subclasses override specific steps.

**Structure:**
- Abstract Class declares template method and abstract/hook methods for steps
- Concrete Classes override steps to implement specific behavior
- Template method calls steps in defined order

### Trade-offs

**Benefits:**
- Let clients override only certain parts of algorithm
- Pull duplicate code into superclass
- Control which parts of algorithm can be extended

**Drawbacks:**
- Limited by inheritance
- Violates Liskov Substitution Principle if step semantics differ
- Hard to maintain as steps increase
- Template method can become too restrictive

### Implementation Information

**When to use in this project:**
- Standard evaluation workflow with customizable steps
- Document processing pipeline with variant preprocessing
- Test data generation with common structure but different details
- Report generation with standard format but varying content

**Example pattern:**
```python
from abc import ABC, abstractmethod

# Abstract class with template method
class DocumentEvaluator(ABC):
    """Template for document evaluation process."""
    
    def evaluate(self, document: dict) -> dict:
        """
        Template method defining evaluation algorithm.
        
        Subclasses can override specific steps but not the overall structure.
        """
        # Step 1: Load document (common)
        loaded_doc = self._load_document(document)
        
        # Step 2: Preprocess (customizable)
        preprocessed = self.preprocess(loaded_doc)
        
        # Step 3: Parse data (customizable)
        parsed = self.parse_data(preprocessed)
        
        # Step 4: Validate (common)
        if not self._validate_parsed(parsed):
            raise ValueError("Input validation failed")
        
        # Step 5: Calculate metrics (customizable)
        metrics = self.calculate_metrics(parsed)
        
        # Step 6: Format results (common)
        return self._format_results(metrics)
    
    # Common steps (same for all subclasses)
    def _load_document(self, document: dict) -> dict:
        """Load document - common implementation."""
        return {"content": document.get("content", ""), "metadata": {}}
    
    def _validate_parsed(self, parsed: dict) -> bool:
        """Validate parsed data - common implementation."""
        return "data" in parsed
    
    def _format_results(self, metrics: dict) -> dict:
        """Format results - common implementation."""
        return {"metrics": metrics, "timestamp": "2024-01-01"}
    
    # Abstract steps (must be implemented by subclasses)
    @abstractmethod
    def preprocess(self, document: dict) -> dict:
        """Preprocess document - must be implemented."""
        pass
    
    @abstractmethod
    def parse_data(self, document: dict) -> dict:
        """Parse data - must be implemented."""
        pass
    
    @abstractmethod
    def calculate_metrics(self, parsed: dict) -> dict:
        """Calculate metrics - must be implemented."""
        pass
    
    # Hook methods (optional override)
    def log_progress(self, message: str):
        """Hook for logging - can be overridden."""
        pass

# Concrete implementations
class OrderEvaluator(RecordEvaluator):
    """Evaluator specialized for orders."""
    
    def preprocess(self, document: dict) -> dict:
        # Order-specific preprocessing
        content = document["content"].strip()
        return {"content": content, "type": "order"}
    
    def parse_data(self, document: dict) -> dict:
        # Parse order-specific data
        return {"data": {"line_items": [...], "totals": {...}}}
    
    def calculate_metrics(self, parsed: dict) -> dict:
        # Order-specific metrics
        return {"item_accuracy": 0.95, "total_accuracy": 0.99}

class InvoiceEvaluator(RecordEvaluator):
    """Evaluator specialized for invoices."""
    
    def preprocess(self, document: dict) -> dict:
        # Invoice-specific preprocessing
        content = document["content"].upper()  # Different preprocessing
        return {"content": content, "type": "invoice"}
    
    def parse_data(self, document: dict) -> dict:
        # Parse invoice-specific data
        return {"data": {"invoice_number": "...", "line_items": [...]}}
    
    def calculate_metrics(self, parsed: dict) -> dict:
        # Invoice-specific metrics
        return {"parse_rate": 0.92, "line_item_accuracy": 0.87}
    
    def log_progress(self, message: str):
        # Override hook to add logging
        print(f"Invoice Evaluation: {message}")

# Usage
order = {"content": "Order #1001: 2x Widget..."}
invoice = {"content": "Invoice #INV-2024-001..."}

order_eval = OrderEvaluator()
order_result = order_eval.evaluate(order)

invoice_eval = InvoiceEvaluator()
invoice_result = invoice_eval.evaluate(invoice)
```

**Project-specific guidance:**
- Use for evaluation workflows with common structure
- Good for document processing pipelines
- Useful for test data generation with variations
- Framework view hierarchies often use Template Method pattern
- Keep template methods focused—don't make them too long
- Document which methods are required vs. optional hooks

---

## Visitor

### Intent
Separate algorithms from the objects on which they operate. Add new operations to existing object structures without modifying them.

### Problem Space
- You need to perform operations across a set of objects with different classes
- An operation needs to be performed on all elements of a complex object structure
- You want to keep related operations together and separate unrelated ones
- Object classes rarely change but new operations are added frequently

### Solution
Create a visitor class with a visit method for each type of element. Elements accept visitors and call the appropriate visit method. This allows adding new operations without modifying elements.

**Structure:**
- Visitor interface declares visit methods for each element type
- Concrete Visitors implement operations for each element type
- Element interface declares accept method that takes a visitor
- Concrete Elements implement accept by calling appropriate visit method
- Client creates visitors and passes them to elements

### Trade-offs

**Benefits:**
- Open/Closed Principle: add new operations without changing elements
- Single Responsibility Principle: group related operations in visitor
- Visitor accumulates state while traversing structure
- Can work with objects from different hierarchies

**Drawbacks:**
- Must update all visitors when adding new element types
- Elements might expose internal state to visitors
- Complicated pattern that's often overkill

### Implementation Information

**When to use in this project:**
- Performing different analyses on nested data structures
- Generating various reports from hierarchical results
- Implementing multiple export formats
- Applying different validation rules to data structures

**Example pattern:**
```python
from abc import ABC, abstractmethod

# Element interfaces
class CatalogElement(ABC):
    """Element that can be visited."""
    
    @abstractmethod
    def accept(self, visitor: 'CatalogVisitor'):
        pass

# Concrete Elements
class LineItem(CatalogElement):
    def __init__(self, description: str, item_type: str):
        self.description = description
        self.item_type = item_type
    
    def accept(self, visitor: 'CatalogVisitor'):
        visitor.visit_line_item(self)

class Adjustment(CatalogElement):
    def __init__(self, name: str, amount: float):
        self.name = name
        self.amount = amount
    
    def accept(self, visitor: 'CatalogVisitor'):
        visitor.visit_adjustment(self)

class Order(CatalogElement):
    """Composite element containing other elements."""
    
    def __init__(self):
        self.line_items = []
        self.adjustments = []
    
    def accept(self, visitor: 'CatalogVisitor'):
        visitor.visit_order(self)
        
        # Visit all children
        for item in self.line_items:
            item.accept(visitor)
        for adj in self.adjustments:
            adj.accept(visitor)

# Visitor interface
class CatalogVisitor(ABC):
    """Visitor for catalog elements."""
    
    @abstractmethod
    def visit_order(self, order: Order):
        pass
    
    @abstractmethod
    def visit_line_item(self, item: LineItem):
        pass
    
    @abstractmethod
    def visit_adjustment(self, adj: Adjustment):
        pass

# Concrete Visitors (different operations)
class DescriptionLengthVisitor(CatalogVisitor):
    """Count total description length across all items."""
    
    def __init__(self):
        self.total_chars = 0
    
    def visit_order(self, order: Order):
        pass  # No description at order level
    
    def visit_line_item(self, item: LineItem):
        self.total_chars += len(item.description)
    
    def visit_adjustment(self, adj: Adjustment):
        self.total_chars += len(adj.name)

class ExportVisitor(CatalogVisitor):
    """Export order to JSON."""
    
    def __init__(self):
        self.data = {}
    
    def visit_order(self, order: Order):
        self.data = {
            "type": "order",
            "line_items": [],
            "adjustments": []
        }
    
    def visit_line_item(self, item: LineItem):
        self.data["line_items"].append({
            "description": item.description,
            "type": item.item_type
        })
    
    def visit_adjustment(self, adj: Adjustment):
        self.data["adjustments"].append({
            "name": adj.name,
            "amount": adj.amount
        })

class ValidationVisitor(CatalogVisitor):
    """Validate order elements."""
    
    def __init__(self):
        self.errors = []
    
    def visit_order(self, order: Order):
        if len(order.line_items) == 0:
            self.errors.append("Order must have at least one line item")
    
    def visit_line_item(self, item: LineItem):
        if len(item.description) < 3:
            self.errors.append(f"Item description too short: {item.description[:20]}")
    
    def visit_adjustment(self, adj: Adjustment):
        if adj.amount == 0:
            self.errors.append(f"Adjustment '{adj.name}' has zero amount")

# Usage
order = Order()
order.line_items.append(LineItem("Widget A — blue, large", "product"))
order.line_items.append(LineItem("Widget B — red, small", "product"))
order.adjustments.append(Adjustment("Volume Discount", -15.00))

# Apply different operations using visitors
desc_counter = DescriptionLengthVisitor()
order.accept(desc_counter)
print(f"Total chars: {desc_counter.total_chars}")

exporter = ExportVisitor()
order.accept(exporter)
print(f"Exported: {exporter.data}")

validator = ValidationVisitor()
order.accept(validator)
if validator.errors:
    print(f"Validation errors: {validator.errors}")
```

**Project-specific guidance:**
- Use for multiple operations on nested data structures
- Good for report generation in different formats
- Useful for validation with different rule sets
- Can implement different analysis metrics as visitors
- Consider if you need many operations on stable structures
- Alternative: use dictionary dispatch or pattern matching (Python 3.10+)

**When NOT to use:**
- If element types change frequently (breaks all visitors)
- If operations are simple (visitor overhead not worth it)
- Consider Strategy pattern if you only need one operation with variants
