# 01 — Python Refresher: Variables, Types & How Python Thinks

> **Before you read this:** This is not a "hello world" tutorial. If you've never written Python before, go spend a week with the official Python tutorial first, then come back. This lesson assumes you've *seen* Python but may be fuzzy on what's actually happening under the hood — which matters enormously once Django starts doing things you didn't ask for.

---

## Why This Lesson Exists

Django is opinionated. It makes hundreds of tiny decisions for you — how data gets stored, how forms validate, how URLs map to code. If you don't have a firm grip on how Python itself thinks about data, you will spend weeks debugging things that aren't bugs. You'll fight the framework instead of using it.

This lesson is about building the mental model, not memorizing syntax.

---

## 1. Everything in Python Is an Object

This is the single most important thing to understand about Python. Not "variables store values." Variables are **labels** that point to objects. Objects live in memory. Multiple labels can point to the same object.

```python
x = [1, 2, 3]
y = x          # y is NOT a copy. y is another label for the SAME list.

y.append(4)
print(x)       # [1, 2, 3, 4]  ← x changed because x and y point to the same object
```

This will burn you in Django when you pass a list or dict into a function expecting it to be unchanged afterward. Always know whether you're working with the original or a copy.

```python
# To actually copy:
y = x.copy()        # shallow copy — nested objects still shared
y = x[:]            # same as above
import copy
y = copy.deepcopy(x)  # deep copy — completely independent
```

**Check if two names point to the same object:**
```python
x is y    # True if same object in memory (identity)
x == y    # True if same value (equality)
```

Django's ORM will return queryset objects, not raw lists. Understanding that these are objects with their own behavior — not simple arrays — is crucial.

---

## 2. Python's Built-in Types — The Ones That Actually Matter

### 2.1 Integers and Floats

```python
age = 25          # int
price = 19.99     # float
big = 10_000_000  # underscores for readability — valid Python

# Integer division vs float division
7 / 2    # → 3.5   (always returns float)
7 // 2   # → 3     (floor division, drops the decimal)
7 % 2    # → 1     (modulo — remainder)
2 ** 10  # → 1024  (exponentiation)
```

**Django relevance:** Your model fields have types — `IntegerField`, `FloatField`, `DecimalField`. Django will validate these. If you try to store a string in an IntegerField, Django will raise a `ValidationError`. Know your types.

### 2.2 Strings

Strings in Python are **immutable** — you cannot change a string in place. Every operation that looks like it modifies a string actually creates a new one.

```python
name = "django"
name.upper()     # returns "DJANGO" — original 'name' is unchanged
name = name.upper()  # now name points to the new string "DJANGO"
```

**The string methods you'll use constantly in Django:**

```python
s = "  Hello, Django World!  "

s.strip()              # "Hello, Django World!"   ← removes whitespace from both ends
s.lower()              # "  hello, django world!  "
s.upper()              # "  HELLO, DJANGO WORLD!  "
s.replace("Django", "Python")  # "  Hello, Python World!  "
s.split(", ")          # ["  Hello", "Django World!  "]
s.startswith("  Hel")  # True
s.endswith("!  ")      # True
"Django" in s          # True  ← substring check

# String formatting — use f-strings, always
user = "Anurag"
greeting = f"Hello, {user}! You have {2 + 2} messages."
# "Hello, Anurag! You have 4 messages."

# Multiline strings
sql_like = """
    SELECT *
    FROM users
    WHERE active = true
"""
```

**Django uses strings everywhere** — URL patterns, template names, field names, error messages. f-strings were introduced in Python 3.6. Django 4+ requires Python 3.10+. Use f-strings, not `.format()` or `%` formatting.

### 2.3 Booleans

```python
is_active = True
is_deleted = False

# These are all falsy in Python — they evaluate to False in an if statement:
# False, None, 0, 0.0, "", [], {}, set(), ()

# These are all truthy:
# True, any non-zero number, any non-empty string/list/dict/set

# This is a real Django pattern:
users = User.objects.filter(is_active=True)
if users:          # True if queryset has at least one result
    print("Found users")
```

**Critical:** `None` is Python's "nothing." It is not `0`, not `""`, not `False`. It is the absence of a value. In Django, database fields that allow null will return `None` when empty.

```python
x = None
x is None      # True  ← correct way to check for None
x == None      # True  ← works but not idiomatic
if not x:      # True  ← also works, but catches 0 and "" too, so be careful
```

### 2.4 Lists

Ordered, mutable, allows duplicates.

