# Project Structure Guide

Guide to organizing Django projects for maintainability and scalability.

## Recommended Project Structure

```
myproject/
├── myproject/                 # Project configuration
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py           # Common settings
│   │   ├── development.py    # Dev settings
│   │   ├── production.py     # Prod settings
│   │   └── test.py           # Test settings
│   ├── urls.py
│   ├── wsgi.py
│   ├── asgi.py
│   └── celery.py             # Celery configuration
├── apps/                     # Application code
│   ├── accounts/             # User management
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   ├── services.py
│   │   ├── selectors.py
│   │   ├── permissions.py
│   │   ├── tasks.py          # Celery tasks
│   │   ├── signals.py
│   │   ├── admin.py
│   │   ├── tests/
│   │   │   ├── __init__.py
│   │   │   └── test_models.py
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── urls.py
│   │   │   │   └── views.py
│   │   │   └── __init__.py
│   │   ├── migrations/
│   │   │   └── __init__.py
│   │   └── constants.py
│   ├── orders/
│   ├── products/
│   └── blog/
├── common/                   # Shared code
│   ├── __init__.py
│   ├── models.py             # Base models
│   ├── views.py              # Base views
│   ├── serializers.py        # Base serializers
│   ├── utils.py
│   └── exceptions.py
├── services/                 # Cross-app services
│   ├── __init__.py
│   ├── notification.py
│   └── payment.py
├── templates/                # HTML templates
│   ├── base.html
│   └── accounts/
├── static/                   # Static files
│   ├── css/
│   ├── js/
│   └── images/
├── media/                    # User uploads
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
├── .env                      # Environment variables
├── .gitignore
├── manage.py
├── pytest.ini
├── docker-compose.yml
├── Dockerfile
└── README.md
```

## Base Model Pattern

```python
# common/models.py
from django.db import models
from django.utils import timezone


class TimeStampedModel(models.Model):
    """Abstract base model with created and updated timestamps."""
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True


class UUIDModel(models.Model):
    """Abstract base model with UUID primary key."""
    import uuid
    
    id = models.UUIDField(
        primary_key=True,
        default=uuid.uuid4,
        editable=False
    )
    
    class Meta:
        abstract = True


# Usage
class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
```

## Settings Structure

```python
# settings/base.py
from pathlib import Path
import os

BASE_DIR = Path(__file__).resolve().parent.parent


SECRET_KEY = os.environ.get('SECRET_KEY')

DEBUG = False

ALLOWED_HOSTS = []


# settings/development.py
from .base import *

DEBUG = True

ALLOWED_HOSTS = ['*']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}


# settings/production.py
from .base import *

DEBUG = False

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': os.environ.get('DB_PORT'),
    }
}
```

## URL Configuration

```python
# project/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/accounts/', include('apps.accounts.api.v1.urls')),
    path('api/v1/orders/', include('apps.orders.api.v1.urls')),
    path('', include('apps.blog.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## Requirements Files

```python
# requirements/base.txt
Django>=4.2,<5.0
djangorestframework>=3.14,<4.0
psycopg2-binary>=2.9
django-filter>=23.0


# requirements/production.txt
-r base.txt
gunicorn>=20.0
whitenoise>=6.0
redis>=4.0
celery>=5.0
django-storages>=1.13
boto3>=1.26


# requirements/development.txt
-r base.txt
pytest>=7.0
pytest-django>=4.5
pytest-cov>=4.0
faker>=18.0
```

## Application Pattern

```python
# apps/orders/__init__.py
default_app_config = 'apps.orders.apps.OrdersConfig'


# apps/orders/apps.py
from django.apps import AppConfig


class OrdersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.orders'
    verbose_name = 'Orders'
    
    def ready(self):
        import apps.orders.signals  # noqa


# apps/orders/api/v1/__init__.py
# apps/orders/api/v1/urls.py
from django.urls import path
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register('orders', views.OrderViewSet, basename='order')

urlpatterns = router.urls
```

## Summary

1. **Use apps/ directory** for all applications
2. **Separate settings** for dev/prod
3. **Create common/ for shared code**
4. **Use base models** (TimeStampedModel, UUIDModel)
5. **Organize API by version** (v1/, v2/)
6. **Group related files** (models, views, services)
7. **Use requirements/** for dependencies
8. **Use .env for secrets**
9. **Follow Django conventions** for file naming
10. **Keep apps focused** (single responsibility)
