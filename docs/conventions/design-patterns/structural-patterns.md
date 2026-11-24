# Structural Design Patterns

Structural patterns explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient.

---

## Adapter

### Intent
Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

### Problem Space
- You need to use an existing class but its interface doesn't match what you need
- You want to integrate third-party libraries with incompatible interfaces
- You need to reuse several existing subclasses that lack common functionality
- It's impractical to modify the source code of the existing class

### Solution
Create an adapter class that wraps the incompatible object and exposes the interface clients expect. The adapter translates calls from the client interface to the wrapped object's interface.

**Structure:**
- Target interface defines the domain-specific interface used by client
- Client code works with objects through the target interface
- Adaptee is the incompatible class that needs adapting
- Adapter implements the target interface and wraps an adaptee instance

### Trade-offs

**Benefits:**
- Single Responsibility Principle: separate interface conversion from business logic
- Open/Closed Principle: introduce new adapters without changing existing code
- Allows integration of incompatible interfaces
- Promotes code reuse

**Drawbacks:**
- Overall complexity increases (additional abstraction layer)
- Sometimes simpler to just change the service class interface
- Performance overhead from indirection

### Implementation Information

**When to use in this project:**
- Integrating different AI model providers (OpenAI, Anthropic, etc.) with a common interface
- Adapting document parsers from different libraries to a unified interface
- Converting external data formats to internal representations
- Wrapping third-party evaluation metrics libraries

**Example pattern:**
```python
from abc import ABC, abstractmethod

# Target interface
class DocumentExtractor(ABC):
    """Common interface expected by our code."""
    
    @abstractmethod
    def extract_text(self, document) -> str:
        pass
    
    @abstractmethod
    def extract_metadata(self, document) -> dict:
        pass

# Adaptee (incompatible third-party library)
class ThirdPartyCSVParser:
    """External library with different interface."""
    
    def get_content(self, csv_file):
        return "CSV content..."
    
    def get_properties(self, csv_file):
        return {"pages": 10, "author": "John"}

# Adapter
class CSVParserAdapter(DataExtractor):
    """Adapts third-party parser to our interface."""
    
    def __init__(self, csv_parser: ThirdPartyCSVParser):
        self._parser = csv_parser
    
    def extract_text(self, document) -> str:
        # Translate interface
        return self._parser.get_content(document)
    
    def extract_metadata(self, document) -> dict:
        props = self._parser.get_properties(document)
        # Transform to our expected format
        return {
            "page_count": props["pages"],
            "author": props["author"]
        }

# Usage
parser = ThirdPartyCSVParser()
adapter = CSVParserAdapter(parser)
text = adapter.extract_text(document)  # Unified interface
```

**Project-specific guidance:**
- Use when integrating different model APIs (OpenAI, Anthropic, local models)
- Create adapters for document parsing libraries
- Keep adapters thin—only translate interfaces, don't add business logic
- Consider using Protocol classes (PEP 544) for structural subtyping

---

## Bridge

### Intent
Decouple an abstraction from its implementation so the two can vary independently. Split a large class or closely related classes into two separate hierarchies (abstraction and implementation).

### Problem Space
- You have a class with multiple dimensions of variation (e.g., Shape × Color)
- You want to avoid a cartesian product of subclasses
- You need to switch implementations at runtime
- You want to divide and organize a monolithic class with multiple variants

### Solution
Split the class into two separate hierarchies: abstraction (high-level control layer) and implementation (low-level operations). The abstraction contains a reference to an implementation object and delegates the actual work to it.

**Structure:**
- Abstraction provides high-level control logic, delegates to implementation
- Refined Abstraction extends abstraction with variations
- Implementation interface defines operations for all concrete implementations
- Concrete Implementations contain platform-specific code

### Trade-offs

**Benefits:**
- Separate platform-independent code from platform-specific code
- Open/Closed Principle: introduce new abstractions and implementations independently
- Single Responsibility Principle: abstraction focuses on high-level logic
- Avoids exponential subclass growth

