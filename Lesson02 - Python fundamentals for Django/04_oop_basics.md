# 04 — OOP Basics: Classes, Objects, and Why Django Speaks Object

> **Django is object-oriented to its core.** Every model is a class. Every view can be a class. Every form is a class. Every middleware is a class. If OOP feels abstract or theoretical to you right now, this lesson will fix that — because we're going to build a mental model that maps directly onto how Django thinks.

---

## 1. Why Object-Oriented Programming Exists

Before classes, programs were just functions and data floating around separately. As programs grew, it became hard to keep track of what data a function was supposed to work with.

OOP says: **bundle the data (attributes) and the functions that work on that data (methods) together into one thing — an object.**

```python
# Procedural approach — data and functions separate
user_name = "Anurag"
user_email = "anurag@example.com"
user_is_active = True

def get_user_display(name, email, is_active):
    status = "active" if is_active else "inactive"
    return f"{name} <{email}> [{status}]"

get_user_display(user_name, user_email, user_is_active)

# OOP approach — data and behavior bundled
class User:
    def __init__(self, name, email, is_active=True):
        self.name = name
        self.email = email
        self.is_active = is_active

    def get_display(self):
        status = "active" if self.is_active else "inactive"
        return f"{self.name} <{self.email}> [{status}]"

user = User("Anurag", "anurag@example.com")
user.get_display()
```

The OOP version is more code for this tiny example, but scales much better. When you have 100 users, the `User` class handles them all without needing to pass 3 separate variables to every function.

---

## 2. Classes and Instances — The Blueprint vs. The Building

A **class** is a blueprint. An **instance** (also called an object) is a specific thing created from that blueprint.

```python
class Car:             # the blueprint
    pass

my_car = Car()         # an instance of Car
your_car = Car()       # another instance, completely separate from my_car

print(type(my_car))    # <class '__main__.Car'>
print(isinstance(my_car, Car))   # True
```

**Django equivalent:**

```python
class Article(models.Model):   # the blueprint (Django model)
    title = models.CharField(max_length=200)
    content = models.TextField()

# Each row in the database becomes an instance:
article1 = Article(title="Python Tips", content="...")
article2 = Article(title="Django ORM", content="...")
```

Every row in your `Article` table is an instance of the `Article` class. When you do `Article.objects.get(id=1)`, Django constructs an instance and hands it to you.

---

## 3. `__init__` — The Constructor

`__init__` runs automatically when you create an instance. It's where you set up the object's initial state.

```python
class BlogPost:
    def __init__(self, title, author, content=""):
        # self.attribute = value — creates instance attributes
        self.title = title
        self.author = author
        self.content = content
        self.is_published = False   # always starts unpublished
        self.view_count = 0         # always starts at 0
        self._internal_id = id(self)  # underscore = internal, not for outside use

post = BlogPost("Learning Django", "Anurag")
print(post.title)         # "Learning Django"
print(post.is_published)  # False
print(post.view_count)    # 0
```

**`self` is the instance itself.** When you write `self.title = title`, you're saying "store `title` as an attribute on THIS specific instance, not shared across all posts."

This is not optional. Every method in a class must have `self` as its first parameter. Python passes it automatically — you never explicitly pass it when calling.

```python
post.get_summary()    # Python translates to: BlogPost.get_summary(post)
```

---

## 4. Instance Attributes vs. Class Attributes

**Instance attributes:** Belong to one specific instance. Defined in `__init__` (or anywhere with `self.attr = value`).

**Class attributes:** Belong to the class itself. Shared across ALL instances.

```python
class Article:
    # Class attributes — shared by ALL instances
    site_name = "My Blog"
    total_count = 0

    def __init__(self, title, author):
        # Instance attributes — unique to this instance
        self.title = title
        self.author = author
        Article.total_count += 1   # increment shared counter

a1 = Article("Post 1", "Alice")
a2 = Article("Post 2", "Bob")

print(a1.site_name)        # "My Blog"  — class attribute
print(a2.site_name)        # "My Blog"  — same value, same attribute
print(Article.total_count) # 2          — both instances incremented it
print(a1.total_count)      # 2          — accessible from instance too
print(a1.title)            # "Post 1"   — instance attribute (unique to a1)
print(a2.title)            # "Post 2"   — instance attribute (unique to a2)
```

**The shadow trap:**

```python
class Config:
    debug = False   # class attribute

c = Config()
c.debug = True       # this creates an INSTANCE attribute on c, doesn't change class
print(Config.debug)  # False — class attribute unchanged
print(c.debug)       # True  — instance attribute shadows the class attribute
```

**Django relevance:** Django model fields are class attributes. When you write:

