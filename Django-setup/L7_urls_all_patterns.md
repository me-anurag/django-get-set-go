# ---- myproject/urls.py (ROOT) ----

from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('',       include('blog.urls')),
    path('users/', include('users.urls')),
    path('shop/',  include('shop.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL,  document_root=settings.MEDIA_ROOT)
    urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)


# ---- blog/urls.py ----

from django.urls import path
from . import views

app_name = 'blog'  # enables namespacing: {% url 'blog:list' %}

urlpatterns = [
    # ---- BASIC ----
    path('',                     views.PostListView.as_view(),   name='list'),
    path('create/',              views.PostCreateView.as_view(), name='create'),

    # ---- WITH INT PARAMETER ----
    path('<int:pk>/',            views.PostDetailView.as_view(), name='detail'),
    path('<int:pk>/edit/',       views.PostUpdateView.as_view(), name='edit'),
    path('<int:pk>/delete/',     views.PostDeleteView.as_view(), name='delete'),

    # ---- WITH SLUG PARAMETER ----
    path('<slug:slug>/',         views.post_detail,              name='detail-slug'),

    # ---- WITH STRING PARAMETER ----
    path('category/<str:name>/', views.category_view,            name='category'),

    # ---- WITH UUID PARAMETER ----
    # path('<uuid:uid>/',        views.some_view,                name='uuid-view'),

    # ---- NESTED PARAMETERS ----
    path('<int:year>/<int:month>/<slug:slug>/', views.archive_detail, name='archive'),
]

# URL parameter types:
# <int:name>   → matches integers only:           /post/42/
# <slug:name>  → matches slugs (letters, -, _):   /post/my-title/
# <str:name>   → matches any non-empty string:    /post/anything/
# <uuid:name>  → matches UUID:                    /post/abc123.../
# <path:name>  → matches including slashes:       /files/a/b/c/


# ---- USING URLS IN CODE ----

from django.urls import reverse, reverse_lazy

# In views (FBV):
url = reverse('blog:detail', kwargs={'slug': 'my-post'})
return redirect('blog:list')
return redirect('blog:detail', slug='my-post')

# In views (CBV) — use reverse_lazy because class loads before URLs:
success_url = reverse_lazy('blog:list')

# In templates:
# {% url 'blog:list' %}
# {% url 'blog:detail' slug=post.slug %}
# {% url 'blog:archive' year=2024 month=6 slug=post.slug %}