**Drawbacks:**
- Increases complexity for highly cohesive classes
- More difficult to understand initially
- Requires careful interface design

### Implementation Information

**When to use in this project:**
- Separating processing strategies from data source types
- Decoupling evaluation logic from metric implementations
- Separating storage mechanisms from business logic
- Implementing cross-cutting concerns (logging, caching) across different components

**Example pattern:**
```python
from abc import ABC, abstractmethod

# Implementation hierarchy
class PricingStrategy(ABC):
    """Implementation interface."""
    
    @abstractmethod
    def extract(self, content: str) -> dict:
        pass

class FlatRatePricing(PricingStrategy):
    """Concrete implementation."""
    
    def extract(self, content: str) -> dict:
        # Flat rate pricing logic
        return {"method": "regex", "data": [...]}

class TieredPricing(PricingStrategy):
    """Concrete implementation."""
    
    def extract(self, content: str) -> dict:
        # Tiered pricing logic
        return {"method": "ai", "data": [...]}

# Abstraction hierarchy
class DocumentProcessor(ABC):
    """Abstraction with reference to implementation."""
    
    def __init__(self, strategy: PricingStrategy):
        self._strategy = strategy
    
    @abstractmethod
    def process(self, document) -> dict:
        pass

class StandardOrderProcessor(OrderProcessor):
    """Refined abstraction for standard orders."""
    
    def process(self, document) -> dict:
        # High-level standard order processing logic
        content = document.get_content()
        extracted = self._strategy.extract(content)
        # Post-process for standard orders
        processed["order_type"] = "standard"
        return extracted

class PermitProcessor(DocumentProcessor):
    """Refined abstraction for permits."""
    
    def process(self, document) -> dict:
        content = document.get_content()
        extracted = self._strategy.extract(content)
        # Post-process for permits
        extracted["document_type"] = "permit"
        return extracted

# Usage: Abstractions and implementations can vary independently
order_flat = StandardOrderProcessor(FlatRatePricing())
order_tiered = StandardOrderProcessor(TieredPricing())
subscription_tiered = SubscriptionProcessor(TieredPricing())
```

**Project-specific guidance:**
- Use when you have orthogonal dimensions (e.g., order types × pricing strategies)
- Prefer composition over inheritance
- Consider Strategy pattern if you only need to vary one dimension
- Useful for separating test doubles from production implementations

---

## Composite

### Intent
Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.

### Problem Space
- Your domain involves part-whole hierarchies or tree structures
- You want clients to treat simple and composite objects uniformly
- You need to work with tree-like structures (directories, UI components, documents)
- Operations should work recursively across the structure

### Solution
Define a component interface for both simple and composite objects. Composite objects contain children and delegate operations to them. Leaf objects perform operations directly.

**Structure:**
- Component interface declares operations common to simple and composite elements
- Leaf represents end objects with no children
- Composite stores children and implements operations by delegating to them
- Client works with all elements through the component interface

### Trade-offs

**Benefits:**
- Work with complex tree structures more conveniently
- Open/Closed Principle: introduce new element types without breaking existing code
- Uniform treatment of simple and composite objects
- Simplifies client code

**Drawbacks:**
- Can make design overly general
- Hard to restrict component types in a composite
- Can be difficult to provide common interface for very different classes

### Implementation Information

**When to use in this project:**
- Representing nested data hierarchies (order → line items → adjustments → sub-adjustments)
- Modeling evaluation metrics that aggregate sub-metrics
- Building complex validation rules from simpler rules
- Representing document structure trees

