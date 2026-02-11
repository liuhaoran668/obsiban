# Refactoring to Patterns - Step-by-Step Guide

Based on "持续重构 (Continuous Refactoring)" principles from 设计模式之美.

---

## Core Refactoring Philosophy

**持续重构 > 一次性设计**

Don't try to design perfectly upfront. Instead:

1. **Write MVP code** - Focus on making it work
2. **Add tests** - Enable safe refactoring
3. **Identify smells** - Through real usage, not speculation
4. **Refactor incrementally** - Small steps, verify after each
5. **Apply patterns** - Only when justified by actual pain

---

## Refactoring Workflow

### Step 1: Establish Safety Net

**Before any refactoring, ensure test coverage**:

```python
# Original code (no tests)
def process_payment(amount, method):
    if method == "stripe":
        # 50 lines of Stripe logic
    elif method == "paypal":
        # 50 lines of PayPal logic

# Step 1: Add characterization tests
def test_stripe_payment():
    result = process_payment(100, "stripe")
    assert result["status"] == "success"
    assert result["amount"] == 100

def test_paypal_payment():
    result = process_payment(100, "paypal")
    assert result["status"] == "success"
    assert result["amount"] == 100

# Now safe to refactor
```

### Step 2: Identify Code Smell

**Common smells and their pattern solutions**:

| Code Smell | Example | Target Pattern |
|------------|---------|----------------|
| Long Method | 100+ line method | Extract Method → Template/Strategy |
| Large Class | 20+ methods, 1000+ lines | Extract Class → Facade |
| Long Parameter List | 7+ parameters | Builder/Parameter Object |
| Divergent Change | Class changes for multiple reasons | Extract Class (SRP) |
| Shotgun Surgery | One change requires editing many classes | Move Method/Field |
| Feature Envy | Method uses another class more than its own | Move Method |
| Data Clumps | Same group of data appears together | Extract Class |
| Primitive Obsession | Using primitives instead of small objects | Replace with Value Object |
| Switch Statements | Long switch on type codes | Replace with Polymorphism (Strategy/State) |
| Lazy Class | Class does too little | Inline Class |
| Speculative Generality | "Might need it someday" abstraction | Remove dead code |
| Temporary Field | Field only set in certain circumstances | Extract Class |
| Message Chains | `a.getB().getC().getD()` | Hide Delegate |
| Middle Man | Class delegates most work to another | Remove Middle Man |
| Inappropriate Intimacy | Class accesses internals of another | Move Method/Extract Class |
| Alternative Classes with Different Interfaces | Similar classes, different interfaces | Rename Method/Extract Superclass |
| Incomplete Library Class | Library missing feature | Introduce Foreign Method/Wrapper |
| Data Class | Class with only getters/setters | Move Method (add behavior) |
| Refused Bequest | Subclass doesn't use parent methods | Replace Inheritance with Delegation |
| Comments | Excessive comments explaining complex code | Extract Method/Rename |

### Step 3: Choose Refactoring Strategy

**Decision tree**:

```
What's the primary smell?

COMPLEXITY SMELL
├─ Long if-else chains → Strategy or State
├─ Repeated code → Template Method or Extract Method
└─ Complex creation → Factory or Builder

COUPLING SMELL
├─ Tight coupling between modules → Observer or Mediator
├─ Class knows too much about others → Facade or Law of Demeter
└─ Hard to swap implementations → Dependency Injection + Strategy

RIGIDITY SMELL
├─ Hard to add new features → Open/Closed via Polymorphism
├─ Change ripples through system → Decouple with interfaces
└─ Can't test → Dependency Injection + Mock
```

### Step 4: Refactor Incrementally

**Example: Refactoring to Strategy Pattern**

