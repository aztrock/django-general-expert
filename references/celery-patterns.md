# Celery Patterns Guide

Best practices for Celery tasks in Django applications. Celery is essential for handling long-running operations asynchronously.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Django App                             │
│                                                             │
│  View ──► Service ──► Task.delay() ──► Message Broker       │
│                           │                                 │
│                           ▼                                 │
│                    ┌──────────────┐                         │
│                    │ Celery Worker│                         │
│                    └──────────────┘                         │
│                           │                                 │
│                           ▼                                 │
│                    ┌──────────────┐                         │
│                    │   Database   │                         │
│                    └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

## Celery Configuration

### Basic Configuration

```python
# settings.py
from celery import Celery

app = Celery('myproject')

app.config_from_object('django.conf:settings', namespace='CELERY')

app.autodiscover_tasks()


# Celery settings
CELERY_BROKER_URL = os.environ.get('CELERY_BROKER_URL', 'redis://localhost:6379/0')
CELERY_RESULT_BACKEND = os.environ.get('CELERY_RESULT_BACKEND', 'redis://localhost:6379/0')
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TIMEZONE = 'UTC'
CELERY_ENABLE_UTC = True
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60  # 30 minutes
CELERY_TASK_SOFT_TIME_LIMIT = 25 * 60  # 25 minutes
CELERY_WORKER_PREFETCH_MULTIPLIER = 4
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000
```

### Advanced Configuration

```python
# settings.py
# Retry configuration
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True

# Rate limiting
CELERY_TASK_DEFAULT_RATE_LIMIT = '100/m'

# Result expiration
CELERY_RESULT_EXPIRES = 60 * 60 * 24  # 24 hours

# Scheduled tasks (Celery Beat)
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'

# Monitoring
CELERY_SEND_TASK_SENT_EVENT = True
CELERY_SEND_TASK_ERROR_EMAILS = True

# Security
CELERY_SECURITY_KEY = os.environ.get('CELERY_SECURITY_KEY')
CELERY_SECURITY_CERTIFICATE = os.environ.get('CELERY_SECURITY_CERTIFICATE')
CELERY_SECURITY_CACERT = os.environ.get('CELERY_SECURITY_CACERT')
```


### Django Celery Beat (Database-backed periodic tasks)

```bash
# Install
pip install django-celery-beat
```

```python
# settings.py
INSTALLED_APPS = [
    'django_celery_beat',
]

# Use database scheduler for periodic tasks
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

```python
# models.py - Create periodic tasks in Django admin
from django_celery_beat.models import PeriodicTask, IntervalSchedule
import json


# Create interval schedule
schedule, _ = IntervalSchedule.objects.get_or_create(
    every=60,
    period=IntervalSchedule.SECONDS,
)

# Create periodic task
PeriodicTask.objects.create(
    interval=schedule,
    name='Cleanup old sessions',
    task='myapp.tasks.cleanup_old_sessions',
    args=json.dumps([]),
    kwargs=json.dumps({}),
    enabled=True,
)


# Or create crontab schedule
from django_celery_beat.models import CrontabSchedule

schedule, _ = CrontabSchedule.objects.get_or_create(
    minute='0',
    hour='3',
    day_of_week='*',
    day_of_month='*',
    month_of_year='*',
)

PeriodicTask.objects.create(
    crontab=schedule,
    name='Send daily digest',
    task='myapp.tasks.send_daily_digest',
    enabled=True,
)
```

**Benefits of django-celery-beat:**
- Manage periodic tasks from Django Admin
- No need to restart Celery beat when adding/modifying tasks
- Tasks can be enabled/disabled dynamically
- Store schedules in database instead of code

**Run migrations:**
```bash
python manage.py migrate django_celery_beat
```

**Start beat:**
```bash
celery -A myproject beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
```

## Task Structure

### Basic Task Pattern

```python
# tasks.py
from celery import shared_task
import logging

logger = logging.getLogger(__name__)


