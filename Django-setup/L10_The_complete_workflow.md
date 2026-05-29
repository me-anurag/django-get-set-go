# ================================================================
# COMPLETE PROJECT SETUP FROM ZERO — copy and run in order
# ================================================================

# 1. Create and enter project folder
mkdir myproject
cd myproject

# 2. Create virtual environment
python -m venv venv

# 3. Activate it (PowerShell)
.\venv\Scripts\Activate.ps1

# 4. Install packages
pip install django pillow               # pillow needed for ImageField
pip install djangorestframework         # if building an API
pip install python-decouple             # for .env file (secrets)
pip install psycopg2-binary             # if using PostgreSQL

# 5. Save your dependencies
pip freeze > requirements.txt

# 6. On another machine, install from requirements.txt
pip install -r requirements.txt

# 7. Create Django project
django-admin startproject myproject .

# 8. Create apps
python manage.py startapp blog
python manage.py startapp users

# 9. Register apps in settings.py (manually edit the file)

# 10. Run first migration (sets up auth, sessions tables)
python manage.py migrate

# 11. Create admin user
python manage.py createsuperuser

# 12. Run server
python manage.py runserver



# ================================================================
# .env FILE — keep secrets out of settings.py
# ================================================================

# Install: pip install python-decouple

# Create a .env file in root (next to manage.py):
# SECRET_KEY=your-actual-secret-key
# DEBUG=True
# DB_NAME=mydb
# DB_USER=myuser
# DB_PASSWORD=mypassword

# In settings.py:
from decouple import config

SECRET_KEY = config('SECRET_KEY')
DEBUG      = config('DEBUG', default=False, cast=bool)

# Add .env to .gitignore — NEVER commit it


# ================================================================
# QUERYSET CHEATSHEET — everything you need for the ORM
# ================================================================

from blog.models import Post

Post.objects.all()                         # all records
Post.objects.filter(status='published')    # WHERE status = 'published'
Post.objects.exclude(status='draft')       # WHERE status != 'draft'
Post.objects.get(pk=1)                     # get exactly one — raises error if not found
Post.objects.first()                       # first record
Post.objects.last()                        # last record
Post.objects.count()                       # COUNT(*)
Post.objects.order_by('-created_at')       # ORDER BY created_at DESC
Post.objects.order_by('title')             # ORDER BY title ASC
Post.objects.filter(author__username='ali')  # follow FK with double underscore
Post.objects.filter(title__icontains='django')  # case-insensitive LIKE
Post.objects.filter(created_at__year=2024)
Post.objects.filter(views__gte=100)        # >=
Post.objects.filter(views__lte=50)         # <=
Post.objects.filter(views__gt=0)           # >
Post.objects.values('title', 'status')     # returns dicts, not objects
Post.objects.values_list('title', flat=True)  # returns flat list of titles
Post.objects.select_related('author', 'category')  # JOIN — avoids N+1 for FK
Post.objects.prefetch_related('tags')             # prefetch M2M
Post.objects.only('title', 'slug')         # fetch only these fields
Post.objects.defer('body')                 # fetch all EXCEPT these fields

# Create
post = Post.objects.create(title='Hello', slug='hello', body='...', author=user)

# Update
Post.objects.filter(status='draft').update(status='published')

# Delete
Post.objects.filter(author=user).delete()

# Save
post = Post(title='Hello', slug='hello')
post.author = request.user
post.save()


# ================================================================
# SHELL — test code interactively without running the server
# ================================================================

# python manage.py shell

from blog.models import Post
Post.objects.all()
Post.objects.create(title='test', slug='test', body='body', author_id=1)


# ================================================================
# COMMON MANAGE.PY COMMANDS
# ================================================================

# python manage.py runserver          → start development server
# python manage.py makemigrations     → detect model changes
# python manage.py migrate            → apply changes to DB
# python manage.py createsuperuser    → create admin user
# python manage.py shell              → interactive Python shell with Django loaded
# python manage.py dbshell            → raw SQL shell
# python manage.py collectstatic      → copy static files to STATIC_ROOT (deployment)
# python manage.py startapp name      → create new app
# python manage.py check              → check for project problems
# python manage.py test               → run tests
# python manage.py showmigrations     → list all migrations and status
# python manage.py sqlmigrate app 0001 → show SQL for a migration
# python manage.py flush              → wipe all data from DB (careful!)


# ================================================================
# .gitignore — what to NEVER commit
# ================================================================

# venv/
# __pycache__/
# *.pyc
# db.sqlite3
# .env
# media/
# staticfiles/
# *.log


<!-- 1. cd into your project folder
2. .\venv\Scripts\Activate.ps1      ← activate venv
3. python manage.py runserver       ← start server
4. Make changes to models?          → makemigrations + migrate
5. New app created?                 → add to INSTALLED_APPS in settings.py
6. New URL?                         → add to app/urls.py + include in root urls.py
7. New view?                        → create in views.py + wire in urls.py
8. New template?                    → create in app/templates/appname/file.html -->