# Performance Optimization

Guide to optimizing Django applications for speed and scalability. Performance optimization should be data-driven - profile first, then optimize.

## Performance Optimization Workflow

```
1. Identify bottleneck (profiling)
     │
     ▼
2. Measure current performance
     │
     ▼
3. Optimize the critical path
     │
     ▼
4. Verify improvement
     │
     ▼
5. Repeat for next bottleneck
```

## Database Query Optimization

### The N+1 Problem

The N+1 query problem occurs when fetching a list of objects and then accessing related objects, causing one query per related object.

```python
# ❌ BAD: N+1 queries
# 1 query for posts + N queries for authors
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # Each access triggers a query!


# ✅ GOOD: Single query with JOIN
# Only 1 query total
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.username)  # No additional query


# ❌ BAD: N+1 for M2M
posts = Post.objects.all()
for post in posts:
    for tag in post.tags.all():  # N queries!
        print(tag.name)


# ✅ GOOD: Prefetch M2M
posts = Post.objects.prefetch_related('tags').all()
for post in posts:
    for tag in post.tags.all():  # No additional queries
        print(tag.name)
```

### select_related vs prefetch_related

```python
# select_related: For ForeignKey and OneToOne (single related object)
# Uses SQL JOIN, reduces queries to 1

# ✅ GOOD: Single query with JOIN
posts = Post.objects.select_related('author', 'category')
# Generates: SELECT ... FROM post JOIN author ON ... JOIN category ON ...


# prefetch_related: For ManyToMany and reverse ForeignKey (multiple objects)
# Uses separate query + Python join

# ✅ GOOD: Two queries + Python join
posts = Post.objects.prefetch_related('tags', 'comments')
# Query 1: SELECT * FROM post
# Query 2: SELECT * FROM tag WHERE post_id IN (...)
# Query 3: SELECT * FROM comment WHERE post_id IN (...)


# Prefetch with custom queryset
from django.db.models import Prefetch

posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(status='approved').select_related('author')
    )
)


# Prefetch with limit
from django.db.models import Prefetch

posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.order_by('-created_at')[:5]
    )
)
```

### Query Optimization Techniques

```python
# Only fetch needed fields
posts = Post.objects.only('id', 'title', 'slug', 'published_at')

# Exclude unneeded fields
posts = Post.objects.defer('content', 'raw_html')


# Use values() for dictionaries - no model instantiation
post_dicts = Post.objects.values('id', 'title', 'slug')
# Returns: [{'id': 1, 'title': 'Post 1', 'slug': 'post-1'}, ...]


# Use values_list() for flat lists - avoids generators
titles = Post.objects.values_list('title', flat=True)
# Returns: ['Post 1', 'Post 2', ...]


# Use first() instead of exists() when you need the object
post = Post.objects.filter(status='published').first()
if not post:
    return Response({'error': 'No posts'})
# Use post directly - no second query needed


# Only use exists() for boolean checks
if not Post.objects.filter(status='published').exists():
    logger.warning('No published posts')


# Use annotate() for complex filters on related fields
from django.db.models import Count, Q

# Instead of complex filter on related field
authors = User.objects.annotate(
    published_count=Count('posts', filter=Q(posts__status='published'))
).filter(published_count__gt=10)


# Use annotate() for aggregations
from django.db.models import Count, Avg, Sum, Max

# Count related objects
posts = Post.objects.annotate(comment_count=Count('comments'))

# Aggregate in queryset
authors = User.objects.annotate(
    post_count=Count('posts'),
    avg_rating=Avg('posts__rating')
)


# Use aggregate() for totals
from django.db.models import Sum

total_revenue = Order.objects.aggregate(total=Sum('total'))
# Returns: {'total': Decimal('12345.67')}


# Use Q objects for complex queries
from django.db.models import Q

posts = Post.objects.filter(
    Q(status='published') | Q(is_featured=True)
).filter(
    Q(published_at__isnull=False) & ~Q(category=None)
)
```


### Custom QuerySet Managers

