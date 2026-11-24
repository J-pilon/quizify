# Creational Design Patterns

Creational patterns provide various object creation mechanisms that increase flexibility and reuse of existing code.

---

## Factory Method

### Intent
Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

### Problem Space
- You need to create objects but don't know their exact types until runtime
- You want to provide a library of products, but only reveal their interfaces
- You want to delegate object creation logic to specialized classes
- Direct instantiation with `new` creates tight coupling between code and concrete classes

### Solution
Replace direct object construction calls with calls to a special factory method. Objects returned by the factory method are often referred to as "products."

**Structure:**
- Define a creator interface/abstract class with a factory method
- Concrete creators override the factory method to return different product types
- All products share a common interface
- Client code works with products through the common interface

### Trade-offs

**Benefits:**
- Decouples client code from concrete product classes
- Single Responsibility Principle: creation logic is centralized
- Open/Closed Principle: introduce new product types without breaking existing code
- Simplifies testing by allowing mock products

**Drawbacks:**
- Can create parallel class hierarchies (Creator + Product)
- May introduce unnecessary complexity for simple cases
- More classes to maintain

### Implementation Information

**When to use in this project:**
- Creating different types of report formatters (e.g., `PDFFormatter`, `CSVFormatter`, `JSONFormatter`)
- Instantiating different processing strategies based on configuration
- Creating parser instances for different input formats (JSON, XML, CSV)

**Example pattern:**
```python
from abc import ABC, abstractmethod

class DocumentParser(ABC):
    """Abstract creator."""
    
    @abstractmethod
    def create_extractor(self) -> Extractor:
        """Factory method to create extractors."""
        pass
    
    def parse_document(self, doc):
        """Template method using the factory."""
        extractor = self.create_extractor()
        return extractor.extract(doc)

class OrderParser(DocumentParser):
    """Concrete creator for order documents."""
    
    def create_extractor(self) -> Extractor:
        return OrderExtractor()

class InvoiceParser(DocumentParser):
    """Concrete creator for invoice documents."""
    
    def create_extractor(self) -> Extractor:
        return InvoiceExtractor()
```

**Framework notes:**
- Consider using factory methods in repository or service layers for complex object creation
- Useful for creating test fixtures with varying configurations (see `testing-conventions.md`)
- Can be combined with a registry pattern for dynamic type instantiation

---

## Abstract Factory

### Intent
Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

### Problem Space
- You need to create families of related objects that must be used together
- You want to enforce constraints that certain objects should be compatible
- Your system should be independent of how its products are created
- You need to provide multiple variants of a product family (e.g., different themes, platforms)

### Solution
Declare explicit interfaces for each distinct product in the family. Make all product variants follow those interfaces. Declare an abstract factory with creation methods for all products. For each product family variant, create a separate factory class.

**Structure:**
- Abstract factory interface declares creation methods for each product type
- Concrete factories implement these methods to return specific product variants
- Products share interfaces but have different implementations per variant
- Client code works only with abstract interfaces

### Trade-offs

**Benefits:**
- Ensures product compatibility within a family
- Isolates concrete classes from client code
- Makes exchanging product families easy
- Promotes consistency among products

**Drawbacks:**
- Complexity: many interfaces and classes
- Adding new product types requires changing all factories
- Can be overkill for systems with few product variations

### Implementation Information

**When to use in this project:**
- Creating related evaluation components (evaluator + metrics + reporter) that must work together
- Supporting different AI model providers (OpenAI, Anthropic) with their specific clients and formatters
- Building test fixtures that require multiple related objects

**Example pattern:**
```python
from abc import ABC, abstractmethod

class AIProviderFactory(ABC):
    """Abstract factory for AI provider components."""
    
    @abstractmethod
    def create_client(self) -> AIClient:
        pass
    
    @abstractmethod
    def create_formatter(self) -> PromptFormatter:
        pass
    
    @abstractmethod
    def create_parser(self) -> ResponseParser:
        pass

class OpenAIFactory(AIProviderFactory):
    """Concrete factory for OpenAI components."""
    
    def create_client(self) -> AIClient:
        return OpenAIClient()
    
    def create_formatter(self) -> PromptFormatter:
        return OpenAIFormatter()
    
    def create_parser(self) -> ResponseParser:
        return OpenAIParser()

class AnthropicFactory(AIProviderFactory):
    """Concrete factory for Anthropic components."""
    
    def create_client(self) -> AIClient:
        return AnthropicClient()
    
    def create_formatter(self) -> PromptFormatter:
        return AnthropicFormatter()
    
    def create_parser(self) -> ResponseParser:
        return AnthropicParser()
```

