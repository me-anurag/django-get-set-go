myproject/
├── users/
│   └── models.py   ← has a User model
├── blog/
│   └── models.py   ← wants to use User
└── shop/
    └── models.py   ← also wants to use User


# ---- blog/models.py ----

from django.db import models

# Import from another app using full path
from users.models import UserProfile

class Post(models.Model):
    author = models.ForeignKey(UserProfile, on_delete=models.CASCADE)
    title  = models.CharField(max_length=200)


# ---- shop/models.py ----

from users.models import UserProfile   # same pattern

class Order(models.Model):
    customer = models.ForeignKey(UserProfile, on_delete=models.CASCADE)


# ---- AVOID CIRCULAR IMPORTS ----
# If users/models.py also imports from blog/models.py → circular error!
# Solution: use Django's string reference instead of direct import

class Post(models.Model):
    author = models.ForeignKey('users.UserProfile', on_delete=models.CASCADE)
    #                           ^ string: 'appname.ModelName' — no import needed