# 11 — Type Hints: Writing Python That Documents Itself

> **Type hints don't change how your program runs. They change how you and your tools understand it.** A function signature with type hints is a contract written in code — it tells every reader, every IDE, and every static analysis tool exactly what flows in and what comes out. In a Django project with 50 models, 100 views, and a team of developers, that contract is the difference between confident changes and terrified guessing.

---

## 1. What Type Hints Are — And What They Are Not

Type hints are **annotations**. They're metadata attached to variables, parameters, and return values. At runtime, Python ignores them entirely.

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

# At runtime, this is identical to:
def greet(name):
    return f"Hello, {name}!"

# The hints are stored but not enforced:
greet(42)         # works fine at runtime — Python doesn't check
greet.__annotations__  # {"name": <class 'str'>, "return": <class 'str'>}
```

**What they give you:**
- IDE autocomplete that actually knows what type you're working with
- Static type checkers (`mypy`, `pyright`) that catch type errors before runtime
- Documentation that can't go stale (it's in the code, not a comment)
- Explicit contracts between functions — especially valuable across team boundaries

**What they don't give you:**
- Runtime validation (use `pydantic` or Django forms for that)
- Performance improvements
- Protection against wrong types at runtime (unless you add runtime checking)

---

## 2. Basic Syntax — Variables, Parameters, Return Types

```python
# Variable annotations
age: int = 28
name: str = "Anurag"
is_active: bool = True
score: float = 9.5

# Annotating without assignment (common in class bodies)
class User:
    id: int
    username: str
    email: str
    is_staff: bool

# Function parameters and return type
def add(x: int, y: int) -> int:
    return x + y

def greet(name: str, title: str = "Mr.") -> str:
    return f"Hello, {title} {name}"

# Return None explicitly
def log_message(message: str) -> None:
    print(message)
    # No return value

# Functions that never return (raise or sys.exit)
def fail(message: str) -> Never:   # from typing import Never (Python 3.11+)
    raise RuntimeError(message)
```

---

## 3. The `typing` Module — Before Python 3.9

Before Python 3.9, you had to import generic types from `typing`:

```python
from typing import List, Dict, Tuple, Set, Optional, Union, Any, Callable