**Project-specific guidance:**
- Use when adding support for new model providers
- Keep factories focused on object creation, not business logic
- Consider configuration files or environment variables to select which factory to use

---

## Builder

### Intent
Construct complex objects step by step. Allows producing different types and representations of an object using the same construction code.

### Problem Space
- Object construction requires many steps or has many optional parameters
- Constructor has too many parameters (telescoping constructor anti-pattern)
- You need to create different representations of the same object
- Construction logic is complex and should be isolated from the object itself

### Solution
Extract object construction code into separate builder objects. Organize construction steps into a series of method calls. Optionally, use a director class to define the order of construction steps.

**Structure:**
- Builder interface declares construction steps common to all builders
- Concrete builders provide different implementations of construction steps
- Product is the resulting object
- Director (optional) defines the order to execute building steps
- Client associates a builder with a director, then triggers construction

### Trade-offs

**Benefits:**
- Construct objects step by step, defer or run steps recursively
- Reuse construction code for different product representations
- Single Responsibility Principle: isolate complex construction logic
- Improves readability with named steps vs. long parameter lists

**Drawbacks:**
- Overall complexity increases (more classes)
- Can be overkill for simple objects
- Potential for invalid intermediate states if not careful

### Implementation Information

**When to use in this project:**
- Building complex evaluation configurations with many optional parameters
- Constructing test documents with various attributes (parties, clauses, terms)
- Creating query objects for complex database searches
- Building ground-truth datasets with specific characteristics

**Example pattern:**
```python
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class EvaluationConfig:
    """Complex product."""
    model_name: str
    metrics: List[str] = field(default_factory=list)
    threshold: float = 0.8
    include_provenance: bool = False
    sample_size: Optional[int] = None
    exclude_categories: List[str] = field(default_factory=list)

class EvaluationConfigBuilder:
    """Builder for evaluation configurations."""
    
    def __init__(self, model_name: str):
        self._config = EvaluationConfig(model_name=model_name)
    
    def with_metrics(self, *metrics: str) -> 'EvaluationConfigBuilder':
        self._config.metrics = list(metrics)
        return self
    
    def with_threshold(self, threshold: float) -> 'EvaluationConfigBuilder':
        self._config.threshold = threshold
        return self
    
    def include_provenance(self) -> 'EvaluationConfigBuilder':
        self._config.include_provenance = True
        return self
    
    def with_sample_size(self, size: int) -> 'EvaluationConfigBuilder':
        self._config.sample_size = size
        return self
    
    def excluding_categories(self, *categories: str) -> 'EvaluationConfigBuilder':
        self._config.exclude_categories = list(categories)
        return self
    
    def build(self) -> EvaluationConfig:
        """Return the constructed configuration."""
        return self._config

# Usage:
config = (EvaluationConfigBuilder("gpt-4")
    .with_metrics("accuracy", "precision", "recall")
    .with_threshold(0.85)
    .include_provenance()
    .with_sample_size(100)
    .build())
```

**Project-specific guidance:**
- Use method chaining (fluent interface) for better readability
- Consider using dataclasses for the product objects
- Useful in test factories for creating complex test fixtures
- Can combine with Factory pattern for different builder types

---

## Prototype

### Intent
Create new objects by copying existing objects (prototypes) rather than creating them from scratch.

### Problem Space
- Object creation is expensive (requires database queries, network calls, heavy computation)
- You need to create many similar objects with slight variations
- You want to avoid subclassing just to differ in initial values
- Object initialization logic is complex or requires specific state

### Solution
Delegate the cloning process to the objects being cloned. Declare a common interface with a `clone()` method. An object that supports cloning is called a prototype.

**Structure:**
- Prototype interface declares the cloning method
- Concrete prototypes implement cloning, handling all edge cases
- Client creates new objects by asking prototypes to clone themselves
- Optional prototype registry stores frequently used prototypes

### Trade-offs

**Benefits:**
- Avoid expensive initialization when creating similar objects
- Reduce subclassing for initialization variations
- Can add/remove prototypes at runtime
- Specify new objects by varying values rather than structure