@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_order_confirmation(self, order_id: int):
    """
    Send order confirmation email.
    
    Uses bind=True to access task instance for retries.
    """
    logger.info(f'Sending confirmation for order {order_id}')
    
    # Validate order exists
    try:
        order = Order.objects.get(pk=order_id)
    except Order.DoesNotExist:
        logger.warning(f'Order {order_id} not found, skipping')
        return {'status': 'skipped', 'reason': 'not_found'}
    
    # Check if already sent
    if order.confirmation_sent:
        logger.info(f'Confirmation already sent for order {order_id}')
        return {'status': 'skipped', 'reason': 'already_sent'}
    
    try:
        # Send email
        send_email(
            to=order.customer.email,
            subject=f'Order Confirmation #{order.public_id}',
            template='emails/order_confirmation.html',
            context={'order': order}
        )
        
        # Update order
        order.confirmation_sent = True
        order.save(update_fields=['confirmation_sent'])
        
        logger.info(f'Confirmation sent for order {order_id}')
        return {'status': 'success', 'order_id': order_id}
    
    except EmailServiceError as e:
        logger.error(f'Failed to send confirmation for order {order_id}: {e}')
        
        # Retry with exponential backoff
        try:
            raise self.retry(exc=e, countdown=60 * (2 ** self.request.retries))
        except self.MaxRetriesExceededError:
            logger.error(f'Max retries exceeded for order {order_id}')
            return {'status': 'failed', 'error': str(e)}
```

### Periodic Tasks

```python
# tasks.py
from celery import shared_task
from datetime import datetime, timedelta
from django.contrib.sessions.models import Session
from external_api import InventoryAPI
import logging

logger = logging.getLogger(__name__)
from django.contrib.sessions.models import Session
import logging

logger = logging.getLogger(__name__)


@shared_task
def cleanup_old_sessions():
    """
    Clean up expired sessions.
    Run daily at 3 AM.
    """
    expired = Session.objects.filter(
        expire_date__lt=datetime.now()
    )
    count = expired.count()
    expired.delete()
    
    logger.info(f'Cleaned up {count} expired sessions')
    return {'status': 'success', 'deleted': count}


@shared_task
def send_daily_digest():
    """
    Send daily digest email to users.
    Run at 8 AM daily.
    """
    yesterday = datetime.now() - timedelta(days=1)
    
    users = User.objects.filter(
        preferences__daily_digest=True,
        is_active=True
    )
    
    for user in users:
        posts = Post.objects.filter(
            published_at__gte=yesterday,
            author=user
        )
        
        if posts.exists():
            send_email(
                to=user.email,
                subject='Your Daily Digest',
                template='emails/daily_digest.html',
                context={'posts': posts, 'user': user}
            )
    
    return {'status': 'success', 'users_notified': users.count()}


@shared_task
def sync_inventory():
    """
    Sync inventory with external system.
    Run every hour.
    """
    api = InventoryAPI()
    
    products = Product.objects.filter(sync_inventory=True)
    
    for product in products:
        try:
            external_stock = api.get_stock(product.sku)
            
            product.inventory.quantity = external_stock
            product.inventory.last_synced = datetime.now()
            product.inventory.save(update_fields=['quantity', 'last_synced'])
        
        except Exception as e:
            logger.error(f'Failed to sync {product.sku}: {e}')
    
    return {'status': 'success', 'synced': products.count()}


# Celery Beat Schedule
CELERY_BEAT_SCHEDULE = {
    'cleanup-old-sessions': {
        'task': 'tasks.cleanup_old_sessions',
        'schedule': crontab(hour=3, minute=0),
    },
    'send-daily-digest': {
        'task': 'tasks.send_daily_digest',
        'schedule': crontab(hour=8, minute=0),
    },
    'sync-inventory-hourly': {
        'task': 'tasks.sync_inventory',
        'schedule': 60 * 60,  # Every hour
    },
}
```

## Error Handling

### Task Error Handling Patterns

```python
from celery import shared_task
from celery.exceptions import MaxRetriesExceededError
import sentry_sdk


