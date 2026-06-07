# 09 — Iterators & Generators: How Django's QuerySets Are Lazy

> **Django's ORM is built on lazy evaluation.** A queryset like `Article.objects.filter(is_published=True)` does NOT hit the database when you write it. It hits the database only when you actually consume the data — when you loop over it, call `len()`, convert it to a list, or render it in a template. This is not magic. It's Python's iterator protocol. Understanding it gives you precise control over when database queries happen, which is one of the most important performance skills in Django.

---

## 1. The Iterator Protocol — Python's Secret Engine

An **iterable** is anything you can loop over with `for`. An **iterator** is an object that produces values one at a time, on demand.

The protocol requires two dunder methods:

- `__iter__()` — returns the iterator itself (or a new iterator)
- `__next__()` — returns the next value, or raises `StopIteration` when exhausted

```python
# A for loop is syntactic sugar for:
iterable = [1, 2, 3]

iterator = iter(iterable)      # calls iterable.__iter__()
while True:
    try:
        value = next(iterator) # calls iterator.__next__()
        print(value)
    except StopIteration:
        break                  # loop ends
```

Python does this automatically with `for`. You never call `iter()` and `next()` manually in normal code — but understanding that they exist is essential.

### 1.1 Building an Iterator From Scratch

```python
class CountUp:
    """Counts from start to end, inclusive."""

    def __init__(self, start, end):
        self.current = start
        self.end = end

    def __iter__(self):
        return self   # the object IS its own iterator

    def __next__(self):
        if self.current > self.end:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

counter = CountUp(1, 5)
for n in counter:
    print(n)   # 1, 2, 3, 4, 5

# Manual iteration:
counter2 = CountUp(1, 3)
print(next(counter2))   # 1
print(next(counter2))   # 2
print(next(counter2))   # 3
print(next(counter2))   # StopIteration!
```

**Iterables vs Iterators — the difference matters:**

```python
# A list is ITERABLE but NOT an iterator
my_list = [1, 2, 3]
next(my_list)   # TypeError: 'list' object is not an iterator
iter(my_list)   # returns a list_iterator — that's the actual iterator

# You can iterate a list multiple times:
for x in my_list: print(x)  # 1, 2, 3
for x in my_list: print(x)  # 1, 2, 3 again — list is reusable

# An iterator is consumed — once exhausted, it's done:
it = iter([1, 2, 3])
for x in it: print(x)  # 1, 2, 3
for x in it: print(x)  # nothing — iterator is exhausted
```

---

## 2. Generators — Iterators You Can Write in 5 Lines

Writing a class with `__iter__` and `__next__` is verbose. **Generators** are a shorthand: any function with a `yield` statement becomes a generator function. Calling it returns a generator object (which is an iterator).

```python
def count_up(start, end):
    current = start
    while current <= end:
        yield current      # ← suspends function, returns value, resumes on next()
        current += 1

gen = count_up(1, 5)
print(type(gen))           # <class 'generator'>

next(gen)   # 1
next(gen)   # 2
next(gen)   # 3

for n in count_up(1, 5):
    print(n)               # 1, 2, 3, 4, 5

# Equivalent to CountUp class above — less code, same behavior
```

### 2.1 How `yield` Works — The Suspension Model

`yield` is not `return`. `return` exits the function. `yield` **suspends** the function — preserving all local variables and execution state — and returns a value. When `next()` is called again, execution resumes exactly where it left off.

```python
def demonstrate_yield():
    print("Start")
    yield 1              # suspends here, returns 1
    print("After first yield")
    yield 2              # suspends here, returns 2
    print("After second yield")
    yield 3
    print("Function ends — StopIteration raised automatically")

gen = demonstrate_yield()

print(next(gen))
# Start
# 1

print(next(gen))
# After first yield
# 2

print(next(gen))
# After second yield
# 3

print(next(gen))
# Function ends — StopIteration raised automatically
# StopIteration raised
```

This suspension mechanism is what makes generators memory-efficient. The function body doesn't run all at once — it runs incrementally, producing one value at a time.

---

## 3. Why Lazy Evaluation Matters — The Memory Argument

