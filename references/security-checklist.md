# Security Checklist

Comprehensive security guide for Django applications covering OWASP Top 10 vulnerabilities and Django-specific security best practices.

## OWASP Top 10 in Django

### 1. SQL Injection

**Django ORM protects you automatically, but there are exceptions.**

```python
# ✅ SAFE: ORM parameterizes queries
User.objects.filter(email=user_input)
User.objects.get(pk=user_id)

# ✅ SAFE: Raw queries with parameters
User.objects.raw('SELECT * FROM users WHERE email = %s', [user_input])

# ✅ SAFE: Extra select parameters
User.objects.extra(where=["email = %s"], params=[user_input])

# ❌ DANGEROUS: String formatting
User.objects.raw(f'SELECT * FROM users WHERE email = "{user_input}"')

# ❌ DANGEROUS: f-string in filter
User.objects.filter(email=user_input)  # Safe in Django ORM
# BUT:
User.objects.filter(username="'; DROP TABLE users; --")  # NEVER do this

# ❌ DANGEROUS: extra() with string formatting
User.objects.extra(
    where=["email = '%s'" % user_input]  # NEVER do this
)


# ✅ GOOD: Using Q objects
from django.db.models import Q
User.objects.filter(
    Q(email=user_input) | Q(username=user_input)
)


# ✅ GOOD: Using Like with proper escaping
from django.db.models import CharField, Value
from django.db.models.functions import Concat
User.objects.annotate(
    search=Concat('email', Value(' '), 'username')
).filter(search__icontains=user_input)
```

### 2. Cross-Site Scripting (XSS)

**Django templates auto-escape by default, but there are exceptions.**

```django
{# ✅ SAFE: Auto-escaped by default #}
<p>Hello, {{ user.name }}</p>
<p>{{ user.bio }}</p>
<p>{{ post.content }}</p>


{# ❌ DANGEROUS: Marks as safe, disables escaping #}
<p>{{ user_bio|safe }}</p>
{% autoescape off %}
    {{ user_input }}  # Danger!
{% endautoescape %}


{# ✅ SAFE: Using escape filter explicitly #}
<p>{{ user_input|escape }}</p>


{# ✅ GOOD: Using conditional escaping #}
{% autoescape off %}
    {% if user.is_trusted %}
        {{ user.html_content }}
    {% else %}
        {{ user.html_content|escape }}
    {% endif %}
{% endautoescape %}
```

**In Python code:**

```python
from django.utils.safestring import mark_safe

# ❌ DANGEROUS: mark_safe with user input
mark_safe(user_input)  # NEVER

# ✅ GOOD: mark_safe with trusted content
mark_safe('<b>Trusted content</b>')

# ✅ GOOD: Using format_html
from django.utils.html import format_html, format_html_join

format_html('<span>{}</span>', user_input)  # Auto-escaped
format_html_join(
    '',
    '<p>{}</p>',
    [(u.name,) for u in users]
)
```

### 3. Cross-Site Request Forgery (CSRF)

**Django's CSRF protection is enabled by default.**

```django
{# ✅ REQUIRED: Include CSRF token in forms #}
<form method="post">
    {% csrf_token %}
    ...
</form>


{# ✅ GOOD: AJAX with CSRF #}
<script>
    function getCookie(name) {
        let cookieValue = null;
        if (document.cookie && document.cookie !== '') {
            const cookies = document.cookie.split(';');
            for (let i = 0; i < cookies.length; i++) {
                const cookie = cookies[i].trim();
                if (cookie.substring(0, name.length + 1) === (name + '=')) {
                    cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                    break;
                }
            }
        }
        return cookieValue;
    }
    
    fetch('/api/data/', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRFToken': getCookie('csrftoken')
        },
        body: JSON.stringify(data)
    });
</script>


{# ❌ DANGEROUS: Disable CSRF for public forms #}
{% csrf_token %}
{# If you must disable (rare), use @csrf_exempt in views only #}
```

**In views:**

```python
from django.views.decorators.csrf import csrf_exempt

# Only use when truly necessary (e.g., webhooks)
@csrf_exempt
def webhook_handler(request):
    ...
```

### 4. Broken Authentication

**Use Django's built-in authentication properly.**

```python
# ✅ GOOD: Use Django's password hashing
from django.contrib.auth.models import User

user = User.objects.create_user(
    username='john',
    email='john@example.com',
    password='raw_password'  # Django hashes it automatically
)

# Check password
user.check_password('raw_password')  # Returns True/False


# ❌ DANGEROUS: Storing passwords in plain text
user.password = 'raw_password'  # NEVER do this
user.save()


# ❌ DANGEROUS: Using weak hashing
import hashlib
user.password = hashlib.md5(password).hexdigest()  # NEVER


# ✅ GOOD: Custom password validation
from django.contrib.auth.password_validation import (
    validate_password,
    MinimumLengthValidator,
    CommonPasswordValidator,
    NumericPasswordValidator,
)

# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
        "OPTIONS": {
            "min_length": 9,
        },
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
]


# ✅ GOOD: Password reset with proper token
from django.contrib.auth.tokens import default_token_generator
from django.utils.http import urlsafe_base64_encode
from django.utils.encoding import force_bytes

def password_reset_request(request):
    if request.method == 'POST':
        email = request.POST.get('email')
        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return JsonResponse({'error': 'Email not found'})
        
        token = default_token_generator.make_token(user)
        uid = urlsafe_base64_encode(force_bytes(user.pk))
        
        # Send email with reset link
        send_password_reset_email(user, token, uid)
        
        return JsonResponse({'message': 'Password reset email sent'})


### Password Hashers

```python
# settings.py
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
]

