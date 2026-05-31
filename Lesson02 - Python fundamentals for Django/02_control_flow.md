# 02 — Control Flow: Making Decisions and Repeating Work

> **The difference between a beginner and an intermediate programmer** is not knowing more syntax — it's making better decisions about *which* control structure fits the situation, and why. By the end of this lesson, you should be asking "what's the cleanest way to express this logic?" not "how do I write an if statement?"

---

## 1. Conditionals — Beyond the Basics

You know `if/elif/else`. Let's talk about what actually happens and where it gets subtle.

### 1.1 Truthiness vs. Truth

Python doesn't require a boolean in an `if` condition. It evaluates *truthiness*:

```python
# All of these are FALSY — if block does NOT run:
if False: ...
if None: ...
if 0: ...
if 0.0: ...
if "": ...
if []: ...
if {}: ...
if set(): ...
if (): ...

# All of these are TRUTHY — if block runs:
if True: ...
if 1: ...
if -1: ...         # any non-zero number
if "anything": ... # any non-empty string
if [0]: ...        # a list containing one False-y item is still truthy
if {"key": None}: ...  # a dict with one key is truthy
```

**This is where subtle bugs happen:**

```python
def get_user_count():
    return 0

count = get_user_count()
if count:
    print("We have users!")    # WRONG — 0 users is falsy, but it's valid data
else:
    print("No users yet")

# Correct approach when 0 is meaningful:
if count is not None:
    print(f"We have {count} users")

# Or be explicit:
if count > 0:
    print("We have users!")
```

**Django relevance:** `if queryset:` is fine — an empty queryset is falsy. But `if user_count:` when count could legitimately be 0 is a bug.

### 1.2 Comparison Operators — The Ones That Surprise People

```python
# Chaining comparisons — Python allows this, unlike most languages
x = 5
if 0 < x < 10:   # equivalent to: if 0 < x and x < 10
    print("x is between 0 and 10")

# is vs ==
a = [1, 2, 3]
b = [1, 2, 3]
a == b     # True  — same values
a is b     # False — different objects in memory

# Small integers are cached in CPython (implementation detail, don't rely on it)
x = 256
y = 256
x is y    # True  — CPython caches integers -5 to 256
x = 257
y = 257
x is y    # False — outside the cache range

# Always use == for value comparison, is only for None, True, False
```

### 1.3 The Ternary (Conditional) Expression

```python
# Standard form
status = "active" if user.is_logged_in else "guest"

# Equivalent to:
if user.is_logged_in:
    status = "active"
else:
    status = "guest"

# Nested ternary — technically works, but avoid if it hurts readability
label = "admin" if is_admin else "moderator" if is_mod else "user"
# Better written as:
if is_admin:
    label = "admin"
elif is_mod:
    label = "moderator"
else:
    label = "user"
```

**Django uses ternary expressions in templates too (differently), so knowing Python's version matters.**

### 1.4 `match` Statement (Python 3.10+)

Django 4.x+ targets Python 3.10+. The `match` statement is Python's pattern matching — more powerful than a switch statement.

```python
def handle_http_status(status_code):
    match status_code:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 403 | 401:         # multiple values
            return "Auth Error"
        case code if code >= 500:  # guard condition
            return f"Server Error: {code}"
        case _:                 # default
            return "Unknown Status"

# Matching against structure
def process_response(response):
    match response:
        case {"status": "success", "data": data}:
            print(f"Got data: {data}")
        case {"status": "error", "message": msg}:
            print(f"Error: {msg}")
        case _:
            print("Unexpected response format")
```

You won't use `match` in basic Django, but you'll see it in production codebases. Know it exists.

---

## 2. Loops — What's Actually Happening

### 2.1 `for` Loops — Iterating Over Iterables

Python's `for` loop doesn't work with indexes like C or Java. It works with *iterables* — any object that can yield items one at a time.

