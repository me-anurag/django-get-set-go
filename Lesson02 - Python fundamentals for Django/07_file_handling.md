# 07 — File Handling: Reading, Writing, and the Context Manager Pattern

> **Web applications handle files constantly** — user-uploaded profile pictures, exported CSVs, generated PDFs, log files, media assets, JSON configs. Django has high-level tools for file handling, but they all sit on top of Python's raw file I/O. If you don't understand the layer below, Django's file handling will feel magical and unpredictable. When something breaks, you'll have no idea where to look.

---

## 1. The Fundamental Model: Files Are Streams

A file is not a list of lines or a single string. It's a **stream** — a sequential flow of bytes. When you open a file, you get a file object that maintains a **cursor** (a position in the stream). Reading or writing advances the cursor.

```python
f = open("data.txt", "r")

f.read(5)     # reads next 5 bytes, cursor moves to position 5
f.read(5)     # reads the NEXT 5 bytes (5-10), not the first 5 again
f.tell()      # returns current cursor position: 10

f.seek(0)     # move cursor back to the beginning
f.read(5)     # now reads the first 5 bytes again

f.seek(0, 2)  # seek to end (0 bytes from end=2)
f.tell()      # file size in bytes

f.close()     # MUST close — releases OS file handle
```

This cursor model matters in Django when you're handling uploaded files — an `InMemoryUploadedFile` works the same way, and if you read it once, you need to `seek(0)` before reading it again.

---

## 2. Opening Files — `open()` and All Its Modes

```python
open(file, mode='r', encoding=None, buffering=-1, errors=None, newline=None)
```

### 2.1 Mode Strings

| Mode | Meaning | File must exist? | Truncates? |
|------|---------|-----------------|-----------|
| `'r'` | Read text | Yes | No |
| `'w'` | Write text | No (creates) | Yes — wipes file |
| `'a'` | Append text | No (creates) | No |
| `'x'` | Exclusive create | No (fails if exists) | N/A |
| `'r+'` | Read + write | Yes | No |
| `'w+'` | Read + write | No (creates) | Yes |
| `'b'` | Binary flag | — | — |
| `'t'` | Text flag (default) | — | — |

```python
# Common combinations:
open("file.txt", "r")    # read text (default)
open("file.txt", "w")    # write text — DANGEROUS: wipes existing file
open("file.txt", "a")    # append text — safe for logging
open("file.bin", "rb")   # read binary (images, PDFs, any non-text)
open("file.bin", "wb")   # write binary
open("file.txt", "r+")   # read and write without truncating
```

**`'w'` is destructive.** The moment you `open("file.txt", "w")`, the file is emptied — even before you write anything. If you wanted to update a file, use `'r+'` or read the content first, modify it, then write back.

### 2.2 Encoding — Always Specify It

```python
# BAD — encoding depends on platform (UTF-8 on Mac/Linux, cp1252 on Windows)
f = open("file.txt", "r")

# GOOD — always explicit
f = open("file.txt", "r", encoding="utf-8")
f = open("file.txt", "w", encoding="utf-8")
```

**Always use `encoding="utf-8"` for text files in Django projects.** Django itself is UTF-8 throughout. Files written without explicit encoding will produce platform-dependent behavior that breaks in production.

---

## 3. The Context Manager — `with` Is Not Optional

Every time you `open()` a file, you're asking the OS for a **file descriptor** — a limited resource. The OS has a cap (typically 1024 per process). If you forget to close files, you'll hit `OSError: [Errno 24] Too many open files` in production.

```python
# The wrong way — what most beginners write
f = open("data.txt", "r")
content = f.read()
# ... what if an exception happens here? f.close() never runs
f.close()

# The right way — always
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
# f is guaranteed closed here, even if an exception occurred inside the block
```

**`with` calls `__enter__` when entering and `__exit__` when leaving — even if an exception is raised.** This is the same context manager protocol you built in the OOP lesson. `open()` returns an object whose `__exit__` calls `f.close()`.

```python
# Multiple files at once
with open("input.txt", "r", encoding="utf-8") as infile, \
     open("output.txt", "w", encoding="utf-8") as outfile:
    for line in infile:
        outfile.write(line.upper())
# Both files closed automatically
```