```python
# Eager — compute everything upfront, store everything in memory
def get_all_squares_eager(n):
    result = []
    for i in range(n):
        result.append(i ** 2)
    return result   # returns a list with n items — all in RAM at once

squares = get_all_squares_eager(10_000_000)   # 10M items × ~28 bytes = ~280MB RAM

# Lazy — compute one at a time, constant memory
def get_all_squares_lazy(n):
    for i in range(n):
        yield i ** 2   # one item at a time — no list built

for square in get_all_squares_lazy(10_000_000):   # ~200 bytes RAM regardless of n
    process(square)
```

**Django QuerySet equivalent:**

```python
# Eager — hits database AND loads all 1M users into RAM
all_users = list(User.objects.all())    # SELECT * FROM users → 1M objects in memory

# Lazy — evaluates in chunks, constant memory
for user in User.objects.all():         # still one SELECT, but Django chunks the results
    send_email(user)                    # process one at a time

# Even better for huge datasets — iterator() avoids Django's queryset cache
for user in User.objects.all().iterator(chunk_size=2000):
    send_email(user)   # fetches 2000 rows at a time, no caching
```

---

## 4. Generator Patterns — The Toolkit

### 4.1 Infinite Generators

```python
def counter(start=0, step=1):
    """Infinite counter — produces values forever."""
    n = start
    while True:
        yield n
        n += step

# Must limit consumption with islice or break:
from itertools import islice

first_ten = list(islice(counter(), 10))   # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Or with a for loop and break:
for n in counter(start=1, step=2):
    if n > 20:
        break
    print(n)   # 1, 3, 5, 7, 9, 11, 13, 15, 17, 19
```

### 4.2 Pipeline Generators — Chaining Transformations

This is the most powerful generator pattern. Each generator feeds into the next, with constant memory usage:

```python
def read_log_file(filepath):
    """Lazily yield lines from a log file."""
    with open(filepath, encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n")

def filter_errors(lines):
    """Yield only error lines."""
    for line in lines:
        if "ERROR" in line or "CRITICAL" in line:
            yield line

def parse_log_line(lines):
    """Parse each line into a structured dict."""
    for line in lines:
        parts = line.split(" | ", maxsplit=3)
        if len(parts) == 4:
            yield {
                "timestamp": parts[0],
                "level": parts[1],
                "module": parts[2],
                "message": parts[3],
            }

def enrich_with_metadata(records):
    """Add derived fields."""
    for record in records:
        record["is_critical"] = record["level"] == "CRITICAL"
        yield record

# Pipeline — each stage is lazy, entire process uses constant memory
# regardless of log file size (could be 100GB)
pipeline = enrich_with_metadata(
    parse_log_line(
        filter_errors(
            read_log_file("/var/log/django.log")
        )
    )
)

for record in pipeline:
    store_in_database(record)
```

### 4.3 `yield from` — Delegating to Sub-Generators

```python
def flatten(nested):
    """Flatten a nested list structure of arbitrary depth."""
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)   # delegate to recursive call
        else:
            yield item

list(flatten([1, [2, 3], [4, [5, 6]], 7]))
# [1, 2, 3, 4, 5, 6, 7]
```

`yield from iterable` is equivalent to `for item in iterable: yield item`, but more efficient and works correctly with coroutines (advanced use).

### 4.4 Generator Expressions — Inline Generators

Like list comprehensions, but lazy:

```python
# List comprehension — eager, creates list immediately
squares_list = [x ** 2 for x in range(1000000)]   # 1M items in RAM

# Generator expression — lazy, computes on demand
squares_gen = (x ** 2 for x in range(1000000))    # just a generator object, no RAM

# Both work with sum, any, all, min, max — which accept iterables:
total = sum(x ** 2 for x in range(1000000))   # no list created ever

# Use generator expressions when:
# - You only need to consume once
# - The result feeds directly into another function
# - Memory matters

# Use list comprehensions when:
# - You need to access by index
# - You need to iterate multiple times
# - You need to pass to something requiring a list
```

---

## 5. Django QuerySets — Laziness in Production

Now that you understand the iterator protocol, QuerySets make complete sense.

### 5.1 The QuerySet is Lazy Until Evaluated

