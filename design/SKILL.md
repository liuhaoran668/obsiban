---
name: design-patterns-apply
description: |
  Apply design patterns correctly based on actual problems, not theoretical needs.
  Use when code exhibits specific smells (long if-else chains, duplicated code,
  complex object creation), when refactoring to improve extensibility, or when
  deciding between pattern solutions. Prevents over-engineering and under-design.
  Keywords: design pattern, refactoring, SOLID, code quality, extensibility
---

# Design Patterns - Principles Over Patterns

## Core Philosophy

**Principles ("心法") > Patterns ("招式")**

Design patterns are tools, not goals. The purpose is **improving code quality**
(readability, extensibility, maintainability), not showing off knowledge.

**Golden Rule**: If applying a pattern makes code harder to read or maintain,
it's over-engineering.

---

## When to Apply Design Patterns

### ✅ Use Patterns When

**Problem-first approach**:
- ✅ Code has clear pain points (see Recognition Triggers below)
- ✅ Pattern demonstrably improves extensibility
- ✅ Complexity justifies the abstraction cost
- ✅ Team understands the pattern and can maintain it

**Examples of justified use**:
```python
# Problem: Adding new payment methods requires modifying checkout code
# Solution: Strategy pattern to decouple payment algorithms

# Problem: 20+ parameter constructor, hard to understand
# Solution: Builder pattern for complex object construction

# Problem: if-else chains with 10+ payment gateway checks
# Solution: Factory pattern + Strategy pattern
```

### ❌ Don't Use Patterns When

**Anti-patterns of pattern usage**:
- ❌ "Might need it in the future" (YAGNI violation)
- ❌ Just learned the pattern, looking for places to apply it
- ❌ Simple code becomes complex after applying pattern
- ❌ Problem doesn't exist yet (premature optimization)

**Examples of over-engineering**:
```python
# Problem: Only 2 fixed algorithms, never change at runtime
# Bad: Use Strategy pattern with 5 classes
# Good: Simple if-else or function parameter

# Problem: Simple CRUD with 3 fields
# Bad: Builder pattern with fluent interface
# Good: Constructor with 3 parameters

# Problem: Single database connection
# Bad: Singleton + Factory + Proxy
# Good: Simple class with instance method
```

---

## Recognition Triggers: Code Smells → Pattern Solutions

### Creational Patterns

| Code Smell | Problem | Pattern | Caution |
|------------|---------|---------|---------|
| Global unique instance needed | Config, ID generator, connection pool | **Singleton** | ⚠️ Hides dependencies, hard to test. Prefer DI/IOC container |
| Complex creation logic | Multiple steps, conditional creation | **Factory** | Only if creation is truly complex |
| 5+ constructor params | Hard to read, many optional params | **Builder** | Not worth it for 2-3 params |
| Object creation is expensive | Heavy I/O, computation | **Prototype** | Deep copy can be expensive too |

### Structural Patterns

| Code Smell | Problem | Pattern | Caution |
|------------|---------|---------|---------|
| Non-functional concerns mixed in | Logging, auth, caching scattered everywhere | **Proxy** (AOP) | Use dynamic proxy, not static |
| Interface incompatibility | Legacy APIs, third-party libs | **Adapter** | Last resort - refactor if possible |
| Layered function enhancement | Nested wrappers (e.g., Java I/O) | **Decorator** | Can create deep class hierarchies |
| Complex subsystem calls | Multiple fine-grained APIs | **Facade** | Balance interface granularity |
| Tree-like data + operations | File system, org chart | **Composite** | Only if data is truly tree-structured |
| Massive duplicate objects | String pool, cached integers | **Flyweight** | Must be immutable |

### Behavioral Patterns

| Code Smell | Problem | Pattern | Caution |
|------------|---------|---------|---------|
| 10+ if-else for algorithms | Payment methods, export formats | **Strategy** | Need 3+ runtime variants to justify |
| Tight coupling between core and non-core | Email notifications in checkout | **Observer** | Use async to avoid blocking |
| Repeated algorithm skeleton | Template code across subclasses | **Template Method** | Inheritance limits flexibility |
| Sequential request handlers | Filters, interceptors | **Chain of Responsibility** | Long chains hurt performance |
| Complex state transitions | Game states, workflow | **State** | Simple flags sufficient for 2-3 states |
| Complex multi-object interaction | UI widget coordination | **Mediator** | Mediator can become god class |

---

## Decision Framework

### Step 1: Identify the Real Problem

**Ask**:
1. What is the current pain point? (Be specific)
2. Have I encountered this problem at least 3 times?
3. Will this problem recur in the future?

