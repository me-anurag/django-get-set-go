# 12 — Pre-Django Checklist: Are You Actually Ready?

> **This is not a feel-good exercise.** This checklist exists because most people who struggle with Django don't struggle with Django — they struggle with Python, and Django just exposes it. Every item on this list maps directly to something Django does. If you're honest with yourself and find gaps, go back and fill them. A week of Python fundamentals now saves months of confused Django debugging later.

---

## How to Use This Checklist

For every item:

- ✅ **Can do it from memory** — no looking up, no guessing
- 🔶 **Familiar but need to look it up** — go review, then come back
- ❌ **Not confident** — go back to that lesson, do the exercises, then return

**Target: every item is ✅ before you start Lesson03.**

You are not being graded. No one is watching. Be ruthlessly honest.

---

## Section 1 — Python's Object Model

These are things Django's ORM and template engine depend on directly.

```python
# 1.1 — Given this code, what does it print and why?
a = {"tags": ["python", "django"]}
b = a.copy()
b["tags"].append("web")
print(a["tags"])
```
> You must answer without running it. If you're not certain, re-read lesson 01 (shallow copy vs deep copy).

```python
# 1.2 — What's wrong here? Fix it.
class Post:
    views = []   # all posts share this list

    def add_view(self, viewer_id):
        self.views.append(viewer_id)

p1 = Post()
p2 = Post()
p1.add_view(1)
print(p2.views)   # what does this print?
```

```python
# 1.3 — Predict the output:
x = None
y = ""
z = []

print(bool(x), bool(y), bool(z))
print(x is None, y is None, z is None)
print(x == False, y == False, z == False)
```

```python
# 1.4 — Which of these are falsy in Python?
# 0, 0.0, "", [], {}, (), set(), False, None, "0", [0], {"": ""}
```

**If you got 1.1-1.4 completely right:** ✅
**If you had to think hard or guess:** 🔶 — re-read lesson 01, section 2 and 3

---

## Section 2 — Control Flow

```python
# 2.1 — What does this print?
for i in range(5):
    if i == 3:
        break
    print(i)
else:
    print("no break")
```

```python
# 2.2 — Rewrite this without nested ifs (use early returns):
def process(user, amount):
    result = None
    if user:
        if user.get("is_active"):
            if amount > 0:
                result = amount * 1.18
    return result
```

```python
# 2.3 — Write this without a loop using any() or all():
# "Return True if at least one user in the list is both active and verified"
users = [
    {"is_active": True, "is_verified": False},
    {"is_active": True, "is_verified": True},
    {"is_active": False, "is_verified": True},
]
```

```python
# 2.4 — What's the difference between:
users = User.objects.filter(is_active=True)
if users:        # (a)
    pass

if len(users):   # (b)
    pass

if users.exists():  # (c)
    pass
```
> (b) and (c) both "work" but one is significantly more efficient. Which one and why?

**If you got 2.1-2.4 right:** ✅
**If not:** 🔶 — re-read lesson 02

---

## Section 3 — Functions

```python
# 3.1 — What does this print? Why is this a bug?
def add_permission(user, permissions=[]):
    permissions.append("read")
    user["permissions"] = permissions
    return user

u1 = add_permission({"name": "Alice"})
u2 = add_permission({"name": "Bob"})
print(u1["permissions"])
print(u2["permissions"])
```

```python
# 3.2 — Write this function signature correctly using *args and **kwargs:
# A function that accepts: one required string 'action',
# any number of positional items to apply the action to,
# optional keyword args 'dry_run' (bool, default False) and 'verbose' (bool, default False)
```

```python
# 3.3 — What does @functools.wraps do and when is it required?
# Write a decorator @timer that measures execution time.
# Show what breaks without @functools.wraps.
```

```python
# 3.4 — Rewrite this as a decorator with arguments:
# @validate(required=["title", "content"], max_length={"title": 200})
# It should validate a Django request.POST dict before the view runs.
```

**If you got 3.1-3.4:** ✅
**If not:** 🔶 — re-read lessons 03 and 10

---

## Section 4 — OOP Basics and Advanced

