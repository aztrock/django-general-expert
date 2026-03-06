# Logging Guide

Guide to implementing logging in Django applications.

## Log Context Constants

Define log contexts as constants for consistency:

```python
# constants.py
class LogContext:
    INTEGRATION = "INTEGRATION"
    SIGNALS = "SIGNALS"
    CELERY = "CELERY"
    HTTP = "HTTP"
```

Or as simple constants:

```python
# constants.py
LOG_CONTEXT_INTEGRATION = "INTEGRATION"
LOG_CONTEXT_SIGNALS = "SIGNALS"
LOG_CONTEXT_CELERY = "CELERY"
LOG_CONTEXT_HTTP = "HTTP"
```

## Basic Configuration

```python
# settings.py
import os


class ContextFormatter(logging.Formatter):
    """Custom formatter that includes context from extra."""
    
    def format(self, record):
        if hasattr(record, 'context') and record.context:
            record.message = f"[{record.context}] {record.getMessage()}"
        return super().format(record)


LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    
    'formatters': {
        'verbose': {
            'format': '%(asctime)s | %(levelname)-8s | %(name)-25s | %(message)s',
            'datefmt': '%Y-%m-%d %H:%M:%S',
        },
        'simple': {
            'format': '%(levelname)s %(message)s',
        },
        'json': {
            'format': '{"time": "%(asctime)s", "level": "%(levelname)s", "module": "%(module)s", "message": "%(message)s"}',
        },
        'context': {
            '()': ContextFormatter,
            'format': '%(asctime)s | %(levelname)-8s | %(name)-25s | %(message)s',
            'datefmt': '%Y-%m-%d %H:%M:%S',
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
            'formatter': 'context',
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

**Output example:**
```
2026-03-06 14:54:12 | INFO    | myapp.orders         | [INTEGRATION] Starting to process order 456
2026-03-06 14:54:13 | DEBUG   | myapp.orders         | [INTEGRATION] Order status: pending
2026-03-06 14:54:14 | WARNING | myapp.orders         | [INTEGRATION] Order 789 not found
2026-03-06 14:54:15 | INFO    | myapp.users          | [INTEGRATION] Creating user email=john@example.com
2026-03-06 14:54:16 | ERROR   | myapp.orders         | [INTEGRATION] Error processing order 123: DoesNotExist(...)
```

## Using Logging in Code

```python
import logging
from constants import LogContext

logger = logging.getLogger(__name__)


def process_order(order_id):
    logger.info(f"Starting to process order {order_id}", extra={"context": LogContext.INTEGRATION})
    
    try:
        order = Order.objects.get(pk=order_id)
        
        logger.debug(f"Order status: {order.status}", extra={"context": LogContext.INTEGRATION, "order_id": order_id})
        
        return order
    
    except Order.DoesNotExist:
        logger.warning(f"Order {order_id} not found", extra={"context": LogContext.INTEGRATION})
        return None
    
    except Exception as e:
        logger.error(f"Error processing order {order_id}: {e}", exc_info=True, extra={"context": LogContext.INTEGRATION})
        raise


def create_user(email, data):
    logger.info(
        "Creating user",
        extra={
            "context": LogContext.INTEGRATION,
            "email": email,
            "data_keys": list(data.keys()),
        }
    )
```

## Logging in Services

```python
# services.py
import logging
from django.db import transaction
from constants import LogContext

logger = logging.getLogger(__name__)


class OrderService:
    @property
    def _logger(self):
        return logging.getLogger(f'{__name__}.{self.__class__.__name__}')
    
    @transaction.atomic
    def create_order(self, data):
        self._logger.info(
            "Creating order",
            extra={
                "context": LogContext.INTEGRATION,
                "customer_id": data.customer_id,
                "item_count": len(data.items),
            }
        )
        
        try:
            order = Order.objects.create(...)
            
            self._logger.info(
                "Order created successfully",
                extra={
                    "context": LogContext.INTEGRATION,
                    "order_id": order.id,
                    "total": float(order.total),
                }
            )
            
            return order
        
        except Exception as e:
            self._logger.error(
                f"Failed to create order: {e}",
                exc_info=True,
                extra={
                    "context": LogContext.INTEGRATION,
                    "customer_id": data.customer_id,
                }
            )
            raise
```

## Logging in Django Signals

```python
# signals.py
import logging
from constants import LogContext

logger = logging.getLogger(__name__)


def post_save_signal(sender, **kwargs):
    instance = kwargs.get('instance')
    created = kwargs.get('created')
    
    if created:
        logger.info(
            f"Created {sender.__name__} with id {instance.pk}",
            extra={"context": LogContext.SIGNALS}
        )
    else:
        logger.debug(
            f"Updated {sender.__name__} with id {instance.pk}",
            extra={"context": LogContext.SIGNALS}
        )
```

## Logging in Celery Tasks

```python
# tasks.py
from celery import shared_task
import logging
from constants import LogContext

logger = logging.getLogger(__name__)


@shared_task(bind=True, max_retries=3)
def process_order_task(self, order_id):
    logger.info(f"Starting order processing", extra={"context": LogContext.CELERY, "order_id": order_id})
    
    try:
        order = Order.objects.get(pk=order_id)
        
        logger.info(
            "Order processed successfully",
            extra={
                "context": LogContext.CELERY,
                "order_id": order_id,
                "status": order.status,
            }
        )
        
        return {'status': 'success'}
    
    except Exception as e:
        logger.error(
            f"Failed to process order: {e}",
            exc_info=True,
            extra={
                "context": LogContext.CELERY,
                "order_id": order_id,
                "retry": self.request.retries,
            }
        )
        raise
```

## Logging HTTP Requests

```python
# middleware.py
import logging
import time
from constants import LogContext

logger = logging.getLogger(__name__)


class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        start_time = time.time()
        
        logger.info(
            f"Request started: {request.method} {request.path}",
            extra={
                "context": LogContext.HTTP,
                "method": request.method,
                "path": request.path,
                "user": request.user.id if request.user.is_authenticated else None,
            }
        )
        
        response = self.get_response(request)
        
        duration = time.time() - start_time
        logger.info(
            f"Request completed: {request.method} {request.path}",
            extra={
                "context": LogContext.HTTP,
                "method": request.method,
                "path": request.path,
                "status_code": response.status_code,
                "duration": duration,
            }
        )
        
        return response
```

## Logging SQL Queries (Optional)

Useful for debugging N+1 problems and query performance. Only enable in development.

```python
# settings.py - Add to LOGGING
LOGGING = {
    ...
    'loggers': {
        ...
        # Log all SQL queries - ONLY in development!
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}

# In development, this will log every SQL query:
# (0.001) SELECT "orders_order"."id", ... FROM "orders_order" WHERE ...
```

**Note:** Set level to `'WARNING'` in production to avoid filling up logs with queries.

1. **Configure logging** in settings.py
2. **Use proper log levels**: DEBUG, INFO, WARNING, ERROR, CRITICAL
3. **Include context** in extra dict
4. **Use exc_info=True** for exceptions
5. **Rotate log files** to prevent disk full
6. **Log in middleware** for HTTP request/response
7. **Log in services** for business logic
8. **Log in signals** sparingly
9. **Use JSON formatter** for log aggregation