```python
# This does NOT hit the database:
articles = Article.objects.filter(is_published=True)       # SQL not sent yet
articles = articles.filter(author=request.user)            # still not sent
articles = articles.order_by("-created_at")                # still not sent
articles = articles.select_related("author", "category")   # still not sent

# SQL is sent ONLY when you evaluate:
list(articles)          # evaluation — hits DB
for a in articles: ...  # evaluation — hits DB
articles[0]             # evaluation — hits DB (LIMIT 1)
articles[:10]           # evaluation — hits DB (LIMIT 10)
len(articles)           # evaluation — hits DB (or uses cache if already evaluated)
bool(articles)          # evaluation — hits DB
articles.count()        # evaluation — but uses SELECT COUNT(*), not fetching rows
articles.exists()       # evaluation — uses SELECT 1 LIMIT 1, faster than count()
```

**Why does this matter?**

```python
# Without understanding laziness — you accidentally hit the DB 3 times:
articles = Article.objects.filter(is_published=True)
total = len(articles)       # DB hit 1 — fetches all rows
first_ten = articles[:10]   # DB hit 2 — fetches 10 rows
for a in articles: ...      # DB hit 3 — fetches all rows again

# With understanding — the queryset caches after first evaluation:
articles = Article.objects.filter(is_published=True)
list_of_articles = list(articles)   # DB hit 1 — fetches and caches
total = len(list_of_articles)       # no DB hit — uses list
first_ten = list_of_articles[:10]   # no DB hit — slices list

# Or use the right tools:
total = Article.objects.filter(is_published=True).count()  # SELECT COUNT(*) — efficient
exists = Article.objects.filter(is_published=True).exists()  # SELECT 1 LIMIT 1 — fastest
```

### 5.2 The N+1 Problem — Laziness Biting Back

```python
# The classic N+1 query bug:
articles = Article.objects.filter(is_published=True)   # 1 query

for article in articles:
    print(article.author.name)   # 1 query PER ARTICLE — N queries!
    # Total: 1 + N queries for N articles

# Fix with select_related (for ForeignKey/OneToOne — SQL JOIN):
articles = Article.objects.filter(is_published=True).select_related("author")
# 1 query total — author fetched in the same JOIN

for article in articles:
    print(article.author.name)   # no query — author already fetched

# Fix with prefetch_related (for ManyToMany/reverse FK — separate optimized query):
articles = Article.objects.filter(is_published=True).prefetch_related("tags")
# 2 queries total — articles, then tags in a single IN query

for article in articles:
    for tag in article.tags.all():   # no query — tags already fetched
        print(tag.name)
```

### 5.3 Queryset as Generator — `iterator()` for Huge Datasets

```python
# Loading 500K users — DON'T do this:
for user in User.objects.all():
    send_birthday_email(user)
# Django caches ALL 500K User objects in the queryset. RAM explodes.

# DO this:
for user in User.objects.all().iterator(chunk_size=1000):
    send_birthday_email(user)
# Fetches 1000 rows at a time, discards after processing. Constant RAM.

# Note: iterator() disables the queryset cache — you can't re-use it
```

### 5.4 Writing Generator-Based Django Utilities

```python
# Export any queryset to CSV lazily — handles millions of rows
def queryset_to_csv_rows(queryset, fields):
    """
    Generator that yields CSV rows from a queryset.
    Memory usage is proportional to chunk_size, not queryset size.
    """
    yield fields   # header row

    for obj in queryset.iterator(chunk_size=2000):
        yield [str(getattr(obj, field, "")) for field in fields]

# Django view using StreamingHttpResponse for large exports:
import csv
from django.http import StreamingHttpResponse

class EchoBuffer:
    """Minimal file-like object for StreamingHttpResponse."""
    def write(self, value):
        return value

def export_all_users(request):
    fields = ["username", "email", "date_joined", "is_active"]
    queryset = User.objects.all().order_by("id")

    rows = queryset_to_csv_rows(queryset, fields)

    pseudo_buffer = EchoBuffer()
    writer = csv.writer(pseudo_buffer)

    response = StreamingHttpResponse(
        (writer.writerow(row) for row in rows),
        content_type="text/csv",
    )
    response["Content-Disposition"] = 'attachment; filename="users.csv"'
    return response
```

This exports millions of rows without loading them all into memory — using Python generators end-to-end.

---

## 6. `itertools` — The Generator Toolkit

`itertools` is a standard library module of lazy, composable iterator building blocks. These are used constantly in data processing.

