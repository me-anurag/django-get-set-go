# ---- blog/admin.py ----

from django.contrib import admin
from .models import Post, Category, Tag


# Simple registration
admin.site.register(Category)
admin.site.register(Tag)


# Full customization
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display   = ['title', 'author', 'status', 'created_at']  # columns shown
    list_filter    = ['status', 'category', 'created_at']          # sidebar filters
    search_fields  = ['title', 'body']                             # search box
    prepopulated_fields = {'slug': ('title',)}                     # auto-fill slug from title
    raw_id_fields  = ['author']                                    # search popup for FK
    date_hierarchy = 'created_at'                                  # date drill-down
    ordering       = ['-created_at']
    list_editable  = ['status']                                    # edit inline in list


# ---- CREATE SUPERUSER ----
# python manage.py createsuperuser
# Visit: http://127.0.0.1:8000/admin


# ---- AUTHENTICATION IN VIEWS ----

from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.decorators import login_required
from django.contrib import messages


def login_view(request):
    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']
        user = authenticate(request, username=username, password=password)
        if user is not None:
            login(request, user)
            messages.success(request, 'Welcome back!')
            return redirect('blog:list')
        else:
            messages.error(request, 'Invalid username or password.')
    return render(request, 'users/login.html')


def logout_view(request):
    logout(request)
    messages.info(request, 'You have been logged out.')
    return redirect('users:login')


@login_required
def dashboard(request):
    # request.user → the logged-in user object
    # request.user.is_authenticated → True/False
    # request.user.is_staff         → True if admin
    # request.user.is_superuser     → True if superuser
    my_posts = Post.objects.filter(author=request.user)
    return render(request, 'users/dashboard.html', {'posts': my_posts})


# ---- MESSAGE LEVELS ----
messages.debug(request,   'Debug message')
messages.info(request,    'Info message')
messages.success(request, 'It worked!')
messages.warning(request, 'Watch out!')
messages.error(request,   'Something went wrong.')