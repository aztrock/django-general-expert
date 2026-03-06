# Signals Guide

Guide to using Django signals effectively. Signals allow decoupled applications to receive notifications when certain events occur.

## When to Use Signals

Signals are appropriate when you need **cross-cutting concerns** that affect multiple parts of your application without direct dependencies:

- **Audit logging** - Track changes to models
- **Denormalization** - Update cached/count fields
- **Cross-app communication** - Notify other apps of changes
- **External integrations** - Trigger actions in external systems
- **Cache invalidation** - Clear related caches

## When to Avoid Signals

Signals can make code harder to debug because the side effects aren't visible in the code that triggers them. Avoid signals for:

- **Simple business logic** - Use service layer instead
- **Complex multi-step operations** - Use tasks/celery
- **Modifying multiple models in chains** - Use transactions
- **Logic tightly coupled to the model** - Put in model methods

## Signal Types

### Model Signals

```python
from django.db.models.signals import (
    pre_init, post_init,
    pre_save, post_save,
    pre_delete, post_delete,
    m2m_changed
)
from django.dispatch import receiver


# Called before model is initialized
@receiver(pre_init, sender=Order)
def order_pre_init(sender, **kwargs):
    instance = kwargs.get('instance')
    print(f'Order {instance} is being initialized')


# Called after model is initialized
@receiver(post_init, sender=Order)
def order_post_init(sender, **kwargs):
    instance = kwargs.get('instance')
    print(f'Order {instance} initialized')


# Called before model is saved
@receiver(pre_save, sender=Order)
def order_pre_save(sender, **kwargs):
    instance = kwargs.get('instance')
    if instance.pk:
        print(f'Order {instance.pk} is being updated')
    else:
        print('New order is being created')


# Called after model is saved
@receiver(post_save, sender=Order)
def order_post_save(sender, **kwargs):
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created:
        print(f'New order {instance.pk} created')
    else:
        print(f'Order {instance.pk} updated')


# Called before model is deleted
@receiver(pre_delete, sender=Order)
def order_pre_delete(sender, **kwargs):
    instance = kwargs.get('instance')
    print(f'Order {instance.pk} is being deleted')


# Called after model is deleted
@receiver(post_delete, sender=Order)
def order_post_delete(sender, **kwargs):
    instance = kwargs.get('instance')
    print(f'Order {instance.pk} deleted')


# Called when M2M relationship changes
@receiver(m2m_changed, sender=Post.tags.through)
def post_tags_changed(sender, **kwargs):
    action = kwargs.get('action')
    instance = kwargs.get('instance')
    
    if action == 'post_add':
        print(f'Tags added to post {instance.pk}')
    elif action == 'post_remove':
        print(f'Tags removed from post {instance.pk}')
    elif action == 'post_clear':
        print(f'Tags cleared from post {instance.pk}')
```

### Request Signals

```python
from django.core.signals import request_started, request_finished
from django.http import JsonResponse


@receiver(request_started)
def request_started_handler(**kwargs):
    """Log when a request starts."""
    print('Request started')


@receiver(request_finished)
def request_finished_handler(**kwargs):
    """Log when a request finishes."""
    print('Request finished')
```

### Custom Signals

```python
import django.dispatch


# Define a custom signal
order_processed = django.dispatch.Signal()
order_cancelled = django.dispatch.Signal()


# Send custom signal
def process_order(order):
    # Process order
    order.status = OrderStatus.PROCESSED
    order.save()
    
    # Send signal
    order_processed.send(sender=Order, order=order)


# Connect to custom signal
@receiver(order_processed)
def handle_order_processed(sender, order, **kwargs):
    print(f'Order {order.pk} processed')
    send_notification(order.customer)


@receiver(order_cancelled)
def handle_order_cancelled(sender, order, **kwargs):
    print(f'Order {order.pk} cancelled')
    refund_payment(order)
```

## Common Patterns

### Pattern 1: Audit Logging

