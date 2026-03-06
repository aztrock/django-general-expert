# Filters Guide

Guide to using django-filters in Django REST Framework for advanced filtering capabilities.

## Installation

```bash
pip install django-filter
```

## Configuration

### Settings Configuration

```python
# settings.py
INSTALLED_APPS = [
    'django_filters',
    'rest_framework',
]

# Configure globally - avoid repeating in views
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

## Basic FilterSet

### Simple Filters

```python
# filters.py
import django_filters
from .models import Product, Category


class ProductFilter(django_filters.FilterSet):
    # Exact match filter
    status = django_filters.CharFilter(field_name='status')
    
    # Exact match (default for non-string fields)
    category_id = django_filters.NumberFilter(field_name='category_id')
    
    # Boolean filter
    is_active = django_filters.BooleanFilter(field_name='is_active')
    
    # Date filters
    created_after = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='gte'
    )
    created_before = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='lte'
    )
    
    class Meta:
        model = Product
        fields = [
            'status',
            'category_id',
            'is_active',
        ]
```

### Using Filters in Views

```python
# views.py
from rest_framework import viewsets
from .filters import ProductFilter
from .serializers import ProductSerializer


class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filterset_class = ProductFilter
    
    # filter_backends NOT needed when using filterset_class
```

### Filter URL Patterns

```
# GET /api/products/
# GET /api/products/?status=active
# GET /api/products/?category_id=5
# GET /api/products/?is_active=true
# GET /api/products/?created_after=2024-01-01
# GET /api/products/?created_before=2024-12-31
# GET /api/products/?status=active&category_id=5
```

## Advanced Filters

### Lookup Expressions

```python
import django_filters


class ProductFilter(django_filters.FilterSet):
    # Exact (default)
    name = django_filters.CharFilter(field_name='name', lookup_expr='exact')
    
    # Contains (case-sensitive)
    name_contains = django_filters.CharFilter(
        field_name='name',
        lookup_expr='contains'
    )
    
    # icontains (case-insensitive)
    name_icontains = django_filters.CharFilter(
        field_name='name',
        lookup_expr='icontains'
    )
    
    # Starts with
    name_starts = django_filters.CharFilter(
        field_name='name',
        lookup_expr='startswith'
    )
    
    # Ends with
    name_ends = django_filters.CharFilter(
        field_name='name',
        lookup_expr='endswith'
    )
    
    # Greater than or equal
    price_min = django_filters.NumberFilter(
        field_name='price',
        lookup_expr='gte'
    )
    
    # Less than or equal
    price_max = django_filters.NumberFilter(
        field_name='price',
        lookup_expr='lte'
    )
    
    # Range
    price_range = django_filters.RangeFilter(field_name='price')
    
    # In list
    ids = django_filters.CharFilter(
        field_name='id',
        lookup_expr='in'
    )
    
    # Is null
    no_category = django_filters.BooleanFilter(
        field_name='category',
        lookup_expr='isnull'
    )
    
    class Meta:
        model = Product
        fields = ['status', 'is_active']
```

### URL Patterns for Advanced Filters

```
# GET /api/products/?name_contains=phone
# GET /api/products/?price_min=100&price_max=500
# GET /api/products/?price_range_min=100&price_range_max=500
# GET /api/products/?ids=1,2,3,4,5
# GET /api/products/?no_category=true
```

### Date/Time Filters

```python
import django_filters
from django_filters import DateFromToRangeFilter, DateTimeFromToRangeFilter


class OrderFilter(django_filters.FilterSet):
    # Simple date filter
    created_date = django_filters.DateFilter(field_name='created_at__date')
    
    # Date range
    created_range = DateFromToRangeFilter(field_name='created_at')
    
    # DateTime range
    updated_range = DateTimeFromToRangeFilter(field_name='updated_at')
    
    # Relative date (last N days)
    class Meta:
        model = Order
        fields = ['status', 'created_range']
```

### Filtering by Related Fields

```python
import django_filters


class ProductFilter(django_filters.FilterSet):
    # Filter by foreign key field
    category = django_filters.CharFilter(field_name='category__slug')
    category_name = django_filters.CharFilter(field_name='category__name')
    
    # Filter by nested foreign key
    author_username = django_filters.CharFilter(
        field_name='author__username'
    )
    
    # Filter by M2M field
    tag = django_filters.CharFilter(field_name='tags__slug')
    
    class Meta:
        model = Product
        fields = ['category', 'status']


