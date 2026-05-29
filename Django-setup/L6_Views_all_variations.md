# ---- blog/views.py ----

from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpResponse, JsonResponse
from django.views import View
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView, TemplateView
)
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from .models import Post, Category
from .forms import PostForm


# ======================================
# TYPE 1: Function-Based Views (FBV)
# ======================================

def post_list(request):
    posts = Post.objects.filter(status='published')
    return render(request, 'blog/list.html', {'posts': posts})


def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug)  # returns 404 page if not found
    return render(request, 'blog/detail.html', {'post': post})


@login_required   # redirect to login if not authenticated
def create_post(request):
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('blog:detail', slug=post.slug)
    else:
        form = PostForm()
    return render(request, 'blog/create.html', {'form': form})


def post_delete(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.method == 'POST':
        post.delete()
        return redirect('blog:list')
    return render(request, 'blog/confirm_delete.html', {'post': post})


# Return JSON (for APIs or AJAX)
def post_api(request, pk):
    post = get_object_or_404(Post, pk=pk)
    data = {'title': post.title, 'body': post.body}
    return JsonResponse(data)


# ======================================
# TYPE 2: Class-Based Views (CBV)
# ======================================

class PostListView(ListView):
    model = Post
    template_name = 'blog/list.html'     # default would be blog/post_list.html
    context_object_name = 'posts'         # default would be 'object_list'
    paginate_by = 10

    def get_queryset(self):
        return Post.objects.filter(status='published')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()  # add extra data
        return context


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/detail.html'
    context_object_name = 'post'
    slug_field = 'slug'          # which model field to look up by
    slug_url_kwarg = 'slug'      # what the URL parameter is called


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/create.html'
    success_url = reverse_lazy('blog:list')

    def form_valid(self, form):
        form.instance.author = self.request.user   # set author before save
        return super().form_valid(form)


class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/edit.html'
    success_url = reverse_lazy('blog:list')


class PostDeleteView(LoginRequiredMixin, DeleteView):
    model = Post
    template_name = 'blog/confirm_delete.html'
    success_url = reverse_lazy('blog:list')


# ======================================
# TYPE 3: Generic TemplateView
# ======================================

class HomeView(TemplateView):
    template_name = 'home.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['latest_posts'] = Post.objects.filter(status='published')[:5]
        return context