```python
# models.py - Use model's custom manager/queryset for complex relations
# This reduces SQL complexity and pushes data processing to the database

class PostQuerySet(models.QuerySet):
    def with_stats(self):
        return self.annotate(
            comment_count=Count('comments'),
            view_count=Count('views')
        )
    
    def published(self):
        return self.filter(status='published')
    
    def with_author(self):
        return self.select_related('author')


class PostManager(models.Manager):
    def get_queryset(self):
        return PostQuerySet(self.model, using=self._db)
    
    def with_stats(self):
        return self.get_queryset().with_stats()
    
    def published(self):
        return self.get_queryset().published()
    
    def with_author(self):
        return self.get_queryset().with_author()


class Post(models.Model):
    title = models.CharField(max_length=200)
    status = models.CharField(max_length=20)
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    comments = models.ManyToManyField(Comment)
    views = models.ManyToManyField(View)
    
    objects = PostManager()


# Usage - clean and reusable
posts = Post.objects.published().with_stats().with_author()
```

### Raw SQL Optimization

```python
# Use raw() for complex queries
posts = Post.objects.raw(
    '''
    SELECT p.*, COUNT(c.id) as comment_count
    FROM blog_post p
    LEFT JOIN blog_comment c ON p.id = c.post_id
    WHERE p.status = 'published'
    GROUP BY p.id
    ORDER BY comment_count DESC
    '''
)


# Use database functions
from django.db.models import F, Func

# Case-insensitive search (use database index if available)
posts = Post.objects.filter(title__icontains='django')


# Use Q objects for complex queries
from django.db.models import Q

posts = Post.objects.filter(
    Q(status='published') | Q(is_featured=True)
).filter(
    Q(published_at__isnull=False) & ~Q(category=None)
)
```

## Database Indexing

### When to Index

```python
# ✅ Index: Fields used in WHERE clauses
class Post(models.Model):
    title = models.CharField(max_length=200)
    status = models.CharField(max_length=20)  # Indexed for WHERE status = ?
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [
            # Single field index
            models.Index(fields=['status']),
            
            # Composite index for common queries
            models.Index(fields=['status', 'created_at']),
            
            # Index for ordering
            models.Index(fields=['-created_at']),
            
            # Partial index (not supported in Django directly, use raw SQL)
        ]


# ✅ Index: ForeignKey fields (automatic in PostgreSQL, manual in MySQL)
# Django creates indexes on FK automatically in most cases


# ✅ Index: Fields used in joins
class Comment(models.Model):
    post = models.ForeignKey(Post, on_length=20)  # Already indexed
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    
    class Meta:
        indexes = [
            # Composite for common query pattern
            models.Index(fields=['post', 'created_at']),
        ]


# ✅ Index: Fields used in LIKE prefix searches
class Product(models.Model):
    sku = models.CharField(max_length=50, db_index=True)
    name = models.CharField(max_length=200)
    
    # For LIKE 'django%' queries
    # You might need: CREATE INDEX ... USING BTREE (name text_pattern_ops)


# ❌ Don't index: Low-cardinality fields (boolean, gender)
# Unless combined with other fields


# ❌ Don't index: Fields rarely queried
```

### Django-Specific Indexes

```python
# Expression index (Django 3.2+)
from django.db.models import Index, Func, Value


class Product(models.Model):
    name = models.CharField(max_length=200)
    category = models.CharField(max_length=50)
    
    class Meta:
        indexes = [
            # Lowercase index for case-insensitive search
            Index(Func('name', function='LOWER'), name='product_name_lower_idx'),
            
            # Partial index (use RawSQL or check if your DB supports it)
        ]


# Functional index for PostgreSQL
class Customer(models.Model):
    email = models.EmailField()
    
    class Meta:
        indexes = [
            Index(
                Lower('email'),
                name='customer_email_lower_idx'
            ),
        ]
```

## Caching

### Django Caching Levels

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 60 * 15,  # 15 minutes default
    }
}
```

### View Caching

```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie


# Cache entire view
@cache_page(60 * 15)  # 15 minutes
def post_list(request):
    posts = Post.objects.all()
    return render(request, 'posts/list.html', {'posts': posts})


