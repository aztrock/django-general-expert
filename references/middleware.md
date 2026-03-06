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
from django.utils.deprecation import MiddlewareMixin


class RequestLoggingMiddleware(MiddlewareMixin):
    def process_request(self, request):
        # Log request
        print(f'{request.method} {request.path}')
    
    def process_response(self, request, response):
        # Log response
        print(f'Response: {response.status_code}')
        return response


# With error handling
class ErrorLoggingMiddleware(MiddlewareMixin):
    def process_exception(self, request, exception):
        # Log exception
        logger.exception(f'Error processing {request.path}: {exception}')
        return None  # Let Django handle it


# Using async (Django 3.1+)
import logging

logger = logging.getLogger(__name__)


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
from django.contrib.auth import get_user_model
from django.utils.deprecation import MiddlewareMixin


class UserLastActiveMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if request.user.is_authenticated:
            # Update last active
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
                    from django.shortcuts import redirect
                    return redirect('password_change')
        
        return None
```

### Request Validation

```python
# middleware.py
from django.http import JsonResponse
from django.core.exceptions import ValidationError
import json


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
        # Simple rate limiting
        from django.core.cache import cache
        
        ip = request.META.get('REMOTE_ADDR')
        key = f'rate_{ip}'
        
        count = cache.get(key, 0)
        
        if count > 100:  # 100 requests per minute
            return JsonResponse({
                'error': 'Rate limit exceeded'
            }, status=429)
        
        cache.set(key, count + 1, 60)
        
        return None
```

### Response Processing

```python
# middleware.py
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
                # Don't modify in debug mode
                if not settings.DEBUG:
                    # Whitespace removal would go here
                    pass
        return response


class GZipMiddleware(MiddlewareMixin):
    def process_response(self, request, response):
        # Check if client accepts gzip
        if 'gzip' in request.META.get('HTTP_ACCEPT_ENCODING', ''):
            # Would implement compression here
            pass
        return response
```

### CORS Middleware

```python
# middleware.py
from django.http import JsonResponse
from django.utils.deprecation import MiddlewareMixin


class CORSMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if request.method == 'OPTIONS':
            response = JsonResponse({})
            response['Access-Control-Allow-Origin'] = '*'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
            return response
        
        return None
    
    def process_response(self, request, response):
        response['Access-Control-Allow-Origin'] = '*'
        response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        return response


# Or use django-cors-headers
# pip install django-cors-headers
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