```python
# 4.1 — Without running it, what does this print?
class Animal:
    sound = "..."

    def speak(self):
        return f"I say {self.sound}"

class Dog(Animal):
    sound = "Woof"

class Cat(Animal):
    def speak(self):
        return f"I say Meow (not {self.sound})"

animals = [Dog(), Cat(), Animal()]
for animal in animals:
    print(animal.speak())
```

```python
# 4.2 — What's wrong? Fix it.
class TimestampMixin:
    def save(self, *args, **kwargs):
        from datetime import datetime
        self.updated_at = datetime.now()
        # missing something critical here

class Article(TimestampMixin, models.Model):
    title = models.CharField(max_length=200)
    updated_at = models.DateTimeField(auto_now=True)
```

```python
# 4.3 — What is the MRO for this class? What does D().hello() print?
class A:
    def hello(self): return "A"

class B(A):
    def hello(self): return "B"

class C(A):
    def hello(self): return "C"

class D(B, C):
    pass
```

```python
# 4.4 — Write a @property 'full_name' and a 'last_name' setter that:
# - Returns "FirstName LastName"
# - When setting last_name, validates it's at least 2 characters
# - Stores it as self._last_name
class Person:
    def __init__(self, first_name: str, last_name: str):
        self.first_name = first_name
        self.last_name = last_name  # should trigger the setter
```

```python
# 4.5 — Why does this Django pattern always use super()?
class ArticleCreateView(LoginRequiredMixin, CreateView):
    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)   # ← why is this line non-negotiable?
```

**If you got 4.1-4.5:** ✅
**If not:** 🔶 — re-read lessons 04 and 05

---

## Section 5 — Modules and Packages

```python
# 5.1 — What's the difference between these two imports? When would you use each?
import blog.models
from blog.models import Article
from blog import models
from . import models       # inside blog/views.py
from .models import Article  # inside blog/views.py
```

```python
# 5.2 — What does __init__.py do? What would you put in blog/__init__.py
# to allow this import to work:
from blog import Article   # instead of: from blog.models import Article
```

```python
# 5.3 — A new developer gets this error running the Django dev server:
# ModuleNotFoundError: No module named 'blog'
# List every possible cause and how to diagnose each one.
```

```python
# 5.4 — Why does this fail, and how do you fix it?
# models.py:
from blog.signals import on_article_saved

# signals.py:
from blog.models import Article

# Both files import each other. What happens? What are 3 different ways to fix it?
```

**If you got 5.1-5.4:** ✅
**If not:** 🔶 — re-read lesson 06

---

## Section 6 — File Handling

```python
# 6.1 — What's wrong with this view?
def export_csv(request):
    with open("users.csv", "w") as f:
        writer = csv.writer(f)
        for user in User.objects.all():
            writer.writerow([user.username, user.email])
    with open("users.csv", "r") as f:
        return HttpResponse(f.read(), content_type="text/csv")
```
> There are at least 4 things wrong. List them all.

```python
# 6.2 — Write the correct version of this:
# A Django view that exports all active users to a downloadable CSV
# without writing any file to disk
```

```python
# 6.3 — This view processes an uploaded file:
def process_upload(request):
    f = request.FILES["document"]
    content = f.read()
    size = len(f.read())           # bug is here
    words = content.decode().split()
    ...
# What is the bug? What is the fix?
```

```python
# 6.4 — What does this path compute and why is it the right way to do it?
BASE_DIR = Path(__file__).resolve().parent.parent
```

**If you got 6.1-6.4:** ✅
**If not:** 🔶 — re-read lesson 07

---

## Section 7 — Error Handling

```python
# 7.1 — Rank these from best to worst practice, and explain why:
# (a)
try:
    user = User.objects.get(id=1)
except:
    pass

# (b)
try:
    user = User.objects.get(id=1)
except Exception as e:
    print(e)

# (c)
try:
    user = User.objects.get(id=1)
except User.DoesNotExist:
    raise Http404("User not found")

# (d)
user = get_object_or_404(User, id=1)
```

```python
# 7.2 — What does transaction.atomic() do?
# What happens to the database if an exception is raised inside it?
# Write an example where NOT using it would leave the database inconsistent.
```

```python
# 7.3 — Design a custom exception hierarchy for a payment system.
# It must have: base exception, payment failure, insufficient funds,
# card declined, and expired card exceptions.
# Show how a view would catch each with appropriate HTTP status codes.
```