```python
import itertools

# chain — iterate multiple iterables as one
for item in itertools.chain([1, 2], [3, 4], [5]):
    print(item)   # 1, 2, 3, 4, 5

# islice — lazy slicing (unlike list slicing, doesn't consume to get to start)
first_five = list(itertools.islice(big_generator(), 5))

# takewhile / dropwhile — conditional slicing
data = [2, 4, 6, 7, 8, 10]   # ascending, then a gap
evens_at_start = list(itertools.takewhile(lambda x: x % 2 == 0, data))
# [2, 4, 6] — stops at first odd

after_first_odd = list(itertools.dropwhile(lambda x: x % 2 == 0, data))
# [7, 8, 10] — drops until first odd, then yields everything

# groupby — group consecutive equal elements (input must be sorted by key)
data = [
    {"dept": "Engineering", "name": "Alice"},
    {"dept": "Engineering", "name": "Bob"},
    {"dept": "Marketing", "name": "Charlie"},
]
for dept, members in itertools.groupby(data, key=lambda x: x["dept"]):
    print(dept, list(members))

# product — cartesian product
sizes = ["S", "M", "L"]
colors = ["red", "blue"]
for size, color in itertools.product(sizes, colors):
    print(f"{size}-{color}")   # S-red, S-blue, M-red, M-blue, L-red, L-blue

# combinations and permutations
list(itertools.combinations([1, 2, 3], 2))    # [(1,2), (1,3), (2,3)]
list(itertools.permutations([1, 2, 3], 2))   # [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]

# count — infinite counter
for i in itertools.count(start=10, step=5):
    if i > 30:
        break
    print(i)   # 10, 15, 20, 25, 30

# cycle — infinite cycle through an iterable
statuses = itertools.cycle(["active", "idle", "busy"])
for _ in range(6):
    print(next(statuses))   # active, idle, busy, active, idle, busy

# accumulate — running totals
import operator
data = [1, 2, 3, 4, 5]
list(itertools.accumulate(data))                  # [1, 3, 6, 10, 15]
list(itertools.accumulate(data, operator.mul))    # [1, 2, 6, 24, 120] (factorial)
```

---

## 7. Generator-Based Coroutines — A Glimpse Into Async

Generators can also **receive** values, not just yield them. This is the foundation of Python's `async/await` system.

```python
def running_average():
    """Coroutine that computes running average — receives values via send()."""
    total = 0
    count = 0
    average = None

    while True:
        value = yield average   # yield sends out the average, receives the next value
        if value is None:
            break
        total += value
        count += 1
        average = total / count

# Using a coroutine:
avg = running_average()
next(avg)            # prime the coroutine — advance to first yield
avg.send(10)         # sends 10, gets back 10.0
avg.send(20)         # sends 20, gets back 15.0
avg.send(30)         # sends 30, gets back 20.0
avg.send(None)       # signals done — raises StopIteration
```

Django Channels and async Django views build on this exact mechanism. `async def` functions that use `await` are syntactically cleaner coroutines under the hood.

---

## 8. Real-World Generator Patterns for Django

### 8.1 Batch Processing

```python
def batch(iterable, size):
    """Split any iterable into fixed-size batches."""
    it = iter(iterable)
    while True:
        batch = list(itertools.islice(it, size))
        if not batch:
            return
        yield batch

# Send emails in batches of 100:
all_users = User.objects.filter(wants_newsletter=True).iterator(chunk_size=500)
for user_batch in batch(all_users, 100):
    send_bulk_email(user_batch)
```

### 8.2 Streaming Large Data Transformations

```python
def process_large_import(csv_file):
    """
    Import a large CSV file into the database without loading it all.
    Uses generators at every stage.
    """
    def read_rows():
        reader = csv.DictReader(csv_file)
        for row in reader:
            yield row

    def validate_rows(rows):
        for row in rows:
            errors = []
            if not row.get("email"):
                errors.append(f"Row missing email: {row}")
            if not row.get("username"):
                errors.append(f"Row missing username: {row}")
            if errors:
                for err in errors:
                    logger.warning(err)
                continue   # skip invalid rows
            yield row

    def transform_rows(rows):
        for row in rows:
            yield {
                "username": row["username"].strip().lower(),
                "email": row["email"].strip().lower(),
                "is_active": row.get("status", "active") == "active",
            }

    def create_in_batches(rows, batch_size=500):
        for row_batch in batch(rows, batch_size):
            User.objects.bulk_create(
                [User(**row) for row in row_batch],
                ignore_conflicts=True
            )
            yield len(row_batch)   # yield batch size for progress tracking

    pipeline = create_in_batches(
        transform_rows(
            validate_rows(
                read_rows()
            )
        )
    )

    total = sum(pipeline)   # drives the pipeline, collects totals
    return total
```

