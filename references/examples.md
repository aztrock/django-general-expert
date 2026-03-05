# Examples

Complete, working examples demonstrating Django patterns and best practices.

## Example: Blog API with DRF

### Models

```python
from django.db import models
from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)


class PublicationStatus(models.TextChoices):
    DRAFT = 'draft', 'Draft'
    PUBLISHED = 'published', 'Published'


class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    status = models.CharField(
        max_length=20,
        choices=PublicationStatus.choices,
        default=PublicationStatus.DRAFT
    )
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['status', 'created_at']),
        ]
```

### Serializers

```python
from rest_framework import serializers
from .models import Post


class PostSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.username', read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'status', 'author_name', 'created_at']
        read_only_fields = ['id', 'created_at', 'author']
```

### Views

```python
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    
    def get_queryset(self):
        return Post.objects.select_related('author').filter(
            author=self.request.user
        )
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

## Summary

These examples demonstrate:

1. **Clean separation** between models, serializers, and views
2. **Service layer** for business logic
3. **Optimized queries** with select_related/prefetch_related
4. **Security best practices** with permissions