**Example pattern:**
```python
from abc import ABC, abstractmethod
from typing import List

class ClauseComponent(ABC):
    """Component interface."""
    
    @abstractmethod
    def get_text(self) -> str:
        pass
    
    @abstractmethod
    def word_count(self) -> int:
        pass
    
    @abstractmethod
    def find_terms(self, term: str) -> List[str]:
        pass

class Clause(ClauseComponent):
    """Leaf: individual clause."""
    
    def __init__(self, text: str):
        self._text = text
    
    def get_text(self) -> str:
        return self._text
    
    def word_count(self) -> int:
        return len(self._text.split())
    
    def find_terms(self, term: str) -> List[str]:
        return [self._text] if term in self._text else []

class ClauseGroup(ClauseComponent):
    """Composite: group of clauses."""
    
    def __init__(self, title: str):
        self._title = title
        self._children: List[ClauseComponent] = []
    
    def add(self, component: ClauseComponent):
        self._children.append(component)
    
    def remove(self, component: ClauseComponent):
        self._children.remove(component)
    
    def get_text(self) -> str:
        texts = [f"# {self._title}"]
        texts.extend(child.get_text() for child in self._children)
        return "\n".join(texts)
    
    def word_count(self) -> int:
        return sum(child.word_count() for child in self._children)
    
    def find_terms(self, term: str) -> List[str]:
        results = []
        for child in self._children:
            results.extend(child.find_terms(term))
        return results

# Usage: Treat individual clauses and groups uniformly
clause1 = Clause("Party A agrees to...")
clause2 = Clause("Payment terms are...")
section = ClauseGroup("Payment Section")
section.add(clause1)
section.add(clause2)

catalog = ItemGroup("Product Catalog")
catalog.add(category)

# Recursive operation
total_items = catalog.item_count()  # Works on entire tree
```

**Project-specific guidance:**
- Useful for document structure representation
- Consider for building complex evaluation criteria
- Can represent ground-truth hierarchies
- Be careful with deep recursion—add cycle detection if needed

---

## Decorator

### Intent
Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

### Problem Space
- You need to add behavior to individual objects without affecting other objects
- Responsibilities need to be added/removed at runtime
- Extension by subclassing is impractical (sealed classes, too many combinations)
- You want to avoid feature-laden classes with many optional behaviors

### Solution
Wrap the target object in decorator objects that add new behaviors. Decorators implement the same interface as the wrapped object and delegate operations to it while adding their own behavior before/after delegation.

**Structure:**
- Component interface defines operations that can be altered by decorators
- Concrete Component is the object being decorated
- Base Decorator wraps a component and delegates operations to it
- Concrete Decorators add behaviors before/after delegating to wrapped component

### Trade-offs

**Benefits:**
- More flexible than static inheritance
- Extend object behavior without modifying it
- Add/remove responsibilities at runtime
- Combine several decorators to get complex behavior
- Single Responsibility Principle: divide functionality into separate classes

**Drawbacks:**
- Can result in many small objects
- Hard to remove specific wrapper from stack
- Decorators aren't identical to wrapped objects
- Order of decoration matters and can be confusing

### Implementation Information

**When to use in this project:**
- Adding logging, caching, or timing to extractors
- Applying validation or sanitization to inputs/outputs
- Adding retry logic or error handling to operations
- Implementing feature flags or experimental variations

