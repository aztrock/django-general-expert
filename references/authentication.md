# Django Authentication Guide

Guide to implementing authentication in Django applications.

## Custom User Model

```python
# models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    # Additional fields
    phone = models.CharField(max_length=20, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    bio = models.TextField(max_length=500, blank=True)
    
    # Verification
    is_verified = models.BooleanField(default=False)
    verification_token = models.CharField(max_length=100, blank=True)
    
    # Preferences
    email_notifications = models.BooleanField(default=True)
    language = models.CharField(max_length=10, default='en')
    timezone = models.CharField(max_length=50, default='UTC')
    
    class Meta:
        db_table = 'users'
    
    def __str__(self):
        return self.username
    
    def get_full_name(self):
        return f'{self.first_name} {self.last_name}'.strip() or self.username


# settings.py
AUTH_USER_MODEL = 'accounts.User'
```

## Login/Logout

```python
# views.py
from django.contrib.auth import authenticate, login, logout
from django.views.generic import FormView, View, RedirectView
from django.shortcuts import redirect
from django.contrib import messages


class LoginView(FormView):
    template_name = 'accounts/login.html'
    form_class = None  # Use Django's AuthenticationForm if needed
    success_url = 'dashboard'

    def form_valid(self, form):
        username = self.request.POST.get('username')
        password = self.request.POST.get('password')
        
        user = authenticate(self.request, username=username, password=password)
        
        if user is not None:
            login(self.request, user)
            next_page = self.request.GET.get('next', self.success_url)
            return redirect(next_page)
        else:
            messages.error(self.request, 'Invalid username or password')
            return self.form_invalid(form)


class LogoutView(RedirectView):
    url = 'login'
    
    def get_redirect_url(self, *args, **kwargs):
        logout(self.request)
        return super().get_redirect_url(*args, **kwargs)


# urls.py
from django.urls import path
from .views import LoginView, LogoutView

urlpatterns = [
    path('login/', LoginView.as_view(), name='login'),
    path('logout/', LogoutView.as_view(), name='logout'),
]
```

## Registration

```python
# forms.py
from django import forms
from django.contrib.auth.forms import UserCreationForm
from .models import User


class UserRegistrationForm(UserCreationForm):
    email = forms.EmailField(required=True)
    phone = forms.CharField(max_length=20, required=False)
    
    class Meta:
        model = User
        fields = ['username', 'email', 'phone', 'password1', 'password2']
    
    def clean_email(self):
        email = self.cleaned_data.get('email')
        
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('Email already registered')
        
        return email
    
    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data['email']
        
        if commit:
            user.save()
            send_verification_email(user)
        
        return user


# views.py
from django.views.generic import FormView
from django.contrib.auth import login
from django.shortcuts import redirect


class RegisterView(FormView):
    template_name = 'accounts/register.html'
    form_class = UserRegistrationForm
    success_url = 'dashboard'

    def form_valid(self, form):
        user = form.save()
        login(self.request, user)
        return redirect(self.success_url)


# urls.py
from django.urls import path
from .views import RegisterView

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
]
```

## Password Management

```python
# forms.py
from django.contrib.auth.forms import (
    PasswordChangeForm,
    PasswordResetForm,
    SetPasswordForm
)


class CustomPasswordChangeForm(PasswordChangeForm):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        for field in self.fields.values():
            field.widget.attrs['class'] = 'form-control'


# views.py
from django.contrib.auth.views import (
    PasswordChangeView,
    PasswordResetView,
    PasswordResetDoneView,
    PasswordResetConfirmView,
    PasswordResetCompleteView
)
from django.urls import reverse_lazy


class CustomPasswordChangeView(PasswordChangeView):
    form_class = CustomPasswordChangeForm
    success_url = reverse_lazy('password_change_done')
    template_name = 'accounts/password_change.html'


# urls.py
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('password_change/', 
         auth_views.PasswordChangeView.as_view(
             template_name='accounts/password_change.html'
         ), 
         name='password_change'),
    path('password_change/done/', 
         auth_views.PasswordChangeDoneView.as_view(
             template_name='accounts/password_change_done.html'
         ), 
         name='password_change_done'),
    path('password_reset/', 
         auth_views.PasswordResetView.as_view(
             template_name='accounts/password_reset.html',
             email_template_name='accounts/password_reset_email.html',
         ), 
         name='password_reset'),
    path('password_reset/done/', 
         auth_views.PasswordResetDoneView.as_view(
             template_name='accounts/password_reset_done.html'
         ), 
         name='password_reset_done'),
    path('reset/<uidb64>/<token>/', 
         auth_views.PasswordResetConfirmView.as_view(
             template_name='accounts/password_reset_confirm.html'
         ), 
         name='password_reset_confirm'),
    path('reset/done/', 
         auth_views.PasswordResetCompleteView.as_view(
             template_name='accounts/password_reset_complete.html'
         ), 
         name='password_reset_complete'),
]
```

## JWT Authentication (DRF)

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

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
}

# For custom user model with UUID primary key:
# SIMPLE_JWT = {
#     'USER_ID_FIELD': 'public_id',
#     'USER_ID_CLAIM': 'user_id',
# }

# Install: pip install djangorestframework-simplejwt


# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenBlacklistView,
)
from django.urls import path

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/blacklist/', TokenBlacklistView.as_view(), name='token_blacklist'),
]


# Custom token claims
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer


class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # Add custom claims
        token['email'] = user.email
        token['is_verified'] = user.is_verified
        
        return token