**Drawbacks:**
- Cloning complex objects with circular references can be tricky
- Deep copy vs. shallow copy decisions can be challenging
- Not always clear when to use vs. just creating new objects

### Implementation Information

**When to use in this project:**
- Creating variations of ground-truth test data
- Duplicating document structures with minor modifications for testing
- Cloning evaluation configurations with slight parameter changes
- Creating template documents as starting points

**Example pattern:**
```python
from copy import deepcopy
from typing import Dict, Any

class Cloneable(ABC):
    """Prototype interface."""
    
    @abstractmethod
    def clone(self) -> 'Cloneable':
        """Return a copy of this object."""
        pass

class GroundTruthDocument(Cloneable):
    """Concrete prototype."""
    
    def __init__(self, content: str, metadata: Dict[str, Any]):
        self.content = content
        self.metadata = metadata
        self.annotations = []
    
    def clone(self) -> 'GroundTruthDocument':
        """Create a deep copy of this document."""
        new_doc = GroundTruthDocument(
            content=self.content,
            metadata=deepcopy(self.metadata)
        )
        new_doc.annotations = deepcopy(self.annotations)
        return new_doc

# Usage:
template = GroundTruthDocument(
    content="Standard order template...",
    metadata={"type": "order", "region": "CA"}
)

# Create variations
doc1 = template.clone()
doc1.metadata["client"] = "Client A"

doc2 = template.clone()
doc2.metadata["client"] = "Client B"
```

**Project-specific guidance:**
- Python's `copy` module provides `copy()` and `deepcopy()` functions
- Be careful with ORM instances—cloning may require custom logic to avoid sharing primary keys
- Consider using `dataclasses` with `replace()` method for immutable variants
- Useful in test fixtures to avoid repeated database queries

---

## Singleton

### Intent
Ensure a class has only one instance and provide a global point of access to it.

### Problem Space
- You need exactly one instance of a class (e.g., database connection, configuration)
- You want controlled access to a single shared resource
- The single instance should be extensible through subclassing
- Lazy initialization is needed (create only when first requested)

### Solution
Make the constructor private and create a static method that acts as a constructor. This method either creates a new instance (first call) or returns the existing instance (subsequent calls).

**Structure:**
- Private constructor prevents direct instantiation
- Static method provides global access point
- Static field stores the single instance

### Trade-offs

**Benefits:**
- Guarantees only one instance exists
- Global access point
- Instance is initialized only when needed (lazy initialization)
- Easy to control number of instances (can extend to limited pool)

**Drawbacks:**
- Violates Single Responsibility Principle (controls creation AND logic)
- Can mask poor design (global state)
- Difficult to unit test (global state between tests)
- Requires special treatment in multi-threaded environments
- Complicates subclassing

### Implementation Information

**When to use in this project:**
- Database connection pools (usually managed by framework or ORM)
- Application configuration managers
- Logging coordinators
- Resource pools (with modifications for pooling)

**Example pattern:**
```python
from threading import Lock

class EvaluationCoordinator:
    """Singleton to coordinate evaluation runs."""
    
    _instance = None
    _lock = Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._initialized = False
        return cls._instance
    
    def __init__(self):
        if self._initialized:
            return
        self._initialized = True
        self._active_runs = {}
        self._results = []
    
    def register_run(self, run_id: str):
        self._active_runs[run_id] = {"status": "running"}
    
    def complete_run(self, run_id: str, result):
        self._results.append(result)
        del self._active_runs[run_id]

# Usage:
coordinator1 = EvaluationCoordinator()
coordinator2 = EvaluationCoordinator()
assert coordinator1 is coordinator2  # Same instance
```

**Project-specific guidance:**
- **Use sparingly** in this project—often a sign of global state issues
- Consider dependency injection instead for better testability
- Your framework may provide its own singleton patterns (settings, app registry) — prefer those over rolling your own
- If you must use it, prefer module-level instances in Python:
  ```python
  # evaluation_coordinator.py
  class _EvaluationCoordinator:
      def __init__(self):
          self._active_runs = {}
  
  # Module-level singleton
  evaluation_coordinator = _EvaluationCoordinator()
  ```
- For thread-safety, use `threading.Lock()`
- In tests, provide a reset method or use dependency injection

**Alternatives to consider:**
- Dependency Injection (preferred for testability)
- Module-level objects (Pythonic singleton)
- App config patterns for application-wide state — check your framework's conventions first
- Context managers for scoped single instances
