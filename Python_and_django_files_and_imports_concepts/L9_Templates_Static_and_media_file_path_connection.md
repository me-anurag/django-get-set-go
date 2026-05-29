# ---- TEMPLATE LOADING ----
# Django looks for templates in two places:
# 1. BASE_DIR/templates/  (global)
# 2. each_app/templates/  (app-level, if APP_DIRS=True in settings)

# In views.py:
from django.shortcuts import render

def index(request):
    return render(request, 'blog/index.html', {'key': 'value'})
    #                        ^ Django finds this at:
    #                          myapp/templates/blog/index.html
    #                          OR
    #                          BASE_DIR/templates/blog/index.html


# ---- STATIC FILES ----
# In settings.py:
STATIC_URL = '/static/'

# In templates:
# {% load static %}
# <link href="{% static 'css/style.css' %}">
# Django finds: myapp/static/css/style.css

# In Python code:
from django.templatetags.static import static
url = static('css/style.css')   # returns '/static/css/style.css'


# ---- MEDIA FILES (user uploads) ----
# In models.py:
from django.db import models

class Article(models.Model):
    cover = models.ImageField(upload_to='covers/')
    # file saved to: MEDIA_ROOT/covers/filename.jpg
    # accessed via:  MEDIA_URL/covers/filename.jpg  →  /media/covers/filename.jpg

# In templates:
# <img src="{{ article.cover.url }}">


# ---- READING ANY FILE WITH BASE_DIR ----
import os
from django.conf import settings

file_path = os.path.join(settings.BASE_DIR, 'data', 'cities.json')
# same thing with pathlib:
file_path = settings.BASE_DIR / 'data' / 'cities.json'

with open(file_path) as f:
    data = f.read()