class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```


### Sliding Tokens vs Regular JWT

**Regular JWT (default):**
- Short access token, separate refresh token
- Better security (stateless)
- Requires refresh before expiration

**Sliding Tokens:**
- Single token that refreshes on each use
- Better user experience (session-like)
- Use when: user experience priority over maximum security

```python
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'SLIDING_TOKEN_LIFETIME': timedelta(days=1),
    'SLIDING_TOKEN_REFRESH_LIFETIME': timedelta(days=1),
}
```

## Permission Classes

```python
# permissions.py
from rest_framework import permissions


class IsOwner(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user or request.user.is_staff


class IsVerifiedUser(permissions.BasePermission):
    def has_permission(self, request, view):
        return (
            request.user and 
            request.user.is_authenticated and 
            request.user.is_verified
        )


class IsAdminOrReadOnly(permissions.BasePermission):
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_staff


# views.py
class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwner]
```

## Login Required

```python
# Function-based view
from django.contrib.auth.decorators import login_required


@login_required
def dashboard(request):
    return render(request, 'dashboard.html')


# Class-based view
from django.contrib.auth.mixins import LoginRequiredMixin


class DashboardView(LoginRequiredMixin, View):
    login_url = '/login/'
    redirect_field_name = 'next'
    
    def get(self, request):
        return render(request, 'dashboard.html')


# With verification
from django.contrib.auth.mixins import UserPassesTestMixin


class VerifiedUserMixin(UserPassesTestMixin):
    def test_func(self):
        return self.request.user.is_authenticated and self.request.user.is_verified
```

## Social Authentication

```python
# settings.py
# pip install django-allauth

INSTALLED_APPS = [
    'django.contrib.auth',
    'django.contrib.sites',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.github',
]

SITE_ID = 1

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

# Social providers
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'APP': {
            'client_id': os.environ.get('GOOGLE_CLIENT_ID'),
            'secret': os.environ.get('GOOGLE_SECRET'),
            'key': '',
        }
    }
}

LOGIN_REDIRECT_URL = 'dashboard'
ACCOUNT_LOGOUT_REDIRECT_URL = 'home'
```


## Brute Force Protection (django-axes)

```python
# settings.py
# pip install django-axes
from datetime import timedelta

INSTALLED_APPS = [
    'axes',
]

AUTHENTICATION_BACKENDS = [
    'axes.backends.AxesBackend',
    'django.contrib.auth.backends.ModelBackend',
]

AXES_FAILURE_LIMIT = 5
AXES_COOLOFF_TIME = timedelta(minutes=15)

# Handler options:
# 'axes.handlers.database.AxesDatabaseHandler' - Store in DB
# 'axes.handlers.cache.AxesCacheHandler' - Store in cache (Redis)
# 'axes.handlers.redis.AxesRedisHandler' - Store in Redis
AXES_HANDLER = 'axes.handlers.database.AxesDatabaseHandler'

# urls.py
from django.urls import path

urlpatterns = [
    path('admin/axes/', include('axes.urls')),
]
```


## dj-rest-auth (DRF Authentication Endpoints)

```python
# settings.py
# pip install dj-rest-auth

INSTALLED_APPS = [
    'rest_framework',
    'rest_framework.authtoken',
    'dj_rest_auth',
]

REST_AUTH = {
    'USER_SERIALIZERS': {
        'USER': 'myapp.serializers.UserSerializer',
    },
    'LOGIN_SERIALIZERS': {
        'LOGIN': 'myapp.serializers.LoginSerializer',
    },
    'PASSWORD_RESET_SERIALIZERS': {
        'PASSWORD_RESET': 'myapp.serializers.PasswordResetSerializer',
    },
}


# urls.py
from django.urls import path
from dj_rest_auth.views import (
    LoginView,
    LogoutView,
    UserDetailsView,
    PasswordChangeView,
    PasswordResetView,
    PasswordResetConfirmView,
)

urlpatterns = [
    path('api/auth/login/', LoginView.as_view(), name='rest_login'),
    path('api/auth/logout/', LogoutView.as_view(), name='rest_logout'),
    path('api/auth/user/', UserDetailsView.as_view(), name='rest_user_details'),
    path('api/auth/password/change/', PasswordChangeView.as_view(), name='rest_password_change'),
    path('api/auth/password/reset/', PasswordResetView.as_view(), name='rest_password_reset'),
    path('api/auth/password/reset/confirm/', PasswordResetConfirmView.as_view(), name='rest_password_reset_confirm'),
]
```

## Session Management

```python
# settings.py
# Session security
SESSION_COOKIE_AGE = 60 * 60 * 24  # 24 hours
SESSION_COOKIE_SECURE = True
SESSION_EXPIRE_AT_BROWSER_CLOSE = False

# Login throttling
from django.contrib.auth import login
from django.core.cache import cache


def login_with_throttle(request, user):
    # Rate limiting
    ip = request.META.get('REMOTE_ADDR')
    cache_key = f'login_attempts_{ip}'
    attempts = cache.get(cache_key, 0)
    
    if attempts >= 5:
        raise ValidationError('Too many login attempts. Please try again later.')
    
    cache.set(cache_key, attempts + 1, 60 * 15)  # 15 minutes
    
    login(request, user)
```

## Summary

1. **Use custom user model** from the start
2. **Use built-in auth forms** when possible
3. **Use JWT for APIs**, sessions for template views
4. **Implement proper permissions** at view level
5. **Use login_required decorator** for protected views
6. **Handle password reset** securely
7. **Use social auth** with django-allauth
8. **Implement rate limiting** to prevent brute force
9. **Verify email** for important actions
10. **Use HTTPS** in production for authentication