# URL: /api/products/?category=electronics
# URL: /api/products/?author_username=jdoe
# URL: /api/products/?tag=django
```

## SearchFilter

### Basic Search

```python
# views.py - filter_backends already in settings
from rest_framework import viewsets


class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    search_fields = ['name', 'description', 'category__name']
    # Generates: WHERE name ILIKE '%query%' OR description ILIKE '%query%'
```

### Advanced Search

```python
# views.py - filter_backends already in settings
class ProductViewSet(viewsets.ModelViewSet):
    # Simple fields
    search_fields = ['name', 'description']

    # With lookups (Django field lookups)
    search_fields = [
        '=name',      # Exact match (faster)
        'name',       # Partial match (LIKE %name%)
        '^name',      # Starts with
        '$name',      # Regex
    ]

    # Related fields
    search_fields = [
        'name',
        'category__name',
        'author__username',
    ]

    # Custom search behavior
    def get_search_fields(self, view, request):
        if request.query_params.get('version') == 'v2':
            return ['name', 'description', 'sku']
        return ['name', 'description']
```

## OrderingFilter

### Basic Ordering

```python
# views.py - filter_backends already in settings
from rest_framework import viewsets


class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    ordering_fields = ['created_at', 'price', 'name']
    ordering = ['-created_at']  # Default ordering
```

### Multiple Ordering

```python
# Allow ordering by multiple fields
# GET /api/products/?ordering=-price,name
# Orders by price descending, then by name ascending
```

## Combined Filtering

### Full Example

```python
# filters.py
import django_filters
from .models import Product, Category


class ProductFilter(django_filters.FilterSet):
    # Price filters
    min_price = django_filters.NumberFilter(
        field_name='price',
        lookup_expr='gte'
    )
    max_price = django_filters.NumberFilter(
        field_name='price',
        lookup_expr='lte'
    )
    
    # Category filter
    category = django_filters.CharFilter(field_name='category__slug')
    
    # Status filter
    status = django_filters.CharFilter()
    
    # Search (SearchFilter already in settings)
    # Custom filter method
    search = django_filters.CharFilter(method='filter_search')
    
    def filter_search(self, queryset, name, value):
        return queryset.filter(
            models.Q(name__icontains=value) |
            models.Q(description__icontains=value) |
            models.Q(sku__icontains=value)
        )
    
    class Meta:
        model = Product
        fields = ['status', 'category', 'min_price', 'max_price']


# views.py - filter_backends already in settings
from rest_framework import viewsets


class ProductViewSet(viewsets.ModelViewSet):
    serializer_class = ProductSerializer
    filterset_class = ProductFilter
    
    # SearchFilter settings (filter_backends in settings)
    search_fields = ['name', 'description', 'sku']
    
    # OrderingFilter settings (filter_backends in settings)
    ordering_fields = ['price', 'created_at', 'name']
    ordering = ['-created_at']
    
    def get_queryset(self):
        return Product.objects.select_related('category', 'author')
```

### URL Examples

```
# Filter by status
GET /api/products/?status=active

# Filter by category
GET /api/products/?category=electronics

# Filter by price range
GET /api/products/?min_price=10&max_price=100

# Search
GET /api/products/?search=phone

# Order by price descending
GET /api/products/?ordering=-price

# Combined
GET /api/products/?status=active&category=electronics&min_price=50&ordering=-price&search=laptop
```

## Best Practices

### Organization

```python
# Structure: apps/products/api/v1/products/filters.py

# Or: apps/products/filters/product_filters.py
```

### Performance

```python
# Always use select_related and prefetch_related in views
class ProductViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Product.objects.select_related(
            'category', 'author'
        ).prefetch_related('tags')
```

### Validation

```python
# filters.py
class ProductFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(
        field_name='price',
        lookup_expr='gte',
        min_value=0  # Validate minimum value
    )
    max_price = django_filters.NumberFilter(
        field_name='price',
        lookup_expr='lte',
        max_value=1000000
    )
```

## Summary

1. **Configure globally** in settings, not in views
2. **Use FilterSet** for complex filtering needs
3. **Combine with SearchFilter and OrderingFilter** for full API functionality
4. **Use filter_fields** for simple cases
5. **Use select_related/prefetch_related** for performance
6. **Organize by API endpoint** in separate files
7. **Use help_text** for documentation
8. **Import constants** from constants.py
9. **No magic numbers** - use constants
10. **Test filters** - ensure expected behavior
