# ---- myproject/settings.py ----

from pathlib import Path
import os

BASE_DIR = Path(__file__).resolve().parent.parent
# BASE_DIR is the ROOT of your project (where manage.py lives)
# Path(__file__)            → full path to settings.py
# .parent                   → myproject/ config folder
# .parent.parent            → root myproject/ folder

SECRET_KEY = 'your-secret-key'
DEBUG = True

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'blog',               # your app (short name)
    'shop.apps.ShopConfig', # your app using AppConfig class (preferred)
]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',   # path joining with /
    }
}

TEMPLATES = [{
    'DIRS': [BASE_DIR / 'templates'],  # global templates folder
}]

STATIC_URL  = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'   # collectstatic puts files here

MEDIA_URL  = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'          # uploaded files go here


# ---- Accessing settings anywhere in code ----

from django.conf import settings

def my_view(request):
    debug_mode  = settings.DEBUG
    media_path  = settings.MEDIA_ROOT
    secret      = settings.SECRET_KEY