# Design Patterns Catalog - Detailed Reference

This file provides comprehensive details on all 23 GoF patterns based on "设计模式之美" principles.

---

## Creational Patterns (创建型)

### 1. Singleton (单例模式)

**Intent**: Ensure a class has only one instance and provide global access point.

**适用场景**:
- Configuration managers (配置管理器)
- ID generators (全局ID生成器)
- Connection pools (连接池)
- Thread pools (线程池)
- Logging (日志对象)

**实现要点**:
```python
# Thread-safe singleton (Python)
class Singleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

**反模式警告** (Why it's often an anti-pattern):
1. **隐藏依赖**: Dependencies not visible in constructor
2. **难以测试**: Can't mock or replace in tests
3. **不支持有参构造**: Can't pass parameters to constructor
4. **扩展性差**: Hard to extend to multi-instance later

**更好的替代方案**:
```python
# Use IoC container instead (Spring example)
@Configuration
class AppConfig:
    @Bean
    @Scope("singleton")
    def database_connection(self):
        return DatabaseConnection()

# Or simple module-level instance (Python)
_connection = DatabaseConnection()

def get_connection():
    return _connection
```

**结合使用**:
- Often combined with **Factory** or **IoC container** to enforce uniqueness
- Don't hardcode singleton; let framework manage lifecycle

---

### 2. Factory (工厂模式)

**Intent**: Encapsulate object creation logic.

**适用场景**:
- Creation logic is complex (involves multiple steps, validation)
- Need to decouple object creation from usage
- Create different objects based on runtime parameters

**三种工厂模式对比**:

| Type | Use Case | Example |
|------|----------|---------|
| **Simple Factory** | Single factory method, limited types | `createShape(type)` |
| **Factory Method** | Subclasses decide which class to instantiate | `createButton()` overridden by platform |
| **Abstract Factory** | Families of related objects | UI toolkit (Button, Checkbox, etc.) |

**实现示例**:
```python
# Simple Factory
class ShapeFactory:
    @staticmethod
    def create(shape_type):
        if shape_type == "circle":
            return Circle()
        elif shape_type == "square":
            return Square()
        raise ValueError(f"Unknown shape: {shape_type}")

# Factory Method
class Button(ABC):
    @abstractmethod
    def render(self): pass

class WindowsButton(Button):
    def render(self):
        return "<Windows Button>"

class MacButton(Button):
    def render(self):
        return "<Mac Button>"

class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: pass

class WindowsFactory(UIFactory):
    def create_button(self) -> Button:
        return WindowsButton()
```

**常见错误**:
- ❌ Simple objects that can be directly `new`-ed don't need factory
- ❌ Don't create factory for every class "just in case"

**结合使用**:
- **Factory + Strategy**: Factory creates strategy instances
- **Factory + Singleton**: Factory ensures singleton creation

---

### 3. Builder (建造者模式)

**Intent**: Separate construction of complex object from its representation.

**适用场景**:
- Constructor has 5+ parameters (especially many optional ones)
- Parameters have validation dependencies
- Need to create immutable objects

**实现示例**:
```python
class HttpRequest:
    def __init__(self, url, method, headers, body, timeout):
        self.url = url
        self.method = method
        self.headers = headers
        self.body = body
        self.timeout = timeout

# Builder pattern
class HttpRequestBuilder:
    def __init__(self):
        self._url = None
        self._method = "GET"
        self._headers = {}
        self._body = None
        self._timeout = 30
    
    def url(self, url):
        self._url = url
        return self
    
    def method(self, method):
        self._method = method
        return self
    
    def header(self, key, value):
        self._headers[key] = value
        return self
    
    def body(self, body):
        self._body = body
        return self
    
    def build(self):
        if not self._url:
            raise ValueError("URL is required")
        return HttpRequest(
            self._url, self._method, 
            self._headers, self._body, self._timeout
        )

# Usage
request = (HttpRequestBuilder()
    .url("https://api.example.com")
    .method("POST")
    .header("Content-Type", "application/json")
    .body({"key": "value"})
    .build())
