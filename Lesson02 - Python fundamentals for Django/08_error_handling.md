# 08 — Error Handling: Writing Code That Fails Gracefully

> **Production code doesn't crash. It fails gracefully, reports what happened, and recovers where possible.** Amateur code either ignores errors (and produces silent wrong results) or crashes loudly with no useful information. Django's entire request/response cycle is wrapped in error handling machinery. Understanding Python's exception system is what separates code that works in development from code that survives in production.

---

## 1. Python's Exception Hierarchy — The Full Map

Every exception in Python is a class that inherits from `BaseException`. Understanding the hierarchy tells you what you can catch and how broad your net is.

```
BaseException
├── SystemExit                  ← raised by sys.exit() — don't catch this
├── KeyboardInterrupt           ← Ctrl+C — don't catch this either
├── GeneratorExit               ← generator cleanup — never catch
└── Exception                  ← everything catchable lives here
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   ├── OverflowError
    │   └── FloatingPointError
    ├── LookupError
    │   ├── IndexError          ← list[999] on a 3-item list
    │   └── KeyError            ← dict["missing_key"]
    ├── ValueError              ← right type, wrong value: int("abc")
    ├── TypeError               ← wrong type: "string" + 5
    ├── AttributeError          ← obj.nonexistent_attribute
    ├── NameError               ← using undefined variable
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── OSError (IOError)
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── RuntimeError
    │   └── RecursionError
    ├── StopIteration           ← signals end of an iterator
    ├── MemoryError
    └── NotImplementedError     ← abstract method not implemented
```

**Why this matters:**

```python
# Catching a parent class catches all its children too
try:
    result = my_dict[key]
except LookupError:
    # catches BOTH IndexError and KeyError
    handle_missing()

try:
    value = int(user_input)
except Exception:
    # catches EVERYTHING in the hierarchy above
    # too broad — you'll swallow bugs you didn't expect
    pass

# Never catch BaseException — it catches SystemExit and KeyboardInterrupt
# which means Ctrl+C won't work and sys.exit() won't exit
```

---

## 2. `try / except / else / finally` — The Complete Structure

Most tutorials only show `try/except`. The full structure has four clauses and each one has a specific purpose.

```python
try:
    # Code that might raise an exception
    result = risky_operation()

except SpecificError as e:
    # Runs ONLY if that specific exception occurred
    # 'e' is the exception instance
    handle_specific_error(e)

except (AnotherError, YetAnotherError) as e:
    # Multiple exception types in one handler
    handle_these(e)

except Exception as e:
    # Broad catch — use as a last resort, always log
    logger.error(f"Unexpected error: {e}", exc_info=True)
    raise   # re-raise — don't just swallow it

else:
    # Runs ONLY if NO exception occurred in the try block
    # This is important: it's NOT inside the try, so exceptions here
    # are NOT caught by the above except clauses
    use_result(result)

finally:
    # ALWAYS runs — exception or not, return or not
    # Use for cleanup: close files, release locks, close connections
    cleanup()
```

**The `else` clause is often overlooked but important:**

```python
# Without else — misleading structure
try:
    data = fetch_from_database()
    process(data)           # BUG: if process() raises, the except might handle it
except DatabaseError:
    log("DB failed")

# With else — clear separation of concerns
try:
    data = fetch_from_database()    # only this can raise DatabaseError
except DatabaseError:
    log("DB failed")
else:
    process(data)   # runs only if fetch succeeded; exceptions here propagate normally
```

---

## 3. Exception Information — What You Can Access

```python
try:
    result = 1 / 0
except ZeroDivisionError as e:
    print(type(e))           # <class 'ZeroDivisionError'>
    print(e)                 # division by zero
    print(str(e))            # "division by zero"
    print(repr(e))           # ZeroDivisionError('division by zero')
    print(e.args)            # ('division by zero',)

    # Get the full traceback:
    import traceback
    traceback.print_exc()    # prints to stderr
    tb_string = traceback.format_exc()   # full traceback as string (for logging)
```