---

## 4. Reading Files — Every Method Explained

```python
with open("article.txt", "r", encoding="utf-8") as f:

    # read() — entire file as one string
    content = f.read()            # "Line 1\nLine 2\nLine 3"

    # read(n) — exactly n characters
    f.seek(0)
    chunk = f.read(1024)          # first 1KB

    # readline() — one line including the \n
    f.seek(0)
    line1 = f.readline()          # "Line 1\n"
    line2 = f.readline()          # "Line 2\n"

    # readlines() — ALL lines as a list
    f.seek(0)
    lines = f.readlines()         # ["Line 1\n", "Line 2\n", "Line 3"]

    # iterating directly — BEST for large files (lazy, one line at a time)
    f.seek(0)
    for line in f:
        process(line.rstrip("\n"))  # strip the trailing newline
```

**For large files, always iterate directly:**

```python
# BAD for large files — loads entire file into memory
with open("huge_log.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()    # 2GB file → 2GB in RAM
    for line in lines:
        process(line)

# GOOD — lazy, constant memory usage regardless of file size
with open("huge_log.txt", "r", encoding="utf-8") as f:
    for line in f:           # reads one line at a time
        process(line.rstrip())
```

---

## 5. Writing Files

```python
with open("output.txt", "w", encoding="utf-8") as f:

    # write() — write a string, returns number of characters written
    f.write("Hello, world!\n")   # \n must be explicit

    # writelines() — write a list of strings (no newlines added automatically)
    lines = ["line 1\n", "line 2\n", "line 3\n"]
    f.writelines(lines)

    # print() can write to a file
    print("Hello from print", file=f)
    print("With newline", file=f)    # print adds \n automatically

# Appending — doesn't wipe existing content
with open("log.txt", "a", encoding="utf-8") as f:
    f.write("New log entry\n")
```

### 5.1 Buffering — Why Your Writes Might Not Appear Immediately

Python buffers writes — it accumulates data in memory and writes in chunks for performance. This means a crash before the buffer is flushed can lose data.

```python
with open("log.txt", "a", encoding="utf-8") as f:
    f.write("Something happened\n")
    f.flush()   # force write to OS immediately (but still in OS cache)

# For critical data:
import os
with open("critical.txt", "w", encoding="utf-8") as f:
    f.write("Important data\n")
    f.flush()
    os.fsync(f.fileno())   # force OS to write to physical disk
```

For most Django work, the default buffering is fine. But if you're writing a log file that needs to survive a crash, `flush()` matters.

---

## 6. Working with File Paths — `pathlib` Is the Modern Way

Before `pathlib` (Python 3.4+), path manipulation was done with string concatenation and `os.path`. Don't use those anymore.

```python
from pathlib import Path

# Creating paths
p = Path("/home/anurag/projects/djangoblog")
p = Path.home() / "projects" / "djangoblog"   # / operator joins paths

# Path components
p.name           # "djangoblog"  — filename or final component
p.stem           # "djangoblog"  — filename without extension
p.suffix         # ""            — file extension (e.g., ".py")
p.parent         # Path("/home/anurag/projects")
p.parts          # ("/", "home", "anurag", "projects", "djangoblog")

# File operations
p.exists()       # True/False
p.is_file()      # True if it's a file
p.is_dir()       # True if it's a directory
p.stat().st_size # file size in bytes
p.stat().st_mtime  # last modified time (Unix timestamp)

# Creating directories
p.mkdir(parents=True, exist_ok=True)   # creates all intermediate dirs

# Listing directory contents
for item in p.iterdir():
    print(item.name)

# Glob patterns
for py_file in p.glob("**/*.py"):    # all .py files recursively
    print(py_file)

# Reading and writing directly via pathlib
content = Path("file.txt").read_text(encoding="utf-8")   # entire file as string
Path("output.txt").write_text("Hello!\n", encoding="utf-8")

raw = Path("image.png").read_bytes()     # entire file as bytes
Path("copy.png").write_bytes(raw)
```

**Django's settings use `pathlib`:**