```python
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from datetime import datetime


class AuditLog(models.Model):
    action = models.CharField(max_length=10)
    model_name = models.CharField(max_length=100)
    object_id = models.PositiveIntegerField()
    changes = models.JSONField(default=dict)
    user_id = models.PositiveIntegerField(null=True)
    timestamp = models.DateTimeField(auto_now_add=True)


@receiver(post_save)
def audit_save(sender, **kwargs):
    """Log all model saves."""
    # Skip if model is AuditLog itself
    if sender == AuditLog:
        return
    
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    # Get changes if updating
    changes = {}
    if not created and hasattr(instance, '_state'):
        # Track field changes
        for field in instance._meta.fields:
            old_value = getattr(instance, field.name)
            # Note: old_value would need to be fetched from DB
            changes[field.name] = str(old_value)
    
    AuditLog.objects.create(
        action='CREATE' if created else 'UPDATE',
        model_name=sender._meta.model_name,
        object_id=instance.pk,
        changes=changes,
    )


@receiver(post_delete)
def audit_delete(sender, **kwargs):
    """Log all model deletes."""
    if sender == AuditLog:
        return
    
    instance = kwargs.get('instance')
    
    AuditLog.objects.create(
        action='DELETE',
        model_name=sender._meta.model_name,
        object_id=instance.pk,
        changes={},
    )
```

### Pattern 2: Denormalized Counts

```python
from django.db.models.signals import post_save, post_delete, m2m_changed
from django.dispatch import receiver


class Author(models.Model):
    name = models.CharField(max_length=100)
    post_count = models.PositiveIntegerField(default=0)  # Denormalized
    comment_count = models.PositiveIntegerField(default=0)


class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)


class Comment(models.Model):
    content = models.TextField()
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)


@receiver(post_save, sender=Post)
def update_author_post_count(sender, **kwargs):
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created:
        Author.objects.filter(pk=instance.author_id).update(
            post_count=models.F('post_count') + 1
        )


@receiver(post_delete, sender=Post)
def decrement_author_post_count(sender, **kwargs):
    instance = kwargs.get('instance')
    
    Author.objects.filter(pk=instance.author_id).update(
        post_count=models.F('post_count') - 1
    )


@receiver(m2m_changed, sender=Post.likes.through)
def update_post_like_count(sender, **kwargs):
    instance = kwargs.get('instance')
    action = kwargs.get('action')
    
    if action in ['post_add', 'post_remove']:
        instance.refresh_from_db()
        instance.like_count = instance.likes.count()
        instance.save(update_fields=['like_count'])
```

### Pattern 3: Cache Invalidation

```python
from django.core.cache import cache
from django.db.models.signals import post_save, post_delete


CACHE_KEYS = {
    'homepage_posts': 'homepage_posts',
    'featured_posts': 'featured_posts',
    'user_profile': lambda user_id: f'user_profile_{user_id}',
}


@receiver(post_save, sender=Post)
def invalidate_post_cache(sender, **kwargs):
    """Invalidate post-related caches when posts change."""
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    # Invalidate homepage cache
    cache.delete(CACHE_KEYS['homepage_posts'])
    
    # Invalidate featured posts cache
    cache.delete(CACHE_KEYS['featured_posts'])
    
    # If status changed to published, also invalidate
    if created or instance.status == 'published':
        # Additional cache invalidation
        pass


@receiver(post_delete, sender=Post)
def invalidate_post_cache_on_delete(sender, **kwargs):
    cache.delete(CACHE_KEYS['homepage_posts'])
    cache.delete(CACHE_KEYS['featured_posts'])
```

### Pattern 4: Signal + Celery

For heavy operations, use signals to trigger Celery tasks:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver


@receiver(post_save, sender=Order)
def trigger_order_processing(sender, **kwargs):
    """
    Trigger async processing when order is created.
    """
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created and instance.status == OrderStatus.PENDING:
        # Don't import inside function if possible
        from .tasks import process_new_order
        
        process_new_order.delay(instance.id)


@receiver(post_save, sender=Post)
def trigger_notifications(sender, **kwargs):
    """
    Send notifications when post is published.
    """
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    # Check if post was just published
    if created or instance.status == PublicationStatus.PUBLISHED:
        if not created:
            # Check if status changed to published
            old_instance = Post.objects.get(pk=instance.pk)
            if old_instance.status != PublicationStatus.PUBLISHED:
                from .tasks import send_post_notifications
                send_post_notifications.delay(instance.id)
```

### Pattern 5: Conditional Signals

```python
from django.db.models.signals import post_save
from django.dispatch import receiver


