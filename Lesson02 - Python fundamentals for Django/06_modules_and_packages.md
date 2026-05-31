# 06 — Modules & Packages: The Architecture of a Python Project

> **A Django project is not a script. It's a package.** The moment you run `django-admin startproject`, Django generates a folder structure built entirely on Python's module and package system. If you don't understand how `import` actually works — how Python finds files, what `__init__.py` does, what relative vs absolute imports mean — you will spend hours fighting import errors you can't diagnose.

---

## 1. What a Module Is — Really

A module is just a `.py` file. That's it. When Python executes `import something`, it finds `something.py`, runs it from top to bottom, and gives you back a reference to the resulting namespace.

```python
# math_utils.py  ← this file IS the module
PI = 3.14159

def circle_area(radius):
    return PI * radius ** 2

def circle_circumference(radius):
    return 2 * PI * radius

class Circle:
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return circle_area(self.radius)
```

```python
# main.py
import math_utils

print(math_utils.PI)                       # 3.14159
print(math_utils.circle_area(5))           # 78.53975
c = math_utils.Circle(3)
print(c.area())                            # 28.27431
```

When you write `import math_utils`, Python:
1. Looks for `math_utils.py` (or a `math_utils/` package) in the search path
2. Executes the entire file
3. Stores the module in `sys.modules` (a cache — importing twice doesn't re-execute)
4. Binds the name `math_utils` in your current namespace

**Step 3 is important:** modules are only executed once per interpreter session, no matter how many times you import them.

```python
import sys
"math_utils" in sys.modules   # True after first import
```

---

## 2. Import Styles — Choose the Right One

```python
# Style 1: import the module — access everything through the namespace
import math
math.sqrt(16)   # 4.0 — clear where sqrt comes from

# Style 2: import specific names — less typing, less clarity about origin
from math import sqrt, pi
sqrt(16)   # 4.0 — but where does sqrt come from? (in large files, hard to tell)

# Style 3: import with alias — for long names or avoiding name collisions
import numpy as np
import pandas as pd
np.array([1, 2, 3])

# Style 4: import everything — almost always a bad idea
from math import *   # imports ALL public names into current namespace
# sqrt(16)  — works, but now you have 40+ names polluting your namespace
```

**Which style to use:**

| Situation | Preferred Style |
|---|---|
| Standard library module | `import module` or `from module import name` |
| Third-party with long name | `import module as alias` |
| A few specific names you'll use 50+ times | `from module import name` |
| `*` imports | Almost never — only in `__init__.py` for controlled re-exports |

**Django projects use a consistent style:**

```python
# Django conventions you'll see everywhere
from django.db import models
from django.contrib.auth.models import User
from django.urls import path, include
from django.http import HttpResponse, JsonResponse
from django.shortcuts import render, redirect, get_object_or_404
from django.conf import settings
```

Always `from django.x import y` — never `import django` and then `django.db.models.CharField`.

---

## 3. How Python Finds Modules — `sys.path`

When you write `import something`, Python searches these locations in order:

1. The directory containing the script being run
2. Directories in the `PYTHONPATH` environment variable
3. Installation-dependent defaults (where `pip` installs packages)

```python
import sys
print(sys.path)
# ['', '/usr/lib/python3.11', '/usr/lib/python3.11/lib-dynload', ...]
# '' means the current directory
```

**Django manipulates `sys.path` automatically** when you run `manage.py`. That's why you can do `from myapp.models import Article` from anywhere in the project — Django ensures the project root is on `sys.path`.

```python
# This is roughly what manage.py does:
import sys
sys.path.insert(0, "/path/to/your/django/project")
```

When you get an `ImportError: No module named 'myapp'`, the diagnosis is always: **the directory containing `myapp/` is not on `sys.path`.**

---

## 4. Packages — Modules Organized into Folders

A **package** is a directory containing a special file: `__init__.py`. That file makes Python treat the directory as a package.

```
myproject/
├── manage.py
├── myproject/
│   ├── __init__.py       ← makes myproject/ a package
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── blog/
    ├── __init__.py       ← makes blog/ a package
    ├── models.py
    ├── views.py
    ├── urls.py
    ├── forms.py
    ├── admin.py
    └── tests.py
```

```python
# Importing from a package
from blog.models import Article       # blog/models.py → Article class
from blog.views import ArticleListView
from myproject.settings import DEBUG
```

**Without `__init__.py`:** In Python 3.3+, directories without `__init__.py` are still importable as "namespace packages" — but Django apps require `__init__.py`. Always create it.

### 4.1 What `__init__.py` Does

By default, it's empty — just a marker that says "this directory is a package."

But it can also be used to **control what gets exposed when someone imports the package:**

```python
# blog/__init__.py  — empty by default in Django apps
```

```python
# utils/__init__.py  — re-exporting selected names
from .string_helpers import slugify, truncate
from .date_helpers import format_date, time_ago
from .validators import validate_email, validate_phone

# Now users can do:
from utils import slugify    # instead of: from utils.string_helpers import slugify
from utils import time_ago   # instead of: from utils.date_helpers import time_ago
```

This is the **facade pattern** — `__init__.py` is the public API of the package. Internal structure can change without breaking callers.

**Django uses this in its own source code:**

```python
# django/db/models/__init__.py exports Model, Field, Q, F, etc.
# That's why you can write:
from django.db import models
models.Model    # works, even though Model is defined deep in django/db/models/base.py
models.Q        # works, even though Q is in django/db/models/query_utils.py
```

---

## 5. Absolute vs. Relative Imports

### 5.1 Absolute Imports

Start from the top of the package hierarchy. Always explicit, always clear.

```python
# In blog/views.py:
from blog.models import Article          # absolute — unambiguous
from blog.forms import ArticleForm
from django.shortcuts import render
```

### 5.2 Relative Imports

Use `.` to mean "current package", `..` to mean "parent package."

```python
# In blog/views.py:
from .models import Article              # . = same package (blog)
from .forms import ArticleForm           # . = same package (blog)
from ..utils import format_date          # .. = parent package
```

**When to use which:**

| | Absolute | Relative |
|---|---|---|
| **Clarity** | Always clear where it comes from | Can be confusing in deep hierarchies |
| **Refactoring** | Must update if package is renamed | Still works if package is renamed |
| **Django convention** | Preferred in most Django code | Common in reusable app internals |
| **In `__init__.py`** | Both fine | Relative is common |

**PEP 8 says:** prefer absolute imports. Django's own code uses a mix.

```python
# In your Django app — use absolute for clarity:
from blog.models import Article
from blog.serializers import ArticleSerializer

# Using relative in the same app file is also fine:
from .models import Article
```

---

## 6. The Module Execution Model — What Runs and When

**Everything at the top level of a module runs when imported.** This has consequences.

```python
# dangerous_module.py
import time

print("This module is being imported!")   # runs on import
time.sleep(5)                              # also runs on import — bad!

def useful_function():
    return "hello"
```

```python
import dangerous_module
# prints "This module is being imported!" and sleeps 5 seconds
# even though you only wanted useful_function()
```

**The `if __name__ == "__main__"` guard:**

```python
# safe_module.py
def main():
    print("Running as main script")
    run_some_expensive_setup()

def useful_function():
    return "hello"

if __name__ == "__main__":
    # This block only runs when the file is executed directly:
    # $ python safe_module.py  → __name__ is "__main__", block runs
    # import safe_module       → __name__ is "safe_module", block doesn't run
    main()
```

**Django's `manage.py` uses this:**

```python
# manage.py
if __name__ == "__main__":
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
    from django.core.management import execute_from_command_line
    execute_from_command_line(sys.argv)
```

Running `python manage.py runserver` triggers the block. Importing `manage` from another module doesn't.

---

## 7. Virtual Environments — Isolation Is Not Optional

Every Python project should have its own virtual environment. This is not a suggestion.

**The problem without venvs:**

```
Project A needs Django 3.2
Project B needs Django 4.2
Your system Python can only have ONE version of Django installed
→ One of these projects will always be broken
```

**Creating and using a virtual environment:**

```bash
# Create a venv in your project directory
python -m venv venv

# Activate it (Linux/Mac)
source venv/bin/activate

# Activate it (Windows)
venv\Scripts\activate

# Your prompt changes to show (venv) — you're now isolated
(venv) $ pip install django

# Deactivate when done
deactivate
```

**What actually happens:**

When you activate a venv, it:
1. Prepends `venv/bin/` (or `venv/Scripts/`) to your `PATH`
2. So `python` and `pip` now point to the venv's copies
3. `sys.path` points to `venv/lib/pythonX.Y/site-packages/`
4. Packages installed with `pip` go into `venv/` — NOT the system Python

```bash
# Before activation:
which python    # /usr/bin/python  (system)
which pip       # /usr/bin/pip     (system)

# After activation:
which python    # /your/project/venv/bin/python
which pip       # /your/project/venv/bin/pip
```

### 7.1 `requirements.txt` — Recording What You Need

```bash
# After installing your packages:
pip freeze > requirements.txt   # write all installed packages and versions

# Someone else sets up the project:
pip install -r requirements.txt  # install everything at once
```

```
# requirements.txt looks like:
Django==4.2.7
Pillow==10.0.1
psycopg2-binary==2.9.7
python-decouple==3.8
```

**Pin exact versions in `requirements.txt`.** "I'll just write `Django>=4.0`" sounds flexible but six months later `pip install -r requirements.txt` might install Django 5.x which breaks your code.

### 7.2 `requirements.txt` Variations — A Real Project Pattern

```
requirements/
├── base.txt       ← packages needed everywhere
├── development.txt ← extra tools for dev (debugger, test tools)
└── production.txt  ← production-only (gunicorn, sentry-sdk)
```

```
# base.txt
Django==4.2.7
psycopg2-binary==2.9.7

# development.txt
-r base.txt           # include everything from base.txt
django-debug-toolbar==4.2.0
pytest-django==4.7.0

# production.txt
-r base.txt
gunicorn==21.2.0
sentry-sdk==1.32.0
```

```bash
pip install -r requirements/development.txt
```

---

## 8. `pip` — Package Management

```bash
# Install latest version
pip install django

# Install specific version
pip install django==4.2.7

# Install minimum version
pip install "django>=4.0,<5.0"

# Upgrade a package
pip install --upgrade django

# Uninstall
pip uninstall django

# Show installed packages
pip list

# Show info about a specific package
pip show django
# Name: Django
# Version: 4.2.7
# Location: /venv/lib/python3.11/site-packages
# Requires: asgiref, sqlparse

# Search PyPI (deprecated in newer pip — use pypi.org instead)
pip search django   # might not work
```

**Never `pip install` outside a virtual environment for project work.** You'll pollute your system Python and create version conflicts.

---

## 9. A Django Project's Import Architecture

Let's walk through a real Django project's import structure so everything clicks:

```
djangoblog/           ← project root (on sys.path)
├── manage.py
├── djangoblog/       ← project package
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py       ← root URLconf
│   └── wsgi.py
└── blog/             ← app package
    ├── __init__.py
    ├── models.py
    ├── views.py
    ├── urls.py
    ├── forms.py
    ├── admin.py
    ├── signals.py
    └── tests/
        ├── __init__.py
        ├── test_models.py
        └── test_views.py
```

```python
# blog/models.py
from django.db import models               # django package → db subpackage → models module
from django.contrib.auth.models import User  # django → contrib → auth → models → User

class Article(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    content = models.TextField()

    def __str__(self):
        return self.title
```

```python
# blog/views.py
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required

from .models import Article    # relative: same package (blog)
from .forms import ArticleForm  # relative: same package (blog)

@login_required
def article_create(request):
    if request.method == "POST":
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.author = request.user
            article.save()
            return redirect("article-list")
    else:
        form = ArticleForm()
    return render(request, "blog/article_form.html", {"form": form})
```

```python
# blog/admin.py
from django.contrib import admin
from .models import Article    # relative import within app

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "created_at"]
    search_fields = ["title", "content"]
```

```python
# djangoblog/urls.py (root URLconf)
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("blog/", include("blog.urls")),   # string form — Django imports this lazily
]
```

---

## 10. Common Import Patterns in Django

### 10.1 Lazy Imports — Avoiding Circular Imports

Circular imports happen when A imports B and B imports A. The solution: import inside the function, not at the top level.

```python
# signals.py — signals often cause circular imports
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender="blog.Article")   # use string reference, not direct import
def on_article_saved(sender, instance, created, **kwargs):
    if created:
        # Import here to avoid circular import
        from blog.tasks import send_notification_email
        send_notification_email.delay(instance.id)
```

### 10.2 Django's `apps.get_model()` — Runtime Model Resolution

```python
# When you can't import the model directly due to circular imports:
from django.apps import apps

def get_article_count():
    Article = apps.get_model("blog", "Article")  # get the model class at runtime
    return Article.objects.count()
```

### 10.3 `TYPE_CHECKING` — Imports Only for Type Hints

```python
from __future__ import annotations    # makes all annotations lazy strings
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    # These imports only run when a type checker (mypy) analyzes the code
    # They are NOT executed at runtime, so they can't cause circular imports
    from blog.models import Article

def process(article: "Article") -> None:
    ...
```

---

## 11. Writing Reusable Apps — Package Design

A well-structured Django app is a self-contained package:

```
blog/
├── __init__.py
├── apps.py              ← AppConfig — Django's app registry entry point
├── models.py
├── views.py
├── urls.py
├── forms.py
├── admin.py
├── signals.py
├── managers.py          ← custom model managers
├── mixins.py            ← reusable view/form mixins
├── utils.py             ← helper functions
├── constants.py         ← app-level constants
├── exceptions.py        ← custom exceptions
├── serializers.py       ← DRF serializers if using API
├── permissions.py       ← custom DRF permissions
├── tasks.py             ← Celery tasks
├── tests/
│   ├── __init__.py
│   ├── factories.py     ← test data factories
│   ├── test_models.py
│   ├── test_views.py
│   ├── test_forms.py
│   └── test_api.py
├── migrations/
│   ├── __init__.py
│   └── 0001_initial.py
├── templates/
│   └── blog/            ← namespaced templates
│       ├── article_list.html
│       └── article_detail.html
└── static/
    └── blog/            ← namespaced static files
        ├── css/
        └── js/
```

```python
# blog/apps.py — the AppConfig
from django.apps import AppConfig

class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"
    verbose_name = "Blog"

    def ready(self):
        """Called when Django starts. Import signals here."""
        import blog.signals   # noqa: F401
```

```python
# blog/__init__.py
default_app_config = "blog.apps.BlogConfig"
# In Django 3.2+, this is auto-detected if AppConfig sets name correctly
```

---

## ⚠️ Common Mistakes

**Mistake 1: Naming your file the same as a standard library module**
```python
# DON'T create a file called email.py, datetime.py, string.py, json.py, etc.
# It will shadow the standard library module:
# import email  →  imports YOUR email.py, not Python's email module
```

**Mistake 2: Importing at module level what should be imported lazily**
```python
# models.py
from myapp.signals import setup_signals  # circular import!
setup_signals()

# Fix: use AppConfig.ready()
# apps.py
class MyAppConfig(AppConfig):
    def ready(self):
        from myapp import signals  # safe here — all apps are loaded
```

**Mistake 3: Forgetting to activate the virtual environment**
```bash
# Symptom: ModuleNotFoundError: No module named 'django'
# Diagnosis: you're using system Python, not the venv
which python   # should show venv path
# Fix: source venv/bin/activate
```

**Mistake 4: Modifying `sys.path` manually in Django**
```python
# Don't do this in Django code:
import sys
sys.path.append("/some/path")

# Django's manage.py handles sys.path. If you need a package, install it properly.
```

**Mistake 5: Wildcard imports in non-`__init__.py` files**
```python
from django.db.models import *   # pollutes namespace, hides where things come from
# Fine for __init__.py re-exports; never for application code
```

---

## 🧪 Exercises

**Exercise 1:** Create a small package called `validators/` with:
- `__init__.py` that re-exports `validate_email`, `validate_phone`, `validate_url`
- `string_validators.py` with `validate_email` and `validate_url` functions
- `number_validators.py` with `validate_phone` function

Import from the package using both absolute style (`from validators import validate_email`) and by making `__init__.py` expose only the public API.

**Exercise 2:** Investigate a circular import. Create two files:
- `module_a.py`: imports something from `module_b`
- `module_b.py`: imports something from `module_a`

Observe the error. Then fix it using a lazy import inside a function.

**Exercise 3:** Set up a Python project from scratch:
1. Create a project folder
2. Create a virtual environment inside it
3. Activate it
4. Install `django` and `requests`
5. Run `pip freeze > requirements.txt`
6. Deactivate
7. Create a new venv in a different folder
8. `pip install -r requirements.txt` in the new venv
9. Verify both venvs have the same packages but are completely separate

**Exercise 4:** Open an existing Django project (or create a fresh one with `django-admin startproject`). Trace the import chain for this line in a view file:
```python
from django.contrib.auth.models import User
```
Find the actual file on disk where `User` is defined inside your venv's `site-packages`. What does `django/contrib/auth/__init__.py` contain? What does `django/contrib/auth/models.py` import from?

---

## ✅ Before Moving On

- [ ] You can explain what `sys.path` is and why Django manipulates it
- [ ] You know what `__init__.py` does and how to use it as a public API
- [ ] You understand the difference between absolute and relative imports
- [ ] You know why `if __name__ == "__main__":` exists and when to use it
- [ ] You always use a virtual environment for Python projects
- [ ] You know how to create, activate, and reproduce a virtual environment
- [ ] You understand what causes circular imports and how to fix them
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

The entire Django project is a package. Every app you create is a package. When Django starts up, it walks through `INSTALLED_APPS`, imports each app's `AppConfig`, calls `ready()`, imports models for the ORM, imports URL patterns for routing, and imports middleware classes.

All of that is Python's module system at work. When you get an `ImportError` in Django, you now know exactly where to look: `sys.path`, circular dependencies, missing `__init__.py`, or an activated virtualenv.

**Next:** [07 — File Handling: Reading, Writing, and the Context Manager Pattern](./07_file_handling.md)
