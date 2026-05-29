# ---- 1. CIRCULAR IMPORT — use string reference ----
# BAD: blog/models.py imports from users/models.py AND users/models.py imports from blog
# FIX: use a string
class Post(models.Model):
    author = models.ForeignKey('users.UserProfile', on_delete=models.CASCADE)


# ---- 2. IMPORT BEFORE DJANGO SETUP ----
# If you run a script outside Django (not manage.py), set up Django first:
import django
import os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')
django.setup()
# NOW you can import models
from blog.models import Post


# ---- 3. LAZY IMPORT INSIDE FUNCTION (breaks circular imports) ----
def get_user_posts(user_id):
    from blog.models import Post   # import inside function, not at top
    return Post.objects.filter(author_id=user_id)


# ---- 4. APPS NOT IN INSTALLED_APPS ----
# If you get: "No module named 'blog'" or "App not found"
# Check settings.py → INSTALLED_APPS has 'blog' or 'blog.apps.BlogConfig'


# ---- 5. RELATIVE vs ABSOLUTE — when to use which ----
from .models  import Post          # relative: good INSIDE the same app
from blog.models import Post       # absolute: good when importing FROM another app or script


# ---- 6. IMPORTING FROM NESTED FOLDERS ----
# Structure: myapp/services/email_service.py
from myapp.services.email_service import send_welcome_email
from .services.email_service import send_welcome_email  # relative version


# ---- 7. RE-EXPORTING via __init__.py ----
# myapp/services/__init__.py
from .email_service import send_welcome_email
from .sms_service   import send_sms

# Now anywhere:
from myapp.services import send_welcome_email, send_sms  # clean!


# ---- 8. ENVIRONMENT-SPECIFIC SETTINGS ----
# Split settings into files:
# myproject/settings/
#   __init__.py
#   base.py       ← common settings
#   dev.py        ← development
#   prod.py       ← production

# dev.py:
from .base import *
DEBUG = True

# Run with:
# python manage.py runserver --settings=myproject.settings.dev


# ---- 9. USING os.path vs pathlib ----
import os
from pathlib import Path

# Old style (os.path):
path = os.path.join(os.path.dirname(__file__), 'templates')

# New style (pathlib — preferred in Django 3+):
path = Path(__file__).parent / 'templates'


# ---- 10. QUICK REFERENCE: what imports what ----

# views.py  → imports from: models, forms, utils, shortcuts, decorators
# urls.py   → imports from: views, django.urls
# models.py → imports from: django.db.models, other models (carefully)
# admin.py  → imports from: models, django.contrib.admin
# forms.py  → imports from: django.forms, models
# signals.py→ imports from: models, django.db.models.signals, django.dispatch
# apps.py   → imports from: django.apps
# tests.py  → imports from: django.test, models, views, urls