```python
# settings.py — what Django generates:
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
# Path(__file__)  → /home/anurag/projects/djangoblog/djangoblog/settings.py
# .parent         → /home/anurag/projects/djangoblog/djangoblog/
# .parent.parent  → /home/anurag/projects/djangoblog/  (the project root)

MEDIA_ROOT = BASE_DIR / "media"
STATIC_ROOT = BASE_DIR / "static"
TEMPLATES = [{"DIRS": [BASE_DIR / "templates"]}]
```

The `/` operator on `Path` objects builds paths safely — no string concatenation, no platform-specific separators. Always use `pathlib` for paths in Django.

---

## 7. Binary Files — Images, PDFs, Everything Non-Text

Text mode (`'r'`, `'w'`) decodes bytes to strings using an encoding. Binary mode (`'rb'`, `'wb'`) gives you raw bytes — no decoding, no newline translation.

```python
# Copy a binary file (image, PDF, etc.)
with open("photo.jpg", "rb") as source, open("photo_copy.jpg", "wb") as dest:
    # Read in chunks — never read a potentially large file all at once
    chunk_size = 65536   # 64KB chunks
    while True:
        chunk = source.read(chunk_size)
        if not chunk:   # empty bytes means EOF
            break
        dest.write(chunk)

# Or with pathlib for small files:
from pathlib import Path
Path("photo_copy.jpg").write_bytes(Path("photo.jpg").read_bytes())
```

**Working with binary data:**

```python
# bytes and bytearray
data = b"Hello, binary world!"   # bytes literal — immutable
mutable_data = bytearray(data)   # bytearray — mutable

# Converting between str and bytes
text = "Hello"
encoded = text.encode("utf-8")      # str → bytes: b"Hello"
decoded = encoded.decode("utf-8")   # bytes → str: "Hello"

# This is what Django does with HTTP request bodies:
raw_body = request.body    # bytes
text_body = raw_body.decode("utf-8")   # decode to work with it as text
```

---

## 8. Working with CSV Files

CSV is the most common data exchange format in web apps — reports, imports, exports.

```python
import csv

# Writing CSV
data = [
    ["Name", "Email", "Age"],
    ["Alice", "alice@example.com", 30],
    ["Bob", "bob@example.com", 25],
]

with open("users.csv", "w", newline="", encoding="utf-8") as f:
    # newline="" is important on Windows — csv handles its own line endings
    writer = csv.writer(f)
    writer.writerows(data)   # write all rows at once

# Writing CSV from dicts — much cleaner in real code
users = [
    {"name": "Alice", "email": "alice@example.com", "age": 30},
    {"name": "Bob", "email": "bob@example.com", "age": 25},
]

with open("users.csv", "w", newline="", encoding="utf-8") as f:
    fieldnames = ["name", "email", "age"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(users)

# Reading CSV
with open("users.csv", "r", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)   # reads each row as a dict
    for row in reader:
        print(row["name"], row["email"])
        # {"name": "Alice", "email": "alice@example.com", "age": "30"}
        # Note: all values are strings — you must convert age to int yourself
```

**Generating CSV responses in Django:**

```python
# views.py
import csv
from django.http import HttpResponse

def export_users_csv(request):
    response = HttpResponse(content_type="text/csv")
    response["Content-Disposition"] = 'attachment; filename="users.csv"'

    writer = csv.writer(response)   # HttpResponse works like a file!
    writer.writerow(["Username", "Email", "Date Joined"])

    for user in User.objects.all().values_list("username", "email", "date_joined"):
        writer.writerow(user)

    return response
```

---

## 9. Working with JSON Files

JSON is the language of APIs and configuration files.

```python
import json

# Python → JSON string
data = {
    "name": "Anurag",
    "age": 28,
    "skills": ["Python", "Django"],
    "is_active": True,
    "score": None   # Python None → JSON null
}

# Serialize to string
json_string = json.dumps(data)
# '{"name": "Anurag", "age": 28, "skills": ["Python", "Django"], "is_active": true, "score": null}'

# Pretty-print
json_pretty = json.dumps(data, indent=2, sort_keys=True)

# Serialize to file
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
    # ensure_ascii=False preserves non-ASCII characters (Hindi, Arabic, etc.)

# JSON string → Python
parsed = json.loads(json_string)
parsed["name"]   # "Anurag"

# File → Python
with open("data.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
```