**Getting exception info in `finally` or after the handler:**

```python
import sys

try:
    risky()
except Exception:
    exc_type, exc_value, exc_traceback = sys.exc_info()
    # exc_type: the exception class
    # exc_value: the exception instance
    # exc_traceback: traceback object
```

---

## 4. Raising Exceptions — Your Code Should Scream When Something Is Wrong

Don't return `None` or `False` when something genuinely goes wrong. Raise an exception. Silence hides bugs.

```python
# Bad — the caller has to check the return value every time
def get_user(user_id):
    if user_id <= 0:
        return None
    ...

user = get_user(-1)
if user is None:       # easy to forget this check
    ...
user.name              # AttributeError if you forgot to check

# Good — fail loudly, fail early
def get_user(user_id):
    if user_id <= 0:
        raise ValueError(f"user_id must be positive, got {user_id}")
    ...
```

### 4.1 `raise` Forms

```python
# Raise a new exception
raise ValueError("Something is wrong")
raise ValueError(f"Expected positive int, got {value!r}")

# Raise an exception instance
error = ValueError("Something is wrong")
raise error

# Re-raise the current exception (inside an except block)
try:
    risky()
except Exception as e:
    log(e)
    raise   # re-raises the SAME exception with SAME traceback — correct

# Chain exceptions — show the context
try:
    connect_to_database()
except ConnectionRefusedError as e:
    raise RuntimeError("Failed to start application") from e
    # The 'from e' preserves the original exception as __cause__

# Suppress context (when you don't want to show the cause)
raise NewException("message") from None
```

### 4.2 Exception Chaining — `__cause__` and `__context__`

```python
try:
    int("not a number")
except ValueError as e:
    raise RuntimeError("Configuration failed") from e

# Output:
# ValueError: invalid literal for int() with base 10: 'not a number'
# The above exception was the direct cause of the following exception:
# RuntimeError: Configuration failed
```

This is crucial for debugging. When you catch one exception and raise another, always use `from original_exception` to preserve the cause chain. Losing the original exception makes production debugging a nightmare.

---

## 5. Custom Exceptions — Building a Domain-Specific Error Hierarchy

One of the most powerful things you can do for code quality: define your own exception hierarchy for your application's domain.

```python
# exceptions.py — in your Django app

class BlogException(Exception):
    """Base exception for all blog-related errors."""
    pass

class ArticleError(BlogException):
    """Errors related to articles."""
    pass

class ArticleNotFoundError(ArticleError):
    """Raised when an article doesn't exist."""
    def __init__(self, article_id):
        self.article_id = article_id
        super().__init__(f"Article with id={article_id} does not exist")

class ArticlePermissionError(ArticleError):
    """Raised when a user doesn't have permission to modify an article."""
    def __init__(self, user, article):
        self.user = user
        self.article = article
        super().__init__(
            f"User '{user.username}' cannot modify article '{article.title}'"
        )

class ArticlePublishError(ArticleError):
    """Raised when an article cannot be published due to validation issues."""
    def __init__(self, article, reasons):
        self.article = article
        self.reasons = reasons
        reasons_str = "; ".join(reasons)
        super().__init__(f"Cannot publish '{article.title}': {reasons_str}")

class PaymentError(BlogException):
    """Errors related to payments."""
    pass

class InsufficientFundsError(PaymentError):
    def __init__(self, required, available):
        self.required = required
        self.available = available
        super().__init__(
            f"Insufficient funds: need {required}, have {available}"
        )
```

**Why a hierarchy?** Callers can choose how specific to be:

```python
try:
    publish_article(article, user)
except ArticleNotFoundError:
    return HttpResponse("Article not found", status=404)
except ArticlePermissionError:
    return HttpResponse("Permission denied", status=403)
except ArticlePublishError as e:
    return JsonResponse({"errors": e.reasons}, status=400)
except ArticleError:
    # catches any ArticleError we didn't handle above
    return HttpResponse("Article operation failed", status=500)
except BlogException:
    # catches any blog-related error
    logger.error("Unexpected blog error", exc_info=True)
    return HttpResponse("Server error", status=500)
```