**Original Code (100 lines, 3 payment methods)**:
```python
def process_payment(order, method):
    if method == "stripe":
        # Validate Stripe credentials
        api_key = get_stripe_key()
        if not api_key:
            raise ValueError("Stripe not configured")
        
        # Create Stripe charge
        charge = stripe.Charge.create(
            amount=order.total * 100,
            currency="usd",
            source=order.stripe_token,
            description=f"Order {order.id}"
        )
        
        # Update order
        order.payment_id = charge.id
        order.status = "paid"
        order.save()
        
        # Send confirmation email
        send_email(order.customer, "Payment confirmed")
        
        return {"status": "success", "transaction_id": charge.id}
    
    elif method == "paypal":
        # Similar 30 lines for PayPal
        ...
    
    elif method == "crypto":
        # Similar 30 lines for Crypto
        ...
    
    else:
        raise ValueError(f"Unknown payment method: {method}")
```

**Iteration 1: Extract Methods (Prepare for Strategy)**

```python
def process_payment(order, method):
    if method == "stripe":
        return _process_stripe_payment(order)
    elif method == "paypal":
        return _process_paypal_payment(order)
    elif method == "crypto":
        return _process_crypto_payment(order)
    else:
        raise ValueError(f"Unknown payment method: {method}")

def _process_stripe_payment(order):
    # All Stripe logic here
    api_key = get_stripe_key()
    if not api_key:
        raise ValueError("Stripe not configured")
    
    charge = stripe.Charge.create(...)
    order.payment_id = charge.id
    order.status = "paid"
    order.save()
    send_email(order.customer, "Payment confirmed")
    
    return {"status": "success", "transaction_id": charge.id}

# Similar for _process_paypal_payment and _process_crypto_payment
```

**Run tests → All pass → Commit**

---

**Iteration 2: Introduce Strategy Interface**

```python
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def process(self, order) -> dict:
        """Process payment and return result"""
        pass

class StripePaymentStrategy(PaymentStrategy):
    def process(self, order):
        # Move _process_stripe_payment logic here
        api_key = get_stripe_key()
        if not api_key:
            raise ValueError("Stripe not configured")
        
        charge = stripe.Charge.create(...)
        order.payment_id = charge.id
        order.status = "paid"
        order.save()
        send_email(order.customer, "Payment confirmed")
        
        return {"status": "success", "transaction_id": charge.id}

# Similar for PayPalPaymentStrategy and CryptoPaymentStrategy

# Update process_payment to use strategies
def process_payment(order, method):
    strategies = {
        "stripe": StripePaymentStrategy(),
        "paypal": PayPalPaymentStrategy(),
        "crypto": CryptoPaymentStrategy(),
    }
    
    strategy = strategies.get(method)
    if not strategy:
        raise ValueError(f"Unknown payment method: {method}")
    
    return strategy.process(order)
```

**Run tests → All pass → Commit**

---

**Iteration 3: Extract Factory**

```python
class PaymentStrategyFactory:
    _strategies = {
        "stripe": StripePaymentStrategy,
        "paypal": PayPalPaymentStrategy,
        "crypto": CryptoPaymentStrategy,
    }
    
    @classmethod
    def create(cls, method: str) -> PaymentStrategy:
        strategy_class = cls._strategies.get(method)
        if not strategy_class:
            raise ValueError(f"Unknown payment method: {method}")
        return strategy_class()
    
    @classmethod
    def register(cls, method: str, strategy_class: type):
        """Allow registering new payment methods"""
        cls._strategies[method] = strategy_class

# Simplified process_payment
def process_payment(order, method):
    strategy = PaymentStrategyFactory.create(method)
    return strategy.process(order)
```

**Run tests → All pass → Commit**

---

**Iteration 4: Introduce Payment Processor (Context)**

```python
class PaymentProcessor:
    def __init__(self, strategy: PaymentStrategy):
        self._strategy = strategy
    
    def process(self, order):
        return self._strategy.process(order)
    
    def set_strategy(self, strategy: PaymentStrategy):
        """Allow runtime strategy switching"""
        self._strategy = strategy

# Usage
processor = PaymentProcessor(
    PaymentStrategyFactory.create(order.payment_method)
)
result = processor.process(order)
```