**Example pattern:**
```python
from abc import ABC, abstractmethod
import time
from functools import wraps

# Component interface
class DataExtractor(ABC):
    @abstractmethod
    def extract(self, document) -> dict:
        pass

# Concrete component
class BaseExtractor(DataExtractor):
    def extract(self, document) -> dict:
        return {"parties": [...], "terms": [...]}

# Base decorator
class ExtractorDecorator(DataExtractor):
    """Base decorator with delegation."""
    
    def __init__(self, extractor: DataExtractor):
        self._extractor = extractor
    
    def extract(self, document) -> dict:
        return self._extractor.extract(document)

# Concrete decorators
class LoggingExtractor(ExtractorDecorator):
    """Adds logging behavior."""
    
    def extract(self, document) -> dict:
        print(f"Extracting from document: {document.id}")
        result = self._extractor.extract(document)
        print(f"Extracted {len(result)} items")
        return result

class TimingExtractor(ExtractorDecorator):
    """Adds timing measurement."""
    
    def extract(self, document) -> dict:
        start = time.time()
        result = self._extractor.extract(document)
        elapsed = time.time() - start
        result["_timing"] = elapsed
        return result

class CachingExtractor(ExtractorDecorator):
    """Adds caching behavior."""
    
    def __init__(self, extractor: DataExtractor):
        super().__init__(extractor)
        self._cache = {}
    
    def extract(self, document) -> dict:
        if document.id in self._cache:
            return self._cache[document.id]
        result = self._extractor.extract(document)
        self._cache[document.id] = result
        return result

# Usage: Stack decorators
extractor = BaseExtractor()
extractor = CachingExtractor(extractor)
extractor = TimingExtractor(extractor)
extractor = LoggingExtractor(extractor)

result = extractor.extract(document)  # All decorators applied
```

**Python-specific alternative:**
```python
# Python function decorators for simpler cases
def timing_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"Took {time.time() - start:.2f}s")
        return result
    return wrapper

@timing_decorator
def extract_data(document):
    return {"data": [...]}
```

**Project-specific guidance:**
- Use for cross-cutting concerns (logging, metrics, caching)
- Python decorators (`@decorator`) work well for functions
- For objects, consider using the formal Decorator pattern
- Useful for adding evaluation instrumentation
- Can wrap data-access methods with caching or logging

---

## Facade

### Intent
Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

### Problem Space
- You have a complex subsystem with many interdependent classes
- You want to provide a simple interface to a complex library or framework
- You need to layer your system (presentation, business logic, data access)
- You want to reduce dependencies between client code and implementation

### Solution
Create a facade class that provides simple methods for complex subsystem operations. The facade delegates client requests to appropriate subsystem objects.

**Structure:**
- Facade provides convenient access to subsystem functionality
- Subsystem classes implement detailed functionality but are unaware of the facade
- Client uses the facade instead of calling subsystem objects directly

### Trade-offs

**Benefits:**
- Isolates clients from subsystem components
- Reduces number of objects clients deal with
- Promotes weak coupling between subsystems and clients
- Doesn't prevent using subsystem classes directly if needed

**Drawbacks:**
- Can become a god object coupled to all subsystem classes
- May hide too much, limiting advanced users
- Can become a single point of change

### Implementation Information

**When to use in this project:**
- Simplifying complex evaluation orchestration
- Providing simple interface to document processing pipeline
- Wrapping complex AI model interactions
- Creating a clean API surface for complex subsystems

**Example pattern:**
```python
# Complex subsystem
class DocumentParser:
    def parse(self, file_path):
        return {"content": "..."}

class TextPreprocessor:
    def clean(self, text):
        return text.strip().lower()

class EntityExtractor:
    def extract(self, text):
        return {"parties": [...], "dates": [...]}

class ProvenanceTracker:
    def track(self, entities, source):
        for entity in entities:
            entity["source"] = source

class ResultValidator:
    def validate(self, entities):
        return all("source" in e for e in entities)

# Facade: Simple interface to complex subsystem
class ProcessingFacade:
    """Simplified interface for the processing pipeline."""
    
    def __init__(self):
        self._parser = DocumentParser()
        self._preprocessor = TextPreprocessor()
        self._extractor = EntityExtractor()
        self._tracker = ProvenanceTracker()
        self._validator = ResultValidator()
    
    def extract_from_document(self, file_path: str) -> dict:
        """
        One simple method instead of orchestrating multiple subsystem calls.
        """
        # Orchestrate complex interactions
        parsed = self._parser.parse(file_path)
        cleaned = self._preprocessor.clean(parsed["content"])
        entities = self._extractor.extract(cleaned)
        self._tracker.track(entities, file_path)
        
        if not self._validator.validate(entities):
            raise ValueError("Processing validation failed")
        
        return entities

# Usage: Simple for clients
facade = ProcessingFacade()
result = facade.process("order-123")  # One call, complex work done
```