---

## 6. Django's Exception Hierarchy

Django defines its own exception classes that you need to know:

```python
from django.core.exceptions import (
    ObjectDoesNotExist,      # base for model .DoesNotExist
    MultipleObjectsReturned, # .get() found more than one result
    ValidationError,         # form/model validation failure
    PermissionDenied,        # user lacks permission — triggers 403
    SuspiciousOperation,     # e.g. bad file path, bad cookie — triggers 400
    ImproperlyConfigured,    # something wrong in settings
    FieldDoesNotExist,       # model has no such field
    FieldError,              # invalid field lookup in queryset
)

from django.http import Http404   # triggers 404 response

from django.db import (
    IntegrityError,          # DB constraint violation (unique, FK, etc.)
    OperationalError,        # DB connection/query failure
    DatabaseError,           # base for all DB errors
)
```

### 6.1 The Model `.DoesNotExist` Pattern

```python
# Each model gets its own DoesNotExist (and MultipleObjectsReturned):
Article.DoesNotExist    # subclass of ObjectDoesNotExist and Exception
User.DoesNotExist

# The safe get pattern:
try:
    article = Article.objects.get(slug=slug)
except Article.DoesNotExist:
    raise Http404(f"No article with slug '{slug}'")

# Django shortcut that does the same thing:
from django.shortcuts import get_object_or_404
article = get_object_or_404(Article, slug=slug)
# If not found → automatically raises Http404 → Django returns 404 page
```

### 6.2 `ValidationError` — The Most Used Django Exception

```python
from django.core.exceptions import ValidationError

# In a model's clean() method:
class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published_at = models.DateTimeField(null=True, blank=True)

    def clean(self):
        errors = {}

        if self.published_at and not self.content:
            errors["content"] = ValidationError(
                "Published articles must have content."
            )

        if self.title and len(self.title) < 10:
            errors["title"] = ValidationError(
                "Title must be at least 10 characters.",
                code="title_too_short"  # machine-readable code
            )

        if errors:
            raise ValidationError(errors)  # dict form — field-specific errors

# In a form:
class ArticleForm(forms.ModelForm):
    def clean_title(self):
        title = self.cleaned_data["title"]
        if Article.objects.filter(title=title).exists():
            raise ValidationError("An article with this title already exists.")
        return title

    def clean(self):
        cleaned_data = super().clean()
        start = cleaned_data.get("start_date")
        end = cleaned_data.get("end_date")
        if start and end and start > end:
            raise ValidationError("Start date must be before end date.")
        return cleaned_data
```

### 6.3 `IntegrityError` — Database Constraint Violations

```python
from django.db import IntegrityError, transaction

def create_user_profile(user, bio):
    try:
        profile = UserProfile.objects.create(user=user, bio=bio)
        return profile
    except IntegrityError:
        # Unique constraint violation — profile already exists for this user
        return UserProfile.objects.get(user=user)

# Race condition safe version — use get_or_create:
profile, created = UserProfile.objects.get_or_create(
    user=user,
    defaults={"bio": bio}
)
```

---

## 7. Context Managers for Error Handling

### 7.1 `contextlib.suppress` — Intentional Silence

Sometimes you genuinely want to ignore a specific exception. `suppress` makes this intent explicit instead of hiding it in a bare `except: pass`.

```python
from contextlib import suppress

# BAD — unclear intent
try:
    os.remove("temp_file.txt")
except FileNotFoundError:
    pass

# GOOD — intent is clear: if the file doesn't exist, that's fine
with suppress(FileNotFoundError):
    os.remove("temp_file.txt")

# Multiple exceptions:
with suppress(FileNotFoundError, PermissionError):
    os.remove("temp_file.txt")
```

### 7.2 `contextlib.contextmanager` — Writing Context Managers as Generators

