# ---- REGISTER THE APP IN settings.py ----
# After creating an app you MUST add it to INSTALLED_APPS or Django ignores it

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # YOUR APPS — two valid ways:

    # Way 1: short name (simple)
    'blog',
    'users',
    'shop',

    # Way 2: AppConfig path (preferred — gives you more control)
    'blog.apps.BlogConfig',
    'users.apps.UsersConfig',
    'shop.apps.ShopConfig',
]


# ---- blog/apps.py ---- (auto-generated, just know what it is)
from django.apps import AppConfig

class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'   # must match the folder name exactly

    # Optional: run code when app starts
    def ready(self):
        import blog.signals   # connect signals when app loads