@receiver(post_save, sender=User)
def send_welcome_email(sender, **kwargs):
    """
    Only send welcome email for new users.
    """
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created and instance.email and not instance.is_superuser:
        # Use update_or_create to avoid duplicate sends
        user_created = UserCreatedNotification.objects.update_or_create(
            user=instance,
            defaults={'email_sent': True}
        )
        
        if user_created[1]:  # Was created (first time)
            from .tasks import send_welcome_email_task
            send_welcome_email_task.delay(instance.id)
```

## Signal Pitfalls

### Pitfall 1: Circular Imports

```python
# ❌ BAD: Import in signal handler
@receiver(post_save, sender=Order)
def my_handler(sender, **kwargs):
    from .models import SomeModel  # This is fine inside function
    # But avoid at module level if it causes circular imports


# ✅ GOOD: Use AppConfig to import signals
# apps/orders/apps.py
from django.apps import AppConfig


class OrdersConfig(AppConfig):
    name = 'orders'
    
    def ready(self):
        import orders.signals  # Register signals here
```

### Pitfall 2: Saving in post_save

```python
# ❌ BAD: Save in post_save can trigger infinite loop
@receiver(post_save, sender=Order)
def bad_handler(sender, **kwargs):
    instance = kwargs.get('instance')
    instance.slug = generate_slug(instance.title)
    instance.save()  # This triggers post_save again!


# ✅ GOOD: Use pre_save or update with specific fields
@receiver(pre_save, sender=Order)
def good_handler(sender, **kwargs):
    instance = kwargs.get('instance')
    if not instance.slug:
        instance.slug = generate_slug(instance.title)


# ✅ GOOD: If you must save in post_save, use update_fields
@receiver(post_save, sender=Order)
def good_handler2(sender, **kwargs):
    instance = kwargs.get('instance')
    instance.last_modified = timezone.now()
    instance.save(update_fields=['last_modified'])
```

### Pitfall 3: Slow Signals

```python
# ❌ BAD: Heavy operations in signal
@receiver(post_save, sender=Post)
def slow_handler(sender, **kwargs):
    # This blocks the save operation
    send_mass_email(...)
    generate_pdf_report(...)
    sync_with_external_api(...)


# ✅ GOOD: Use Celery tasks
@receiver(post_save, sender=Post)
def fast_handler(sender, **kwargs):
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created:
        process_post_task.delay(instance.id)
```

### Pitfall 4: Testing Signals

```python
# ❌ BAD: Hard to test
@receiver(post_save, sender=Order)
def my_handler(sender, **kwargs):
    # Side effect that's hard to test
    send_email(...)


# ✅ GOOD: Make testable with custom signals
order_completed = Signal()


@receiver(post_save, sender=Order)
def trigger_completed(sender, **kwargs):
    instance = kwargs.get('instance')
    if instance.status == 'completed':
        order_completed.send(sender=Order, order=instance)


# Now you can test either:
# 1. The signal handler
# 2. Connect your own handler to order_completed
# 3. Test the logic directly in the service
```

## Registering Signals

### Method 1: AppConfig

```python
# myapp/apps.py
from django.apps import AppConfig


class MyAppConfig(AppConfig):
    name = 'myapp'
    
    def ready(self):
        import myapp.signals  # noqa
```

### Method 2: Django Settings

```python
# settings.py
INSTALLED_APPS = [
    ...
]

# Or explicitly configure
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

## Testing Signals

```python
from django.test import TestCase
from unittest.mock import patch, MagicMock


class SignalTest(TestCase):
    @patch('myapp.signals.send_email')
    def test_post_save_signal(self, mock_send_email):
        """Test that signal sends email on post save."""
        post = Post.objects.create(
            title='Test',
            content='Content',
            author=self.user
        )
        
        # Signal should have been called
        mock_send_email.assert_called_once()
    
    def test_signal_order(self):
        """Test signal execution order."""
        # Use assertNumQueries if testing DB operations
        pass
```

## Summary

**Use signals for:**
- Audit logging
- Cache invalidation
- Denormalized counts
- Triggering async tasks
- Cross-app communication

**Avoid signals for:**
- Business logic (use services)
- Complex operations (use Celery)
- Tightly coupled logic (put in model)

**Best practices:**
1. Keep signals simple and focused
2. Use pre_save for modifications
3. Use post_save for side effects
4. Avoid saving in post_save handlers
5. Use Celery for heavy operations
6. Test signal handlers explicitly
7. Document signal dependencies
