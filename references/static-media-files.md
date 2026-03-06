# Static & Media Files Guide

Guide to managing static and media files in Django.

## Basic Configuration

```python
# settings.py
import os

# Base directory
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

# Static files (CSS, JS, images)
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'static'),
]

# Media files (user uploads)
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# Finders
STATICFILES_FINDERS = [
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    'django.contrib.staticfiles.finders.CompressorFinder',
]


# Whitenoise for production
MIDDLEWARE = [
    # ...
    'whitenoise.middleware.WhiteNoiseMiddleware',
]

# WhiteNoise configuration
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

## FileField Configuration

```python
# models.py
import os
from django.db import models


def upload_to(instance, filename):
    """Generate upload path based on model and date."""
    ext = filename.split('.')[-1]
    filename = f'{instance.pk or "new"}.{ext}'
    
    return os.path.join(
        'uploads',
        instance.__class__.__name__.lower(),
        filename
    )


class Document(models.Model):
    title = models.CharField(max_length=200)
    file = models.FileField(upload_to='documents/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.title


class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    avatar = models.ImageField(
        upload_to='avatars/',
        default='avatars/default.png',
        blank=True
    )
    
    def get_avatar_url(self):
        if self.avatar:
            return self.avatar.url
        return '/static/img/default-avatar.png'


# Image field with thumbnails
class Post(models.Model):
    title = models.CharField(max_length=200)
    image = models.ImageField(upload_to='posts/')
    thumbnail = models.ImageField(
        upload_to='posts/thumbnails/',
        blank=True,
        null=True
    )
```

## Serving Media in Development

```python
# urls.py
from django.conf import settings
from django.conf.urls.static import static
from django.contrib.staticfiles.urls import staticfiles_urlpatterns

urlpatterns = [
    # ... your url patterns
]

# Development only
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
    urlpatterns += staticfiles_urlpatterns()
```

## Image Processing with Pillow

```python
# utils.py
from PIL import Image
import os


def create_thumbnail(image_path, thumbnail_path, size=(300, 300)):
    """Create thumbnail from image."""
    img = Image.open(image_path)
    img.thumbnail(size, Image.Resampling.LANCZOS)
    img.save(thumbnail_path)


def optimize_image(image_path, quality=85):
    """Optimize image size."""
    img = Image.open(image_path)
    
    if img.mode == 'RGBA':
        img = img.convert('RGB')
    
    img.save(image_path, 'JPEG', quality=quality, optimize=True)


# Using in model
from django.db import models
from django.dispatch import receiver
from django.core.files import File
import tempfile


@receiver(models.signals.post_save, sender=Post)
def create_post_thumbnail(sender, instance, **kwargs):
    if instance.image and not instance.thumbnail:
        # Create thumbnail
        img = Image.open(instance.image.path)
        img.thumbnail((300, 300), Image.Resampling.LANCZOS)
        
        # Save to temp file
        with tempfile.NamedTemporaryFile(suffix='.jpg', delete=False) as tmp:
            img.save(tmp.name, 'JPEG')
            
            # Save to thumbnail field
            instance.thumbnail.save(
                f'thumb_{instance.image.name}',
                File(open(tmp.name, 'rb')),
                save=True
            )
            
            os.unlink(tmp.name)
```

## File Validation

```python
# models.py
from django.core.exceptions import ValidationError
import os


def validate_file_extension(value):
    """Validate file extension."""
    ext = os.path.splitext(value.name)[1].lower()
    allowed_extensions = ['.jpg', '.jpeg', '.png', '.gif', '.pdf']
    
    if ext not in allowed_extensions:
        raise ValidationError(
            f'File type not allowed. Allowed types: {", ".join(allowed_extensions)}'
        )


def validate_file_size(value):
    """Validate file size."""
    limit = 10 * 1024 * 1024  # 10MB
    
    if value.size > limit:
        raise ValidationError('File too large. Maximum size is 10MB.')


class Document(models.Model):
    file = models.FileField(
        upload_to='documents/',
        validators=[validate_file_extension, validate_file_size]
    )
```

## S3/Media Storage

```python
# settings.py
# Using django-storages with S3
INSTALLED_APPS = [
    # ...
    'storages',
]

# AWS S3 Configuration
AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
AWS_S3_REGION_NAME = 'us-east-1'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'

# Media files (user uploads)
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'


# Custom storage classes
class MediaStorage(S3Boto3Storage):
    location = 'media'
    file_overwrite = False


class StaticStorage(S3Boto3Storage):
    location = 'static'
    default_acl = 'public-read'


# Use specific storage
class Document(models.Model):
    file = models.FileField(
        upload_to='documents/',
        storage=MediaStorage()
    )
```

## Summary

1. **Use STATIC_URL** for CSS/JS/images
2. **Use MEDIA_URL** for user uploads
3. **Set STATIC_ROOT** for collectstatic
4. **Use MEDIA_ROOT** for file storage
5. **Serve media in dev** with static() url
6. **Use WhiteNoise** for production static
7. **Validate file types** and sizes
8. **Create thumbnails** for images
9. **Use django-storages** for S3
10. **Use upload_to** for organized paths
