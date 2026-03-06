# Middleware Guide

Guide to creating and using Django middleware for request/response processing.

## What is Middleware?

Middleware is a framework of hooks into Django's request/response processing. It's a light, low-level plugin system for globally altering Django's input or output.

```
Request → Middleware 1 → Middleware 2 → ... → View → Middleware N → Response
```

## Built-in Middleware

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

## Custom Middleware

### Basic Middleware

```python
# middleware.py
import logging
from django.utils.deprecation import MiddlewareMixin

logger = logging.getLogger(__name__)


class RequestLoggingMiddleware(MiddlewareMixin):
    def process_request(self, request):
        logger.info(f'{request.method} {request.path}')
    
    def process_response(self, request, response):
        logger.info(f'Response: {response.status_code}')
        return response


class ErrorLoggingMiddleware(MiddlewareMixin):
    def process_exception(self, request, exception):
        logger.exception(f'Error processing {request.path}: {exception}')
        return None


class AsyncRequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    async def __call__(self, request):
        logger.info(f'{request.method} {request.path}')
        
        response = await self.get_response(request)
        
        logger.info(f'Response: {response.status_code}')
        
        return response
```

### Authentication Middleware

```python
# middleware.py
import logging
from django.contrib.auth import get_user_model
from django.shortcuts import redirect
from django.utils.deprecation import MiddlewareMixin
from django.utils import timezone

logger = logging.getLogger(__name__)


class UserLastActiveMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if request.user.is_authenticated:
            User = get_user_model()
            User.objects.filter(pk=request.user.pk).update(
                last_active_at=timezone.now()
            )


class ForcePasswordChangeMiddleware(MiddlewareMixin):
    EXEMPT_URLS = [
        '/accounts/password_change/',
        '/accounts/logout/',
    ]
    
    def process_request(self, request):
        if not request.user.is_authenticated:
            return None
        
        if request.user.force_password_change:
            if request.path not in self.EXEMPT_URLS:
                if not request.path.startswith('/accounts/password_change/'):
                    return redirect('password_change')
        
        return None
```

### Request Validation

```python
# middleware.py
import json
from django.core.cache import cache
from django.http import JsonResponse
from django.utils.deprecation import MiddlewareMixin


class JSONBodyValidationMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if request.method in ['POST', 'PUT', 'PATCH']:
            if request.content_type == 'application/json':
                try:
                    if request.body:
                        request.json_data = json.loads(request.body)
                except json.JSONDecodeError:
                    return JsonResponse({
                        'error': 'Invalid JSON'
                    }, status=400)
        
        return None


class RateLimitMiddleware(MiddlewareMixin):
    def process_request(self, request):
        ip = request.META.get('REMOTE_ADDR')
        key = f'rate_{ip}'
        
        count = cache.get(key, 0)
        
        if count > 100:
            return JsonResponse({
                'error': 'Rate limit exceeded'
            }, status=429)
        
        cache.set(key, count + 1, 60)
        
        return None
```

### Response Processing

```python
# middleware.py
from django.conf import settings
from django.utils.deprecation import MiddlewareMixin


class AddHeadersMiddleware(MiddlewareMixin):
    def process_response(self, request, response):
        response['X-Application'] = 'MyDjangoApp'
        response['X-Frame-Options'] = 'DENY'
        return response


class RemoveWhitespaceMiddleware(MiddlewareMixin):
    def process_response(self, request, response):
        if response.status_code == 200:
            if 'text/html' in response.get('Content-Type', ''):
                if not settings.DEBUG:
                    pass
        return response
```

### CORS Middleware

**Use django-cors-headers (recommended for production):**

```bash
pip install django-cors-headers
```

```python
# settings.py
INSTALLED_APPS = [
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
]

CORS_ALLOWED_ORIGINS = [
    'https://example.com',
    'https://www.example.com',
]

CORS_ALLOW_CREDENTIALS = True
CORS_EXPOSE_HEADERS = ['Content-Length', 'Content-Range']
CORS_PREFLIGHT_CACHE = 86400
```

## Using Middleware in Settings

```python
# settings.py
MIDDLEWARE = [
    # Security
    'django.middleware.security.SecurityMiddleware',
    
    # Session & Auth
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    
    # Custom
    'myapp.middleware.RequestLoggingMiddleware',
    'myapp.middleware.RateLimitMiddleware',
    'myapp.middleware.CORSMiddleware',
    
    # Common
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    
    # Messages & Clickjacking
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

## Summary

1. **Middleware processes every request/response**
2. **Order matters** in MIDDLEWARE setting
3. **Keep middleware lightweight** - avoid heavy operations
4. **Use built-in middleware** when possible
5. **Return None** to continue processing
6. **Return response** to short-circuit
7. **Handle exceptions** in process_exception
8. **Consider async** for Django 3.1+