```python
# 7.4 — What's the difference between these and when would you use each?
logger.error("Something failed")
logger.error("Something failed: %s", error)
logger.exception("Something failed")
logger.exception("Something failed: %s", error)
```

**If you got 7.1-7.4:** ✅
**If not:** 🔶 — re-read lesson 08

---

## Section 8 — Iterators and Generators

```python
# 8.1 — What is the difference between these two? When does each hit the database?
# (a)
articles = Article.objects.filter(is_published=True)

# (b)
articles = list(Article.objects.filter(is_published=True))
```

```python
# 8.2 — This code causes a performance problem. Diagnose it and fix it.
articles = Article.objects.all()
for article in articles:
    print(f"{article.title} by {article.author.username}")
    # Django: N+1 queries! 1 for articles + 1 per article for author
```

```python
# 8.3 — When would you use .iterator() on a queryset?
# What does it disable, and why might that matter?
```

```python
# 8.4 — Write a generator function batch(iterable, size) that yields
# fixed-size chunks of any iterable. Then use it to bulk-email 100K users
# in batches of 500, processing with iterator() to avoid memory issues.
```

```python
# 8.5 — What does this do, and why is it more memory efficient than
# the list comprehension version?
total = sum(article.word_count for article in Article.objects.iterator())
```

**If you got 8.1-8.5:** ✅
**If not:** 🔶 — re-read lesson 09

---

## Section 9 — Decorators

```python
# 9.1 — Write this decorator from scratch, correctly:
# @retry(max_attempts=3, delay=1, exceptions=(ConnectionError, TimeoutError))
# It should retry the function up to max_attempts times,
# wait delay seconds between attempts,
# and only retry on the specified exception types.
# Include @functools.wraps.
```

```python
# 9.2 — What's the execution order when a request hits this view?
@never_cache
@login_required
@permission_required("blog.change_article")
@require_POST
def edit_article(request, pk):
    ...
```

```python
# 9.3 — Why doesn't this work, and what's the fix?
class ArticleDetailView(DetailView):
    @login_required          # this doesn't work on CBV methods
    def get(self, request, *args, **kwargs):
        ...
```

```python
# 9.4 — Write a decorator @json_api that:
# - Parses request.body as JSON
# - Returns 400 if JSON is malformed
# - Catches ValidationError and returns field errors as JSON
# - Logs unexpected exceptions and returns 500
# - Works correctly when stacked with @login_required and @require_POST
```

**If you got 9.1-9.4:** ✅
**If not:** 🔶 — re-read lesson 10

---

## Section 10 — Type Hints

```python
# 10.1 — Add type hints to every parameter, return type, and local variable:
def get_active_users_by_city(city, include_staff=False):
    users = User.objects.filter(city=city, is_active=True)
    if not include_staff:
        users = users.filter(is_staff=False)
    return list(users)
```

```python
# 10.2 — When do you use TypeVar? Write an example.
# Write a generic function first_or_default(items, default) that returns
# the first item of a list or a default value, where mypy knows the return
# type matches the list element type.
```

```python
# 10.3 — What's the difference between these? When would you use each?
from typing import Any, Union, Optional, Protocol, TypedDict
```

**If you got 10.1-10.3:** ✅
**If not:** 🔶 — re-read lesson 11

---

## The 10 Things Django Will Immediately Test You On

These are the top 10 concepts from Lesson02 that show up in your very first lines of real Django code. If any of these feels uncertain, stop and solidify it before Lesson03.

| # | Concept | Where it appears in Django |
|---|---------|---------------------------|
| 1 | Mutable default arguments | Model `__init__`, form defaults |
| 2 | `super().__init__(*args, **kwargs)` | Every class-based view, every model |
| 3 | MRO and cooperative inheritance | CBV mixins, LoginRequiredMixin stacking |
| 4 | `@functools.wraps` in decorators | `@login_required`, every custom decorator |
| 5 | When a queryset evaluates vs stays lazy | Every view that uses the ORM |
| 6 | N+1 queries and `select_related` | Every view that accesses related objects |
| 7 | `try/except Model.DoesNotExist` | Every `.get()` call without `get_object_or_404` |
| 8 | `transaction.atomic()` | Every multi-step write operation |
| 9 | `pathlib.Path` for `BASE_DIR` | `settings.py` from line 1 |
| 10 | `with open(..., encoding="utf-8")` | File exports, CSV generation, config loading |