def process(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}
```

**Python 3.9+ — use built-in types directly:**

```python
# Python 3.9+: no imports needed for these
def process(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

def get_coordinates() -> tuple[float, float]:
    return (40.7128, -74.0060)

def get_tags() -> set[str]:
    return {"python", "django"}
```

**Python 3.10+ — use `|` instead of `Union`:**

```python
# Old way (still valid):
from typing import Union, Optional
def find_user(user_id: int) -> Union[User, None]:
    ...

# Python 3.10+ way — cleaner:
def find_user(user_id: int) -> User | None:
    ...

# Optional[X] is just Union[X, None]:
def find(id: int) -> Optional[str]:   # old
def find(id: int) -> str | None:      # new — prefer this
```

---

## 4. Optional, Union, Any — Handling Uncertainty

### 4.1 `Optional` — Values That Can Be None

```python
from typing import Optional

# This parameter might not be provided:
def create_article(
    title: str,
    content: str,
    author: str | None = None,   # Python 3.10+
    tags: list[str] | None = None,
) -> dict:
    return {
        "title": title,
        "content": content,
        "author": author or "Anonymous",
        "tags": tags or [],
    }
```

**In Django models:** any field with `null=True` should have `Optional[...]` in type hints.

```python
from typing import Optional
from datetime import datetime

class Article:
    id: int
    title: str
    published_at: Optional[datetime]   # null=True in DB → Optional in hints
    deleted_at: datetime | None        # same thing, Python 3.10+ style
```

### 4.2 `Union` — Multiple Possible Types

```python
# A function that accepts either str or int as an ID:
def get_user(user_id: int | str) -> User | None:
    if isinstance(user_id, str):
        return User.objects.filter(username=user_id).first()
    return User.objects.filter(id=user_id).first()

# A function that returns different types based on a flag:
def serialize(obj: Article, as_json: bool = False) -> dict | str:
    data = {"title": obj.title, "author": obj.author}
    if as_json:
        import json
        return json.dumps(data)
    return data
```

### 4.3 `Any` — The Escape Hatch

```python
from typing import Any

# Use Any when you genuinely don't know or don't care about the type
# — rare in good code
def log(value: Any) -> None:
    print(repr(value))

# Common in Django for things like raw request data before validation:
def process_webhook(payload: dict[str, Any]) -> None:
    ...
```

**Don't overuse `Any`** — it defeats the purpose of type hints. If you find yourself reaching for `Any` often, that's a signal your code could be better structured.

---

## 5. Collections — Being Precise About What's Inside

```python
# Lists
def process_names(names: list[str]) -> list[str]:
    return [name.strip().title() for name in names]

# Dicts
def build_context(user: User, articles: list[Article]) -> dict[str, Any]:
    return {"user": user, "articles": articles, "count": len(articles)}

# Tuples — fixed length, each position has a type
def get_lat_lng(address: str) -> tuple[float, float]:
    ...

# Variable-length tuple of one type:
def sum_all(*args: float) -> float:
    return sum(args)

# Sets
def get_unique_tags(articles: list[Article]) -> set[str]:
    return {tag for article in articles for tag in article.tags}

# Nested collections
def get_user_permissions(user: User) -> dict[str, list[str]]:
    return {
        "read": ["article", "comment"],
        "write": ["article"],
        "admin": [],
    }
```

---

## 6. Callables — Typing Functions as Arguments

```python
from typing import Callable

# A function that takes a function as argument:
def apply(func: Callable[[int, int], int], x: int, y: int) -> int:
    return func(x, y)

apply(lambda a, b: a + b, 3, 4)   # 7
apply(lambda a, b: a * b, 3, 4)   # 12

# Callable[[arg_types...], return_type]
# Callable[[str, int], bool] = function taking str and int, returning bool

# For complex callables, use Protocol (see below)
# For any callable: Callable[..., ReturnType]
def run_with_retry(func: Callable[..., Any], retries: int = 3) -> Any:
    for attempt in range(retries):
        try:
            return func()
        except Exception:
            if attempt == retries - 1:
                raise
```

---

## 7. `TypeVar` — Generic Functions

When a function returns the same type it receives, use `TypeVar`:

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T | None:
    return items[0] if items else None

# The return type is the SAME as the element type of the input list:
result = first([1, 2, 3])        # mypy knows: result is int | None
result = first(["a", "b"])       # mypy knows: result is str | None
result = first([User(), User()]) # mypy knows: result is User | None
```

```python
# TypeVar with constraints — T must be one of these types
from typing import TypeVar
Number = TypeVar("Number", int, float)

def double(x: Number) -> Number:
    return x * 2

double(5)     # returns int
double(3.14)  # returns float
double("a")   # mypy error: str doesn't satisfy Number
```

---

## 8. `Protocol` — Structural Typing (Duck Typing, Typed)

Python's duck typing says "if it has the right methods, it works." `Protocol` lets you express that in the type system without requiring inheritance.

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None:
        ...

class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

class Triangle:
    pass   # no draw() method

def render(shape: Drawable) -> None:
    shape.draw()

render(Circle())    # fine — Circle has draw()
render(Square())    # fine — Square has draw()
render(Triangle())  # mypy error — Triangle doesn't satisfy Drawable
```

No inheritance needed. `Circle` and `Square` don't know about `Drawable` — they just happen to implement the right interface. This is how Django's ORM works: anything that quacks like a queryset is treated like one.

**A Django-relevant Protocol:**

```python
from typing import Protocol, runtime_checkable

@runtime_checkable   # allows isinstance() checks at runtime
class Serializable(Protocol):
    def to_dict(self) -> dict[str, Any]:
        ...

    def to_json(self) -> str:
        ...

def send_to_api(obj: Serializable) -> None:
    data = obj.to_dict()
    # ... send data

# Any class implementing to_dict() and to_json() satisfies Serializable
```

---

## 9. `TypedDict` — Typed Dictionaries

Regular `dict[str, Any]` loses all information about what keys exist and what types they have. `TypedDict` fixes this.

```python
from typing import TypedDict, Required, NotRequired   # NotRequired: Python 3.11+

class ArticleData(TypedDict):
    title: str
    content: str
    author_id: int
    tags: list[str]

# With required vs optional fields:
class UserProfile(TypedDict, total=False):
    # total=False means all fields are optional by default
    bio: str
    avatar_url: str
    website: str

class UserData(TypedDict):
    # Mix required and optional in Python 3.11+:
    username: str                       # required
    email: str                          # required
    bio: NotRequired[str]               # optional
    avatar_url: NotRequired[str | None] # optional, can be None

def create_user(data: UserData) -> User:
    return User.objects.create(
        username=data["username"],
        email=data["email"],
        bio=data.get("bio", ""),
    )

# mypy knows the shape of the dict:
user_data: UserData = {
    "username": "anurag",
    "email": "anurag@example.com",
    # bio and avatar_url are optional
}
create_user(user_data)
```

**Django context dicts as TypedDict:**

```python
class ArticleDetailContext(TypedDict):
    article: Article
    related_articles: list[Article]
    comment_form: CommentForm
    user_has_liked: bool

def article_detail(request, pk: int) -> HttpResponse:
    article = get_object_or_404(Article, pk=pk)

    context: ArticleDetailContext = {
        "article": article,
        "related_articles": list(Article.objects.filter(
            tags__in=article.tags.all()
        ).exclude(pk=pk)[:5]),
        "comment_form": CommentForm(),
        "user_has_liked": article.likes.filter(user=request.user).exists(),
    }

    return render(request, "blog/article_detail.html", context)
```

---

## 10. `dataclass` with Type Hints — The Best of Both Worlds

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class ArticleDTO:
    """Data Transfer Object for article data."""
    title: str
    content: str
    author_id: int
    tags: list[str] = field(default_factory=list)
    published_at: datetime | None = None
    view_count: int = 0

    @property
    def is_published(self) -> bool:
        return self.published_at is not None

    @property
    def word_count(self) -> int:
        return len(self.content.split())

# Fully typed, __init__ auto-generated, __repr__ auto-generated:
article = ArticleDTO(
    title="Django Tips",
    content="Here are some tips...",
    author_id=42,
    tags=["django", "python"],
)

article.is_published   # False
article.word_count     # 4
```

---

## 11. Type Hints in Django — Practical Patterns

### 11.1 Typed Views

```python
from django.http import HttpRequest, HttpResponse, JsonResponse
from django.views.decorators.http import require_http_methods

def article_list(request: HttpRequest) -> HttpResponse:
    articles = Article.objects.filter(is_published=True)
    return render(request, "blog/list.html", {"articles": articles})

def article_api(request: HttpRequest, pk: int) -> JsonResponse:
    article = get_object_or_404(Article, pk=pk)
    return JsonResponse({
        "id": article.id,
        "title": article.title,
    })
```

### 11.2 Typed Model Methods

```python
from django.db import models
from django.contrib.auth.models import User
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from django.db.models import QuerySet   # import only for type checking

class ArticleManager(models.Manager):
    def published(self) -> "QuerySet[Article]":
        return self.filter(is_published=True)

    def by_author(self, user: User) -> "QuerySet[Article]":
        return self.filter(author=user)

class Article(models.Model):
    title: str
    content: str
    author: User
    is_published: bool

    objects: ArticleManager = ArticleManager()  # type: ignore[assignment]

    def get_absolute_url(self) -> str:
        from django.urls import reverse
        return reverse("article-detail", kwargs={"pk": self.pk})

    def can_be_edited_by(self, user: User) -> bool:
        return self.author == user or user.is_staff
```

### 11.3 Typed Service Functions

```python
from typing import NamedTuple

class PublishResult(NamedTuple):
    success: bool
    article: Article | None
    errors: list[str]

def publish_article(article_id: int, requesting_user: User) -> PublishResult:
    try:
        article = Article.objects.get(id=article_id)
    except Article.DoesNotExist:
        return PublishResult(success=False, article=None, errors=["Article not found"])

    if article.author != requesting_user and not requesting_user.is_staff:
        return PublishResult(success=False, article=None, errors=["Permission denied"])

    errors: list[str] = []
    if not article.content:
        errors.append("Article has no content")
    if not article.title:
        errors.append("Article has no title")

    if errors:
        return PublishResult(success=False, article=article, errors=errors)

    article.is_published = True
    article.save()
    return PublishResult(success=True, article=article, errors=[])

# Caller knows exactly what to expect:
result = publish_article(pk, request.user)
if not result.success:
    return JsonResponse({"errors": result.errors}, status=400)
return redirect("article-detail", pk=result.article.pk)
```

---

## 12. `from __future__ import annotations` — Deferred Evaluation

In older Python, you can't reference a class that hasn't been defined yet in type hints:

```python
class Article:
    def get_siblings(self) -> list[Article]:   # NameError in Python < 3.10!
        ...
```

Fix:

```python
from __future__ import annotations   # makes ALL annotations lazy strings

class Article:
    def get_siblings(self) -> list[Article]:   # works — "Article" is just a string
        ...
```

Or use a string literal:

```python
class Article:
    def get_siblings(self) -> "list[Article]":   # string form — always works
        ...
```

Put `from __future__ import annotations` at the top of every file that has forward references.

---

## 13. Running `mypy` — Making Type Hints Do Work

Type hints without a type checker are just documentation. With `mypy`, they become automated bug detection.

```bash
pip install mypy django-stubs mypy-extensions

# mypy.ini or setup.cfg:
[mypy]
python_version = 3.11
plugins =
    mypy_django_plugin.main

[mypy.plugins.django-stubs]
django_settings_module = myproject.settings

[mypy-*.migrations.*]
ignore_errors = True   # don't type-check auto-generated migrations
```

```bash
# Run on your project:
mypy blog/

# Common output:
# blog/views.py:45: error: Argument 1 to "filter" has incompatible type "str"; expected "int"
# blog/models.py:12: error: Item "None" of "Optional[User]" has no attribute "username"
```

**`mypy` in CI:** Add `mypy .` to your test pipeline. Catch type errors before they reach production.

---

## ⚠️ Common Mistakes

**Mistake 1: Using `List`, `Dict` from `typing` in Python 3.9+**
```python
from typing import List, Dict   # outdated
def f(items: List[str]) -> Dict[str, int]: ...

# Modern (Python 3.9+):
def f(items: list[str]) -> dict[str, int]: ...
```

**Mistake 2: Annotating everything as `Any`**
```python
def process(data: Any) -> Any:   # might as well have no hints
    ...
```

**Mistake 3: Using string literals when not needed**
```python
# Not needed if not a forward reference:
def greet(name: "str") -> "str":   # unnecessary quotes
    ...
# Just:
def greet(name: str) -> str:
    ...
```

**Mistake 4: Forgetting `-> None` for procedures**
```python
def save_to_db(article):   # ambiguous — does it return something?
    article.save()

def save_to_db(article: Article) -> None:   # explicit — nothing returned
    article.save()
```

**Mistake 5: Type hints in `__init__` without understanding self**
```python
class Article:
    def __init__(self, title: str) -> None:  # always None — __init__ never returns a value
        self.title: str = title              # annotate the attribute assignment
```

---

## 🧪 Exercises

**Exercise 1:** Take this untyped Django service module and add complete, accurate type hints to every function and class:
```python
def get_or_create_profile(user, defaults=None):
    profile, created = UserProfile.objects.get_or_create(user=user, defaults=defaults or {})
    return profile, created

def bulk_update_status(user_ids, status):
    updated = User.objects.filter(id__in=user_ids).update(is_active=status)
    return updated

def get_user_stats(user):
    return {
        "articles": user.article_set.count(),
        "published": user.article_set.filter(is_published=True).count(),
        "tags_used": list(user.article_set.values_list("tags__name", flat=True).distinct()),
    }
```

**Exercise 2:** Define a `TypedDict` for a REST API response envelope that your Django views return:
```python
# Should handle both success and error cases:
# {"status": "success", "data": {...}, "meta": {"page": 1, "total": 100}}
# {"status": "error", "errors": ["field is required"], "code": 400}
```

**Exercise 3:** Write a generic `paginate` function using `TypeVar` that:
- Takes any `list[T]`
- Takes `page: int` and `per_page: int`
- Returns a `TypedDict` with `items: list[T]`, `total: int`, `page: int`, `pages: int`
- mypy should be able to infer the type of `items` based on the input list type

**Exercise 4:** Write a `Protocol` called `CacheBackend` with methods:
- `get(key: str) -> Any`
- `set(key: str, value: Any, timeout: int | None = None) -> None`
- `delete(key: str) -> bool`
- `clear() -> None`

Then write a `cached_view` decorator factory that takes a `CacheBackend` and a `timeout: int`, and caches view responses. The decorator must be fully typed.

---

## ✅ Before Moving On

- [ ] You understand that type hints are not enforced at runtime
- [ ] You use `X | None` (Python 3.10+) instead of `Optional[X]`
- [ ] You use `list[str]`, `dict[str, int]` (Python 3.9+) instead of `List`, `Dict` from `typing`
- [ ] You know when to use `TypeVar`, `Protocol`, `TypedDict`
- [ ] You've added type hints to a real piece of Django code and run `mypy` on it
- [ ] You understand `from __future__ import annotations` and when to use it
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

Type hints in Django have reached the point where Django itself ships with type stubs (`django-stubs`). The Django REST Framework has `djangorestframework-stubs`. When you add type hints and run mypy, you'll catch:

- Passing the wrong type to ORM methods
- Accessing attributes on `None` (the most common Django bug)
- Inconsistent return types in views and services
- Missing fields in context dicts

Teams that adopt type hints and mypy in Django projects consistently report fewer production bugs and much faster onboarding of new developers — because the code tells you what it expects, rather than making you read through 5 files to figure it out.

**Next:** [12 — Pre-Django Checklist: Are You Actually Ready?](./12_python_for_django_checklist.md)