**Project-specific guidance:**
- Create facades for multi-step experiment workflows
- Use for complex query or data-access patterns
- Useful for wrapping third-party library complexity
- Keep facades focused—don't make them do too much
- Good for creating clean interfaces in management commands

---

## Flyweight

### Intent
Use sharing to support large numbers of fine-grained objects efficiently. Minimize memory use by sharing data among multiple objects.

### Problem Space
- Application needs to spawn a huge number of similar objects
- Memory consumption is too high
- Objects contain data that can be shared (intrinsic state)
- Object identity isn't important—only their properties matter

### Solution
Extract shared (intrinsic) state into flyweight objects that can be reused. Pass unique (extrinsic) state to flyweight methods as parameters. Store flyweights in a factory that reuses existing instances.

**Structure:**
- Flyweight stores intrinsic state (shared among many objects)
- Context stores extrinsic state (unique to each object)
- Flyweight Factory creates and manages flyweight objects
- Client computes or stores extrinsic state and uses flyweights

### Trade-offs

**Benefits:**
- Dramatically reduces memory consumption for many similar objects
- May improve performance by reducing object creation overhead
- Centralizes management of shared state

**Drawbacks:**
- Increases code complexity
- CPU cycles traded for RAM savings (computing extrinsic state)
- Only worth it when you have a large number of similar objects

### Implementation Information

**When to use in this project:**
- Storing large numbers of similar annotations or clauses
- Representing document tokens or terms that repeat frequently
- Caching compiled regex patterns used across documents
- Managing evaluation metrics that share configuration

**Example pattern:**
```python
from typing import Dict

# Flyweight: Shared intrinsic state
class ClauseType:
    """Flyweight storing shared clause type information."""
    
    def __init__(self, category: str, required_fields: list):
        # Intrinsic state (shared)
        self.category = category
        self.required_fields = required_fields
    
    def validate(self, clause_data: dict) -> bool:
        """
        Operation that uses both intrinsic and extrinsic state.
        
        Args:
            clause_data: Extrinsic state passed as parameter
        """
        return all(field in clause_data for field in self.required_fields)

# Flyweight Factory
class ClauseTypeFactory:
    """Creates and manages shared clause types."""
    
    _clause_types: Dict[str, ClauseType] = {}
    
    @classmethod
    def get_clause_type(cls, category: str, required_fields: tuple) -> ClauseType:
        """Return existing or create new flyweight."""
        key = f"{category}:{','.join(required_fields)}"
        
        if key not in cls._clause_types:
            cls._clause_types[key] = ClauseType(category, list(required_fields))
        
        return cls._clause_types[key]

# Context: Stores extrinsic state and uses flyweights
class Clause:
    """Context object with extrinsic state."""
    
    def __init__(self, text: str, clause_type: ClauseType):
        # Extrinsic state (unique to this clause)
        self.text = text
        self.position = None
        self.extracted_data = {}
        
        # Reference to shared flyweight
        self._type = clause_type
    
    def validate(self) -> bool:
        return self._type.validate(self.extracted_data)

# Usage: Many clauses share few clause type objects
payment_type = ClauseTypeFactory.get_clause_type(
    "payment",
    ("amount", "due_date")
)

# Thousands of clauses can share the same payment_type flyweight
clause1 = Clause("Payment of $1000...", payment_type)
clause2 = Clause("Payment of $2000...", payment_type)
# ... thousands more clauses

# Memory saved: payment type info stored once, not per clause
```

**Project-specific guidance:**
- Consider for large-scale document processing
- Useful when loading thousands of similar ground-truth items
- Can apply to data fields that repeat across many instances
- Python's string interning is a built-in flyweight
- Profile first—only use if memory is actually a problem

---

## Proxy