**If NO to any** → Don't use patterns yet. Write simple code.

### Step 2: Verify Pattern Applicability

**Check against pattern's core intent**:

```
Problem: "I have 3 payment gateways and need runtime selection"

Pattern candidates:
- Strategy ✅ (encapsulate interchangeable algorithms)
- Factory ✅ (decouple creation)
- Template Method ❌ (not about algorithm skeleton)
- State ❌ (not about state transitions)

Choose: Strategy + Factory
```

### Step 3: Assess Complexity Trade-off

**Before implementation, estimate**:

| Aspect | Without Pattern | With Pattern | Net Benefit |
|--------|----------------|--------------|-------------|
| Lines of code | 50 | 150 | -100 😞 |
| Adding new gateway | Modify 3 places | Add 1 class | ✅ |
| Testability | Hard to mock | Easy to inject | ✅ |
| Team understanding | Everyone gets it | Need pattern knowledge | ⚠️ |

**If costs > benefits** → Don't use the pattern.

### Step 4: Continuous Refactoring Approach

**Recommended workflow** (避免过度设计的最有效方法):

```
1. Write MVP code (最小可行性)
   └─ Focus: Make it work
   
2. Add unit tests
   └─ Protection: Enable safe refactoring
   
3. Identify pain points through usage
   └─ Real problems, not imagined ones
   
4. Refactor in small steps
   └─ Apply patterns incrementally
   
5. Verify improvement
   └─ Code quality metrics, team feedback
```

**Example**:
```python
# Iteration 1: MVP (50 lines)
def process_payment(amount, method):
    if method == "stripe":
        # Stripe logic
    elif method == "paypal":
        # PayPal logic

# Iteration 2: After adding 3rd gateway → Refactor to Strategy
class PaymentProcessor:
    def __init__(self, strategy: PaymentStrategy):
        self._strategy = strategy

# Iteration 3: After needing factory → Add Factory
class PaymentFactory:
    @staticmethod
    def create(method: str) -> PaymentStrategy:
        ...
```

---

## Pattern Selection Guide

### Creational Patterns Comparison

```
Need to create objects?
├─ Complex creation logic? → Factory
├─ Many optional parameters? → Builder
├─ Expensive to create? → Prototype
└─ Global unique instance? → Singleton (consider DI instead)
```

**Specific scenarios**:
- **Singleton**: Configuration, ID generator, connection pool
  - ⚠️ **Risk**: Hidden dependencies, hard to test, poor extensibility
  - **Better**: Use IoC container (Spring, Guice) to enforce uniqueness

- **Factory**: Creation involves conditional logic, need to decouple
  - Combine with **Strategy** to create strategy instances

- **Builder**: 5+ constructor params, especially with many optionals
  - Not worth it for simple objects (< 3 params)

- **Prototype**: Object creation requires heavy I/O or computation
  - ⚠️ **Risk**: Deep copy complexity, shared mutable state

### Structural Patterns Comparison

```
Need to organize classes/objects?
├─ Interface incompatibility? → Adapter (last resort)
├─ Add functionality without subclassing? → Decorator
├─ Control access (logging, auth, cache)? → Proxy
├─ Simplify complex subsystem? → Facade
├─ Tree-structured data? → Composite
└─ Save memory with shared objects? → Flyweight (must be immutable)
```

**Key distinctions**:
- **Proxy vs Decorator**:
  - Proxy: Control access (auth, lazy loading, remote)
  - Decorator: Enhance functionality (add features)
  - Both have same structure, different intent

- **Adapter**: "事后补救" - Fix incompatible interfaces
  - Prefer refactoring over adapting if you control the code

### Behavioral Patterns Comparison

```
Need to handle object interactions?
├─ Multiple interchangeable algorithms? → Strategy
├─ One-to-many event notification? → Observer
├─ Algorithm skeleton with varying steps? → Template Method
├─ Sequential request processing? → Chain of Responsibility
├─ Complex state transitions? → State
└─ Decouple many-to-many interactions? → Mediator
```

**When to use what**:

| Need | Simple Solution | Pattern Solution | Pattern |
|------|----------------|------------------|---------|
| 2 algorithms | if-else | Only if runtime swapping needed | Strategy |
| Event notification | Direct call | Decouple core from non-core | Observer |
| Code reuse | Copy-paste | Shared skeleton, varying steps | Template |
| Request filtering | Single function | Chain multiple handlers | Chain |
| 2-3 states | Boolean flags | Complex state machine | State |

---

## Implementation Checklist

### Before Writing Code

