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
from django.shortcuts import render, redirect
from django.contrib import messages


def user_login(request):
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        
        user = authenticate(request, username=username, password=password)
        
        if user is not None:
            login(request, user)
            
            # Redirect to next page
            next_page = request.GET.get('next', 'dashboard')
            return redirect(next_page)
        else:
            messages.error(request, 'Invalid username or password')
    
    return render(request, 'accounts/login.html')


def user_logout(request):
    logout(request)
    return redirect('login')


# urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('login/', views.user_login, name='login'),
    path('logout/', views.user_logout, name='logout'),
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
            # Send verification email
            send_verification_email(user)
        
        return user


# views.py
def register(request):
    if request.method == 'POST':
        form = UserRegistrationForm(request.POST)
        
        if form.is_valid():
            user = form.save()
            login(request, user)
            return redirect('dashboard')
    else:
        form = UserRegistrationForm()
    
    return render(request, 'accounts/register.html', {'form': form})
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
         auth_views.PasswordChangeView.as_view(), 
         name='password_change'),
    path('password_change/done/', 
         auth_views.PasswordChangeDoneView.as_view(), 
         name='password_change_done'),
    path('password_reset/', 
         auth_views.PasswordResetView.as_view(), 
         name='password_reset'),
    path('password_reset/done/', 
         auth_views.PasswordResetDoneView.as_view(), 
         name='password_reset_done'),
    path('reset/<uidb64>/<token>/', 
         auth_views.PasswordResetConfirmView.as_view(), 
         name='password_reset_confirm'),
    path('reset/done/', 
         auth_views.PasswordResetCompleteView.as_view(), 
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
        token['role'] = user.role
        
        return token


class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
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