```python
from contextlib import contextmanager

@contextmanager
def managed_database_operation(operation_name):
    """Context manager that logs and handles database operation errors."""
    import logging
    logger = logging.getLogger(__name__)

    logger.info(f"Starting: {operation_name}")
    try:
        yield   # ← the 'with' block runs here
        logger.info(f"Completed: {operation_name}")
    except IntegrityError as e:
        logger.error(f"Integrity error in {operation_name}: {e}")
        raise
    except DatabaseError as e:
        logger.critical(f"Database error in {operation_name}: {e}", exc_info=True)
        raise
    finally:
        logger.debug(f"Finished: {operation_name}")

# Usage:
with managed_database_operation("create_user"):
    User.objects.create(username="anurag", email="a@example.com")
```

### 7.3 `transaction.atomic()` — Django's Error Recovery

```python
from django.db import transaction

def transfer_credits(from_user, to_user, amount):
    """Transfer credits atomically — either both succeed or neither does."""
    try:
        with transaction.atomic():
            from_user.credits -= amount
            from_user.save()

            if to_user.is_banned:
                raise ValueError("Cannot transfer to banned user")

            to_user.credits += amount
            to_user.save()

            # If any exception is raised here, BOTH saves are rolled back
    except ValueError as e:
        return {"success": False, "error": str(e)}
    except IntegrityError as e:
        return {"success": False, "error": "Database constraint violation"}

    return {"success": True}
```

`transaction.atomic()` creates a savepoint. If an exception escapes the `with` block, the transaction rolls back. This is how you ensure your database stays consistent even when operations fail mid-way.

---

## 8. Logging — The Right Way to Track Errors

`print()` is for development. `logging` is for production. Django uses Python's built-in `logging` module.

```python
import logging

# Get a logger for this module — ALWAYS use __name__
logger = logging.getLogger(__name__)
# In blog/views.py, __name__ is 'blog.views'
# This creates a hierarchy: root → blog → blog.views

# Log at different severity levels:
logger.debug("Entering view: %s", request.path)       # development info
logger.info("User %s logged in", request.user)        # normal operations
logger.warning("Rate limit approaching for %s", ip)   # something to watch
logger.error("Failed to send email to %s", email)     # something went wrong
logger.critical("Database connection lost")           # immediate action required

# Log exceptions with full traceback:
try:
    risky_operation()
except Exception as e:
    logger.exception("Unexpected error in risky_operation")
    # logger.exception = logger.error + exc_info=True
    # logs the full traceback automatically
```

### 8.1 Django Logging Configuration

```python
# settings.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,

    "formatters": {
        "verbose": {
            "format": "{levelname} {asctime} {module} {process:d} {thread:d} {message}",
            "style": "{",
        },
        "simple": {
            "format": "{levelname} {message}",
            "style": "{",
        },
    },

    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "simple",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": BASE_DIR / "logs" / "django.log",
            "maxBytes": 1024 * 1024 * 5,   # 5MB
            "backupCount": 5,
            "formatter": "verbose",
            "encoding": "utf-8",
        },
        "mail_admins": {
            "class": "django.utils.log.AdminEmailHandler",
            "level": "ERROR",
            # sends emails to settings.ADMINS for errors in production
        },
    },

    "root": {
        "handlers": ["console"],
        "level": "WARNING",
    },

    "loggers": {
        "django": {
            "handlers": ["console", "file"],
            "level": "INFO",
            "propagate": False,
        },
        "blog": {                  # your app's logger
            "handlers": ["console", "file"],
            "level": "DEBUG",      # verbose in dev; set to WARNING in prod
            "propagate": False,
        },
    },
}
```

```python
# In any file in your blog app:
import logging
logger = logging.getLogger(__name__)   # 'blog.views', 'blog.models', etc.
# All these loggers feed into the 'blog' logger configured above
```

---

## 9. Django's Error Views — What Happens When Things Go Wrong

Django maps specific exceptions to HTTP responses:

| Exception | HTTP Status | Django Handler |
|-----------|------------|----------------|
| `Http404` | 404 | `handler404` |
| `PermissionDenied` | 403 | `handler403` |
| `SuspiciousOperation` | 400 | `handler400` |
| Any unhandled exception | 500 | `handler500` |

```python
# Custom error pages — in your project's urls.py:
handler404 = "myproject.views.custom_404"
handler500 = "myproject.views.custom_500"

# views.py:
def custom_404(request, exception):
    return render(request, "errors/404.html", status=404)

def custom_500(request):
    return render(request, "errors/500.html", status=500)
```

```python
# In a view — raise Django exceptions directly:
from django.core.exceptions import PermissionDenied
from django.http import Http404

def article_detail(request, pk):
    article = get_object_or_404(Article, pk=pk)

    if article.is_private and not request.user.is_authenticated:
        raise PermissionDenied   # → 403 page

    if not article.is_published and article.author != request.user:
        raise Http404   # pretend it doesn't exist → 404 page

    return render(request, "blog/article_detail.html", {"article": article})
```

---

## 10. Error Handling Patterns in Django Views

### 10.1 The Safe View Pattern

```python
import logging
from django.http import JsonResponse
from django.views.decorators.http import require_POST
from django.contrib.auth.decorators import login_required
from django.db import IntegrityError, transaction

logger = logging.getLogger(__name__)

@login_required
@require_POST
def api_create_article(request):
    try:
        data = json.loads(request.body)
    except json.JSONDecodeError:
        return JsonResponse({"error": "Invalid JSON"}, status=400)

    title = data.get("title", "").strip()
    content = data.get("content", "").strip()

    if not title:
        return JsonResponse({"error": "Title is required"}, status=400)
    if not content:
        return JsonResponse({"error": "Content is required"}, status=400)

    try:
        with transaction.atomic():
            article = Article.objects.create(
                title=title,
                content=content,
                author=request.user,
            )
    except IntegrityError as e:
        logger.warning("Integrity error creating article: %s", e)
        return JsonResponse({"error": "Article already exists"}, status=409)
    except Exception as e:
        logger.exception("Unexpected error creating article")
        return JsonResponse({"error": "Server error"}, status=500)

    return JsonResponse({"id": article.id, "title": article.title}, status=201)
```

### 10.2 The Service Layer Pattern

Keep error handling logic out of views by using a service layer:

```python
# services.py
from .exceptions import ArticleNotFoundError, ArticlePermissionError, ArticlePublishError

def publish_article(article_id, requesting_user):
    """
    Publish an article. Raises domain-specific exceptions on failure.
    Views catch these and translate to HTTP responses.
    """
    try:
        article = Article.objects.get(id=article_id)
    except Article.DoesNotExist:
        raise ArticleNotFoundError(article_id)

    if article.author != requesting_user and not requesting_user.is_staff:
        raise ArticlePermissionError(requesting_user, article)

    errors = []
    if not article.content:
        errors.append("Article has no content")
    if not article.title:
        errors.append("Article has no title")
    if errors:
        raise ArticlePublishError(article, errors)

    article.is_published = True
    article.published_at = timezone.now()
    article.save()
    return article

# views.py — clean, thin
def publish_article_view(request, pk):
    try:
        article = publish_article(pk, request.user)
    except ArticleNotFoundError:
        return HttpResponse(status=404)
    except ArticlePermissionError:
        return HttpResponse(status=403)
    except ArticlePublishError as e:
        return JsonResponse({"errors": e.reasons}, status=400)

    return redirect("article-detail", pk=article.pk)
```

---

## ⚠️ Common Mistakes

**Mistake 1: Bare `except` — the worst thing you can write**
```python
try:
    risky()
except:           # catches EVERYTHING including KeyboardInterrupt, SystemExit
    pass          # silently swallows all errors — production nightmare

# Fix: always name what you're catching
except Exception as e:
    logger.exception("Unexpected error")
    raise
```

