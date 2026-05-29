# ---- myproject/settings.py ----

from pathlib import Path
import os

# BASE_DIR = the root of your project (where manage.py is)
BASE_DIR = Path(__file__).resolve().parent.parent
# Path(__file__)         → full path to this settings.py file
# .parent                → myproject/ inner config folder
# .parent.parent         → outer myproject/ root ← THIS is BASE_DIR


# ---- SECURITY ----
SECRET_KEY = 'django-insecure-xxxxxxxx'   # change in production, keep secret
DEBUG = True    # True = show error pages | False = hide errors in production
ALLOWED_HOSTS = []   # empty = only localhost
# In production:
# ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com', '123.456.789.0']


# ---- DATABASE ----
DATABASES = {
    'default': {
        # SQLite (default, no setup needed, good for development)
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',

        # PostgreSQL (production recommended)
        # 'ENGINE': 'django.db.backends.postgresql',
        # 'NAME': 'mydb',
        # 'USER': 'myuser',
        # 'PASSWORD': 'mypassword',
        # 'HOST': 'localhost',
        # 'PORT': '5432',

        # MySQL
        # 'ENGINE': 'django.db.backends.mysql',
        # 'NAME': 'mydb',
        # 'USER': 'root',
        # 'PASSWORD': '',
        # 'HOST': 'localhost',
        # 'PORT': '3306',
    }
}


# ---- TEMPLATES ----
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],  # global templates folder
        'APP_DIRS': True,    # also look in each app's templates/ folder
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',  # needed for admin
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]


# ---- STATIC FILES (CSS, JS, images you write) ----
STATIC_URL = '/static/'
STATICFILES_DIRS = [BASE_DIR / 'static']   # where you put your static files
STATIC_ROOT = BASE_DIR / 'staticfiles'     # where collectstatic copies them (deployment)


# ---- MEDIA FILES (user uploads) ----
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'


# ---- AUTH ----
LOGIN_URL = '/users/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'

# Custom user model (if you made one — set BEFORE first migration)
# AUTH_USER_MODEL = 'users.CustomUser'


# ---- EMAIL ----
# Console (dev) — prints emails to terminal
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
# SMTP (production)
# EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
# EMAIL_HOST = 'smtp.gmail.com'
# EMAIL_PORT = 587
# EMAIL_USE_TLS = True
# EMAIL_HOST_USER = 'you@gmail.com'
# EMAIL_HOST_PASSWORD = 'yourpassword'


# ---- TIMEZONE & LANGUAGE ----
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'Asia/Kolkata'   # your timezone
USE_I18N = True
USE_TZ = True                # store datetimes as UTC in DB


# ---- DEFAULT PRIMARY KEY ----
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
# This means every model gets an auto-increment BigInt id by default