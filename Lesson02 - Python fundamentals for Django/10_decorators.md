# 10 — Decorators: The Python Feature Django Can't Live Without

> **Decorators are everywhere in Django.** `@login_required`, `@permission_required`, `@cache_page`, `@require_POST`, `@staticmethod`, `@classmethod`, `@property`, `@receiver`, `@admin.register` — every one of these is a decorator. You've been using them. This lesson is about understanding them so completely that you could write every one of them yourself. Because in real Django work, you will write your own.

---

## 1. What a Decorator Actually Is — No Mystery

A decorator is a **callable that takes a callable and returns a callable**. That's the complete definition. No magic. No special syntax.

```python
def my_decorator(func):        # takes a function
    def wrapper(*args, **kwargs):
        print("Before")
        result = func(*args, **kwargs)   # calls the original
        print("After")
        return result
    return wrapper             # returns a new function

def greet(name):
    print(f"Hello, {name}!")

# Applying the decorator manually:
greet = my_decorator(greet)

greet("Anurag")
# Before
# Hello, Anurag!
# After
```

The `@` syntax is purely shorthand for the reassignment above:

```python
@my_decorator
def greet(name):
    print(f"Hello, {name}!")

# Is EXACTLY equivalent to:
def greet(name):
    print(f"Hello, {name}!")
greet = my_decorator(greet)
```

That's it. `@decorator` just means "pass the function below into `decorator()` and replace it with what comes back."

---

## 2. The Wrapper Pattern — Done Correctly

The minimal correct decorator template. Memorize this structure.

```python
import functools

def my_decorator(func):
    @functools.wraps(func)      # preserves __name__, __doc__, __module__, __qualname__
    def wrapper(*args, **kwargs):
        # --- before the function runs ---
        result = func(*args, **kwargs)
        # --- after the function runs ---
        return result
    return wrapper
```

**Why `*args, **kwargs` in the wrapper?**

Because you don't know what arguments the decorated function takes. The wrapper must be able to accept and pass through anything:

```python
def bad_decorator(func):
    def wrapper():             # only works for zero-argument functions!
        return func()
    return wrapper

@bad_decorator
def greet(name):               # TypeError when called — wrapper takes 0 args
    print(f"Hello, {name}!")

# Fix: always use *args, **kwargs
def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

**Why `@functools.wraps(func)` is non-negotiable:**

```python
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def calculate_tax(amount, rate=0.18):
    """Calculate tax on an amount at a given rate."""
    return amount * rate

# Without @functools.wraps:
calculate_tax.__name__   # "wrapper"  — WRONG
calculate_tax.__doc__    # None       — lost the docstring
help(calculate_tax)      # shows wrapper's signature, not calculate_tax's

# Django's URL router, admin, test runner, and debugger all use __name__
# Breaking it creates subtle, hard-to-diagnose bugs
```

```python
import functools

def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def calculate_tax(amount, rate=0.18):
    """Calculate tax on an amount at a given rate."""
    return amount * rate

calculate_tax.__name__   # "calculate_tax"  — correct
calculate_tax.__doc__    # "Calculate tax on an amount at a given rate."  — preserved
```

---

## 3. Decorators with Arguments — The Factory Pattern

When a decorator needs configuration, you add another layer: a function that takes arguments and **returns** a decorator.

```python
# Without arguments — two layers:
@my_decorator
def func(): ...
# my_decorator(func)

# With arguments — three layers:
@my_decorator(arg1, arg2)
def func(): ...
# my_decorator(arg1, arg2)(func)
```

```python
import functools

