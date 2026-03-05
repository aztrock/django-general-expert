# Signals Guide

Guide to using Django signals effectively.

## When to Use

- Cross-app communication
- Audit logging
- Denormalization

## When to Avoid

- Simple business logic (use service layer)
- Complex multi-step operations
- Modifying multiple models

## Signal + Celery Pattern

For critical processes, use signals to trigger Celery tasks:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver


@receiver(post_save, sender=Order)
def trigger_order_processing(sender, instance, created, **kwargs):
    if created:
        from .tasks import process_order_task
        process_order_task.delay(instance.id)
```

## Summary

- Use for cross-app communication
- Avoid for simple logic
- Combine with Celery for heavy operations