---

## ⚠️ Common Mistakes

**Mistake 1: Iterating a generator twice expecting results the second time**
```python
gen = (x ** 2 for x in range(5))
list(gen)   # [0, 1, 4, 9, 16]
list(gen)   # []  ← exhausted!

# Fix: if you need to iterate multiple times, use a list or re-create the generator
```

**Mistake 2: Forgetting to evaluate a queryset before closing the DB connection**
```python
def get_articles():
    return Article.objects.filter(is_published=True)
    # Returns queryset — lazy! DB not hit yet

articles = get_articles()
# ... some time later, perhaps outside a request/response cycle:
for a in articles:   # DB hit happens here — connection might be gone!
    ...

# Fix: evaluate where the connection is guaranteed available:
articles = list(get_articles())   # evaluates immediately
```

**Mistake 3: Calling `len()` on a queryset when `count()` is correct**
```python
total = len(Article.objects.filter(is_published=True))
# Fetches ALL rows, constructs ALL Python objects, just to count them

total = Article.objects.filter(is_published=True).count()
# SELECT COUNT(*) FROM article WHERE is_published=1 — single fast query
```

**Mistake 4: Using `.iterator()` and then trying to use queryset features**
```python
qs = User.objects.all().iterator()
qs.filter(is_active=True)   # TypeError — iterator() returns a generator, not a queryset
# Fix: apply all filters BEFORE calling .iterator()
qs = User.objects.filter(is_active=True).iterator()
```

**Mistake 5: Not understanding that queryset slicing evaluates**
```python
qs = Article.objects.all()
first = qs[0]    # evaluates — SELECT ... LIMIT 1 OFFSET 0
second = qs[1]   # evaluates again — SELECT ... LIMIT 1 OFFSET 1

# If you need multiple items:
first_ten = list(qs[:10])   # one query for all ten
```

---

## 🧪 Exercises

**Exercise 1:** Write a generator `fibonacci()` that yields Fibonacci numbers infinitely. Then write a function `fibonacci_up_to(limit)` that uses it and returns a list of all Fibonacci numbers below the limit.

**Exercise 2:** Write a generator pipeline that:
- Reads a large text file line by line
- Filters lines containing a search term
- Strips whitespace and converts to lowercase
- Groups every 10 matching lines into a batch (as a list)
- Yields each batch

Test it on a file large enough that you'd notice memory usage if you loaded it all at once.

**Exercise 3:** Write a Django management command `send_newsletter` that:
- Uses `.iterator(chunk_size=500)` to fetch subscribers
- Batches them into groups of 50
- "Sends" (just prints for the exercise) a newsletter to each batch
- Reports total sent count and time taken
- Handles interruption gracefully (try/except KeyboardInterrupt)

**Exercise 4:** Write a function `queryset_diff(qs1, qs2, key_field="id")` that:
- Takes two querysets
- Uses generators and `itertools` (not sets, to remain memory-efficient)
- Yields items in `qs1` not in `qs2`
- Works for querysets too large to load into memory at once

---

## ✅ Before Moving On

- [ ] You can implement `__iter__` and `__next__` from scratch
- [ ] You understand that `yield` suspends execution — not exits
- [ ] You can write generator pipelines and explain why they use constant memory
- [ ] You know the difference between iterables and iterators
- [ ] You understand when Django querysets hit the database
- [ ] You use `.count()` and `.exists()` instead of `len()` and `bool()` on querysets
- [ ] You use `.iterator()` for large querysets
- [ ] You recognize the N+1 problem and fix it with `select_related` / `prefetch_related`
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

QuerySets are the single most important thing you'll interact with in Django. They're lazy iterators. Every time you chain `.filter()`, `.exclude()`, `.order_by()`, `.annotate()` — you're building a lazy computation that hasn't hit the database yet. Knowing exactly when evaluation happens is what separates developers who write 300 queries per page from developers who write 3.

The generator patterns from this lesson also directly apply to management commands for data migrations, CSV exports, email campaigns, and any background job that processes more data than fits in RAM.

**Next:** [10 — Decorators: The Python Feature Django Can't Live Without](./10_decorators.md)