**Run tests → All pass → Commit**

---

**Final State Comparison**:

| Aspect | Before | After |
|--------|--------|-------|
| Lines in main function | 100 | 3 |
| Adding new payment method | Modify `process_payment` | Add new Strategy class |
| Testing | Test giant function | Test each strategy independently |
| Coupling | High (all methods in one function) | Low (strategies independent) |
| Open/Closed | Violates (must modify) | Follows (extend by adding) |

---

## Refactoring to Other Patterns

### Refactoring to Observer Pattern

**Smell**: Notification logic mixed with core business logic

**Before**:
```python
def checkout(order):
    # Core business
    order.status = "completed"
    order.save()
    
    # Non-core concerns mixed in
    send_confirmation_email(order.customer, order)
    update_inventory(order.items)
    log_analytics("order_completed", order.id)
    notify_accounting(order.total)
    # Adding new notification requires modifying checkout!
```

**After** (Observer pattern):
```python
class OrderEvent:
    def __init__(self, event_type, order):
        self.type = event_type
        self.order = order

class OrderObserver(ABC):
    @abstractmethod
    def update(self, event: OrderEvent): pass

class EmailNotificationObserver(OrderObserver):
    def update(self, event):
        if event.type == "completed":
            send_confirmation_email(event.order.customer, event.order)

class InventoryObserver(OrderObserver):
    def update(self, event):
        if event.type == "completed":
            update_inventory(event.order.items)

class OrderSubject:
    def __init__(self):
        self._observers = []
    
    def attach(self, observer: OrderObserver):
        self._observers.append(observer)
    
    def notify(self, event: OrderEvent):
        for observer in self._observers:
            observer.update(event)

# Usage
order_subject = OrderSubject()
order_subject.attach(EmailNotificationObserver())
order_subject.attach(InventoryObserver())
order_subject.attach(AnalyticsObserver())

def checkout(order):
    order.status = "completed"
    order.save()
    
    # Notify all observers
    order_subject.notify(OrderEvent("completed", order))
    # Adding new notification = Add new observer (no modification!)
```

---

### Refactoring to Template Method

**Smell**: Repeated code structure with varying details

**Before**:
```python
def parse_csv(file):
    # Open file
    with open(file) as f:
        # Parse
        data = csv.reader(f)
        # Process
        result = [process_csv_row(row) for row in data]
        # Cleanup
        return result

def parse_json(file):
    # Open file
    with open(file) as f:
        # Parse
        data = json.load(f)
        # Process
        result = [process_json_item(item) for item in data]
        # Cleanup
        return result

# Repeated structure: Open → Parse → Process → Cleanup
```

**After** (Template Method):
```python
class DataParser(ABC):
    def parse(self, file):
        """Template method defining the algorithm skeleton"""
        # Step 1: Open (common)
        raw_data = self._open_file(file)
        
        # Step 2: Parse (varies by format)
        parsed_data = self._parse_data(raw_data)
        
        # Step 3: Process (common)
        result = self._process_data(parsed_data)
        
        # Step 4: Cleanup (common)
        self._cleanup()
        
        return result
    
    def _open_file(self, file):
        with open(file) as f:
            return f.read()
    
    @abstractmethod
    def _parse_data(self, raw_data):
        """Subclass implements format-specific parsing"""
        pass
    
    def _process_data(self, parsed_data):
        """Common processing logic"""
        return [self._transform_item(item) for item in parsed_data]
    
    @abstractmethod
    def _transform_item(self, item):
        """Subclass implements format-specific transformation"""
        pass
    
    def _cleanup(self):
        """Hook method - subclass can override if needed"""
        pass

class CSVParser(DataParser):
    def _parse_data(self, raw_data):
        return csv.reader(raw_data.splitlines())
    
    def _transform_item(self, row):
        return {"name": row[0], "value": row[1]}

class JSONParser(DataParser):
    def _parse_data(self, raw_data):
        return json.loads(raw_data)
    
    def _transform_item(self, item):
        return {"name": item["name"], "value": item["value"]}

# Usage
parser = CSVParser()
result = parser.parse("data.csv")
```