def repeat(n):
    """Decorator factory — takes n, returns a decorator."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            result = None
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator   # decorator factory returns the decorator

@repeat(3)
def say_hello(name):
    print(f"Hello, {name}!")

say_hello("Anurag")
# Hello, Anurag!
# Hello, Anurag!
# Hello, Anurag!
```

**Django's `@cache_page` is exactly this pattern:**

```python
@cache_page(60 * 15)   # cache_page(900) returns a decorator
def my_view(request):
    ...
# Equivalent to: my_view = cache_page(900)(my_view)
```

### 3.1 Optional Arguments — Writing Decorators That Work Both Ways

Sometimes you want a decorator that can be used with or without arguments:

```python
import functools

def log(func=None, *, level="INFO", prefix=""):
    """
    Works as:
        @log
        @log()
        @log(level="DEBUG", prefix="[API]")
    """
    def decorator(f):
        @functools.wraps(f)
        def wrapper(*args, **kwargs):
            print(f"[{level}] {prefix}{f.__name__} called")
            result = f(*args, **kwargs)
            print(f"[{level}] {prefix}{f.__name__} returned {result!r}")
            return result
        return wrapper

    if func is not None:
        # Called as @log without parentheses — func is the decorated function
        return decorator(func)
    # Called as @log() or @log(level="DEBUG") — return the decorator
    return decorator

@log
def add(a, b): return a + b

@log()
def subtract(a, b): return a - b

@log(level="DEBUG", prefix="[MATH] ")
def multiply(a, b): return a * b
```

---

## 4. Class-Based Decorators

Decorators don't have to be functions. Any callable works — including class instances with `__call__`.

```python
import functools

class retry:
    """Retry a function up to max_attempts times on failure."""

    def __init__(self, max_attempts=3, exceptions=(Exception,), delay=0):
        self.max_attempts = max_attempts
        self.exceptions = exceptions
        self.delay = delay

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            import time
            last_exception = None
            for attempt in range(1, self.max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except self.exceptions as e:
                    last_exception = e
                    print(f"Attempt {attempt}/{self.max_attempts} failed: {e}")
                    if attempt < self.max_attempts and self.delay:
                        time.sleep(self.delay)
            raise last_exception
        return wrapper

@retry(max_attempts=3, exceptions=(ConnectionError, TimeoutError), delay=1)
def fetch_data_from_api(url):
    import requests
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()
```

Class-based decorators are useful when the decorator needs to maintain state across calls or when the configuration is complex enough to benefit from a class structure.

---

## 5. Stacking Decorators — Order Matters

When you stack decorators, they're applied **bottom-up** (innermost first):

```python
@decorator_a
@decorator_b
@decorator_c
def my_func():
    pass

# Equivalent to:
my_func = decorator_a(decorator_b(decorator_c(my_func)))
```

```python
def bold(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return f"<b>{func(*args, **kwargs)}</b>"
    return wrapper

def italic(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return f"<i>{func(*args, **kwargs)}</i>"
    return wrapper

def underline(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return f"<u>{func(*args, **kwargs)}</u>"
    return wrapper

@bold
@italic
@underline
def greet(name):
    return f"Hello, {name}!"

greet("Anurag")
# <b><i><u>Hello, Anurag!</u></i></b>
# underline wraps first → italic wraps that → bold wraps that
```

**Django decorator stacking:**

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.views.decorators.http import require_POST
from django.views.decorators.cache import never_cache

@never_cache               # applied last (outermost)
@login_required            # applied third
@permission_required("blog.add_article")  # applied second
@require_POST              # applied first (innermost — wraps the view directly)
def create_article(request):
    ...

# Execution order when a request arrives:
# never_cache → login_required → permission_required → require_POST → create_article
# Each one can short-circuit (redirect/403/405) without calling the next
```

---

## 6. Decorators for Classes and Methods

### 6.1 Decorating an Entire Class

```python
def add_repr(cls):
    """Add a generic __repr__ to any class."""
    def __repr__(self):
        attrs = ", ".join(
            f"{k}={v!r}" for k, v in self.__dict__.items()
            if not k.startswith("_")
        )
        return f"{cls.__name__}({attrs})"
    cls.__repr__ = __repr__
    return cls

@add_repr
class Article:
    def __init__(self, title, author):
        self.title = title
        self.author = author

a = Article("Django Tips", "Anurag")
repr(a)   # "Article(title='Django Tips', author='Anurag')"
```

**Django's `@admin.register` is a class decorator:**

```python
from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "is_published"]
    list_filter = ["is_published", "created_at"]
    search_fields = ["title", "content"]

# Equivalent to:
# class ArticleAdmin(admin.ModelAdmin): ...
# admin.site.register(Article, ArticleAdmin)
```

### 6.2 `@staticmethod` and `@classmethod` Are Decorators

These built-in descriptors you learned in the OOP lesson are just decorators — they transform a function into a different kind of callable:

```python
class MyClass:
    @staticmethod
    def static_method():
        pass
    # equivalent to:
    # static_method = staticmethod(static_method)

    @classmethod
    def class_method(cls):
        pass
    # equivalent to:
    # class_method = classmethod(class_method)

    @property
    def my_property(self):
        return self._value
    # equivalent to:
    # my_property = property(my_property)
```

Everything you've been using is a decorator. Now you see the pattern behind all of it.

---

## 7. Real Django Decorators — Built From Scratch

Now let's build the ones you actually use in Django, from scratch. This cements understanding.

### 7.1 `@login_required` — From Scratch

```python
import functools
from django.shortcuts import redirect
from django.conf import settings

def login_required(view_func=None, login_url=None, redirect_field_name="next"):
    """
    Redirect unauthenticated users to the login page.
    Preserves the original URL as a 'next' parameter.
    """
    actual_login_url = login_url or settings.LOGIN_URL

    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            if request.user.is_authenticated:
                return func(request, *args, **kwargs)

            # Build the redirect URL with the 'next' parameter
            from django.utils.http import urlencode
            from django.urls import reverse

            next_url = request.get_full_path()
            login = f"{actual_login_url}?{urlencode({redirect_field_name: next_url})}"
            return redirect(login)
        return wrapper

    # Handle both @login_required and @login_required(login_url="/login/")
    if view_func is not None:
        return decorator(view_func)
    return decorator
```

### 7.2 `@require_http_methods` — From Scratch

```python
def require_http_methods(allowed_methods):
    """
    Only allow requests with the specified HTTP methods.
    Returns 405 Method Not Allowed otherwise.
    """
    allowed = [m.upper() for m in allowed_methods]

    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            if request.method not in allowed:
                from django.http import HttpResponseNotAllowed
                return HttpResponseNotAllowed(allowed)
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

require_GET = require_http_methods(["GET"])
require_POST = require_http_methods(["POST"])
require_safe = require_http_methods(["GET", "HEAD"])

@require_POST
def create_article(request):
    ...
```

### 7.3 `@cache_page` — Simplified From Scratch

```python
import functools
import hashlib
from django.core.cache import cache

def simple_cache(timeout=60):
    """
    Cache a view's response for `timeout` seconds.
    Cache key is based on the full request URL.
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            # Build cache key from URL
            cache_key = "view_cache:" + hashlib.md5(
                request.get_full_path().encode()
            ).hexdigest()

            cached_response = cache.get(cache_key)
            if cached_response is not None:
                return cached_response

            response = func(request, *args, **kwargs)

            # Only cache successful responses
            if hasattr(response, "status_code") and response.status_code == 200:
                cache.set(cache_key, response, timeout)

            return response
        return wrapper
    return decorator