**Python ↔ JSON type mapping:**

| Python | JSON |
|--------|------|
| `dict` | object `{}` |
| `list`, `tuple` | array `[]` |
| `str` | string `""` |
| `int`, `float` | number |
| `True` / `False` | `true` / `false` |
| `None` | `null` |

**What JSON can't serialize by default:**

```python
import json
from datetime import datetime
from decimal import Decimal

data = {
    "created": datetime.now(),   # TypeError — datetime not JSON serializable
    "price": Decimal("19.99"),   # TypeError — Decimal not JSON serializable
}

json.dumps(data)   # TypeError!

# Fix 1: Custom encoder
class DjangoJSONEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)

json.dumps(data, cls=DjangoJSONEncoder)

# Fix 2: Django ships with its own encoder
from django.core.serializers.json import DjangoJSONEncoder
json.dumps(data, cls=DjangoJSONEncoder)
```

---

## 10. Django's File Handling — How It Works Under the Hood

Understanding Python file I/O makes Django's file handling transparent.

### 10.1 `MEDIA_ROOT` and `MEDIA_URL`

```python
# settings.py
MEDIA_ROOT = BASE_DIR / "media"   # filesystem path where files are stored
MEDIA_URL = "/media/"              # URL prefix to serve them
```

When a user uploads a file, Django saves it to `MEDIA_ROOT/<upload_to>/filename`. When you access `instance.file.url`, Django returns `MEDIA_URL + relative_path`.

### 10.2 Handling Uploaded Files

```python
# models.py
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    avatar = models.ImageField(upload_to="avatars/")   # files go to MEDIA_ROOT/avatars/

# views.py
def upload_avatar(request):
    if request.method == "POST" and request.FILES.get("avatar"):
        uploaded_file = request.FILES["avatar"]

        # uploaded_file is an InMemoryUploadedFile or TemporaryUploadedFile
        # It behaves like a file object:
        print(uploaded_file.name)        # "photo.jpg"
        print(uploaded_file.size)        # size in bytes
        print(uploaded_file.content_type) # "image/jpeg"

        # Reading the file content:
        content = uploaded_file.read()   # reads all bytes
        uploaded_file.seek(0)            # MUST seek back if you want to read again!

        # Django saves it for you via the model:
        profile = request.user.userprofile
        profile.avatar = uploaded_file
        profile.save()   # Django calls storage backend to write the file
```

### 10.3 Generating Files and Serving Them

```python
import io
from django.http import FileResponse
from reportlab.pdfgen import canvas

def generate_pdf(request):
    # Build PDF in memory — no temp file on disk
    buffer = io.BytesIO()

    # Write PDF to buffer
    p = canvas.Canvas(buffer)
    p.drawString(100, 750, f"Hello, {request.user.username}!")
    p.save()

    # Seek to beginning so Django can read it
    buffer.seek(0)

    return FileResponse(
        buffer,
        as_attachment=True,
        filename="report.pdf",
        content_type="application/pdf"
    )
```

`io.BytesIO()` — an in-memory bytes buffer that works like a file. This is the correct way to generate files in Django without writing to disk: build in memory, serve directly.

---

## 11. File System Operations — `os`, `shutil`, and `pathlib`

```python
import os
import shutil
from pathlib import Path

# Check existence (prefer pathlib)
Path("file.txt").exists()
os.path.exists("file.txt")   # old way, still works

# Create directory
Path("media/uploads").mkdir(parents=True, exist_ok=True)

# Copy file
shutil.copy("source.txt", "destination.txt")         # file only
shutil.copy2("source.txt", "destination.txt")        # file + metadata
shutil.copytree("source_dir/", "destination_dir/")  # entire directory tree

# Move / rename
shutil.move("old_name.txt", "new_name.txt")
Path("old.txt").rename("new.txt")   # pathlib way

# Delete
Path("file.txt").unlink()             # delete file
Path("file.txt").unlink(missing_ok=True)  # delete, no error if not found
shutil.rmtree("directory/")          # delete entire directory tree — IRREVERSIBLE

# File info
stat = Path("file.txt").stat()
stat.st_size     # size in bytes
stat.st_mtime    # last modified (Unix timestamp)

import time
last_modified = time.ctime(stat.st_mtime)   # human-readable
```

