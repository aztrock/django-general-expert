# Views & URLs Best Practices

Guide to Django views, URL routing, and keeping business logic decoupled from HTTP handling.

## Function-Based Views vs Class-Based Views

### Decision Guide

**Use Function-Based Views (FBV) when:**
- Simple, linear logic
- Unique behavior that doesn't fit CRUD patterns
- You prefer explicit over implicit
- Handling a single HTTP method
- Maximum readability

**Use Class-Based Views (CBV) when:**
- Standard CRUD operations
- Sharing behavior across views (mixins)
- Handling multiple HTTP methods
- Using Django's generic views
- Building reusable view components

## Function-Based Views (FBV)

### Basic Pattern

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.http import JsonResponse
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods, require_POST


# ✅ GOOD: Simple, explicit view
def post_list(request):
    posts = Post.objects.filter(
        status=PublicationStatus.PUBLISHED
    ).select_related('author').order_by('-created_at')
    
    return render(request, 'blog/post_list.html', {'posts': posts})
```

### Early Return Pattern in Views

```python
# ✅ GOOD: Early return pattern
def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    
    # Guard clauses
    if post.status != PublicationStatus.PUBLISHED:
        if not request.user.is_staff:
            return HttpResponseForbidden("Post not published")
    
    if post.is_premium and not request.user.is_premium:
        return render(request, 'blog/premium_required.html', status=402)
    
    # Main logic
    comments = post.comments.select_related('author').order_by('-created_at')
    
    return render(request, 'blog/post_detail.html', {
        'post': post,
        'comments': comments
    })
```

### Form Handling with Services

```python
# ✅ GOOD: Service layer
from .services import OrderService


def create_order(request):
    # Early return for GET
    if request.method != 'POST':
        form = OrderForm()
        return render(request, 'orders/create.html', {'form': form})
    
    form = OrderForm(request.POST)
    
    # Early return for invalid
    if not form.is_valid():
        return render(request, 'orders/create.html', {'form': form})
    
    # Delegate to service
    try:
        order = OrderService.create_order(
            customer=request.user,
            items_data=form.cleaned_data['items'],
            shipping_address=form.cleaned_data['shipping_address']
        )
        return redirect('order_detail', pk=order.pk)
    except ValidationError as e:
        form.add_error(None, str(e))
        return render(request, 'orders/create.html', {'form': form})
```

## Class-Based Views (CBV)

### Generic Views

```python
from django.views.generic import ListView, DetailView, CreateView, UpdateView
from django.contrib.auth.mixins import LoginRequiredMixin


# ✅ GOOD: ListView with optimization
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 20
    
    def get_queryset(self):
        return Post.objects.filter(
            status=PublicationStatus.PUBLISHED
        ).select_related('author').order_by('-created_at')


# ✅ GOOD: CreateView with service
class OrderCreateView(LoginRequiredMixin, CreateView):
    model = Order
    form_class = OrderForm
    template_name = 'orders/create.html'
    
    def form_valid(self, form):
        try:
            order = OrderService.create_order(
                customer=self.request.user,
                items_data=form.cleaned_data['items'],
                shipping_address=form.cleaned_data['shipping_address']
            )
            return redirect('order_detail', pk=order.pk)
        except ValidationError as e:
            form.add_error(None, str(e))
            return self.form_invalid(form)
```

## URL Configuration

### Basic URL Patterns

```python
# urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # List and detail
    path('', views.PostListView.as_view(), name='post_list'),
    path('post/<int:pk>/', views.PostDetailView.as_view(), name='post_detail'),
    
    # CRUD
    path('post/new/', views.PostCreateView.as_view(), name='post_create'),
    path('post/<int:pk>/edit/', views.PostUpdateView.as_view(), name='post_update'),
    path('post/<int:pk>/delete/', views.PostDeleteView.as_view(), name='post_delete'),
]
```

### URL Namespacing

```python
# project/urls.py
from django.urls import path, include

urlpatterns = [
    path('blog/', include('blog.urls', namespace='blog')),
    path('shop/', include('shop.urls', namespace='shop')),
]

# Usage in templates
{% url 'blog:post_detail' pk=post.pk %}

# Usage in views
reverse('blog:post_detail', kwargs={'pk': 1})
```

## Summary

1. **Use FBV** for simple, explicit logic; **use CBV** for standard CRUD
2. **Apply early return pattern** to avoid nested conditionals
3. **Keep views thin** - delegate to services
4. **Use decorators** to restrict HTTP methods
5. **Always use select_related/prefetch_related** to prevent N+1
6. **Namespace URLs** for clarity
7. **Handle errors gracefully** with custom error views
8. **Use pagination** for list views