@shared_task(bind=True, max_retries=3)
def process_payment(self, payment_id: int):
    """
    Process payment with proper error handling.
    """
    try:
        payment = Payment.objects.get(pk=payment_id)
    except Payment.DoesNotExist:
        logger.error(f'Payment {payment_id} not found')
        return {'status': 'error', 'message': 'Payment not found'}
    
    try:
        result = PaymentGateway.charge(
            amount=payment.amount,
            token=payment.token
        )
        
        payment.status = PaymentStatus.COMPLETED
        payment.transaction_id = result.transaction_id
        payment.save()
        
        return {'status': 'success', 'transaction_id': result.transaction_id}
    
    except PaymentGatewayDeclinedError as e:
        # Handle decline - don't retry
        payment.status = PaymentStatus.DECLINED
        payment.error_message = str(e)
        payment.save()
        
        notify_user_of_decline(payment)
        
        return {'status': 'declined', 'message': str(e)}
    
    except PaymentGatewayTimeoutError as e:
        # Retry with backoff
        try:
            countdown = 60 * (2 ** self.request.retries)
            raise self.retry(exc=e, countdown=countdown)
        except MaxRetriesExceededError:
            payment.status = PaymentStatus.FAILED
            payment.error_message = 'Max retries exceeded'
            payment.save()
            
            notify_admin_of_failure(payment)
            
            return {'status': 'error', 'message': 'Max retries exceeded'}
    
    except Exception as e:
        # Catch-all for unexpected errors
        sentry_sdk.capture_exception(e)
        logger.exception(f'Unexpected error processing payment {payment_id}')
        
        return {'status': 'error', 'message': str(e)}
```

### Dead Letter Queue

```python
from celery import shared_task
from celery.exceptions import Reject


@shared_task(
    bind=True,
    max_retries=3,
    autoretry_for=(Exception,),
    retry_backoff=True,
    retry_backoff_max=600,
    retry_jitter=True
)
def process_message(self, message: dict):
    """
    Process message with dead letter queue on failure.
    """
    try:
        validate_message(message)
        process_message_logic(message)
        
    except ValidationError as e:
        # Don't retry validation errors
        logger.error(f'Invalid message: {e}')
        raise Reject(str(e), requeue=False)
    
    except TemporaryError as e:
        # Retry temporary errors
        logger.warning(f'Temporary error: {e}')
        raise
    
    except PermanentError as e:
        # Send to dead letter queue
        DeadLetterQueue.objects.create(
            task_name=self.name,
            task_args=self.request.args,
            task_kwargs=self.request.kwargs,
            error=str(e),
            traceback=traceback.format_exc()
        )
        raise Requeue = False


# Dead letter queue processor
@shared_task
def process_dead_letters():
    """
    Process messages from dead letter queue.
    Manual intervention required.
    """
    dead_letters = DeadLetterQueue.objects.filter(
        status='pending',
        retry_count__lt=3
    )
    
    for letter in dead_letters:
        try:
            # Attempt to reprocess
            task = celery_app.send_task(
                letter.task_name,
                args=letter.task_args,
                kwargs=letter.task_kwargs
            )
            
            letter.status = 'reprocessing'
            letter.task_id = task.id
            letter.retry_count += 1
            letter.save()
        
        except Exception as e:
            logger.error(f'Failed to reprocess {letter.id}: {e}')
```

## Chained Tasks

```python
from celery import chain, group, chord


@shared_task
def step_one(order_id: int):
    """Step 1: Validate order."""
    order = Order.objects.get(pk=order_id)
    
    if not order.is_valid():
        return {'status': 'error', 'message': 'Invalid order'}
    
    order.status = OrderStatus.VALIDATED
    order.save()
    
    return {'status': 'success', 'order_id': order_id}


@shared_task
def step_two(order_id: int):
    """Step 2: Check inventory."""
    order = Order.objects.get(pk=order_id)
    
    for item in order.items.all():
        if item.quantity > item.product.inventory.quantity:
            return {'status': 'error', 'message': f'Insufficient inventory for {item.product.name}'}
    
    # Reserve inventory
    for item in order.items.all():
        item.product.inventory.quantity -= item.quantity
        item.product.inventory.save()
    
    return {'status': 'success', 'order_id': order_id}


@shared_task
def step_three(order_id: int):
    """Step 3: Process payment."""
    order = Order.objects.get(pk=order_id)
    
    result = PaymentGateway.charge(order.total, order.payment_token)
    
    order.payment.status = PaymentStatus.COMPLETED
    order.payment.transaction_id = result.transaction_id
    order.payment.save()
    
    return {'status': 'success', 'order_id': order_id}