---

## ⚠️ Common Mistakes

**Mistake 1: Opening a file for writing and losing data**
```python
# You want to ADD content to an existing file
with open("log.txt", "w") as f:   # WRONG — 'w' truncates first!
    f.write("New entry\n")

# Fix: use 'a' for append
with open("log.txt", "a", encoding="utf-8") as f:
    f.write("New entry\n")
```

**Mistake 2: Forgetting `encoding` on Windows**
```python
# Works on Mac/Linux (UTF-8 default), breaks on Windows (cp1252 default)
with open("file.txt", "r") as f:
    content = f.read()

# Fix — always explicit
with open("file.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

**Mistake 3: Reading uploaded file twice without seeking**
```python
def process_upload(request):
    f = request.FILES["document"]
    content = f.read()          # cursor now at end
    size = len(f.read())        # reads nothing — cursor is at end!

    # Fix:
    content = f.read()
    f.seek(0)                   # reset cursor
    size = len(f.read())
```

**Mistake 4: Building paths with string concatenation**
```python
# Bad — breaks on Windows (backslash vs forward slash)
path = settings.MEDIA_ROOT + "/uploads/" + filename

# Good — pathlib handles platform differences
path = settings.MEDIA_ROOT / "uploads" / filename
```

**Mistake 5: Loading large files entirely into memory**
```python
with open("huge_file.csv", "r") as f:
    data = f.readlines()   # 5GB file → crash or severe slowdown
    for line in data: ...

# Fix: iterate line by line
with open("huge_file.csv", "r", encoding="utf-8") as f:
    for line in f:
        process(line)
```

---

## 🧪 Exercises

**Exercise 1:** Write a function `merge_csv_files(input_files, output_file)` that:
- Takes a list of CSV file paths
- Reads all of them (each has the same columns)
- Merges them into a single output CSV
- Skips duplicate header rows (only write the header once)
- Reports how many rows were written total

**Exercise 2:** Write a Django view `export_articles_json` that:
- Gets all published articles from the database
- Serializes them to JSON with fields: `id`, `title`, `author_name`, `created_at`, `word_count`
- Returns it as a downloadable `.json` file
- Uses `DjangoJSONEncoder` to handle datetime fields

**Exercise 3:** Write a function `safe_file_write(filepath, content)` that:
- Writes `content` to a temp file first (in the same directory)
- Only replaces the target file if the write succeeds
- If anything fails, the original file is untouched
- Handles both text and bytes content

**Exercise 4:** Using `pathlib`, write a function `analyze_project(root_dir)` that:
- Counts total `.py` files
- Counts total lines of Python code (across all `.py` files)
- Finds the 5 largest `.py` files by size
- Returns a dict with these statistics

---

## ✅ Before Moving On

- [ ] You always use `with open(...)` — never bare `open()` without closing
- [ ] You always specify `encoding="utf-8"` on text files
- [ ] You know the difference between `'w'` (truncates) and `'a'` (appends)
- [ ] You iterate over large files line by line, not `readlines()`
- [ ] You use `pathlib.Path` for all path manipulation (not `os.path` or string concatenation)
- [ ] You know what `seek(0)` is for and when you need it
- [ ] You understand `io.BytesIO` and why Django uses it for in-memory file generation
- [ ] You've done all four exercises

---

## 🔗 What This Has to Do With Django

File handling is everywhere in Django:
- `settings.py` uses `pathlib` for `BASE_DIR`, `MEDIA_ROOT`, `STATIC_ROOT`
- File uploads land in `request.FILES` as file-like objects
- `FileField` and `ImageField` read/write files through storage backends
- Management commands often read from or write to files
- Django's logging system writes to files using Python's `logging` module
- CSV/JSON exports are a staple feature in any data-driven app

Understanding this layer means you can build file import/export features, generate reports, handle user uploads safely, and debug storage-related issues without guesswork.

**Next:** [08 — Error Handling: Writing Code That Fails Gracefully](./08_error_handling.md)
