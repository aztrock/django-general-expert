# Security Checklist

Comprehensive security guide for Django applications covering OWASP Top 10 vulnerabilities.

## OWASP Top 10 in Django

### 1. SQL Injection

**Django ORM protects you automatically.**

```python
# ✅ SAFE: ORM parameterizes queries
User.objects.filter(email=user_input)

# ✅ SAFE: Raw queries with parameters
User.objects.raw('SELECT * FROM users WHERE email = %s', [user_input])

# ❌ DANGEROUS: String formatting
User.objects.raw(f'SELECT * FROM users WHERE email = "{user_input}"')
```

**Rule**: Never use string formatting or concatenation for SQL queries.

### 2. Cross-Site Scripting (XSS)

**Django templates auto-escape by default.**

```django
{# ✅ SAFE: Auto-escaped #}
<p>Hello, {{ user.name }}</p>

{# ❌ DANGEROUS: Marks as safe, disables escaping #}
<p>{{ user_bio|safe }}</p>
```

**Rule**: Never use `|safe` or `mark_safe()` on user-generated content.

### 3. Cross-Site Request Forgery (CSRF)

**Django's CSRF protection is enabled by default.**

```django
{# ✅ REQUIRED: Include CSRF token in forms #}
<form method="post">
  {% csrf_token %}
  ...
</form>
```

**Rule**: Never disable CSRF protection unless you have alternative authentication.

### 4. Broken Authentication

**Use Django's built-in authentication.**

```python
# ✅ GOOD: Use Django's password hashing
user = User.objects.create_user(username='user', password='raw_password')

# ❌ DANGEROUS: Storing passwords in plain text
user.password = 'raw_password'  # NEVER do this
user.save()
```

### 5. Broken Access Control

**Implement proper permissions.**

```python
# ✅ GOOD: Check permissions
from django.contrib.auth.decorators import login_required, permission_required

@login_required
def dashboard(request):
    ...

@permission_required('myapp.view_sensitive_data')
def sensitive_data(request):
    ...
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
ALLOWED_HOSTS = ['mydomain.com', 'www.mydomain.com']

# Security headers
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
```

## Authentication

### Custom User Model

```python
# models.py
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    
    class Meta:
        db_table = 'users'


# settings.py
AUTH_USER_MODEL = 'myapp.User'
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
```

## Summary

1. **SQL Injection**: Django ORM protects you
2. **XSS**: Templates auto-escape by default
3. **CSRF**: Enabled by default, never disable
4. **Authentication**: Use Django's built-in auth
5. **Access Control**: Implement proper permissions
6. **Security Config**: Never hardcode secrets
7. **HTTPS**: Always use HTTPS in production
8. **Input Validation**: Always validate user input
9. **Dependencies**: Keep updated, scan for vulnerabilities
10. **Logging**: Log security events