```python
# Iterating directly — preferred
for fruit in ["apple", "banana", "cherry"]:
    print(fruit)

# When you need the index — use enumerate, not range(len(...))
for index, fruit in enumerate(["apple", "banana", "cherry"]):
    print(f"{index}: {fruit}")

# When you need the index starting from a different number
for index, fruit in enumerate(["apple", "banana", "cherry"], start=1):
    print(f"{index}: {fruit}")   # 1: apple, 2: banana, 3: cherry

# Iterating over a dict
user = {"name": "Anurag", "age": 28, "city": "Mumbai"}
for key in user:           # iterates keys
    print(key)
for key, value in user.items():  # iterates key-value pairs
    print(f"{key} = {value}")

# Iterating two lists in parallel — use zip
names = ["Alice", "Bob", "Charlie"]
scores = [95, 87, 92]
for name, score in zip(names, scores):
    print(f"{name}: {score}")

# zip stops at the shortest list — use zip_longest to fill missing values
from itertools import zip_longest
for name, score in zip_longest(names, scores, fillvalue="N/A"):
    print(f"{name}: {score}")
```

**The `range()` function:**

```python
range(5)        # 0, 1, 2, 3, 4
range(2, 8)     # 2, 3, 4, 5, 6, 7
range(0, 10, 2) # 0, 2, 4, 6, 8  (step=2)
range(10, 0, -1)# 10, 9, 8, ..., 1  (reverse)

# range() is lazy — it doesn't create a list in memory
r = range(1_000_000)  # no memory used yet
list(r)               # NOW it creates the full list — 1M items
```

### 2.2 `while` Loops — Use with Caution

```python
# Basic
count = 0
while count < 5:
    print(count)
    count += 1

# Classic infinite loop with break
while True:
    user_input = input("Enter 'quit' to exit: ")
    if user_input == "quit":
        break
    print(f"You entered: {user_input}")

# While with else (rare but valid — else runs if loop completes without break)
attempts = 0
while attempts < 3:
    password = input("Password: ")
    if password == "secret":
        print("Logged in!")
        break
    attempts += 1
else:
    print("Too many attempts. Account locked.")
```

**Use `while` when:** you don't know how many iterations you need.
**Use `for` when:** you're iterating over a known collection.

In Django, `while` loops are rare. You'll use `for` loops constantly — over querysets, form fields, error lists, template context items.

### 2.3 Loop Control — `break`, `continue`, `pass`

```python
# break — exit the loop entirely
for n in range(10):
    if n == 5:
        break
    print(n)   # prints 0, 1, 2, 3, 4

# continue — skip this iteration, go to next
for n in range(10):
    if n % 2 == 0:
        continue
    print(n)   # prints 1, 3, 5, 7, 9

# pass — do nothing (placeholder)
for n in range(10):
    pass   # valid Python — loop runs but does nothing

# Use pass when Python requires a code block but you have nothing to write yet:
class EmptyView:
    pass   # will add code later

def placeholder_function():
    pass
```

### 2.4 `for...else` — Python's Hidden Feature

Most Python developers don't know this exists. It's genuinely useful.

```python
# The else block runs ONLY if the loop completed without hitting break
def find_prime(numbers):
    for n in numbers:
        for divisor in range(2, n):
            if n % divisor == 0:
                break  # n is not prime
        else:
            # This else belongs to the inner for loop
            # It runs when the inner loop completed without break (n is prime)
            print(f"{n} is prime")

# More readable example
users = [{"name": "Alice", "role": "user"}, {"name": "Bob", "role": "admin"}]
target = "Charlie"

for user in users:
    if user["name"] == target:
        print(f"Found: {user}")
        break
else:
    print(f"{target} not found")  # runs if break never hit
```

---

## 3. Exception Handling — Writing Code That Doesn't Explode

This gets its own full lesson later, but control flow requires understanding the basics.

