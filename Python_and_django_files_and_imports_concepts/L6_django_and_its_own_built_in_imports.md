# ---- MODELS ----
from django.db import models

# ---- VIEWS ----
from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpResponse, HttpResponseNotFound, JsonResponse
from django.views import View
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView

# ---- URLS ----
from django.urls import path, include, reverse, reverse_lazy

# ---- FORMS ----
from django.forms import Form, ModelForm
from django import forms

# ---- AUTHENTICATION ----
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.models import User
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin

# ---- ADMIN ----
from django.contrib import admin

# ---- SETTINGS ----
from django.conf import settings

# ---- SIGNALS ----
from django.db.models.signals import post_save, pre_save, post_delete
from django.dispatch import receiver

# ---- MIDDLEWARE / DECORATORS ----
from django.utils.decorators import method_decorator
from django.views.decorators.csrf import csrf_exempt

# ---- TIMEZONE & DATETIME ----
from django.utils import timezone
from django.utils.timezone import now

# ---- TRANSLATION ----
from django.utils.translation import gettext_lazy as _

# ---- STATIC & MEDIA FILES ----
from django.conf.urls.static import static
from django.contrib.staticfiles.templatetags.staticfiles import static as static_file