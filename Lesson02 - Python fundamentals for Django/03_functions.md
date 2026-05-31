# 03 — Functions: The Building Blocks of Every Django View

> **A function is not just "reusable code."** A well-written function is a contract: it says what it takes, what it does, and what it returns. Django itself is an enormous collection of functions (and methods) that follow this contract. Learning to write functions well is learning to think like a framework designer.

---

## 1. Anatomy of a Function — Every Part Matters

```python
def greet(name: str, greeting: str = "Hello") -> str:
    """
    Return a personalized greeting string.

    Args:
        name: The person's name.
        greeting: The greeting word (default: "Hello").

    Returns:
        A formatted greeting string.
    """
    return f"{greeting}, {name}!"
```

Let's unpack every piece:

- `def` — keyword to define a function
- `greet` — the name (snake_case, always)
- `(name: str, greeting: str = "Hello")` — parameters with type hints and a default value
- `-> str` — return type hint
- `"""..."""` — docstring (not a comment — it's accessible at runtime via `greet.__doc__`)
- `return` — exits the function and sends a value back

**Functions without a `return` statement return `None` implicitly:**

```python
def say_hello(name):
    print(f"Hello, {name}")   # prints but doesn't return

result = say_hello("Anurag")
print(result)   # None
```

This trips people up constantly. **Printing is not returning.**

---

## 2. Parameters vs Arguments

These words are often used interchangeably, but they mean different things:

- **Parameter:** the variable name in the function definition (`name`, `greeting`)
- **Argument:** the actual value passed when calling the function (`"Anurag"`, `"Hey"`)

```python
def add(x, y):   # x and y are PARAMETERS
    return x + y

add(3, 5)         # 3 and 5 are ARGUMENTS
```

### 2.1 Positional Arguments

Arguments passed in the order defined:

```python
def create_user(username, email, age):
    return {"username": username, "email": email, "age": age}

create_user("anurag", "anurag@example.com", 28)
# Order matters — this maps to: username="anurag", email="anurag@example.com", age=28
```

### 2.2 Keyword Arguments

Arguments passed by name — order doesn't matter:

```python
create_user(age=28, username="anurag", email="anurag@example.com")
# Same result, different order — fine because we're using names
```

### 2.3 Default Values

```python
def paginate(queryset, page=1, per_page=20):
    start = (page - 1) * per_page
    end = start + per_page
    return queryset[start:end]

paginate(users)               # page=1, per_page=20 (defaults)
paginate(users, page=2)       # page=2, per_page=20
paginate(users, 3, 10)        # page=3, per_page=10
```

**Critical rule:** Parameters with defaults must come AFTER parameters without defaults:

```python
def bad(x=1, y):    # SyntaxError
def good(x, y=1):   # Fine
```

**The mutable default trap (again — it's that important):**

```python
# This is a bug you will write someday. Here's the warning:
def add_tag(post, tag, tags=[]):
    tags.append(tag)
    post["tags"] = tags
    return post

post1 = add_tag({}, "python")
post2 = add_tag({}, "django")

print(post1)  # {"tags": ["python", "django"]}  ← bug! post1 shouldn't have "django"
print(post2)  # {"tags": ["python", "django"]}

# The list [] is created ONCE when the function is defined, not on each call.
# Fix:
def add_tag(post, tag, tags=None):
    if tags is None:
        tags = []
    tags.append(tag)
    post["tags"] = tags
    return post
```

---

## 3. `*args` and `**kwargs` — The Power Tools

These aren't magic. They're just a way to accept a variable number of arguments.

### 3.1 `*args` — Variable Positional Arguments

```python
def add(*numbers):
    # 'numbers' inside the function is a TUPLE
    return sum(numbers)

add(1, 2)          # 3
add(1, 2, 3, 4, 5) # 15
add()              # 0  — empty tuple

# What's actually happening:
def debug_args(*args):
    print(type(args))  # <class 'tuple'>
    print(args)

debug_args(1, "hello", True)
# <class 'tuple'>
# (1, 'hello', True)
```

**The `*` is the operator, not the name.** You could write `*nums` and it works the same way. Convention is `*args`.

### 3.2 `**kwargs` — Variable Keyword Arguments

```python
def create_model_instance(**fields):
    # 'fields' inside the function is a DICT
    return fields

create_model_instance(name="Anurag", age=28, city="Mumbai")
# {"name": "Anurag", "age": 28, "city": "Mumbai"}
```

**Django uses `**kwargs` constantly:**

```python
# This is essentially what Django's Model.__init__ does under the hood:
user = User(username="anurag", email="anurag@example.com", is_staff=False)

# And when you do:
User.objects.create(**user_data)
# Django receives the dict as **kwargs and maps each key to a model field
```

### 3.3 Combining All Parameter Types

The order must be: positional → `*args` → keyword-only → `**kwargs`

```python
def full_example(pos1, pos2, *args, keyword_only, keyword_with_default=True, **kwargs):
    print(f"pos1: {pos1}")
    print(f"pos2: {pos2}")
    print(f"args: {args}")
    print(f"keyword_only: {keyword_only}")
    print(f"keyword_with_default: {keyword_with_default}")
    print(f"kwargs: {kwargs}")

full_example(1, 2, 3, 4, 5, keyword_only="must provide this", extra="value")
# pos1: 1
# pos2: 2
# args: (3, 4, 5)
# keyword_only: "must provide this"
# keyword_with_default: True
# kwargs: {"extra": "value"}
```

**Keyword-only parameters** (after `*args`) can only be passed by name:

```python
def send_email(to, subject, *, cc=None, bcc=None):
    # cc and bcc are keyword-only — they come after *
    pass

send_email("user@example.com", "Hello")       # fine, cc and bcc are optional
send_email("user@example.com", "Hello", cc="other@example.com")  # fine
send_email("user@example.com", "Hello", "other@example.com")     # TypeError! cc is keyword-only
```

### 3.4 Unpacking Arguments

The `*` and `**` operators also work when **calling** functions:

```python
def add(x, y, z):
    return x + y + z

numbers = [1, 2, 3]
add(*numbers)       # same as add(1, 2, 3)

config = {"x": 1, "y": 2, "z": 3}
add(**config)       # same as add(x=1, y=2, z=3)

# Combining
add(*[1, 2], z=3)   # same as add(1, 2, z=3)
```

**This pattern is everywhere in Django:**

```python
# Merging dicts (Python 3.5+)
defaults = {"color": "blue", "size": "M"}
overrides = {"color": "red"}
final = {**defaults, **overrides}   # {"color": "red", "size": "M"}

# Django URL patterns
from django.urls import path
urlpatterns = [
    path("users/<int:pk>/", views.UserDetailView.as_view(), name="user-detail"),
]

# Django ORM
filters = {"is_active": True, "age__gte": 18}
users = User.objects.filter(**filters)  # unpacks dict as keyword args to filter()
```

---

## 4. Return Values — Functions Can Return Anything

```python
# Return a single value
def square(n):
    return n ** 2

# Return multiple values (actually returns a tuple)
def min_max(numbers):
    return min(numbers), max(numbers)

low, high = min_max([3, 1, 4, 1, 5, 9])
print(low, high)   # 1 9

# Return different types based on condition (avoid this — inconsistent APIs are bad)
def bad_function(x):
    if x > 0:
        return x        # int
    return "negative"   # str — don't do this

# Return None explicitly for clarity
def find_user(user_id):
    for user in users:
        if user["id"] == user_id:
            return user
    return None   # explicit — reader knows function can return None
```

### 4.1 Early Return Pattern

The most important pattern for clean Django views:

```python
# Bad — deeply nested
def create_post(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            if request.user.is_authenticated:
                post = form.save(commit=False)
                post.author = request.user
                post.save()
                return redirect("post-list")

# Good — early returns eliminate nesting
def create_post(request):
    if request.method != "POST":
        return render(request, "create_post.html", {"form": PostForm()})

    form = PostForm(request.POST)
    if not form.is_valid():
        return render(request, "create_post.html", {"form": form})

    if not request.user.is_authenticated:
        return redirect("login")

    post = form.save(commit=False)
    post.author = request.user
    post.save()
    return redirect("post-list")
```

---

## 5. Lambda Functions — Small, Anonymous, One-liners

Lambdas are functions without a name. Use them only for simple, short operations.

```python
# Syntax: lambda parameters: expression
square = lambda x: x ** 2
add = lambda x, y: x + y

# These are equivalent to:
def square(x):
    return x ** 2

# Lambdas are useful as arguments to other functions:
users = [{"name": "Charlie", "age": 30}, {"name": "Alice", "age": 25}]
sorted_users = sorted(users, key=lambda u: u["age"])

# Django ORM sorting:
from django.db.models import F
users = User.objects.order_by("age")  # Django way
# But in Python-land:
users_list = sorted(user_list, key=lambda u: u.age)
```

**When NOT to use lambda:**

```python
# Bad — hard to read, no docstring, can't be tested by name
process = lambda data: [item.strip().lower() for item in data if item.strip()]

# Good — explicit function with a name
def clean_items(data):
    """Remove whitespace and lowercase non-empty items."""
    return [item.strip().lower() for item in data if item.strip()]
```

Rule of thumb: **if a lambda doesn't fit on one readable line, write a proper function.**

---

## 6. Scope and Closures

### 6.1 Revisiting Scope

```python
count = 0   # global scope

def increment():
    count += 1   # UnboundLocalError!

# Python sees the assignment 'count =' and decides count is a local variable
# But then tries to read count before assigning — local doesn't exist yet

# Fix with global keyword (use sparingly):
def increment():
    global count
    count += 1

# Better — return the new value instead of modifying global state:
def increment(count):
    return count + 1

count = increment(count)
```

### 6.2 Closures — Functions That Remember

A closure is a function that "remembers" variables from the enclosing scope even after that scope is gone.

```python
def make_multiplier(n):
    def multiplier(x):
        return x * n   # 'n' is captured from the enclosing scope
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)

double(5)   # 10  — remembers n=2
triple(5)   # 15  — remembers n=3
```

**This is exactly how Django decorators work under the hood:**

```python
# login_required is essentially a closure factory:
def my_login_required(view_func):
    def wrapper(request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect("login")
        return view_func(request, *args, **kwargs)   # view_func captured from outer scope
    return wrapper

@my_login_required
def dashboard(request):
    return render(request, "dashboard.html")
```

---

## 7. Decorators — Functions That Wrap Functions

Decorators are one of the most important Python concepts for Django. They look magical, but they're just functions that take a function and return a modified function.

```python
# A decorator is syntactic sugar for:
dashboard = login_required(dashboard)

# @login_required
# def dashboard(request): ...
# is EXACTLY equivalent to:
# def dashboard(request): ...
# dashboard = login_required(dashboard)
```

### 7.1 Writing Your Own Decorator

```python
import functools
import time

def timer(func):
    @functools.wraps(func)   # preserves the original function's name and docstring
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)   # call the original function
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "done"

slow_function()
# slow_function took 1.0012 seconds
```

### 7.2 Decorators with Arguments

```python
def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    print(f"Hello, {name}!")

greet("Anurag")
# Hello, Anurag!
# Hello, Anurag!
# Hello, Anurag!
```

**Django decorators you'll use constantly:**

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.views.decorators.http import require_POST, require_GET
from django.views.decorators.cache import cache_page

@login_required(login_url="/accounts/login/")
@permission_required("blog.can_publish")
@require_POST
def publish_post(request, post_id):
    ...

@cache_page(60 * 15)   # cache for 15 minutes
def homepage(request):
    ...
```

Decorators stack — they're applied bottom-up. `require_POST` wraps `publish_post` first, then `permission_required` wraps that, then `login_required` wraps the whole thing.

### 7.3 `functools.wraps` — Always Use It

Without `@functools.wraps`, your decorator breaks introspection:

```python
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def my_function():
    """This is my function."""
    pass

my_function.__name__   # "wrapper"  ← wrong!
my_function.__doc__    # None       ← lost the docstring!

# With @functools.wraps:
def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def my_function():
    """This is my function."""
    pass

my_function.__name__   # "my_function"  ← correct
my_function.__doc__    # "This is my function."  ← preserved
```

Django's test runner and debugging tools use `__name__` and `__doc__`. Always use `@functools.wraps`.

---

## 8. Type Hints in Functions — Writing Self-Documenting Code

```python
from typing import Optional, Union, List, Dict, Tuple

def get_user(user_id: int) -> Optional[dict]:
    """Return user dict or None if not found."""
    ...

def process_items(items: List[str], limit: int = 100) -> List[str]:
    return items[:limit]

def merge_configs(base: Dict[str, str], override: Dict[str, str]) -> Dict[str, str]:
    return {**base, **override}

# Python 3.10+ allows using | instead of Union
def find(id: int | str) -> dict | None:
    ...

# Python 3.9+ allows using built-in types directly (no need to import from typing)
def process(items: list[str]) -> dict[str, int]:
    ...
```

Type hints are not enforced at runtime (unless you use tools like `mypy` or `pydantic`). But they:
- Make your code self-documenting
- Enable IDE autocomplete
- Allow static analysis tools to catch bugs before runtime
- Are used by Django REST Framework to serialize/deserialize data

---

## ⚠️ Common Mistakes

**Mistake 1: Not using `functools.wraps` in decorators** (covered above)

**Mistake 2: Returning `None` from functions that should raise exceptions**
```python
# Bad — caller has to check None everywhere
def get_user_or_none(user_id):
    try:
        return User.objects.get(id=user_id)
    except User.DoesNotExist:
        return None

# Better — let exceptions propagate naturally, or raise a domain-specific error
def get_user(user_id):
    return User.objects.get(id=user_id)   # raises DoesNotExist if not found
```

**Mistake 3: Too many parameters — function is doing too much**
```python
# If you have more than 4-5 params, the function probably does too much
def create_order(user_id, product_id, quantity, address, payment_method, coupon_code, is_gift, gift_message):
    ...

# Better — group related data
def create_order(user_id, product_id, quantity, shipping_info, payment_info):
    ...
# Or use a dataclass/dict for complex parameter groups
```

**Mistake 4: Modifying arguments passed as mutable objects**
```python
def normalize_tags(tags):
    tags.append("default")   # modifies the CALLER'S list!
    return tags

user_tags = ["python", "web"]
normalized = normalize_tags(user_tags)
print(user_tags)   # ["python", "web", "default"]  ← side effect!

# Fix — don't modify; return a new object
def normalize_tags(tags):
    return tags + ["default"]   # creates new list
```

---

## 🧪 Exercises

**Exercise 1:** Write a function `make_validator(min_length, max_length, allowed_chars)` that returns a **validator function**. The returned function should take a string and return `(True, "")` if valid, or `(False, "error message")` if invalid. Use closures.

**Exercise 2:** Write a decorator `@log_calls` that prints the function name, arguments, and return value every time a decorated function is called. Make sure it preserves the function's name and docstring.

**Exercise 3:** Write a function `batch_process(items, processor, batch_size=10)` where `processor` is a function that takes a list and returns a list. Your function should process items in batches and return all results combined.

**Exercise 4:** Rewrite this function to use early returns and eliminate nesting:
```python
def process_payment(user, amount, method):
    result = {"success": False, "message": ""}
    if user:
        if user.get("is_active"):
            if amount > 0:
                if method in ("card", "bank", "wallet"):
                    result["success"] = True
                    result["message"] = "Payment processed"
                else:
                    result["message"] = "Invalid payment method"
            else:
                result["message"] = "Amount must be positive"
        else:
            result["message"] = "User is not active"
    else:
        result["message"] = "No user provided"
    return result
```

---

## ✅ Before Moving On

- [ ] You can explain the difference between parameters and arguments
- [ ] You know exactly why mutable default arguments are dangerous
- [ ] You can write functions using `*args` and `**kwargs`
- [ ] You understand closures — what gets captured and when
- [ ] You can write a decorator with and without arguments
- [ ] You always use `@functools.wraps` in decorators
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

Every Django view is a function (or a method on a class). Every Django decorator (`@login_required`, `@permission_required`, `@cache_page`) is built exactly like what you wrote in this lesson. Every form field's `validate()` method, every model's `save()`, every middleware — all functions following the patterns here.

When you later look at a Django view and wonder how `@login_required` works, you'll already know.

**Next:** [04 — OOP Basics: Classes, Objects, and Why Django Speaks Object](./04_oop_basics.md)