```

**何时不用**:
- ❌ Simple objects with 2-3 parameters → Use constructor
- ❌ Parameters have no validation dependencies → Named parameters

---

### 4. Prototype (原型模式)

**Intent**: Create objects by cloning existing instances.

**适用场景**:
- Object creation is expensive (heavy I/O, complex computation)
- Need multiple similar objects with slight variations

**实现要点**:
```python
import copy

class Prototype(ABC):
    @abstractmethod
    def clone(self):
        pass

class ConcretePrototype(Prototype):
    def __init__(self, field1, field2, expensive_data):
        self.field1 = field1
        self.field2 = field2
        self.expensive_data = expensive_data  # Took 5 seconds to compute
    
    def clone(self):
        # Shallow copy
        return copy.copy(self)
    
    def deep_clone(self):
        # Deep copy
        return copy.deepcopy(self)

# Usage
original = ConcretePrototype("A", "B", compute_expensive_data())
clone1 = original.clone()
clone1.field1 = "C"  # Modify without affecting original
```

**深拷贝 vs 浅拷贝**:
- **浅拷贝 (Shallow)**: Copies object, but shares references to nested objects
- **深拷贝 (Deep)**: Recursively copies everything (expensive, complex)

**常见问题**:
- Deep copy can be as expensive as creating from scratch
- Shared mutable objects in shallow copy can cause bugs

**结合使用**:
- **Prototype + Memento**: Save object snapshots

---

## Structural Patterns (结构型)

### 5. Proxy (代理模式)

**Intent**: Control access to an object.

**适用场景** (非业务功能解耦):
- Monitoring (监控统计)
- Authentication/Authorization (鉴权)
- Rate limiting (限流)
- Transaction management (事务)
- Logging (日志)
- RPC (远程调用)
- Caching (缓存)

**三种代理类型**:

| Type | Purpose | Example |
|------|---------|---------|
| **Virtual Proxy** | Lazy initialization | Load large image only when needed |
| **Protection Proxy** | Access control | Check permissions before method call |
| **Remote Proxy** | Remote object representation | RPC stub |

**静态代理 vs 动态代理**:

```python
# Static Proxy (避免 - 类爆炸)
class UserServiceProxy:
    def __init__(self, user_service):
        self._service = user_service
    
    def create_user(self, user):
        log("create_user called")
        return self._service.create_user(user)
    
    # Need to wrap every method manually!

# Dynamic Proxy (推荐)
class LoggingProxy:
    def __init__(self, target):
        self._target = target
    
    def __getattr__(self, name):
        attr = getattr(self._target, name)
        if callable(attr):
            def wrapper(*args, **kwargs):
                log(f"{name} called")
                return attr(*args, **kwargs)
            return wrapper
        return attr

# Usage
service = LoggingProxy(UserService())
service.create_user(user)  # Automatically logged
```

**Proxy vs Decorator**:
- **Proxy**: Control access (focus on when/how to call)
- **Decorator**: Enhance functionality (focus on what additional behavior to add)
- Same structure, different intent

**结合使用**:
- **Spring AOP**: Uses dynamic proxy for cross-cutting concerns

---

### 6. Adapter (适配器模式)

**Intent**: Convert interface of a class into another interface clients expect.

**适用场景** (事后补救 - Last resort):
- Encapsulate defective interfaces (封装有缺陷的接口)
- Unify multiple similar interfaces (统一多个类的接口)
- Replace external dependencies (替换依赖的外部系统)
- Adapt incompatible interfaces (兼容老版本接口)

**类适配器 vs 对象适配器**:

```python
# Class Adapter (使用继承)
class Target(ABC):
    @abstractmethod
    def request(self): pass

class Adaptee:
    def specific_request(self):
        return "Adaptee behavior"

class ClassAdapter(Adaptee, Target):
    def request(self):
        return self.specific_request()

# Object Adapter (使用组合 - 更灵活)
class ObjectAdapter(Target):
    def __init__(self, adaptee: Adaptee):
        self._adaptee = adaptee
    
    def request(self):
        return self._adaptee.specific_request()
```

**真实场景示例**:
```python
# Adapting legacy payment API
class LegacyPaymentAPI:
    def make_payment(self, card_number, amount):
        # Old interface
        pass