@simple_cache(timeout=300)   # cache for 5 minutes
def homepage(request):
    ...
```

### 7.4 `@transaction.atomic` — As a Decorator

`transaction.atomic` is both a context manager AND a decorator — it implements both protocols:

```python
# As context manager:
with transaction.atomic():
    user.save()
    profile.save()

# As decorator:
@transaction.atomic
def create_user_with_profile(username, email):
    user = User.objects.create(username=username, email=email)
    profile = UserProfile.objects.create(user=user)
    return user

# If any exception escapes, BOTH creates are rolled back
```

---

## 8. Writing Your Own Django Decorators — Real Patterns

### 8.1 Rate Limiting Decorator

```python
import functools
from django.core.cache import cache
from django.http import HttpResponse

def rate_limit(max_calls, period=60, key_func=None):
    """
    Limit a view to max_calls requests per period (in seconds).
    key_func: callable(request) → str identifying the caller
              default: user ID for authenticated, IP for anonymous
    """
    def default_key(request):
        if request.user.is_authenticated:
            return f"user:{request.user.id}"
        return f"ip:{request.META.get('REMOTE_ADDR', 'unknown')}"

    get_key = key_func or default_key

    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            cache_key = f"ratelimit:{func.__name__}:{get_key(request)}"

            call_count = cache.get(cache_key, 0)

            if call_count >= max_calls:
                response = HttpResponse(
                    "Rate limit exceeded. Try again later.",
                    status=429
                )
                response["Retry-After"] = period
                return response

            # Increment count; set expiry only on first call
            if call_count == 0:
                cache.set(cache_key, 1, period)
            else:
                cache.incr(cache_key)

            return func(request, *args, **kwargs)
        return wrapper
    return decorator

@rate_limit(max_calls=5, period=60)
def api_submit_comment(request):
    ...

@rate_limit(max_calls=3, period=3600, key_func=lambda req: f"ip:{req.META.get('REMOTE_ADDR')}")
def password_reset_request(request):
    ...