# Cache with per-user variation
@vary_on_cookie
@cache_page(60 * 15)
def dashboard(request):
    return render(request, 'dashboard.html')


# Cache with custom key
from django.views.decorators.cache import cache_control

@cache_control(max_age=3600, public=True)
def public_post_list(request):
    posts = Post.objects.all()
    return render(request, 'posts/list.html', {'posts': posts})
```

### Template Fragment Caching

```django
{% load cache %}

{% cache 500 sidebar %}
<div class="sidebar">
    <h3>Categories</h3>
    {% for category in categories %}
        <a href="{% url 'category' category.slug %}">{{ category.name }}</a>
    {% endfor %}
</div>
{% endcache %}


{% cache 500 "post_list" request.user.id %}
{# User-specific cached content #}
{% endcache %}
```

### Low-Level Cache API

```python
from django.core.cache import cache


# Basic operations
cache.set('my_key', 'my_value', timeout=3600)
value = cache.get('my_key')
value = cache.get('my_key', 'default_value')  # Default if not found


# Multiple keys
cache.set_many({
    'key1': 'value1',
    'key2': 'value2'
})
cache.get_many(['key1', 'key2'])


# Delete
cache.delete('my_key')
cache.delete_pattern('key_*')


# Cache with version
cache.set('key', 'value', version=2)
cache.get('key', version=2)


# Increment/Decrement
cache.set('counter', 0)
cache.incr('counter', 1)
cache.decr('counter', 1)
```

### Cache Invalidation Patterns

```python
from django.core.cache import cache


# Pattern 1: Key-based invalidation
def get_posts():
    cache_key = 'posts_list'
    posts = cache.get(cache_key)
    
    if posts is None:
        posts = Post.objects.all()
        cache.set(cache_key, posts, timeout=3600)
    
    return posts


def invalidate_posts():
    cache.delete('posts_list')


# Pattern 2: Version-based invalidation
POSTS_VERSION = 'posts_version_v1'

def get_posts_cached():
    version = cache.get(POSTS_VERSION)
    cache_key = f'posts_{version}'
    
    posts = cache.get(cache_key)
    if posts is None:
        posts = Post.objects.all()
        cache.set(cache_key, posts, timeout=3600)
    
    return posts


def invalidate_posts():
    # Increment version to invalidate all cached posts
    cache.incr(POSTS_VERSION)


# Pattern 3: Pattern-based invalidation
def invalidate_posts_pattern():
    # Use Redis SCAN pattern in production
    cache.delete_pattern('posts_*')
```

### View Caching with Cache Backend

```python
from django.views.decorators.cache import cache_page
from django.core.cache.backends.redis import RedisCache


@cache_page(3600, cache='redis')
def my_view(request):
    ...
```

## Bulk Operations

### Bulk Create

```python
# ❌ BAD: N queries
for i in range(1000):
    Post.objects.create(title=f'Post {i}')


# ✅ GOOD: Single query
posts = [Post(title=f'Post {i}') for i in range(1000)]
Post.objects.bulk_create(posts)


# ✅ With batch_size
Post.objects.bulk_create(posts, batch_size=100)


# ✅ Bulk create with ignore_conflicts
Post.objects.bulk_create(posts, ignore_conflicts=True)


# ✅ Bulk create with update_conflicts (Django 4.2+)
Post.objects.bulk_create(
    posts,
    update_conflicts=True,
    unique_fields=['slug'],
    update_fields=['title', 'content']
)
```

### Bulk Update

```python
# ❌ BAD: N queries
for post in posts:
    post.view_count += 1
    post.save()


# ✅ GOOD: Single query
Post.objects.filter(id__in=[p.id for p in posts]).update(
    view_count=models.F('view_count') + 1
)


# ✅ Bulk update with values
Post.objects.filter(status='draft').update(
    status='published',
    published_at=timezone.now()
)
```

### Bulk Delete

```python
# ✅ GOOD: Single query
Post.objects.filter(status='archived').delete()
```

### Using annotate with Bulk Operations

```python
from django.db.models import F, Count


# Update based on computed values
User.objects.annotate(
    post_count=Count('posts')
).filter(
    post_count__gt=10
).update(
    is_pro=True
)
```

## Async Views (Django 4.0+)

```python
# Async view
async def async_post_list(request):
    posts = await sync_to_async(Post.objects.all)()
    return render(request, 'posts/list.html', {'posts': posts})


# Async with database
async def async_post_detail(request, slug):
    try:
        post = await sync_to_async(Post.objects.get)(slug=slug)
    except Post.DoesNotExist:
        return JsonResponse({'error': 'Not found'}, status=404)
    
    return JsonResponse({
        'id': post.id,
        'title': post.title,
    })


# Async service call
async def async_order_view(request, order_id):
    order = await sync_to_async(Order.objects.get)(pk=order_id)
    
    # Call async service
    order_data = await get_order_data(order_id)
    
    return JsonResponse(order_data)
```

## Query Monitoring

### Django Debug Toolbar

```python
# Install
# pip install django-debug-toolbar

# settings.py
INSTALLED_APPS = [
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
]

INTERNAL_IPS = ['127.0.0.1']


# urls.py
from debug_toolbar.toolbar import debug_toolbar_urls

urlpatterns = [
    ...
] + debug_toolbar_urls()


# Use in templates to see queries
{% if debug %}
{{ sql_queries|length }} queries
{% endif %}
```

### django-silk

```python
# Install
# pip install django-silk

# settings.py
INSTALLED_APPS = [
    'silk',
]

MIDDLEWARE = [
    'silk.middleware.SilkyMiddleware',
]

ROOT_URLCONF = 'urls'

# urls.py
urlpatterns = [
    ...
    path('silk/', include('silk.urls')),
]


# Enable profiling
SILKY_PYTHON_PROFILER = True
SILKY_PYTHON_PROFILER_ON = True


# For DRF ViewSet
from silk.decorator import silk_profile

class PostViewSet(viewsets.ModelViewSet):
    @silk_profile
    def list(self, request):
        ...
    
    @silk_profile
    def retrieve(self, request, pk=None):
        ...


# For CBV
from silk.decorator import silk_profile

class PostListView(ListView):
    @silk_profile
    def get(self, request):
        ...
```

### Custom Query Logging

```python
# Log slow queries
import logging
from django.db import connection
from django.conf import settings


class QueryLogger:
    def __init__(self, threshold=0.1):  # 100ms
        self.threshold = threshold
    
    def __enter__(self):
        self.queries = connection.queries
        return self
    
    def __exit__(self, *args):
        for query in connection.queries:
            duration = float(query['time'])
            if duration > self.threshold:
                logging.warning(
                    f'Slow query ({duration}s): {query["sql"][:200]}'
                )


# Usage
with QueryLogger(threshold=0.5):
    posts = Post.objects.all().select_related('author')
```

## Profiling

### cProfile

```python
import cProfile
import pstats
from io import StringIO


def profile_view(view_func):
    def wrapper(request, *args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        response = view_func(request, *args, **kwargs)
        
        profiler.disable()
        
        stats = pstats.Stats(profiler)
        stream = StringIO()
        stats.stream = stream
        stats.sort_stats('cumulative')
        stats.print_stats(20)
        
        print(stream.getvalue())
        
        return response
    
    return wrapper
```

## Summary

1. **Profile first** - Don't optimize without data
2. **Use select_related** - For ForeignKey and OneToOne
3. **Use prefetch_related** - For ManyToMany and reverse ForeignKey
4. **Add database indexes** - For WHERE and ORDER BY fields
5. **Use caching** - Redis for production
6. **Use bulk operations** - bulk_create, bulk_update
7. **Use .first()** - Instead of exists() when you need the object
8. **Use exists()** - Only for boolean checks
9. **Use only()/defer()** - Fetch only needed fields
10. **Use values()/values_list()** - For flat data, no model instantiation
11. **Use annotate()** - For aggregations and complex filters
12. **Use custom QuerySet managers** - For reusable query patterns
13. **Monitor queries** - Use debug toolbar or silk
14. **Consider async** - For I/O bound views (Django 4.0+)