```python
fruits = ["apple", "banana", "cherry"]

# Indexing
fruits[0]    # "apple"
fruits[-1]   # "cherry"  ← negative indexing from the end

# Slicing — [start:stop:step] — stop is exclusive
fruits[0:2]  # ["apple", "banana"]  — index 0 and 1, NOT 2
fruits[1:]   # ["banana", "cherry"]
fruits[::-1] # ["cherry", "banana", "apple"]  — reversed

# Modifying
fruits.append("date")           # add to end
fruits.insert(1, "avocado")     # insert at index 1
fruits.remove("banana")         # remove first occurrence
popped = fruits.pop()           # remove and return last item
popped = fruits.pop(0)          # remove and return item at index 0
fruits.sort()                   # in-place sort (modifies original)
sorted_fruits = sorted(fruits)  # returns new sorted list

# Length
len(fruits)   # number of items
```

### 2.5 Tuples

Like lists, but **immutable**. Once created, cannot be changed.

```python
coordinates = (40.7128, -74.0060)  # latitude, longitude
coordinates[0]   # 40.7128
# coordinates[0] = 999  # ← TypeError! Tuples are immutable

# Tuples are faster than lists and signal "this should not change"
# Django uses tuples in settings, like INSTALLED_APPS (though lists work too)
# URL patterns used to be tuples — now they're function calls

# Unpacking
lat, lng = coordinates
print(lat)   # 40.7128

x, *rest = (1, 2, 3, 4, 5)
print(x)     # 1
print(rest)  # [2, 3, 4, 5]
```

### 2.6 Dictionaries

Key-value pairs. Keys must be hashable (strings, numbers, tuples). **Ordered since Python 3.7.**

```python
user = {
    "name": "Anurag",
    "age": 28,
    "is_active": True,
    "scores": [95, 87, 92]
}

# Accessing
user["name"]              # "Anurag"
user.get("email")         # None — safe access, no KeyError
user.get("email", "N/A")  # "N/A" — with default value

# Modifying
user["email"] = "anurag@example.com"   # add or update
del user["age"]                         # delete key

# Checking
"name" in user    # True
"age" in user     # False (we deleted it)

# Iterating
for key in user:                  # iterates over keys
    print(key)

for key, value in user.items():   # iterates over key-value pairs
    print(f"{key}: {value}")

for value in user.values():       # just values
    print(value)

# Merging (Python 3.9+)
defaults = {"color": "blue", "size": "medium"}
custom = {"color": "red"}
merged = defaults | custom   # {"color": "red", "size": "medium"}
```

**Django uses dicts constantly.** The `request.GET`, `request.POST`, and `request.session` objects all behave like dictionaries. Model `__dict__` is a dictionary. Context passed to templates is a dictionary.

### 2.7 Sets

Unordered collection of **unique** items. No duplicates, no indexing.

```python
tags = {"python", "django", "web"}
tags.add("api")
tags.remove("web")   # KeyError if not found
tags.discard("web")  # safe — no error if not found

# Set operations
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
a | b    # {1, 2, 3, 4, 5, 6}  — union
a & b    # {3, 4}               — intersection
a - b    # {1, 2}               — difference
a ^ b    # {1, 2, 5, 6}         — symmetric difference

# Most common use: remove duplicates from a list
items = [1, 2, 2, 3, 3, 3]
unique = list(set(items))   # [1, 2, 3]  — order not guaranteed
```

---

## 3. Type Conversion

```python
int("42")       # 42
int("42.9")     # ValueError — int() doesn't handle decimal strings
int(42.9)       # 42  — truncates (does NOT round)
float("3.14")   # 3.14
str(100)        # "100"
bool(0)         # False
bool("hello")   # True
list("abc")     # ["a", "b", "c"]
list((1,2,3))   # [1, 2, 3]
tuple([1,2,3])  # (1, 2, 3)
```

**Django converts types when saving to the database.** If you pass a string "42" to an IntegerField, Django (or the database) will try to convert it. But this doesn't always work silently — sometimes you get validation errors. Knowing Python's type conversion rules saves hours of debugging.

---

## 4. Variable Naming — The Conventions That Matter

```python
# snake_case for variables and functions (Django style)
user_name = "anurag"
is_authenticated = True

# UPPER_CASE for constants
MAX_LOGIN_ATTEMPTS = 5
SECRET_KEY = "your-secret-key-here"   # you'll see this in Django settings

# Leading underscore = "this is internal, don't touch from outside"
_internal_cache = {}

# Double leading underscore = name mangling in classes (covered in OOP lesson)
__private_attr = "truly private"

# Trailing underscore = avoid conflict with Python keywords
class_ = "grade A"   # 'class' is a Python keyword
type_ = "admin"      # 'type' is a built-in function

# Don't do this — shadows Python built-ins
list = [1, 2, 3]    # ← Now 'list' doesn't work as a function anymore!
id = 42              # ← Now 'id()' built-in is broken in this scope!
```

---

## 5. How Python Resolves Names: LEGB

When Python sees a name (variable, function), it searches in this order:

1. **L**ocal — inside the current function
2. **E**nclosing — in any enclosing functions (for nested functions)
3. **G**lobal — at the module (file) level
4. **B**uilt-in — Python's built-in names (`len`, `print`, `list`, etc.)

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        # x = "local"  ← if this exists, it's used
        print(x)       # prints "enclosing" because no local x

    inner()