```python
# Basic try/except
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Can't divide by zero")

# Catching multiple exceptions
try:
    value = int(input("Enter a number: "))
    result = 10 / value
except ValueError:
    print("That's not a number")
except ZeroDivisionError:
    print("Can't divide by zero")

# Catching multiple in one line
except (ValueError, ZeroDivisionError) as e:
    print(f"Error: {e}")

# else — runs if no exception occurred
try:
    value = int("42")
except ValueError:
    print("Conversion failed")
else:
    print(f"Success: {value}")   # runs only if no exception

# finally — ALWAYS runs, exception or not
try:
    file = open("data.txt")
    content = file.read()
except FileNotFoundError:
    print("File not found")
finally:
    file.close()   # always clean up resources

# Better: use context managers (covered in file handling lesson)
with open("data.txt") as file:
    content = file.read()   # file is closed automatically
```

**Django relevance:** Django has its own exception hierarchy — `ObjectDoesNotExist`, `MultipleObjectsReturned`, `ValidationError`, `PermissionDenied`, `Http404`. You'll wrap Django ORM calls in try/except blocks constantly.

```python
from django.core.exceptions import ObjectDoesNotExist

try:
    user = User.objects.get(id=user_id)
except User.DoesNotExist:
    # Handle missing user
    return HttpResponse("User not found", status=404)
```

---

## 4. The Art of Writing Clean Conditionals

### 4.1 Early Returns — Exit Fast, Avoid Nesting

Deeply nested code is hard to read. Return early when you can.

```python
# Bad — deeply nested
def process_order(order):
    if order is not None:
        if order.is_paid:
            if order.items:
                if order.user.is_active:
                    return ship_order(order)

# Good — early returns
def process_order(order):
    if order is None:
        raise ValueError("Order cannot be None")
    if not order.is_paid:
        raise ValueError("Order must be paid first")
    if not order.items:
        raise ValueError("Order has no items")
    if not order.user.is_active:
        raise PermissionError("User account is not active")

    return ship_order(order)
```

The second version is longer but vastly easier to read, test, and debug.

### 4.2 Avoid Boolean Comparisons

```python
# Redundant — is_active IS a bool
if user.is_active == True:   # don't do this
if user.is_active is True:   # also unnecessary

# Just do this
if user.is_active:

# And for False
if not user.is_active:
if user.is_active == False:  # don't do this
```

### 4.3 Replace Complex Conditions with Variables

```python
# Hard to read
if user.age >= 18 and user.is_verified and not user.is_banned and user.subscription_tier in ("premium", "enterprise"):
    allow_access()

# Better
is_adult = user.age >= 18
is_in_good_standing = user.is_verified and not user.is_banned
has_paid_access = user.subscription_tier in ("premium", "enterprise")

if is_adult and is_in_good_standing and has_paid_access:
    allow_access()
```

### 4.4 `any()` and `all()` — Built-ins You Should Know

```python
permissions = ["read", "write", "delete"]

# any() returns True if at least one item is truthy
if any(p == "admin" for p in permissions):
    print("Has admin permission")

# all() returns True if ALL items are truthy
if all(p in allowed_permissions for p in permissions):
    print("All permissions are valid")

# Practical Django use
required_fields = ["username", "email", "password"]
form_data = request.POST

if not all(field in form_data for field in required_fields):
    return HttpResponse("Missing required fields", status=400)
```

---

## 5. Iteration Patterns — Tools That Replace Messy Loops

### 5.1 `sorted()` and `key=`

```python
users = [
    {"name": "Charlie", "age": 30},
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 35},
]

# Sort by age
by_age = sorted(users, key=lambda u: u["age"])

# Sort by name, descending
by_name_desc = sorted(users, key=lambda u: u["name"], reverse=True)

# Sort by multiple keys (age first, then name)
by_age_then_name = sorted(users, key=lambda u: (u["age"], u["name"]))
```

### 5.2 `filter()` and `map()`

These are built-in functions that work lazily (return iterators, not lists).