```

### 8.2 Ownership Check Decorator

```python
def owned_by_user(model, pk_url_kwarg="pk", owner_field="author"):
    """
    Ensure the requesting user owns the object.
    Raises 404 if object doesn't exist, 403 if user doesn't own it.
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            from django.shortcuts import get_object_or_404
            from django.core.exceptions import PermissionDenied

            pk = kwargs.get(pk_url_kwarg)
            obj = get_object_or_404(model, pk=pk)

            owner = getattr(obj, owner_field, None)

            if owner != request.user and not request.user.is_staff:
                raise PermissionDenied

            # Attach obj to request to avoid a second DB query in the view
            request.owned_object = obj
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

@login_required
@owned_by_user(Article, pk_url_kwarg="article_id", owner_field="author")
def edit_article(request, article_id):
    article = request.owned_object   # already fetched by the decorator
    ...
```

### 8.3 JSON Response Decorator

```python
import json
import functools
from django.http import JsonResponse
from django.core.exceptions import ValidationError

def json_view(func):
    """
    Decorator for API views:
    - Parses request body as JSON and attaches as request.json_data
    - Returns 400 if JSON is invalid
    - Catches ValidationError and returns 400 with errors
    - Catches any other exception and returns 500
    """
    @functools.wraps(func)
    def wrapper(request, *args, **kwargs):
        # Parse JSON body
        if request.content_type == "application/json" and request.body:
            try:
                request.json_data = json.loads(request.body)
            except json.JSONDecodeError as e:
                return JsonResponse({"error": f"Invalid JSON: {e}"}, status=400)
        else:
            request.json_data = {}

        try:
            return func(request, *args, **kwargs)
        except ValidationError as e:
            return JsonResponse({"errors": e.message_dict}, status=400)
        except PermissionError:
            return JsonResponse({"error": "Permission denied"}, status=403)
        except Exception as e:
            import logging
            logging.getLogger(__name__).exception("Unhandled error in API view")
            return JsonResponse({"error": "Internal server error"}, status=500)

    return wrapper

@login_required
@json_view
@require_http_methods(["POST"])
def api_create_article(request):
    data = request.json_data
    article = Article.objects.create(
        title=data["title"],
        content=data["content"],
        author=request.user,
    )
    return JsonResponse({"id": article.id}, status=201)
```

---

## 9. `functools` — The Decorator Utility Belt

```python
import functools

# wraps — already covered, always use it
@functools.wraps(func)

# lru_cache — memoization (cache function results)
@functools.lru_cache(maxsize=128)
def expensive_calculation(n):
    """Result cached after first call with each unique argument."""
    return sum(range(n))

expensive_calculation(1000)   # computed
expensive_calculation(1000)   # returned from cache instantly
expensive_calculation.cache_info()   # CacheInfo(hits=1, misses=1, maxsize=128, currsize=1)
expensive_calculation.cache_clear()  # clear the cache

# cache — same as lru_cache(maxsize=None), Python 3.9+
@functools.cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# partial — fix some arguments of a function
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

square(5)   # 25
cube(4)     # 64

# Useful in Django for URL patterns with pre-filled arguments:
from django.views.generic import TemplateView
about_view = TemplateView.as_view(template_name="about.html")
# as_view() with pre-filled template_name is like a partial application

# reduce — fold a sequence to a single value
from functools import reduce
product = reduce(lambda acc, x: acc * x, [1, 2, 3, 4, 5])   # 120

# total_ordering — define __eq__ and one comparison, get all others
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, major, minor, patch):
        self.major, self.minor, self.patch = major, minor, patch

    def __eq__(self, other):
        return (self.major, self.minor, self.patch) == (other.major, other.minor, other.patch)

    def __lt__(self, other):
        return (self.major, self.minor, self.patch) < (other.major, other.minor, other.patch)

    # functools generates __le__, __gt__, __ge__ automatically from __eq__ and __lt__

v1 = Version(1, 2, 3)
v2 = Version(2, 0, 0)
v1 < v2    # True
v1 > v2    # False — generated by @total_ordering
v1 <= v2   # True  — generated
```

---

## 10. Decorators in Django's Class-Based Views

Function decorators don't work directly on class methods the way you might expect. Django provides `method_decorator` to bridge this:

```python
from django.utils.decorators import method_decorator
from django.views import View
from django.contrib.auth.decorators import login_required
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie

# Option 1: method_decorator on individual method
class ArticleDetailView(View):
    @method_decorator(login_required)
    def get(self, request, pk):
        ...

# Option 2: method_decorator on the class, targeting dispatch
@method_decorator(login_required, name="dispatch")
class ArticleCreateView(View):
    def get(self, request):
        ...
    def post(self, request):
        ...

# Option 3: stack multiple decorators on a CBV
@method_decorator([
    login_required,
    cache_page(60 * 5),
    vary_on_cookie,
], name="dispatch")
class ExpensiveView(View):
    ...

# With generic views:
from django.views.generic import ListView

@method_decorator(login_required, name="dispatch")
class ArticleListView(ListView):
    model = Article
    template_name = "blog/article_list.html"
```

---

## ⚠️ Common Mistakes

**Mistake 1: Forgetting `@functools.wraps`** — covered exhaustively above; never skip it.

**Mistake 2: Calling the decorator instead of passing it**
```python
@login_required()   # WRONG — login_required is not a factory in Django
def my_view(request): ...
# Fix:
@login_required     # correct — no parentheses
@login_required(login_url="/accounts/login/")  # correct — with config
```

**Mistake 3: Stacking order confusion**
```python
# This requires login THEN checks method:
@login_required
@require_POST
def my_view(request): ...
# GET request from unauthenticated user → redirected to login (login_required wins)

# This checks method THEN requires login:
@require_POST
@login_required
def my_view(request): ...
# GET request from unauthenticated user → 405 Method Not Allowed (require_POST wins)
```

Order matters. Think about which check should happen first.

**Mistake 4: Using `@functools.lru_cache` on methods without caution**
```python
class MyClass:
    @functools.lru_cache(maxsize=128)
    def compute(self, n):
        return n ** 2

# self is an argument to the cache — different instances create different cache entries
# WORSE: self is held in the cache, preventing garbage collection of instances!

# Fix: use @functools.cache on module-level functions, or use methodtools.lru_cache
# Or just don't lru_cache instance methods
```

**Mistake 5: Applying function decorators directly to CBV methods**
```python
class MyView(View):
    @login_required    # WRONG — doesn't work on class methods
    def get(self, request): ...

# Fix: use method_decorator
    @method_decorator(login_required)
    def get(self, request): ...
```

---

## 🧪 Exercises

**Exercise 1:** Write a `@timer` decorator that:
- Records start time before calling the function
- Records end time after
- Prints `"{func_name} completed in {elapsed:.4f}s"`
- Also stores the last execution time as `func.last_execution_time`
- Works correctly with `@functools.wraps`

**Exercise 2:** Write a `@validate_json_schema(schema)` decorator that:
- Takes a `schema` dict like `{"title": str, "age": int, "email": str}`
- Parses the request body as JSON
- Validates that all required keys exist and have the correct types
- Returns 400 with field-specific errors if validation fails
- Attaches validated data as `request.validated_data` for the view

**Exercise 3:** Write a `@audit_log` decorator that:
- Logs every call to the decorated Django view with: timestamp, user, view name, method, path, response status
- Saves this to a database model `AuditLog(user, view_name, method, path, status_code, timestamp)`
- Handles anonymous users (user is `None`)
- Doesn't break if the log save fails (catch and log to Python logger instead)

**Exercise 4:** Build a decorator system for a simple permission framework:
- `@requires_permission("blog.publish")` — checks `request.user.has_perm("blog.publish")`
- `@requires_role("editor", "admin")` — checks if user is in any of those groups
- `@requires_own_object(Article)` — checks if the article's author is the requesting user
- All three should raise `PermissionDenied` on failure
- All three should work together when stacked

---

## ✅ Before Moving On

- [ ] You can write a decorator from scratch without looking anything up
- [ ] You always use `@functools.wraps` — no exceptions
- [ ] You can write decorator factories (decorators that take arguments)
- [ ] You understand stacking order and can predict execution order
- [ ] You can write class-based decorators with `__call__`
- [ ] You know how to apply function decorators to CBVs using `method_decorator`
- [ ] You've built at least `@rate_limit` or `@owned_by_user` yourself
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

Every time you write `@login_required`, `@permission_required`, `@cache_page`, `@require_POST`, or `@admin.register` — you're using the exact pattern from this lesson. When you write a class-based view and use `method_decorator`, you're using the factory pattern. When you build an API and want consistent JSON error handling, you write `@json_view`. When you need rate limiting, you write `@rate_limit`.

Decorators are not a feature you learn once and move on from. They're the primary mechanism for adding cross-cutting behavior — authentication, caching, logging, validation, rate limiting — to Django views without cluttering the view's own logic. You'll write your own decorators in every serious Django project.

**Next:** [11 — Type Hints: Writing Python That Documents Itself](./11_type_hints.md)
