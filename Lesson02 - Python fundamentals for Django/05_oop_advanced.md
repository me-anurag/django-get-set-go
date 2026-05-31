# 05 — OOP Advanced: Inheritance, super(), MRO, and Mixins

> **This is where Python OOP gets serious — and where Django's design becomes legible.** Django's class-based views are built entirely on inheritance and mixins. If you don't understand how `super()` works and what the Method Resolution Order is, you will write bugs in Django CBVs that you cannot explain or fix.

---

## 1. Inheritance — Building on What Exists

Inheritance lets one class **extend** another. The child class gets everything the parent has, and can add or override anything.

```python
class Animal:
    def __init__(self, name, sound):
        self.name = name
        self.sound = sound

    def speak(self):
        return f"{self.name} says {self.sound}"

    def breathe(self):
        return f"{self.name} breathes"

class Dog(Animal):   # Dog inherits from Animal
    def fetch(self):
        return f"{self.name} fetches the ball!"

class Cat(Animal):
    def purr(self):
        return f"{self.name} purrs..."

    def speak(self):   # overrides Animal's speak
        return f"{self.name} says MEOW (not {self.sound})"

dog = Dog("Rex", "Woof")
dog.speak()    # "Rex says Woof"  — inherited from Animal
dog.fetch()    # "Rex fetches the ball!"  — Dog's own method
dog.breathe()  # "Rex breathes"  — also inherited

cat = Cat("Whiskers", "Meow")
cat.speak()    # "Whiskers says MEOW (not Meow)"  — overridden
cat.purr()     # "Whiskers purrs..."  — Cat's own method

# isinstance checks the entire chain
isinstance(dog, Dog)    # True
isinstance(dog, Animal) # True — Dog IS an Animal
isinstance(cat, Dog)    # False
```

**Django equivalent — your models inherit from `models.Model`:**

```python
from django.db import models

class TimestampedModel(models.Model):
    """Abstract base model for created_at and updated_at fields."""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True   # doesn't create a DB table, used only for inheritance

class Article(TimestampedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    # Article table automatically has created_at and updated_at columns

class Comment(TimestampedModel):
    text = models.TextField()
    # Comment table also has created_at and updated_at columns
```

This pattern is used in virtually every serious Django project. You define common fields once in an abstract model and inherit everywhere.

---

## 2. `super()` — Calling the Parent Without Naming It

`super()` gives you a proxy object that delegates method calls to the parent class (or the next class in the MRO). It's essential for cooperative inheritance.

### 2.1 Basic Usage

```python
class Vehicle:
    def __init__(self, make, model, year):
        self.make = make
        self.model = model
        self.year = year
        print(f"Vehicle.__init__ called: {make} {model}")

    def start(self):
        return "Vehicle starting..."

class Car(Vehicle):
    def __init__(self, make, model, year, num_doors):
        super().__init__(make, model, year)   # call Vehicle's __init__ FIRST
        self.num_doors = num_doors             # then add Car-specific attributes
        print(f"Car.__init__ called: {num_doors} doors")

    def start(self):
        base_message = super().start()   # get Vehicle's start message
        return f"{base_message} (Car-specific warm-up complete)"

car = Car("Toyota", "Camry", 2023, 4)
# Vehicle.__init__ called: Toyota Camry
# Car.__init__ called: 4 doors

car.start()
# "Vehicle starting... (Car-specific warm-up complete)"
```

**What happens if you DON'T call `super().__init__()`?**

```python
class ElectricCar(Car):
    def __init__(self, make, model, year, num_doors, battery_kwh):
        # WRONG — forget to call super().__init__
        self.battery_kwh = battery_kwh

ec = ElectricCar("Tesla", "Model 3", 2023, 4, 75)
ec.battery_kwh  # 75 — fine
ec.make         # AttributeError! — make was never set because Vehicle.__init__ didn't run
```

**Always call `super().__init__()` in `__init__` unless you have a specific reason not to.**