```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# filter() — keep items where function returns True
evens = list(filter(lambda n: n % 2 == 0, numbers))
# [2, 4, 6, 8, 10]

# map() — transform each item
doubled = list(map(lambda n: n * 2, numbers))
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# Comprehensions are usually more readable — use those instead:
evens = [n for n in numbers if n % 2 == 0]
doubled = [n * 2 for n in numbers]
```

**Use comprehensions over `map()`/`filter()` for simple cases. Use `map()`/`filter()` when passing to functions that expect iterables without materializing a list.**

### 5.3 `itertools` — The Hidden Toolkit

```python
import itertools

# chain — iterate over multiple iterables as one
for item in itertools.chain([1, 2], [3, 4], [5, 6]):
    print(item)   # 1, 2, 3, 4, 5, 6

# groupby — group consecutive items by a key (must be sorted first)
data = [{"city": "NY", "name": "Alice"}, {"city": "NY", "name": "Bob"}, {"city": "LA", "name": "Charlie"}]
data.sort(key=lambda x: x["city"])
for city, group in itertools.groupby(data, key=lambda x: x["city"]):
    print(f"{city}: {list(group)}")

# islice — lazy slicing of any iterator
first_five = list(itertools.islice(range(1_000_000), 5))
# [0, 1, 2, 3, 4]  — no million-item list created

# product — cartesian product
for size, color in itertools.product(["S", "M", "L"], ["red", "blue"]):
    print(f"{size}-{color}")
# S-red, S-blue, M-red, M-blue, L-red, L-blue
```

---

## ⚠️ Common Mistakes

**Mistake 1: Using `range(len(x))` instead of `enumerate`**
```python
# Bad
for i in range(len(items)):
    print(i, items[i])

# Good
for i, item in enumerate(items):
    print(i, item)
```

**Mistake 2: Infinite loops from forgetting to update the condition**
```python
# Bug — n never changes, loop runs forever
n = 0
while n < 10:
    print(n)
    # forgot: n += 1
```

**Mistake 3: Catching too broad an exception**
```python
# Bad — hides real bugs
try:
    result = complex_operation()
except Exception:
    pass  # silently swallows everything

# Good — only catch what you expect
try:
    result = complex_operation()
except SpecificError as e:
    handle_error(e)
```

**Mistake 4: Checking `len(items) > 0` instead of `if items`**
```python
if len(items) > 0:   # works but verbose
if items:            # idiomatic — empty collections are falsy
```

---

## 🧪 Exercises

**Exercise 1:** Write a function `categorize_numbers(numbers)` that takes a list of integers and returns a dict with keys "positive", "negative", and "zero", each mapping to a list of numbers from the input that fall in that category. Do it with comprehensions.

**Exercise 2:** Write a loop that finds the first user in a list who has both `is_active=True` and `is_verified=True`. Use `for...else` to print "No such user found" if none exists.

**Exercise 3:** Given a list of strings that might contain non-numeric values, write code using a comprehension and exception handling (inside the comprehension with a helper function) that converts the valid ones to integers and skips the invalid ones. Input: `["1", "two", "3", "four", "5"]`

**Exercise 4:** You have two lists: `old_users` and `new_users`, each containing user dicts with an "id" field. Write code to find users that exist in `new_users` but NOT in `old_users` (i.e., new additions). Don't use a nested loop — use sets.

---

## ✅ Before Moving On

- [ ] You can explain the difference between falsy and False
- [ ] You use `enumerate()` instead of `range(len(...))`
- [ ] You understand `for...else` and can write a real use case for it
- [ ] You can refactor deeply nested conditionals using early returns
- [ ] You know when to use `any()` and `all()`
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

Django views are full of control flow. A typical view does something like:

1. Check if the request method is POST or GET
2. Check if the user is authenticated
3. Check if the form is valid
4. Try to save to the database
5. Handle exceptions
6. Return appropriate responses

Every one of those steps uses the concepts from this lesson. Without clean control flow skills, your Django views will become an unreadable mess of nested ifs.

**Next:** [03 — Functions: The Building Blocks of Every Django View](./03_functions.md)
