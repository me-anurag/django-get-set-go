# ---- myproject/urls.py (ROOT router) ----

from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),

    # include() delegates to each app's own urls.py
    path('blog/',  include('blog.urls')),
    path('shop/',  include('shop.urls')),
    path('users/', include('users.urls')),

    # include with namespace (use in templates: {% url 'api:list' %})
    path('api/', include(('api.urls', 'api'), namespace='api')),
]

# Serve media files in development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
    urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)


# ---- blog/urls.py (APP-LEVEL router) ----

from django.urls import path
from . import views                        # import all views from this app
from .views import PostListView, PostDetailView   # or import specific ones

app_name = 'blog'   # namespace for {% url 'blog:list' %}

urlpatterns = [
    path('',            views.index,       name='index'),
    path('list/',       PostListView.as_view(), name='list'),
    path('<int:pk>/',   PostDetailView.as_view(), name='detail'),
    path('create/',     views.create_post, name='create'),
    path('<int:pk>/edit/', views.edit_post, name='edit'),
]