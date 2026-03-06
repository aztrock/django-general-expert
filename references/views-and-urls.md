# Views & URLs Best Practices

Guide to Django views, URL routing, and keeping business logic decoupled from HTTP handling.

## Decision Guide

**Use Django CBV (Class-Based Views) when:**
- Standard CRUD operations
- Building web pages with templates
- Need template rendering

**Use DRF ViewSets when:**
- Building REST APIs
- Need standard API operations (list, create, retrieve, update, delete)
- Want automatic URL routing

**Use DRF APIView only when:**
- Custom logic that doesn't fit CRUD patterns
- Complex operations involving multiple models
- Special endpoints that don't map to a single model

## Django Class-Based Views

### Basic CBV Usage

```python
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView


class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 20
    
    def get_queryset(self):
        return Post.objects.filter(
            status=PublicationStatus.PUBLISHED
        ).select_related('author').order_by('-created_at')


class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/create.html'
    success_url = reverse_lazy('post_list')
    
    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)
```

### Ownership Mixin

Create a mixin for ownership checks to avoid code duplication.

```python
# mixins.py
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin


class OwnershipMixin(LoginRequiredMixin, UserPassesTestMixin):
    """Mixin for ownership checks on Update/Delete views."""
    
    def test_func(self):
        obj = self.get_object()
        return obj.author == self.request.user or self.request.user.is_staff


class PostUpdateView(OwnershipMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/update.html'


class PostDeleteView(OwnershipMixin, DeleteView):
    model = Post
    template_name = 'blog/delete.html'
    success_url = reverse_lazy('post_list')
```

## DRF ViewSets

Use ViewSets for standard CRUD operations.

```python
from rest_framework import viewsets


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    
    def get_queryset(self):
        return Order.objects.filter(
            customer=self.request.user
        ).select_related('shipping_address', 'billing_address')
    
    def get_serializer_class(self):
        if self.action == 'create':
            return OrderCreateSerializer
        if self.action == 'list':
            return OrderListSerializer
        return OrderSerializer
    
    def perform_create(self, serializer):
        serializer.save(customer=self.request.user)
```

### Custom Actions

```python
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import status
from rest_framework.exceptions import ValidationError


class OrderViewSet(viewsets.ModelViewSet):
    
    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        order = self.get_object()
        
        if order.status not in [OrderStatus.PENDING, OrderStatus.PROCESSING]:
            raise ValidationError({'detail': 'Cannot cancel this order'})
        
        order = OrderService.cancel_order(order.id)
        
        return Response({'status': OrderStatus.CANCELLED})
    
    @action(detail=False, methods=['get'])
    def summary(self, request):
        orders = self.get_queryset()
        
        return Response({
            'total_orders': orders.count(),
            'total_spent': sum(o.total for o in orders),
            'pending': orders.filter(status=OrderStatus.PENDING).count(),
        })
```

## DRF APIView

Use APIView only for special cases that don't fit CRUD patterns.

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.exceptions import ValidationError


class OrderStatsView(APIView):
    """Custom endpoint for order statistics."""
    
    def get(self, request):
        user = request.user
        
        total_orders = Order.objects.filter(customer=user).count()
        total_spent = Order.objects.filter(
            customer=user,
            status=OrderStatus.COMPLETED
        ).aggregate(total=models.Sum('total'))['total'] or 0
        
        return Response({
            'total_orders': total_orders,
            'total_spent': float(total_spent),
        })
```

## URL Configuration

### Basic URL Patterns

```python
# urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.PostListView.as_view(), name='post_list'),
    path('post/<int:pk>/', views.PostDetailView.as_view(), name='post_detail'),
    path('post/new/', views.PostCreateView.as_view(), name='post_create'),
    path('post/<int:pk>/edit/', views.PostUpdateView.as_view(), name='post_update'),
    path('post/<int:pk>/delete/', views.PostDeleteView.as_view(), name='post_delete'),
]
```

### DRF Router

```python
# urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSet, CommentViewSet

router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')
router.register(r'comments', CommentViewSet, basename='comment')

urlpatterns = router.urls
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

### URL Parameters

```python
# path converters
path('post/<int:pk>/', ...)        # Integer
path('post/<slug:slug>/', ...)     # Slug
path('post/<uuid:public_id>/', ...) # UUID
path('date/<int:year>/', ...)      # Year
path('date/<int:year>/<int:month>/', ...)


# custom converters
class NegativeIntConverter:
    regex = '-?\d+'
    
    def to_python(self, value):
        return int(value)
    
    def to_url(self, value):
        return str(value)


# register converter
from django.urls import register_converter
register_converter(NegativeIntConverter, 'negint')

# use
path('offset/<negint:offset>/', ...)
```

### Named URLs

```python
# Use names for reverse lookups
path('post/<int:pk>/', views.PostDetailView.as_view(), name='post_detail'),

# In templates
<a href="{% url 'post_detail' pk=post.pk %}">{{ post.title }}</a>

# In views
from django.urls import reverse
url = reverse('post_detail', kwargs={'pk': post.pk})

# In redirect
from django.shortcuts import redirect
return redirect('post_detail', pk=post.pk)

# In DRF
from rest_framework.reverse import reverse
url = reverse('post-detail', kwargs={'pk': post.pk})
```

## Error Handling

### Custom Error Views

```python
# views.py
from django.shortcuts import render


def custom_404(request, exception):
    return render(request, 'errors/404.html', status=404)


def custom_500(request):
    return render(request, 'errors/500.html', status=500)


def custom_403(request, exception):
    return render(request, 'errors/403.html', status=403)


# urls.py
handler404 = 'myapp.views.custom_404'
handler500 = 'myapp.views.custom_500'
handler403 = 'myapp.views.custom_403'
```

## Summary

1. **Use Django CBV** for template-based views
2. **Use DRF ViewSets** for standard CRUD APIs
3. **Use APIView** only for special cases
4. **Use mixins** for reusable behavior (OwnershipMixin)
5. **Keep views thin** - delegate to services
6. **Always use select_related/prefetch_related** to prevent N+1
7. **Namespace URLs** for clarity
8. **Configure permissions globally** in settings.py
9. **Use raise ValidationError** instead of return Response with errors
10. **Use constants** for status values