outer()
```

**Django relevance:** You'll see `global` and `nonlocal` occasionally in Django source code. More importantly, understanding scope prevents bugs where you accidentally shadow a variable name.

---

## 6. The `None` Trap — A Real Django Bug

This is so common it deserves its own section:

```python
def get_user_profile(user_id):
    # Imagine this queries the database
    if user_id == 1:
        return {"name": "Anurag"}
    # If user_id is not 1, function falls through and implicitly returns None

profile = get_user_profile(99)
print(profile["name"])   # TypeError: 'NoneType' object is not subscriptable
```

Always handle the `None` case:

```python
profile = get_user_profile(99)
if profile is not None:
    print(profile["name"])

# Or use .get() on dicts
name = (profile or {}).get("name", "Unknown")
```

In Django, `Model.objects.filter()` never returns `None` — it returns an empty queryset. But `Model.objects.get()` raises `DoesNotExist`. And `first()` returns `None` if nothing found. Know which one you're using.

---

## 7. Comprehensions — Write Less, Think More

List comprehensions are not just "shorter loops." They're a fundamentally different way of thinking — transforming a collection into another collection.

```python
# Traditional loop
squares = []
for i in range(10):
    squares.append(i ** 2)

# List comprehension
squares = [i ** 2 for i in range(10)]

# With a condition (filter)
even_squares = [i ** 2 for i in range(10) if i % 2 == 0]

# Dict comprehension
word_lengths = {word: len(word) for word in ["python", "django", "web"]}
# {"python": 6, "django": 6, "web": 3}

# Set comprehension
unique_lengths = {len(word) for word in ["python", "django", "web"]}
# {6, 3}

# Generator expression — like a list comprehension but lazy (doesn't create list in memory)
total = sum(i ** 2 for i in range(1000000))  # no giant list created
```

**Django relevance:** You'll use comprehensions constantly to transform querysets into lists, dicts, or other structures for templates and APIs.

```python
# Real Django pattern
user_emails = [user.email for user in User.objects.filter(is_active=True)]

# Dict from queryset
user_map = {user.id: user.username for user in User.objects.all()}
```

---

## ⚠️ Common Mistakes at This Stage

**Mistake 1: Mutable default arguments**
```python
# BUG — the list is created ONCE and shared across all calls
def add_item(item, items=[]):
    items.append(item)
    return items

add_item("a")  # ["a"]
add_item("b")  # ["a", "b"]  ← bug! expected ["b"]

# Fix
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

**Mistake 2: Comparing with `==` when you mean `is`**
```python
x = None
if x == None:    # works, but wrong style
if x is None:    # correct — None is a singleton
```

**Mistake 3: Modifying a list while iterating over it**
```python
items = [1, 2, 3, 4, 5]
for item in items:
    if item % 2 == 0:
        items.remove(item)   # BUG — skips items

# Fix: iterate over a copy, or use a comprehension
items = [item for item in items if item % 2 != 0]
```

**Mistake 4: String concatenation in loops**
```python
# Slow — creates a new string object every iteration
result = ""
for word in words:
    result += word   # O(n²) — don't do this

# Fast
result = "".join(words)  # O(n) — always use join
```

---

## 🧪 Exercises — Do These, Don't Skip Them

**Exercise 1:** Create a dictionary representing a blog post with keys: `title`, `author`, `tags` (a list), `published` (bool), `views` (int). Write code to add a new tag, increment views by 1, and print a formatted summary using an f-string.

**Exercise 2:** Given a list of numbers `[3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]`, use a comprehension to create a new list of only the unique values greater than 3, sorted in descending order.

**Exercise 3:** Write a function that takes a list of user dictionaries (each with `name` and `is_active` keys) and returns a dictionary mapping each active user's name to their uppercase version. Handle the case where the list might be empty.

**Exercise 4:** Explain in plain English what happens in memory when you do:
```python
a = {"key": [1, 2, 3]}
b = a.copy()
b["key"].append(4)
print(a)  # What does this print and why?
```

---

## ✅ Before Moving On — Check These

- [ ] You understand the difference between `is` and `==`
- [ ] You can explain why `y = x` for a list doesn't create a copy
- [ ] You know which types are mutable (list, dict, set) and which are not (int, str, tuple, bool, None)
- [ ] You can write list, dict, and set comprehensions without looking them up
- [ ] You know what `None` is and how to safely check for it
- [ ] You understand why string concatenation in a loop is a bad idea

---

## 🔗 What This Has to Do With Django

You will use everything in this file before you finish your first Django view. The `request.POST` object is a dict. Template context is a dict. Model instances are objects. Querysets are iterable objects. Django settings are a module full of constants. Form cleaned data is a dict.

None of this is abstract. It's all coming.

**Next:** [02 — Control Flow: Making Decisions and Repeating Work](./02_control_flow.md)