```python
class User(models.Model):
    username = models.CharField(max_length=150)
    email = models.EmailField()
```

`username` and `email` are class attributes that describe the *structure* of the data. When you access `user.username` on an instance, Django's magic returns the actual *value* for that row.

---

## 5. Methods — Three Kinds

### 5.1 Instance Methods (the common ones)

Take `self` as the first parameter. Operate on instance data.

```python
class BlogPost:
    def __init__(self, title, content):
        self.title = title
        self.content = content
        self.is_published = False

    def publish(self):
        """Publish this post."""
        self.is_published = True

    def word_count(self):
        """Return number of words in content."""
        return len(self.content.split())

    def get_summary(self, length=100):
        """Return first `length` characters of content."""
        return self.content[:length] + "..." if len(self.content) > length else self.content

post = BlogPost("Django Guide", "Django is a web framework...")
post.publish()
print(post.is_published)   # True
print(post.word_count())   # 5
print(post.get_summary(10))# "Django is ..."
```

### 5.2 Class Methods (`@classmethod`)

Take `cls` (the class itself) as the first parameter. Used for alternative constructors or operations that don't need instance data.

```python
class BlogPost:
    def __init__(self, title, content, author):
        self.title = title
        self.content = content
        self.author = author

    @classmethod
    def from_dict(cls, data):
        """Create a BlogPost from a dictionary."""
        return cls(
            title=data["title"],
            content=data.get("content", ""),
            author=data["author"]
        )

    @classmethod
    def create_draft(cls, title, author):
        """Create an empty draft post."""
        return cls(title=title, content="", author=author)

# Using alternate constructors:
data = {"title": "My Post", "author": "Anurag", "content": "Hello..."}
post = BlogPost.from_dict(data)
draft = BlogPost.create_draft("Untitled", "Anurag")
```

**Django uses `@classmethod` as `@classmethod` in forms and in `Model.from_db()`:**

```python
class ArticleForm(forms.ModelForm):
    @classmethod
    def for_user(cls, user, *args, **kwargs):
        form = cls(*args, **kwargs)
        form.fields["category"].queryset = Category.objects.filter(owner=user)
        return form
```

### 5.3 Static Methods (`@staticmethod`)

No `self` or `cls`. Just a regular function that happens to live inside a class for organizational reasons.

```python
class PasswordValidator:
    MIN_LENGTH = 8

    @staticmethod
    def has_uppercase(password):
        return any(c.isupper() for c in password)

    @staticmethod
    def has_digit(password):
        return any(c.isdigit() for c in password)

    @staticmethod
    def is_valid(password):
        if len(password) < PasswordValidator.MIN_LENGTH:
            return False
        return PasswordValidator.has_uppercase(password) and PasswordValidator.has_digit(password)

PasswordValidator.is_valid("Secret123")   # True
PasswordValidator.is_valid("short")       # False
```

**When to use which:**
- Use instance methods when you need `self` (access instance data)
- Use `@classmethod` when you need the class itself (alternative constructors, factory methods)
- Use `@staticmethod` when the method doesn't need either — it's just logically grouped with the class

---

## 6. Properties — Computed Attributes with Validation

`@property` turns a method into an attribute-like access. You get computed values without calling a method explicitly.

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius   # store the raw value privately

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError(f"Temperature below absolute zero: {value}")
        self._celsius = value

    @property
    def fahrenheit(self):
        """Computed from celsius — no setter needed, it's read-only."""
        return (self._celsius * 9/5) + 32

    @property
    def kelvin(self):
        return self._celsius + 273.15

temp = Temperature(100)
print(temp.celsius)      # 100    — looks like attribute access, calls the getter
print(temp.fahrenheit)   # 212.0  — computed on the fly
print(temp.kelvin)       # 373.15

temp.celsius = 37        # calls the setter, validates
temp.celsius = -300      # ValueError!