- [ ] **Problem exists**: Identified concrete code smell or pain point
- [ ] **Frequency check**: Problem occurred 3+ times or will recur
- [ ] **Simplicity first**: Verified simpler solutions are insufficient
- [ ] **Team alignment**: Team understands the pattern and its trade-offs
- [ ] **Test coverage**: Unit tests exist to enable safe refactoring

### During Implementation

- [ ] **Follow SOLID principles**:
  - Single Responsibility: Each class has one reason to change
  - Open/Closed: Open for extension, closed for modification
  - Liskov Substitution: Subtypes are substitutable
  - Interface Segregation: No fat interfaces
  - Dependency Inversion: Depend on abstractions

- [ ] **Maintain readability**: Code is clearer than before
- [ ] **Document intent**: Comments explain WHY, not WHAT
- [ ] **Incremental refactoring**: Small steps, verify after each

### After Implementation

- [ ] **Measure improvement**:
  - Extensibility: How easy to add new behavior?
  - Testability: How easy to write tests?
  - Readability: Can team members understand it?
  - Maintainability: Reduced coupling and duplication?

- [ ] **Code review**: Get team feedback on design decisions
- [ ] **Update documentation**: Explain pattern choice and trade-offs

---

## Common Mistakes and Fixes

### Mistake 1: Pattern Before Problem

❌ **Wrong approach**:
```
"I just learned Observer pattern, let me find places to use it"
```

✅ **Correct approach**:
```
"This notification code is tightly coupled to checkout.
How can I decouple? → Observer pattern fits here"
```

### Mistake 2: Future-Proofing

❌ **Over-engineering**:
```python
# "We MIGHT need multiple databases in the future"
class DatabaseFactory:
    def create_connection(self, db_type):
        if db_type == "postgres":
            return PostgresConnection()
        # Only postgres is used, no other types exist

# Current reality: Only PostgreSQL, no plans to change
```

✅ **YAGNI approach**:
```python
# Start simple
connection = PostgresConnection()

# Refactor to Factory WHEN second database is actually needed
```

### Mistake 3: Singleton Abuse

❌ **Anti-pattern**:
```python
class Database(metaclass=Singleton):
    def __init__(self):
        self.connection = connect_to_db()

# Problems:
# - Hidden dependency
# - Hard to test (can't mock)
# - Violates Single Responsibility
```

✅ **Better approach**:
```python
# Use dependency injection
class OrderService:
    def __init__(self, db_connection):
        self._db = db_connection  # Explicit dependency

# In main.py or IoC container:
db = create_database_connection()
service = OrderService(db)  # Testable, explicit
```

### Mistake 4: Premature Abstraction

❌ **Over-engineered**:
```python
# 3 lines of actual logic, 50 lines of pattern code
class PaymentStrategyFactory:
    @staticmethod
    def create(method):
        return _STRATEGIES.get(method)

class PaymentContext: ...
class StripeStrategy: ...
class PayPalStrategy: ...

# Reality: Only 2 payment methods, never change
```

✅ **Start simple**:
```python
def process_payment(amount, method):
    if method == "stripe":
        return stripe.charge(amount)
    elif method == "paypal":
        return paypal.pay(amount)

# Refactor to Strategy WHEN:
# - 3+ payment methods exist
# - Runtime switching is needed
# - if-else becomes unmanageable
```

---

## Pattern Combinations (常见结合)

### Strategy + Factory
```python
# Factory creates strategies
class PaymentFactory:
    @staticmethod
    def create_strategy(method: str) -> PaymentStrategy:
        strategies = {
            "stripe": StripePayment,
            "paypal": PayPalPayment,
        }
        return strategies[method]()

# Strategy handles algorithm variation
processor = PaymentProcessor(PaymentFactory.create_strategy("stripe"))
```

### Template Method + Strategy
```python
# Template defines skeleton
class DataExporter(ABC):
    def export(self):
        data = self.fetch_data()      # Template step
        formatted = self.format(data)  # Varying step (hook)
        self.save(formatted)           # Template step
    
    @abstractmethod
    def format(self, data): pass  # Subclass implements

# Or replace Template with Strategy for more flexibility
class DataExporter:
    def __init__(self, formatter: FormatterStrategy):
        self._formatter = formatter
```

### Observer + Mediator
```python
# Observer for event notification
# Mediator for complex multi-direction communication

# Use Observer when: One-way notifications (email on order)
# Use Mediator when: Complex two-way interactions (UI widgets)
```

### Prototype + Factory
```python
# Factory decides which prototype to clone
class ShapeFactory:
    def __init__(self):
        self._prototypes = {
            "circle": Circle(radius=10),
            "square": Square(side=20),
        }
    
    def create(self, shape_type):
        return self._prototypes[shape_type].clone()
```