**Mistake 2: Catching too broad, doing too little**
```python
try:
    user = User.objects.get(id=user_id)
    send_email(user.email)
    update_record(user)
except Exception:
    print("Something went wrong")   # which of the 3 operations failed? no idea.

# Fix: narrow try blocks, specific exceptions
try:
    user = User.objects.get(id=user_id)
except User.DoesNotExist:
    return not_found_response()

try:
    send_email(user.email)
except SMTPException as e:
    logger.error("Email failed for user %s: %s", user.id, e)
    # continue — email failure shouldn't stop the whole operation
```

**Mistake 3: Swallowing exceptions without re-raising**
```python
try:
    data = fetch_critical_data()
except Exception as e:
    logger.error(e)
    # forgot to raise — function returns None, caller gets no error
    # silent failure is worse than a loud crash

# Fix:
except Exception as e:
    logger.error(e)
    raise   # or raise a domain-specific exception
```

**Mistake 4: Raising `Exception` instead of a specific type**
```python
raise Exception("User not found")    # caller can't handle specifically

raise ValueError("user_id must be positive")   # better — specific type
raise ArticleNotFoundError(article_id)          # best — domain-specific
```

**Mistake 5: Losing the traceback when chaining**
```python
try:
    risky()
except Exception as e:
    raise RuntimeError("Failed") from None   # loses original traceback — bad for debugging
    raise RuntimeError("Failed") from e      # correct — preserves chain
```

---

## 🧪 Exercises

**Exercise 1:** Define a complete exception hierarchy for an e-commerce order system:
- Base: `OrderException`
- Children: `OrderNotFoundError`, `OrderAlreadyCancelledError`, `PaymentFailedError`, `InsufficientStockError`
- Each should carry relevant data (order_id, product, etc.) and produce a meaningful `str()`

**Exercise 2:** Write a function `safe_divide(a, b)` that:
- Returns `a / b` if valid
- Raises `TypeError` with a clear message if either argument is not a number
- Raises `ValueError` with a clear message if `b` is zero
- Includes comprehensive tests using `assert` statements

**Exercise 3:** Write a Django service function `transfer_subscription(from_user_id, to_user_id)` that:
- Checks both users exist (raise `ObjectDoesNotExist` if not)
- Checks `from_user` has an active subscription (raise a custom exception if not)
- Transfers the subscription inside `transaction.atomic()`
- Logs every step with appropriate log levels
- Returns the updated subscription object

**Exercise 4:** Set up a Django logging configuration in `settings.py` that:
- Logs everything at `DEBUG` level to the console in development
- Logs `WARNING` and above to a rotating file in `logs/app.log` (max 10MB, keep 3 backups)
- Emails `ADMINS` on `ERROR` and above (using `AdminEmailHandler`)
- Has separate loggers for `django`, `django.request`, and your app
- Test it by deliberately raising an exception in a view

---

## ✅ Before Moving On

- [ ] You know Python's exception hierarchy — what inherits from what
- [ ] You use all four clauses: `try / except / else / finally` correctly
- [ ] You always catch specific exceptions, never bare `except:`
- [ ] You always re-raise or log — never silently swallow
- [ ] You've built a custom exception hierarchy with domain-specific data
- [ ] You know Django's core exceptions: `Http404`, `PermissionDenied`, `ValidationError`, `IntegrityError`
- [ ] You understand `transaction.atomic()` and why it exists
- [ ] You've configured Django's logging system from scratch
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

Error handling **is** Django views. Every view is one giant try/except — Django's middleware wraps every request in error handling, catching unhandled exceptions and returning 500 responses. Your job as a developer is to handle the expected failures yourself, let the unexpected ones propagate with enough information to debug them, and ensure nothing leaves the database in an inconsistent state.

`ValidationError`, `Http404`, `PermissionDenied`, `IntegrityError` — you'll type these dozens of times. The custom exception hierarchy pattern and service layer pattern from this lesson are what separate spaghetti Django code from maintainable Django code.

**Next:** [09 — Iterators & Generators: How Django's QuerySets Are Lazy](./09_iterators_and_generators.md)