temp.fahrenheit = 100    # AttributeError — no setter defined
```

**Django uses `@property` constantly in models:**

```python
class User(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    date_of_birth = models.DateField()

    @property
    def full_name(self):
        return f"{self.first_name} {self.last_name}"

    @property
    def age(self):
        from datetime import date
        today = date.today()
        return today.year - self.date_of_birth.year - (
            (today.month, today.day) < (self.date_of_birth.month, self.date_of_birth.day)
        )

user = User(first_name="Anurag", last_name="Singh", date_of_birth=date(1996, 3, 15))
print(user.full_name)   # "Anurag Singh"  — no () needed, it's a property
print(user.age)         # computed dynamically
```

In templates, you access properties the same way as regular fields:
```html
{{ user.full_name }}   {# works because full_name is a @property #}
{{ user.age }}         {# also works #}
```

---

## 7. Dunder Methods — Python's Object Protocol

"Dunder" = "double underscore." These special methods let your objects work with Python's built-in operators and functions.

### 7.1 The Essential Dunders

```python
class BlogPost:
    def __init__(self, title, author, views=0):
        self.title = title
        self.author = author
        self.views = views

    def __str__(self):
        """Called by str() and print()."""
        return f"{self.title} by {self.author}"

    def __repr__(self):
        """Called by repr() and in interactive shell — should show how to recreate object."""
        return f"BlogPost(title={self.title!r}, author={self.author!r}, views={self.views})"

    def __len__(self):
        """Called by len()."""
        return len(self.title)

    def __eq__(self, other):
        """Called by ==."""
        if not isinstance(other, BlogPost):
            return NotImplemented
        return self.title == other.title and self.author == other.author

    def __lt__(self, other):
        """Called by < — enables sorting."""
        return self.views < other.views

    def __contains__(self, item):
        """Called by 'in' operator."""
        return item.lower() in self.title.lower()

post = BlogPost("Django Guide", "Anurag", views=1500)
print(post)                     # "Django Guide by Anurag"  — uses __str__
repr(post)                      # "BlogPost(title='Django Guide', author='Anurag', views=1500)"
len(post)                       # 12  (length of "Django Guide")
post == BlogPost("Django Guide", "Anurag")  # True — uses __eq__
"Django" in post                # True — uses __contains__
sorted([post1, post2, post3])   # sorted by views — uses __lt__
```

**`__str__` in Django is critical:**

```python
class Article(models.Model):
    title = models.CharField(max_length=200)

    def __str__(self):
        return self.title

# Without __str__, the Django admin shows: "Article object (1)"
# With __str__, it shows: "Django Fundamentals for Beginners"
```

Always define `__str__` on your Django models. Always.

### 7.2 Container Protocol

```python
class TagCollection:
    def __init__(self):
        self._tags = []

    def __len__(self):
        return len(self._tags)

    def __getitem__(self, index):
        """Called by collection[index]."""
        return self._tags[index]

    def __setitem__(self, index, value):
        """Called by collection[index] = value."""
        self._tags[index] = value

    def __delitem__(self, index):
        """Called by del collection[index]."""
        del self._tags[index]

    def __iter__(self):
        """Called by for loops — must return an iterator."""
        return iter(self._tags)

    def __contains__(self, item):
        """Called by 'in' operator."""
        return item in self._tags

tags = TagCollection()
tags._tags = ["python", "django", "web"]

for tag in tags:     # uses __iter__
    print(tag)

"python" in tags     # True — uses __contains__
tags[0]              # "python" — uses __getitem__
len(tags)            # 3 — uses __len__
```

### 7.3 Context Manager Protocol (`with` statement)

```python
class DatabaseConnection:
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.connection = None

    def __enter__(self):
        """Called when entering 'with' block. Returns the object to bind to 'as'."""
        print(f"Connecting to {self.connection_string}")
        self.connection = True   # simulate connection
        return self   # 'with DatabaseConnection(...) as db:' — db is self

    def __exit__(self, exc_type, exc_value, traceback):
        """Called when leaving 'with' block (even if exception occurred)."""
        print("Closing connection")
        self.connection = None
        return False   # False = don't suppress exceptions; True = suppress

with DatabaseConnection("postgresql://localhost/mydb") as db:
    print(f"Connected: {db.connection}")
    # connection is automatically closed when block exits

# Django's transaction.atomic() uses this exact pattern:
from django.db import transaction

with transaction.atomic():
    user.save()
    profile.save()
    # If any save fails, BOTH are rolled back automatically
```

---

## 8. Bringing It Together — A Real-ish Model

```python
from datetime import datetime

class Article:
    # Class attribute — shared across all articles
    _all_articles = []

    def __init__(self, title, author, content=""):
        if not title:
            raise ValueError("Article must have a title")
        if not author:
            raise ValueError("Article must have an author")

        self.title = title
        self.author = author
        self.content = content
        self._is_published = False
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
        self._tags = []

        Article._all_articles.append(self)

    # Properties
    @property
    def is_published(self):
        return self._is_published

    @property
    def word_count(self):
        return len(self.content.split()) if self.content else 0

    @property
    def tags(self):
        return tuple(self._tags)  # return immutable copy

    @tags.setter
    def tags(self, new_tags):
        self._tags = list(new_tags)
        self.updated_at = datetime.now()

    # Instance methods
    def publish(self):
        if not self.content:
            raise ValueError("Cannot publish an article without content")
        self._is_published = True
        self.updated_at = datetime.now()

    def add_tag(self, tag):
        tag = tag.strip().lower()
        if tag and tag not in self._tags:
            self._tags.append(tag)
            self.updated_at = datetime.now()

    # Class methods
    @classmethod
    def get_all(cls):
        return list(cls._all_articles)

    @classmethod
    def get_published(cls):
        return [a for a in cls._all_articles if a.is_published]

    # Static methods
    @staticmethod
    def is_valid_title(title):
        return bool(title and len(title.strip()) >= 3)

    # Dunders
    def __str__(self):
        return f"{self.title} by {self.author}"

    def __repr__(self):
        return f"Article(title={self.title!r}, author={self.author!r}, published={self._is_published})"

    def __eq__(self, other):
        if not isinstance(other, Article):
            return NotImplemented
        return self.title == other.title and self.author == other.author

    def __contains__(self, tag):
        return tag.lower() in self._tags

# Usage
article = Article("Django ORM Deep Dive", "Anurag", content="QuerySets are lazy...")
article.add_tag("Django")
article.add_tag("Python")
article.publish()

print(article)              # "Django ORM Deep Dive by Anurag"
print(article.word_count)  # 3
print("django" in article) # True
print(Article.get_published())  # [Article(...)]
```

---

## ⚠️ Common Mistakes

**Mistake 1: Forgetting `self`**
```python
class Counter:
    def __init__(self):
        count = 0   # BUG — this is a local variable, not an attribute!
        self.count = 0   # CORRECT

    def increment(self):
        count += 1   # BUG — can't access the instance's count without self
        self.count += 1   # CORRECT
```

**Mistake 2: Confusing class attributes and instance attributes with mutables**
```python
class BadList:
    items = []   # class attribute — SHARED across all instances!

a = BadList()
b = BadList()
a.items.append("hello")
print(b.items)   # ["hello"]  ← b's list was affected!

class GoodList:
    def __init__(self):
        self.items = []   # instance attribute — each instance gets its own list
```

**Mistake 3: Defining `__eq__` without `__hash__`**
```python
# When you define __eq__, Python sets __hash__ to None (making instances unhashable)
# This breaks using objects in sets or as dict keys
class Article:
    def __eq__(self, other):
        return self.title == other.title

article = Article()
{article: "value"}  # TypeError: unhashable type: 'Article'

# Fix: also define __hash__
def __hash__(self):
    return hash(self.title)
```

**Mistake 4: Not calling `super().__init__()` when inheriting (covered next lesson)**

---

## 🧪 Exercises

**Exercise 1:** Build a `ShoppingCart` class with:
- Items stored as a list of dicts with `name`, `price`, `quantity`
- `add_item(name, price, quantity=1)` method
- `remove_item(name)` method
- `total` as a `@property` that calculates the total price
- `item_count` as a `@property`
- `apply_discount(percent)` method
- `__str__` that shows item count and total
- `__len__` that returns number of unique items
- `__contains__` to check if an item name is in the cart

**Exercise 2:** Build a `Stack` class that:
- Stores items in a list
- Has `push(item)`, `pop()`, `peek()` methods
- Raises `IndexError` on pop/peek when empty
- Supports `len()`, `bool()` (empty stack is falsy), `in` operator
- Has a `@classmethod from_list(cls, items)` that creates a Stack from a list

**Exercise 3:** Add `@property` and validation to this class:
```python
class Product:
    def __init__(self, name, price, stock):
        self.name = name
        self.price = price  # must be positive float
        self.stock = stock  # must be non-negative integer
```
The `price` setter should raise `ValueError` if price ≤ 0. The `stock` setter should raise `ValueError` if stock < 0. Add a read-only `is_available` property (True if stock > 0).

---

## ✅ Before Moving On

- [ ] You can explain the difference between a class and an instance
- [ ] You understand why `self` is needed and what it refers to
- [ ] You know the difference between instance and class attributes
- [ ] You can write `@property`, `@classmethod`, and `@staticmethod` correctly
- [ ] You've written `__str__`, `__repr__`, `__eq__`, and `__len__` 
- [ ] You know what `__enter__` and `__exit__` are for
- [ ] You've completed all three exercises

---

## 🔗 What This Has to Do With Django

You now understand the machinery that powers Django's models. When you write:

```python
class Article(models.Model):
    title = models.CharField(max_length=200)

    def __str__(self):
        return self.title

    @property
    def summary(self):
        return self.content[:200]
```

You know exactly what's happening. `models.Model` is a class that `Article` inherits from (next lesson). `CharField` is an instance of a field class being used as a class attribute. `__str__` tells Django's admin how to display the object. `@property` makes `summary` accessible like an attribute in templates.

None of this is magic anymore.

**Next:** [05 — OOP Advanced: Inheritance, super(), and Dunder Methods](./05_oop_advanced.md)