class ModernPaymentInterface(ABC):
    @abstractmethod
    def pay(self, payment_method, amount): pass

class PaymentAdapter(ModernPaymentInterface):
    def __init__(self, legacy_api: LegacyPaymentAPI):
        self._legacy = legacy_api
    
    def pay(self, payment_method, amount):
        # Adapt new interface to old implementation
        card_number = payment_method.get_card_number()
        return self._legacy.make_payment(card_number, amount)
```

**重要原则**:
- Adapter is a **补救措施** (remediation), not first choice
- If you control the code, **refactor** instead of adapting
- Use when integrating third-party libraries or legacy systems

---

### 7. Decorator (装饰器模式)

**Intent**: Attach additional responsibilities to an object dynamically.

**适用场景**:
- Need to add features without modifying existing code
- Support flexible combination of features (nested decorators)
- Alternative to subclassing (avoid class explosion)

**经典示例 - Java I/O**:
```python
from abc import ABC, abstractmethod

# Component
class DataSource(ABC):
    @abstractmethod
    def read(self) -> str: pass
    
    @abstractmethod
    def write(self, data: str): pass

# Concrete Component
class FileDataSource(DataSource):
    def __init__(self, filename):
        self._filename = filename
    
    def read(self):
        with open(self._filename) as f:
            return f.read()
    
    def write(self, data):
        with open(self._filename, 'w') as f:
            f.write(data)

# Decorators
class CompressionDecorator(DataSource):
    def __init__(self, source: DataSource):
        self._source = source
    
    def read(self):
        data = self._source.read()
        return decompress(data)
    
    def write(self, data):
        compressed = compress(data)
        self._source.write(compressed)

class EncryptionDecorator(DataSource):
    def __init__(self, source: DataSource):
        self._source = source
    
    def read(self):
        data = self._source.read()
        return decrypt(data)
    
    def write(self, data):
        encrypted = encrypt(data)
        self._source.write(encrypted)

# Usage - Nested decorators
source = FileDataSource("data.txt")
source = CompressionDecorator(source)
source = EncryptionDecorator(source)
source.write("Hello")  # Encrypted → Compressed → Saved
```

**优缺点**:
- ✅ Flexible combination of features
- ✅ Single Responsibility Principle
- ❌ Many small classes (complexity)
- ❌ Deep decorator stack (debugging difficulty)

---

[Continue with remaining patterns: Facade, Composite, Flyweight, and all Behavioral patterns...]

---

## Pattern Relationships

### Creational Pattern Relationships
```
Factory Method
    ↓ (evolution)
Abstract Factory
    ↓ (combines with)
Builder (for complex products)

Singleton
    ↓ (enforced by)
Factory or IoC Container
```

### Structural Pattern Relationships
```
Adapter (interface conversion)
    vs
Decorator (feature enhancement)
    vs
Proxy (access control)
    ↓ (all use)
Composition over Inheritance
```

### Behavioral Pattern Relationships
```
Strategy (algorithm selection)
    vs
State (state-dependent behavior)
    vs
Template Method (skeleton algorithm)

Observer (event notification)
    vs
Mediator (complex multi-way communication)
```

---

## Anti-Patterns Summary

### Most Abused Patterns

1. **Singleton**: Overused for things that don't need to be global
   - Fix: Use dependency injection

2. **Factory**: Created for every simple class
   - Fix: Use factory only when creation is complex

3. **Observer**: Synchronous implementation blocks main thread
   - Fix: Use async/event loop

4. **Visitor**: Breaks encapsulation, hard to maintain
   - Fix: Avoid unless absolutely necessary

### Pattern Over-Engineering Signs

- Using Strategy for 2 fixed algorithms
- Using Builder for 3 parameters
- Using Factory for direct `new` equivalents
- Using patterns "just in case we need it later"

---

## Further Reading

- Design Patterns: Elements of Reusable Object-Oriented Software (GoF)
- Head First Design Patterns
- Refactoring: Improving the Design of Existing Code (Martin Fowler)
- 设计模式之美 (王争)