# Recommended: Argon2 (install: pip install argon2-cffi)
# Fallback: PBKDF2 (default in Django)
```


### Rate Limiting

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
        'login': '5/minute',
        'api': '100/hour',
    },
}


# views.py - Custom throttle
from rest_framework.throttling import UserRateThrottle


class LoginThrottle(UserRateThrottle):
    scope = 'login'


class PostViewSet(viewsets.ModelViewSet):
    throttle_classes = [LoginThrottle]
    ...


# Scoped throttle for specific endpoints
class SecureViewSet(viewsets.ModelViewSet):
    throttle_classes = [
        'rest_framework.throttling.AnonRateThrottle',
    ]
    throttle_scope = 'api'
```


### Session Management

```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'

SESSION_COOKIE_AGE = 3600 * 24 * 7  # 1 week
SESSION_SAVE_EVERY_REQUEST = False
SESSION_EXPIRE_AT_BROWSER_CLOSE = False

# Secure session cookies
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Lax'
```
```

### 5. Broken Access Control

**Implement proper permissions at every level.**

```python
# ✅ GOOD: Object-level permissions
from rest_framework import permissions


class IsOwner(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user or request.user.is_staff


# ✅ GOOD: View-level permissions
class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def get_queryset(self):
        # Filter by user
        return Post.objects.filter(owner=self.request.user)


# ✅ GOOD: Using @login_required
from django.contrib.auth.decorators import login_required, permission_required

@login_required
def dashboard(request):
    ...


@permission_required('orders.view_order', raise_exception=True)
def order_list(request):
    ...


# ✅ GOOD: Model-level permissions
class Order(models.Model):
    # Use built-in permissions
    class Meta:
        permissions = [
            ("view_order", "Can view orders"),
            ("process_order", "Can process orders"),
        ]
```

### 6. Security Misconfiguration

**Production settings checklist:**

```python
# settings.py - PRODUCTION

# ❌ NEVER IN PRODUCTION
DEBUG = True
SECRET_KEY = 'hardcoded-secret-key'
ALLOWED_HOSTS = ['*']

# ✅ GOOD: Production settings
import os

DEBUG = False
SECRET_KEY = os.environ.get('SECRET_KEY')

# Only allow specific hosts
ALLOWED_HOSTS = [
    'example.com',
    'www.example.com',
    os.environ.get('ALLOWED_HOST', ''),
]

# Security headers
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# Content Security Policy
SECURE_CONTENT_TYPE_NOSNIFF = True

# Prevent clickjacking
X_FRAME_OPTIONS = 'DENY'

# HTTPS only
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')


# Database
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

### 7. Insecure Deserialization

**Be careful with pickle and untrusted data.**

```python
import pickle
import json

# ❌ DANGEROUS: Never unpickle untrusted data
data = pickle.loads(untrusted_data)

# ✅ GOOD: Use JSON for untrusted data
data = json.loads(untrusted_data)

# ✅ GOOD: Use Django's JSON field
class Order(models.Model):
    metadata = models.JSONField(default=dict)
```

### 8. Using Components with Known Vulnerabilities

```bash
# ✅ Keep dependencies updated
pip list --outdated

# ✅ Use safety checking
pip install safety
safety check

# ✅ Use Snyk or similar
pip install snyk
snyk test

# requirements.txt - pin versions
Django==4.2.9
djangorestframework==3.14.0
psycopg2-binary==2.9.9
```

### 9. Insufficient Logging & Monitoring

```python
import logging

logger = logging.getLogger(__name__)


# ✅ GOOD: Log security events
def login_view(request):
    username = request.POST.get('username')
    user = authenticate(username=username, password=password)
    
    if user:
        logger.info(f'User {username} logged in', extra={
            'user_id': user.id,
            'ip': get_client_ip(request)
        })
    else:
        logger.warning(f'Failed login attempt for {username}', extra={
            'ip': get_client_ip(request),
            'user_agent': request.META.get('HTTP_USER_AGENT')
        })


# ✅ GOOD: Log access to sensitive data
def access_sensitive_data(request, data_id):
    data = get_object_or_404(SensitiveData, pk=data_id)
    
    logger.info(f'User {request.user} accessed data {data_id}', extra={
        'user_id': request.user.id,
        'data_id': data_id,
        'ip': get_client_ip(request)
    })
    
    return data


# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'WARNING',
            'class': 'logging.FileHandler',
            'filename': 'security.log',
        },
    },
    'loggers': {
        'django.security': {
            'handlers': ['file'],
            'level': 'WARNING',
        },
    },
}
```

### 10. Server-Side Request Forgery (SSRF)

**Validate and sanitize URLs.**

```python
import requests
from urllib.parse import urlparse

# ❌ DANGEROUS: Allow arbitrary URLs
def fetch_url(url):
    return requests.get(url)


# ✅ GOOD: Validate URL against allowed hosts
ALLOWED_HOSTS = ['example.com', 'api.example.com']

def fetch_url(url):
    parsed = urlparse(url)
    
    if parsed.scheme not in ['http', 'https']:
        raise ValueError('Invalid scheme')
    
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError('Hostname not allowed')
    
    if parsed.port and parsed.port not in [80, 443]:
        raise ValueError('Port not allowed')
    
    return requests.get(url, timeout=10)


# ✅ GOOD: Use Django's URLValidator
from django.core.validators import URLValidator
from django.core.exceptions import ValidationError

validator = URLValidator(schemes=['http', 'https'], host_whitelist=['example.com'])

def validate_url(url):
    try:
        validator(url)
        return True
    except ValidationError:
        return False
```

## Authentication

### Custom User Model

```python
# models.py
from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    phone = models.CharField(max_length=20, blank=True)
    is_verified = models.BooleanField(default=False)
    
    class Meta:
        db_table = 'users'


# settings.py
AUTH_USER_MODEL = 'myapp.User'
```

### JWT Authentication

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Install: pip install djangorestframework-simplejwt


# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

## Input Validation

```python
# forms.py
class ContactForm(forms.Form):
    email = forms.EmailField()
    message = forms.CharField(max_length=1000)
    
    def clean_message(self):
        message = self.cleaned_data['message']
        
        # Check for spam
        spam_words = ['viagra', 'casino', 'lottery']
        if any(word in message.lower() for word in spam_words):
            raise forms.ValidationError('Message contains forbidden words')
        
        return message


# serializers.py
from rest_framework import serializers


class OrderSerializer(serializers.Serializer):
    email = serializers.EmailField()
    quantity = serializers.IntegerField(min_value=1, max_value=100)
    price = serializers.DecimalField(max_digits=10, decimal_places=2, min_value=0)
    
    def validate_email(self, value):
        # Additional email validation
        if value.endswith('@tempmail.com'):
            raise serializers.ValidationError('Temporary emails not allowed')
        return value
```

## File Upload Security

```python
# models.py
class Document(models.Model):
    file = models.FileField(upload_to='documents/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
    
    def save(self, *args, **kwargs):
        # Validate file extension
        allowed_extensions = ['.pdf', '.doc', '.docx', '.txt']
        ext = os.path.splitext(self.file.name)[1].lower()
        
        if ext not in allowed_extensions:
            raise ValidationError('File type not allowed')
        
        # Validate file size (max 10MB)
        if self.file.size > 10 * 1024 * 1024:
            raise ValidationError('File too large')
        
        super().save(*args, **kwargs)


# views.py
from django.core.validators import FileExtensionValidator

def upload_file(request):
    form = UploadForm(request.POST, request.FILES)
    
    if form.is_valid():
        # File is already validated
        file = form.cleaned_data['file']
        
        # Additional checks
        if file.content_type not in ['application/pdf', 'text/plain']:
            return JsonResponse({'error': 'Invalid content type'}, status=400)
        
        # Save securely
        # ...
```

## Security Headers

```python
# settings.py - Using django-csp (recommended)
MIDDLEWARE = [
    ...
    'csp.middleware.CSPMiddleware',
]

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "'unsafe-inline'")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
CSP_IMG_SRC = ("'self'", "data:", "https:")


# settings.py - Without django-csp
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
```

## Summary

1. **SQL Injection**: Django ORM protects you, but avoid raw queries with string formatting
2. **XSS**: Templates auto-escape by default, never use |safe on user input
3. **CSRF**: Enabled by default, use {% csrf_token %} in all forms
4. **Authentication**: Use Django's built-in, never store passwords in plain text
5. **Password Hashers**: Use Argon2 or PBKDF2 (never MD5 or SHA1)
6. **Access Control**: Implement at model, view, and API levels
7. **Security Config**: Never hardcode secrets, use environment variables
8. **Deserialization**: Use JSON, never unpickle untrusted data
9. **Dependencies**: Keep updated, use safety checking
10. **Logging**: Log security events for monitoring
11. **URLs**: Validate and sanitize to prevent SSRF
12. **Files**: Validate type, size, and content
13. **Headers**: Use security headers in production
14. **HTTPS**: Always use in production
15. **Rate limiting**: Protect against abuse (brute force, DoS)
16. **Session Management**: Secure session cookies, proper session expiration
