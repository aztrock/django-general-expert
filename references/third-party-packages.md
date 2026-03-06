# Third-Party Packages Guide

Guide to selecting and integrating third-party packages for Django.

## Core Packages

### REST Framework & API

| Package | Purpose | Install |
|---------|---------|---------|
| djangorestframework | REST APIs | `pip install djangorestframework` |
| django-filter | Advanced filtering | `pip install django-filter` |
| drf-standardized-errors | Consistent error format | `pip install drf-standardized-errors` |
| djangorestframework-simplejwt | JWT auth | `pip install djangorestframework-simplejwt` |

### Database & ORM

| Package | Purpose | Install |
|---------|---------|---------|
| psycopg2-binary | PostgreSQL (dev) | `pip install psycopg2-binary` |
| django-redis | Redis cache | `pip install django-redis` |
| django-candyman | Connection pooling | `pip install django-candyman` |

### Task Queue & Celery

| Package | Purpose | Install |
|---------|---------|---------|
| celery | Async tasks | `pip install celery` |
| django-celery-beat | Scheduled tasks | `pip install django-celery-beat` |
| django-celery-results | Task results | `pip install django-celery-results` |

### File Storage

| Package | Purpose | Install |
|---------|---------|---------|
| django-storages | S3/Cloud storage | `pip install django-storages` |
| boto3 | AWS SDK | `pip install boto3` |

### Security

| Package | Purpose | Install |
|---------|---------|---------|
| django-cors-headers | CORS | `pip install django-cors-headers` |
| django-allauth | Social auth | `pip install django-allauth` |
| django-axes | Brute force protection | `pip install django-axes` |

### Testing

| Package | Purpose | Install |
|---------|---------|---------|
| factory-boy | Test fixtures | `pip install factory-boy` |
| faker | Fake data | `pip install faker` |

### Development

| Package | Purpose | Install |
|---------|---------|---------|
| django-debug-toolbar | Debug | `pip install django-debug-toolbar` |
| django-extensions | Extensions | `pip install django-extensions` |
| ipython | Shell | `pip install ipython` |
| whitenoise | Static files | `pip install whitenoise` |

### Frontend Integration

| Package | Purpose | Install |
|---------|---------|---------|
| django-crispy-forms | Forms | `pip install django-crispy-forms` |
| django-crispy-bootstrap5 | Bootstrap 5 | `pip install crispy-bootstrap5` |

## Package Selection Criteria

### 1. Maintenance Status

```python
# Check package is actively maintained
# - Last commit within 6 months
# - Open issues being addressed
# - Compatible with current Django version
```

### 2. Community Size

```python
# Good indicators:
# - GitHub stars > 1000
# - Active maintainers
# - Stack Overflow questions
# - PyPI downloads
```

### 3. Security

```python
# Always:
# - Check for vulnerabilities
# - Review permissions requested
# - Use official packages
# - Pin versions in production
```

## Installing Packages

```bash
# Requirements files
# requirements/base.txt
Django>=4.2,<5.0
djangorestframework>=3.14,<4.0

# Install from requirements
pip install -r requirements/base.txt


# Specific versions for production
# requirements/production.txt
Django==4.2.9
djangorestframework==3.14.0
psycopg2==2.9.9
```

## Package Configuration

### django-redis

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        }
    }
}
```

### django-cors-headers

```python
# settings.py
INSTALLED_APPS = [
    ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    ...
]

CORS_ALLOWED_ORIGINS = [
    'http://localhost:3000',
    'https://example.com',
]

CORS_ALLOW_CREDENTIALS = True
```

### django-allauth

```python
# settings.py
INSTALLED_APPS = [
    'django.contrib.auth',
    'django.contrib.sites',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
]

SITE_ID = 1

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

LOGIN_REDIRECT_URL = '/dashboard/'
ACCOUNT_LOGOUT_REDIRECT_URL = '/'
```

## Summary

1. **Use established packages** with good community support
2. **Pin versions** in production requirements
3. **Check compatibility** with your Django version
4. **Review security** before installing
5. **Keep updated** but test first
6. **Minimal dependencies** - avoid bloat
7. **Use django-extensions** for development
8. **Use django-filter** for complex APIs
9. **Use whitenoise** for static files in production