@shared_task
def step_four(order_id: int):
    """Step 4: Complete order."""
    order = Order.objects.get(pk=order_id)
    order.status = OrderStatus.COMPLETED
    order.save()
    
    send_order_confirmation.delay(order_id)
    
    return {'status': 'success', 'order_id': order_id}


# Execute chain
def process_order(order_id: int):
    """Process order through chain of tasks."""
    chain(
        step_one.s(order_id),
        step_two.s(),
        step_three.s(),
        step_four.s()
    )()


# Execute group (parallel tasks)
def send_bulk_notification(user_ids: list[int]):
    """Send notifications to multiple users in parallel."""
    group(
        send_notification.s(user_id)
        for user_id in user_ids
    )()


# Execute chord (parallel tasks with callback)
def process_order_with_callback(order_id: int):
    """Process order items in parallel, then complete."""
    chord(
        (process_item.s(item_id) for item_id in item_ids),
        complete_order.s(order_id)
    )()
```

## Monitoring and Logging

```python
import logging
from celery import signals
from celery.signals import task_prerun, task_postrun, task_failure


logger = logging.getLogger(__name__)


@task_prerun.connect
def task_prerun_handler(task_id, task, *args, **kwargs):
    """Log when task starts."""
    logger.info(f'Task {task.name}[{task_id}] started with args={args}, kwargs={kwargs}')


@task_postrun.connect
def task_postrun_handler(task_id, task, *args, **kwargs):
    """Log when task completes."""
    logger.info(f'Task {task.name}[{task_id}] completed')


@task_failure.connect
def task_failure_handler(task_id, task, einfo, *args, **kwargs):
    """Log task failures."""
    logger.error(
        f'Task {task.name}[{task_id}] failed: {einfo.exception}',
        exc_info=einfo.exc_info
    )
    
    # Optional: Send to monitoring
    sentry_sdk.capture_exception(einfo.exception)


@signals.worker_ready.connect
def worker_ready_handler(**kwargs):
    """Log when worker is ready."""
    logger.info('Celery worker is ready')


@signals.worker_shutdown.connect
def worker_shutdown_handler(**kwargs):
    """Log when worker shuts down."""
    logger.info('Celery worker is shutting down')
```

## Best Practices Summary

1. **Always validate entity exists** before processing
2. **Always verify state** before taking actions (idempotency)
3. **Use exponential backoff** for retries
4. **Log meaningful information** - include IDs, context
5. **Use bind=True** for access to task instance and retries
6. **Set time limits** to prevent stuck tasks
7. **Use transactions** for database operations
8. **Handle errors explicitly** - don't catch all exceptions
9. **Use progress tracking** for long-running tasks
10. **Implement dead letter queue** for failed messages
11. **Separate concerns** - one task = one responsibility
12. **Use Celery Beat** for periodic tasks
13. **Monitor with proper logging** and error tracking

## Common Pitfalls

```python
# ❌ BAD: No validation
@shared_task
def send_email(user_id):
    user = User.objects.get(pk=user_id)  # Can raise DoesNotExist
    send_email(user.email, ...)  # Can fail


# ✅ GOOD: Proper validation
@shared_task
def send_email(user_id):
    try:
        user = User.objects.get(pk=user_id)
    except User.DoesNotExist:
        logger.warning(f'User {user_id} not found')
        return {'status': 'skipped', 'reason': 'user_not_found'}
    
    send_email(user.email, ...)


# ❌ BAD: No retry logic
@shared_task
def process_payment(payment_id):
    # No error handling!
    PaymentGateway.charge(...)


# ✅ GOOD: With retry logic
@shared_task(bind=True, max_retries=3)
def process_payment(self, payment_id):
    try:
        PaymentGateway.charge(...)
    except TemporaryError as e:
        raise self.retry(exc=e, countdown=60)


# ❌ BAD: Blocking operations in task
@shared_task
def generate_report():
    # Heavy computation in task - bad!
    data = generate_large_report()
    return data


# ✅ GOOD: Chunked processing
@shared_task
def generate_report(report_id):
    report = Report.objects.get(pk=report_id)
    
    # Process in chunks
    for chunk in chunked_data:
        process_chunk(chunk)
    
    report.status = 'complete'
    report.save()
```