---

## Validation Tests

### Test Pattern Necessity

```python
# Test 1: Is pattern solving a real problem?
def test_extensibility_improvement():
    """Adding new payment method should not modify existing code"""
    # Before pattern: Requires modifying PaymentProcessor
    # After pattern: Just add new Strategy class
    assert can_add_new_strategy_without_modifying_context()

# Test 2: Is implementation correct?
def test_strategy_interchangeability():
    """Strategies should be truly interchangeable"""
    processor = PaymentProcessor(StripePayment())
    result1 = processor.pay(100)
    
    processor.set_strategy(PayPalPayment())
    result2 = processor.pay(100)
    
    assert type(result1) == type(result2)

# Test 3: Does it follow SOLID?
def test_open_closed_principle():
    """Context should not know concrete strategy types"""
    source = inspect.getsource(PaymentProcessor)
    assert "StripePayment" not in source
    assert "PayPalPayment" not in source
```

### Test for Over-Engineering

```python
# Complexity metric
def test_code_complexity():
    """Pattern should reduce complexity, not increase it"""
    before_lines = count_lines(legacy_code)
    after_lines = count_lines(refactored_code)
    
    # If 3x more code for same functionality → Over-engineering
    assert after_lines < before_lines * 2

# Readability metric (team vote)
def test_team_understanding():
    """Team should understand the code"""
    survey_results = team_survey("Do you understand this code?")
    assert survey_results["yes"] > 0.8  # 80%+ should understand
```

---

## Principles (心法) Reference

### SOLID Principles

**Single Responsibility**: Each class has one reason to change
- Violation sign: Class has multiple unrelated methods
- Fix: Split into focused classes

**Open/Closed**: Open for extension, closed for modification
- Violation sign: Adding feature requires modifying existing class
- Fix: Use polymorphism (Strategy, Template Method)

**Liskov Substitution**: Subtypes must be substitutable
- Violation sign: Subclass changes parent behavior in unexpected ways
- Fix: Ensure behavioral compatibility

**Interface Segregation**: No fat interfaces
- Violation sign: Clients forced to implement unused methods
- Fix: Split interfaces into focused contracts

**Dependency Inversion**: Depend on abstractions, not concretions
- Violation sign: High-level module imports low-level module directly
- Fix: Introduce interface/abstract class

### Other Key Principles

**KISS** (Keep It Simple, Stupid):
- Simplest solution is usually the best
- Don't use patterns for simple code

**YAGNI** (You Aren't Gonna Need It):
- Don't implement based on speculation
- Add complexity only when problem exists

**DRY** (Don't Repeat Yourself):
- Extract common logic
- But don't over-abstract (pragmatic DRY)

**LOD** (Law of Demeter):
- Only talk to immediate friends
- Avoid `a.getB().getC().doSomething()`

---

## Quick Reference: Pattern Decision Tree

```
What's the problem?

OBJECT CREATION ISSUES
├─ Complex creation logic → Factory
├─ Too many constructor params → Builder
├─ Expensive to create → Prototype
└─ Need single instance → Singleton (or DI)

STRUCTURAL ISSUES
├─ Interface incompatibility → Adapter
├─ Need to add features → Decorator
├─ Need to control access → Proxy
├─ Complex subsystem → Facade
├─ Tree-like data → Composite
└─ Memory optimization → Flyweight

BEHAVIORAL ISSUES
├─ Multiple algorithms → Strategy
├─ Event notification → Observer
├─ Code reuse skeleton → Template Method
├─ Request chain → Chain of Responsibility
├─ State transitions → State
├─ Complex interactions → Mediator
├─ Undo/redo → Command
└─ Snapshot → Memento

NOT SURE?
└─ Don't use patterns. Keep it simple.
```

---

## References

For detailed pattern implementations, see:
- [references/pattern-catalog.md](./references/pattern-catalog.md) - All patterns with examples
- [references/refactoring-guide.md](./references/refactoring-guide.md) - Step-by-step refactoring
- [references/anti-patterns.md](./references/anti-patterns.md) - Common mistakes
- [references/case-studies.md](./references/case-studies.md) - Real-world examples

## Summary

**High-Quality Code = Theory + Engineering + Refactoring**

1. **Theory**: SOLID principles + Design patterns
2. **Engineering**: Code review + Unit tests + Standards
3. **Refactoring**: Continuous improvement + Evolutionary design

**Remember**:
- Patterns are tools, not goals
- Solve problems, don't create them
- Simple code > Clever code
- Refactor incrementally, test continuously
- Team understanding > Personal preference
