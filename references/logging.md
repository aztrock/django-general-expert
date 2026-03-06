# Logging Guide

Guide to implementing logging in Django applications.

## Basic Configuration

```python
# settings.py
import os

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
        'json': {
            'format': '{"time": "%(asctime)s", "level": "%(levelname)s", "module": "%(module)s", "message": "%(message)s"}',
        },
    },
    
    'filters': {
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
    },
    
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'simple',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': os.path.join(BASE_DIR, 'logs', 'app.log'),
            'maxBytes': 1024 * 1024 * 10,  # 10MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
        'error_file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': os.path.join(BASE_DIR, 'logs', 'error.log'),
            'maxBytes': 1024 * 1024 * 10,
            'backupCount': 5,
            'level': 'ERROR',
            'formatter': 'verbose',
        },
    },
    
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
        },
        'django.server': {
            'handlers': ['console'],
            'level': 'WARNING',
        },
        'myapp': {
            'handlers': ['console', 'file', 'error_file'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
    
    'root': {
        'handlers': ['console', 'file'],
        'level': 'INFO',
    },
}
```

## Using Logging in Code

```python
import logging

logger = logging.getLogger(__name__)


# Basic logging
def process_order(order_id):
    logger.info(f'Starting to process order {order_id}')
    
    try:
        order = Order.objects.get(pk=order_id)
        
        # Process order
        logger.debug(f'Order status: {order.status}')
        
        return order
    
    except Order.DoesNotExist:
        logger.warning(f'Order {order_id} not found')
        return None
    
    except Exception as e:
        logger.error(f'Error processing order {order_id}: {e}', exc_info=True)
        raise


# Structured logging
def create_user(email, data):
    logger.info(
        'Creating user',
        extra={
            'email': email,
            'data_keys': list(data.keys()),
        }
    )
```

## Logging in Services

```python
# services.py
import logging
from django.db import transaction

logger = logging.getLogger(__name__)


class OrderService:
    def __init__(self):
        self.logger = logging.getLogger(f'{__name__}.OrderService')
    
    @transaction.atomic
    def create_order(self, data):
        self.logger.info(
            f'Creating order for customer {data.customer_id}',
            extra={
                'customer_id': data.customer_id,
                'item_count': len(data.items),
            }
        )
        
        try:
            order = Order.objects.create(...)
            
            self.logger.info(
                f'Order {order.id} created successfully',
                extra={
                    'order_id': order.id,
                    'total': float(order.total),
                }
            )
            
            return order
        
        except Exception as e:
            self.logger.error(
                f'Failed to create order: {e}',
                exc_info=True,
                extra={
                    'customer_id': data.customer_id,
                }
            )
            raise
```

## Logging in Django Signals

```python
# signals.py
import logging

logger = logging.getLogger(__name__)


def post_save_signal(sender, **kwargs):
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created:
        logger.info(
            f'Created {sender.__name__} with id {instance.pk}'
        )
    else:
        logger.debug(
            f'Updated {sender.__name__} with id {instance.pk}'
        )
```

## Logging in Celery Tasks

```python
# tasks.py
from celery import shared_task
import logging

logger = logging.getLogger(__name__)


@shared_task(bind=True, max_retries=3)
def process_order_task(self, order_id):
    logger.info(f'Starting order processing: {order_id}')
    
    try:
        order = Order.objects.get(pk=order_id)
        
        # Process
        ...
        
        logger.info(
            f'Order {order_id} processed successfully',
            extra={
                'order_id': order_id,
                'status': order.status,
            }
        )
        
        return {'status': 'success'}
    
    except Exception as e:
        logger.error(
            f'Failed to process order {order_id}: {e}',
            exc_info=True,
            extra={
                'order_id': order_id,
                'retry': self.request.retries,
            }
        )
        raise
```

## Logging HTTP Requests

```python
# middleware.py
import logging
import time

logger = logging.getLogger(__name__)


class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        start_time = time.time()
        
        # Log request
        logger.info(
            f'Request started: {request.method} {request.path}',
            extra={
                'method': request.method,
                'path': request.path,
                'user': request.user.id if request.user.is_authenticated else None,
            }
        )
        
        response = self.get_response(request)
        
        # Log response
        duration = time.time() - start_time
        logger.info(
            f'Request completed: {request.method} {request.path} - {response.status_code} ({duration:.2f}s)',
            extra={
                'method': request.method,
                'path': request.path,
                'status_code': response.status_code,
                'duration': duration,
            }
        )
        
        return response
```

## Summary

1. **Configure logging** in settings.py
2. **Use proper log levels**: DEBUG, INFO, WARNING, ERROR, CRITICAL
3. **Include context** in extra dict
4. **Use exc_info=True** for exceptions
5. **Rotate log files** to prevent disk full
6. **Log in middleware** for HTTP request/response
7. **Log in services** for business logic
8. **Log in signals** sparingly
9. **Use JSON formatter** for log aggregation
