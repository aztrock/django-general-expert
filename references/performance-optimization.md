# Performance Optimization

Guide to optimizing Django applications for speed and scalability.

## Database Query Optimization

### The N+1 Problem

```python
# ❌ BAD: N+1 queries
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # N additional queries!

# ✅ GOOD: Single query with JOIN
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.username)  # 1 query
```

### select_related vs prefetch_related

```python
# select_related: For ForeignKey and OneToOne
posts = Post.objects.select_related('author')

# prefetch_related: For ManyToMany and reverse ForeignKey
authors = Author.objects.prefetch_related('books')
```

## Database Indexing

```python
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['customer', 'status']),
        ]
```

## Caching

### Redis Setup

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

### Cache Decorator

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 minutes
def post_list(request):
    posts = Post.objects.filter(status='published')
    return render(request, 'posts/list.html', {'posts': posts})
```

## Bulk Operations

```python
# ❌ BAD: N queries
for i in range(1000):
    Post.objects.create(title=f'Post {i}')

# ✅ GOOD: 1 query
posts = [Post(title=f'Post {i}') for i in range(1000)]
Post.objects.bulk_create(posts, batch_size=100)
```

## Summary

1. **Prevent N+1 queries** with select_related/prefetch_related
2. **Add database indexes** for frequently queried fields
3. **Use caching** for expensive operations
4. **Use bulk operations** for creating/updating many records
5. **Profile queries** to identify bottlenecks