---

## Final Self-Assessment

Answer each of these honestly. If you can't, go back.

### Conceptual Questions

1. Explain in plain English why `y = x` for a list doesn't create a copy, but `y = x + []` does.

2. What is the MRO, why does Python need it, and how does `super()` use it?

3. A Django view runs 50 database queries on a page that should need 3. Walk me through how you would diagnose and fix it.

4. A user uploads a 2GB CSV file. Your view reads it with `f.readlines()`. What happens? How do you fix it?

5. Explain why `@login_required` is a decorator and how it works internally. No hand-waving — walk through the function calls.

6. Your Django management command processes 2 million users. Walk through the memory-safe approach using iterators.

7. What is the difference between `Article.DoesNotExist` and `ObjectDoesNotExist`? When does catching one not catch the other?

8. Why does this line in `settings.py` work the way it does?
   ```python
   BASE_DIR = Path(__file__).resolve().parent.parent
   ```

9. Explain lazy evaluation in QuerySets. Write a code example that accidentally evaluates a queryset 3 times when it should only be evaluated once.

10. What happens to the database when an unhandled exception occurs inside `transaction.atomic()`?

### Code Challenges

**Challenge 1 — Write From Scratch:**

Build a complete `EventEmitter` class that:
- Maintains a registry of `event_name → list of handler functions`
- Has `on(event, handler)` — registers a handler
- Has `emit(event, *args, **kwargs)` — calls all handlers for that event
- Has `off(event, handler)` — removes a specific handler
- Has `once(event, handler)` — handler fires only on first emit, then auto-removes
- All methods should be chainable (return `self`)
- Use type hints throughout
- Write a decorator `@emits("user_created")` that calls `self.emit("user_created", result)` after the decorated method returns

**Challenge 2 — Debug This:**

```python
class ArticleService:
    _cache = {}   # shared across all instances

    def get_article(self, article_id):
        if article_id in self._cache:
            return self._cache[article_id]
        article = Article.objects.get(id=article_id)
        self._cache[article_id] = article
        return article

    def publish_article(self, article_id, user_id):
        article = self.get_article(article_id)
        user = User.objects.get(id=user_id)
        if user.is_authenticated:
            article.is_published = True
            article.save()
        return article
```

Find all bugs and design problems. There are at least 5.

**Challenge 3 — Build a Django Utility:**

Write a reusable Django view mixin `AjaxResponseMixin` that:
- Detects if the request is an AJAX request (via `X-Requested-With` header)
- For AJAX requests: returns JSON with `{"success": True, "data": {...}}`  or `{"success": False, "errors": [...]}`
- For regular requests: falls through to normal template rendering
- Works with both `FormView` and `DetailView`
- Has full type hints
- Uses proper error handling

---

## What Lesson03 Expects of You

When you start `Lesson03 - What is a framework`, you will be:

- Creating a Django project and app from scratch
- Understanding what `manage.py startproject` generates and why
- Writing your first model with fields and methods
- Running migrations and understanding what they do to the database
- Writing a basic view and wiring it to a URL

**Every single one of these tasks will require:**

- OOP knowledge to understand models as classes
- Modules and packages knowledge to understand the project structure
- Import knowledge to wire views to URLs
- Exception handling for database operations
- Understanding of decorators for view-level authentication
- Iterator knowledge for queryset operations in views

You are not starting fresh in Lesson03. You are applying everything from Lesson02 to a new domain.

---

## The Honest Verdict

**All ✅?** You're genuinely ready. Start Lesson03.

**Mostly ✅, a few 🔶?** Spend one focused day on the 🔶 items. Then start Lesson03.

**Several ❌?** Do not skip ahead. A week of Python fundamentals now prevents months of Django confusion. Go back to the specific lessons, redo the exercises with fresh eyes, and return to this checklist.

The goal is not to pass a test. The goal is to write Django code that you actually understand, can debug confidently, and can extend without fear.

That only happens when the foundation is solid.

---

*End of Lesson02 — Python Fundamentals for Django*

**Next:** [Lesson03 — What is a Framework?](../Lesson03%20-%20What%20is%20a%20framework/README.md)
