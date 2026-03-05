# Filters Guide

Guide to using django-filters in Django REST Framework.

## Installation

```bash
pip install django-filters
```

## Configuration

### Settings

```python
# settings.py
INSTALLED_APPS = [
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
}
```

**IMPORTANT**: Configure in settings, NOT in individual views.

## Basic FilterSet

```python
# apps/orders/api/v1/order_list/filters.py
import django_filters
from myapp.constants import OrderStatus
from myapp.models import Order


class OrderFilter(django_filters.FilterSet):
    status = django_filters.CharFilter(
        field_name='status',
        help_text='Filter by order status'
    )
    
    min_total = django_filters.NumberFilter(
        field_name='total',
        lookup_expr='gte'
    )
    
    max_total = django_filters.NumberFilter(
        field_name='total',
        lookup_expr='lte'
    )
    
    created_after = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='gte'
    )
    
    class Meta:
        model = Order
        fields = ['status', 'customer']
```

## Using in Views

```python
from rest_framework import viewsets
from .filters import OrderFilter


class OrderViewSet(viewsets.ModelViewSet):
    filterset_class = OrderFilter
    
    # filter_backends NOT needed (from settings)
```

## Best Practices

1. **Configure globally** in settings
2. **Organize by API endpoint** in separate files
3. **Use help_text** instead of docstrings
4. **Import constants** from constants.py
5. **No magic numbers**

## Summary

- Configure in settings, not views
- Organize by endpoint
- Use help_text for documentation
- Import constants from constants.py