---

## Refactoring Anti-Patterns

### Anti-Pattern 1: Big Bang Refactoring

❌ **Wrong**:
```
"Let's rewrite the entire payment system using all design patterns!"
[3 weeks later]
[Nothing works, no way to rollback]
```

✅ **Right**:
```
Week 1: Extract payment methods to separate functions + Tests
Week 2: Introduce Strategy interface + Tests
Week 3: Add Factory + Tests
Week 4: Decouple with Observer for notifications + Tests
```

### Anti-Pattern 2: Refactoring Without Tests

❌ **Wrong**:
```python
# No tests
def refactor_complex_code():
    # Change 50 lines
    # Hope it still works
    # Deploy to production
    # 🔥 Everything breaks 🔥
```

✅ **Right**:
```python
# Step 1: Add characterization tests
def test_existing_behavior():
    assert process_payment(100, "stripe") == expected_result

# Step 2: Refactor with confidence
def refactor_complex_code():
    # Change code
    # Run tests
    # All green → Safe to deploy
```

### Anti-Pattern 3: Premature Pattern Application

❌ **Wrong**:
```python
# Day 1: Only Stripe payment exists
# Developer: "We MIGHT add PayPal someday, let's use Strategy!"
[Creates 5 classes for 1 payment method]
[PayPal never gets added]
[Wasted effort]
```

✅ **Right**:
```python
# Day 1: Simple if-else
if method == "stripe":
    return process_stripe(amount)

# Day 30: PayPal requirement confirmed
# Refactor to Strategy now (justified by actual need)
```

---

## Refactoring Checklist

### Before Refactoring

- [ ] **Test coverage exists** (characterization tests if needed)
- [ ] **Version control** (can rollback if needed)
- [ ] **Clear goal** (what smell am I fixing?)
- [ ] **Pattern identified** (which pattern solves this smell?)
- [ ] **Team alignment** (do others understand the plan?)

### During Refactoring

- [ ] **Small steps** (one change at a time)
- [ ] **Run tests after each step** (verify nothing broke)
- [ ] **Commit frequently** (every green test run)
- [ ] **Maintain behavior** (refactoring ≠ adding features)
- [ ] **Follow SOLID** (each step should improve design)

### After Refactoring

- [ ] **All tests pass** (behavior unchanged)
- [ ] **Code review** (get team feedback)
- [ ] **Documentation updated** (explain pattern choice)
- [ ] **Metrics improved** (cyclomatic complexity, coupling)
- [ ] **Team understands** (knowledge transfer complete)

---

## Measuring Refactoring Success

### Quantitative Metrics

```python
# Before refactoring
def process_payment(order, method):
    # Cyclomatic Complexity: 15
    # Lines of Code: 150
    # Coupling: High (knows about Stripe, PayPal, Crypto APIs)
    # Testability: Low (hard to mock)

# After refactoring (Strategy pattern)
class PaymentProcessor:
    # Cyclomatic Complexity: 2
    # Lines of Code: 10
    # Coupling: Low (depends on interface)
    # Testability: High (easy to inject mock strategies)
```

### Qualitative Assessment

**Ask**:
1. Is it easier to add new payment methods now? ✅
2. Is the code more readable? ✅
3. Are tests easier to write? ✅
4. Did complexity decrease? ✅
5. Can team members understand it? (Need to verify)

---

## Resources

- Refactoring: Improving the Design of Existing Code (Martin Fowler)
- Working Effectively with Legacy Code (Michael Feathers)
- Refactoring to Patterns (Joshua Kerievsky)