### 2.2 `super()` in Django Views

This is why `super()` matters in Django:

```python
from django.views.generic import CreateView
from django.contrib.auth.mixins import LoginRequiredMixin

class ArticleCreateView(LoginRequiredMixin, CreateView):
    model = Article
    fields = ["title", "content"]

    def form_valid(self, form):
        # Add the current user as author before saving
        form.instance.author = self.request.user
        return super().form_valid(form)   # Django's form_valid handles the actual save and redirect
        # If you don't call super(), the form never saves!
```

This pattern is so common in Django CBVs that not understanding `super()` makes you unable to use them correctly.

---

## 3. Method Resolution Order (MRO) — Who Gets Called?

When multiple classes are involved in an inheritance chain, Python needs to know *which class's method to use*. This is determined by the **Method Resolution Order (MRO)** — a linearized list of classes to search.

```python
class A:
    def hello(self):
        return "A"

class B(A):
    def hello(self):
        return "B"

class C(A):
    def hello(self):
        return "C"

class D(B, C):   # multiple inheritance
    pass

d = D()
d.hello()   # "B"  — which class won?

# Check the MRO:
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
# D → B → C → A → object
```

Python uses the **C3 linearization algorithm** (don't memorize it — just understand the rules):

1. Start at the class being called
2. Go left-to-right through the parent classes listed in the class definition
3. A class is added to the MRO only after all of its subclasses already appear
4. The base `object` class always appears last

```python
# Visual:
#        object
#          |
#          A
#         / \
#        B   C
#         \ /
#          D

# MRO: D → B → C → A → object
```

**Why does this matter in Django?**

```python
from django.views.generic import UpdateView
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin

class ArticleUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Article
    permission_required = "blog.change_article"
    ...

# MRO: ArticleUpdateView → LoginRequiredMixin → PermissionRequiredMixin → UpdateView → ...
# When dispatch() is called:
# 1. Django looks in ArticleUpdateView → not found
# 2. LoginRequiredMixin.dispatch() is called → checks if logged in
# 3. Calls super().dispatch() → goes to PermissionRequiredMixin.dispatch()
# 4. PermissionRequiredMixin.dispatch() checks permission, calls super().dispatch()
# 5. UpdateView.dispatch() handles the actual request
```

Each mixin in the chain calls `super()`, passing control to the next in line. **This cooperative behavior only works if every class in the chain calls `super()`.**

---

## 4. Cooperative Multiple Inheritance — The Django Way

```python
class LoggingMixin:
    def save(self, *args, **kwargs):
        print(f"[LOG] Saving {self.__class__.__name__}: {self}")
        super().save(*args, **kwargs)   # must call super() — cooperates with other mixins

class ValidationMixin:
    def save(self, *args, **kwargs):
        self.validate()
        super().save(*args, **kwargs)   # also calls super()

    def validate(self):
        raise NotImplementedError("Subclasses must implement validate()")

class Article(LoggingMixin, ValidationMixin, models.Model):
    title = models.CharField(max_length=200)

    def validate(self):
        if not self.title:
            raise ValueError("Title is required")

article = Article(title="Test")
article.save()
# MRO: Article → LoggingMixin → ValidationMixin → models.Model → ...
# Article.save() not defined → LoggingMixin.save() → logs → calls super()
# → ValidationMixin.save() → validates → calls super()
# → models.Model.save() → actually hits the database
```

This chain only works because every `save()` calls `super().save()`. Break that chain anywhere, and the methods below it in the MRO never run.

---

## 5. Abstract Base Classes — Enforcing Contracts

An abstract class defines a structure that subclasses *must* implement. It cannot be instantiated directly.

```python
from abc import ABC, abstractmethod

class BasePaymentProcessor(ABC):
    """All payment processors must implement these methods."""

    @abstractmethod
    def charge(self, amount: float, currency: str) -> dict:
        """Charge the customer. Must return a dict with 'success' and 'transaction_id'."""
        pass

    @abstractmethod
    def refund(self, transaction_id: str, amount: float) -> bool:
        """Refund a transaction. Must return True if successful."""
        pass

    def process_with_retry(self, amount, currency, max_retries=3):
        """Concrete method that uses the abstract methods."""
        for attempt in range(max_retries):
            result = self.charge(amount, currency)
            if result["success"]:
                return result
        raise RuntimeError("Payment failed after retries")

class StripeProcessor(BasePaymentProcessor):
    def charge(self, amount, currency):
        # Stripe-specific implementation
        return {"success": True, "transaction_id": "stripe_123"}

    def refund(self, transaction_id, amount):
        # Stripe-specific refund
        return True

class PayPalProcessor(BasePaymentProcessor):
    def charge(self, amount, currency):
        return {"success": True, "transaction_id": "paypal_456"}

    def refund(self, transaction_id, amount):
        return True

# Can't instantiate abstract class:
processor = BasePaymentProcessor()   # TypeError: Can't instantiate abstract class

# Must implement all abstract methods:
class BrokenProcessor(BasePaymentProcessor):
    def charge(self, amount, currency):
        return {"success": True, "transaction_id": "x"}
    # forgot to implement refund()

broken = BrokenProcessor()   # TypeError: Can't instantiate abstract class (missing refund)
```

**Django's `View` class is essentially abstract** — you're expected to override `get()`, `post()`, etc.

---

## 6. Mixins — Add Behavior Without Full Inheritance

A mixin is a class designed to be used with multiple inheritance to *add specific behavior* without being a full standalone class.

**Rules of a good mixin:**
1. Should not have `__init__` (or if it does, always calls `super().__init__()`)
2. Should be designed to work with other classes, not stand alone
3. Provides a specific, narrow piece of functionality

```python
class SerializeMixin:
    """Add JSON serialization to any model."""
    def to_dict(self):
        return {
            field: getattr(self, field)
            for field in self._serializable_fields
        }

    def to_json(self):
        import json
        return json.dumps(self.to_dict(), default=str)

class TimestampMixin:
    """Add created_at and updated_at tracking."""
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        from datetime import datetime
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

    def touch(self):
        from datetime import datetime
        self.updated_at = datetime.now()

class ValidateMixin:
    """Add validate-before-save behavior."""
    def save(self, *args, **kwargs):
        self.full_clean()   # validate first
        super().save(*args, **kwargs)

    def full_clean(self):
        pass   # subclasses override this

class User(SerializeMixin, TimestampMixin, ValidateMixin):
    _serializable_fields = ["name", "email"]

    def __init__(self, name, email):
        self.name = name
        self.email = email
        super().__init__()   # triggers TimestampMixin.__init__

    def full_clean(self):
        if "@" not in self.email:
            raise ValueError(f"Invalid email: {self.email}")

user = User("Anurag", "anurag@example.com")
print(user.to_json())   # {"name": "Anurag", "email": "anurag@example.com"}
print(user.created_at)  # datetime object
```

### 6.1 Django's Built-in Mixins

Django ships with a rich set of mixins for class-based views:

```python
from django.contrib.auth.mixins import (
    LoginRequiredMixin,        # redirect to login if not authenticated
    PermissionRequiredMixin,   # require specific permissions
    UserPassesTestMixin,       # require a custom test function to pass
)
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView

# A view that requires login AND specific permission:
class ArticleUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Article
    permission_required = "blog.change_article"
    fields = ["title", "content"]
    success_url = "/articles/"

# A view with custom access logic:
class MyProfileView(UserPassesTestMixin, DetailView):
    model = UserProfile

    def test_func(self):
        """Return True if access should be granted."""
        return self.request.user == self.get_object().user
```

---

## 7. `__init_subclass__` — Hooks for Subclassing

A lesser-known but powerful hook that runs when a subclass is created.

```python
class PluginBase:
    _registry = {}

    def __init_subclass__(cls, plugin_name=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if plugin_name:
            PluginBase._registry[plugin_name] = cls
            print(f"Registered plugin: {plugin_name}")

class PaymentPlugin(PluginBase, plugin_name="payment"):
    pass

class EmailPlugin(PluginBase, plugin_name="email"):
    pass

print(PluginBase._registry)
# {"payment": <class 'PaymentPlugin'>, "email": <class 'EmailPlugin'>}
```

**Django uses this internally** to register models, signals, and apps. You'll rarely write it yourself but understanding it demystifies how `class Meta` in Django models works.

---

## 8. Descriptors — How Django Model Fields Actually Work

This is advanced, but it explains something that should feel magical: how `user.name` returns a string even though `name = CharField(...)` is a descriptor object.

```python
class Validator:
    """A descriptor that validates data on assignment."""
    def __init__(self, min_length=0, max_length=None):
        self.min_length = min_length
        self.max_length = max_length
        self.attr_name = None

    def __set_name__(self, owner, name):
        """Called when descriptor is assigned to a class attribute."""
        self.attr_name = f"_{name}_value"

    def __get__(self, obj, objtype=None):
        """Called on attribute access: obj.attr."""
        if obj is None:
            return self   # accessing on the class, not instance
        return getattr(obj, self.attr_name, None)

    def __set__(self, obj, value):
        """Called on attribute assignment: obj.attr = value."""
        if not isinstance(value, str):
            raise TypeError(f"{self.attr_name} must be a string")
        if len(value) < self.min_length:
            raise ValueError(f"Too short (min {self.min_length})")
        if self.max_length and len(value) > self.max_length:
            raise ValueError(f"Too long (max {self.max_length})")
        setattr(obj, self.attr_name, value)

class UserProfile:
    username = Validator(min_length=3, max_length=50)
    bio = Validator(max_length=500)

    def __init__(self, username, bio=""):
        self.username = username   # triggers Validator.__set__
        self.bio = bio

user = UserProfile("anurag")
user.username        # "anurag"  — triggers Validator.__get__
user.username = "x"  # ValueError: Too short (min 3)
user.username = 42   # TypeError: must be a string
```

Django's `CharField`, `IntegerField`, `BooleanField` etc. are all descriptors. When you access `article.title`, Django's field descriptor intercepts it and returns the value from the internal `__dict__`. When you set `article.title = "new title"`, the descriptor validates and stores it. This is the actual mechanism behind `models.CharField(max_length=200)`.

---

## 9. Dataclasses — When You Just Need a Data Container

Python 3.7+ introduced `@dataclass` — for when you want a class that mainly holds data, without writing all the boilerplate.

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Point:
    x: float
    y: float

    def distance_from_origin(self):
        return (self.x ** 2 + self.y ** 2) ** 0.5

# @dataclass automatically generates __init__, __repr__, __eq__:
p1 = Point(3.0, 4.0)
p2 = Point(3.0, 4.0)
p1 == p2            # True — __eq__ generated
print(p1)           # Point(x=3.0, y=4.0) — __repr__ generated
p1.distance_from_origin()  # 5.0

@dataclass
class Config:
    debug: bool = False
    database_url: str = "sqlite:///db.sqlite3"
    allowed_hosts: List[str] = field(default_factory=list)  # mutable default must use field()
    max_connections: int = 10

config = Config(debug=True)
config.allowed_hosts.append("localhost")
```

**Use dataclasses for:** Configuration objects, data transfer objects, simple value objects.

**Use regular classes for:** Anything with complex behavior, anything that needs inheritance hierarchies, Django models.

---

## ⚠️ Common Mistakes

**Mistake 1: Not calling `super().__init__()` in child class**

```python
class Parent:
    def __init__(self):
        self.x = 10

class Child(Parent):
    def __init__(self):
        self.y = 20
        # forgot super().__init__()

c = Child()
c.y   # 20
c.x   # AttributeError — Parent.__init__ never ran
```

**Mistake 2: Breaking the cooperative inheritance chain**

```python
class Mixin:
    def save(self, *args, **kwargs):
        print("Mixin save")
        # forgot to call super().save()

class Model:
    def save(self, *args, **kwargs):
        print("Model save — hits the database")

class MyModel(Mixin, Model):
    pass

obj = MyModel()
obj.save()
# "Mixin save"
# "Model save — hits the database" — NEVER CALLED
# Data never saved to the database!
```

**Mistake 3: Relying on MRO without checking it**

```python
# When something doesn't work as expected, check the MRO:
print(MyView.__mro__)

# Or use:
import inspect
inspect.getmro(MyView)
```

**Mistake 4: Shadowing a parent method without knowing it**

```python
class User(models.Model):
    name = models.CharField(max_length=100)

    def save(self):   # WRONG — missing *args, **kwargs
        self.name = self.name.strip()
        super().save()   # WRONG — not passing args/kwargs through

    def save(self, *args, **kwargs):   # CORRECT
        self.name = self.name.strip()
        super().save(*args, **kwargs)   # CORRECT — pass everything through
```

---

## 🧪 Exercises

**Exercise 1:** Build a class hierarchy for a library system:
- `LibraryItem` base class with `title`, `item_id`, `is_checked_out`, `checkout()`, `return_item()` methods, and `__str__`
- `Book(LibraryItem)` adds `author` and `isbn`
- `DVD(LibraryItem)` adds `director` and `runtime_minutes`
- `Magazine(LibraryItem)` adds `issue_number` and `publisher`
- Each subclass should properly call `super().__init__()` and add its own `__repr__`

**Exercise 2:** Build a `LoggingMixin` and `CachingMixin`. Each has a `get_data()` method. LoggingMixin logs every call. CachingMixin caches results. Write a `DataService` class that uses both mixins and returns data from a slow source. Verify via MRO output that logging happens before caching.

**Exercise 3:** Write an abstract base class `Serializer` with abstract methods `serialize(data)` and `deserialize(raw)`. Implement `JSONSerializer` and `CSVSerializer` as concrete subclasses. Add a `save_to_file(data, filepath)` concrete method on the base class that uses `serialize()`.

**Exercise 4:** Examine this MRO puzzle — predict the output before running:
```python
class A:
    def process(self):
        print("A")
        super().process()

class B(A):
    def process(self):
        print("B")
        super().process()

class C(A):
    def process(self):
        print("C")
        super().process()

class D(B, C):
    def process(self):
        print("D")
        super().process()

# What does D().process() print? In what order?
# What is D.__mro__?
# Why does A call super().process() and not crash?
```

---

## ✅ Before Moving On

- [ ] You can explain what MRO is and why it matters
- [ ] You always call `super().__init__()` in child `__init__`
- [ ] You always pass `*args, **kwargs` through `super()` calls
- [ ] You understand why every mixin method must call `super()`
- [ ] You can use `@abstractmethod` to enforce contracts on subclasses
- [ ] You understand what a descriptor is (even if you wouldn't write one from scratch yet)
- [ ] You've done all four exercises — especially Exercise 4

---

## 🔗 What This Has to Do With Django

You now have the complete mental model for understanding Django's class-based views. When you see:

```python
class ArticleListView(LoginRequiredMixin, ListView):
    model = Article
    template_name = "articles/list.html"

    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)
```

You understand:
- `ArticleListView` inherits from `LoginRequiredMixin` and `ListView`
- The MRO determines which `dispatch()` runs first (LoginRequiredMixin)
- `super().get_queryset()` calls `ListView`'s version, then you add your filter
- If you forget `super()`, you lose Django's default queryset handling

Django's models, forms, views, admin, middleware, signals — all of it is inheritance and mixins built on exactly what you've learned here.

**Next:** [06 — Modules and Packages: The Architecture of a Python Project](./06_modules_and_packages.md)