### Intent
Provide a surrogate or placeholder for another object to control access to it. Create a wrapper that can intercept and control access to the real object.

### Problem Space
- You need lazy initialization (create expensive object only when needed)
- You want to control access to an object (security, logging)
- You need a local representative for a remote object (remote proxy)
- You want to cache results or add functionality without modifying the original

### Solution
Create a proxy class with the same interface as the service object. The proxy intercepts requests, performs additional logic, then passes requests to the real object.

**Structure:**
- Service Interface declares common interface for Service and Proxy
- Service is the actual object doing useful work
- Proxy has a reference to service object and controls access to it
- Client works with services through the same interface

### Trade-offs

**Benefits:**
- Control service object lifecycle without clients knowing
- Works even when service object isn't ready or available
- Open/Closed Principle: introduce proxies without changing service
- Can add security, caching, logging without modifying service

**Drawbacks:**
- Code complexity increases (more classes)
- Response from service might get delayed
- Can be confused with Decorator pattern

### Implementation Information

**When to use in this project:**
- Lazy loading of expensive document objects
- Controlling access to AI model APIs (rate limiting, cost tracking)
- Caching evaluation results
- Adding retry logic or fallback behavior
- Logging API calls without modifying the API client

**Types of proxies:**

1. **Virtual Proxy** (lazy initialization):
```python
class DocumentProxy:
    """Lazy-load expensive document."""
    
    def __init__(self, file_path: str):
        self._file_path = file_path
        self._document = None  # Not loaded yet
    
    def get_content(self) -> str:
        if self._document is None:
            # Load only when first accessed
            self._document = self._load_document()
        return self._document.get_content()
    
    def _load_document(self):
        # Expensive operation
        return Document.from_file(self._file_path)
```

2. **Protection Proxy** (access control):
```python
class SecureModelProxy:
    """Control access to AI model."""
    
    def __init__(self, model: AIModel, user_role: str):
        self._model = model
        self._user_role = user_role
    
    def generate(self, prompt: str) -> str:
        if self._user_role not in ["admin", "researcher"]:
            raise PermissionError("Insufficient permissions")
        
        if len(prompt) > 1000:
            raise ValueError("Prompt too long for your role")
        
        return self._model.generate(prompt)
```

3. **Caching Proxy**:
```python
class CachingEvaluatorProxy:
    """Cache evaluation results."""
    
    def __init__(self, evaluator: Evaluator):
        self._evaluator = evaluator
        self._cache = {}
    
    def evaluate(self, document_id: str, config: dict) -> dict:
        cache_key = f"{document_id}:{hash(str(config))}"
        
        if cache_key not in self._cache:
            self._cache[cache_key] = self._evaluator.evaluate(
                document_id, config
            )
        
        return self._cache[cache_key]
```

4. **Logging/Monitoring Proxy**:
```python
class MonitoredExtractorProxy:
    """Add monitoring to extractor."""
    
    def __init__(self, extractor: Extractor):
        self._extractor = extractor
        self._call_count = 0
        self._total_time = 0
    
    def extract(self, document) -> dict:
        self._call_count += 1
        start = time.time()
        
        result = self._extractor.extract(document)
        
        elapsed = time.time() - start
        self._total_time += elapsed
        
        print(f"Call #{self._call_count} took {elapsed:.2f}s")
        return result
```

**Project-specific guidance:**
- Use virtual proxy for lazy-loading large documents or models
- Protection proxy useful for experiment resource management
- Caching proxy good for expensive evaluation operations
- Lazy-loading mechanisms in ORMs are a form of virtual proxy
- Consider using Python's `__getattr__` for transparent proxying:
  ```python
  class TransparentProxy:
      def __init__(self, obj):
          self._obj = obj
      
      def __getattr__(self, name):
          # Intercept all attribute access
          return getattr(self._obj, name)
  ```
- Proxy vs. Decorator: Proxy manages lifecycle; Decorator adds behavior
