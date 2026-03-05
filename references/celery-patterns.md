# Celery Patterns Guide

Best practices for Celery tasks in Django applications.

## Key Principles

1. **Validate before creating** - Check if entity exists
2. **Verify state before actions** - Check if email already sent
3. **Use exponential backoff** - For retries
4. **Log meaningful information** - Track progress

## Example Task

```python
from celery import shared_task
import logging

logger = logging.getLogger(__name__)


@shared_task(bind=True, max_retries=3)
def send_order_email_task(self, order_id: int):
    logger.info(f'Sending email for order {order_id}')
    
    # Validate existence
    try:
        order = Order.objects.get(pk=order_id)
    except Order.DoesNotExist:
        logger.warning(f'Order {order_id} not found')
        return {'status': 'skipped', 'reason': 'not_found'}
    
    # Verify state
    if order.confirmation_email_sent:
        logger.info(f'Email already sent for order {order_id}')
        return {'status': 'skipped', 'reason': 'already_sent'}
    
    # Process
    send_email(order.customer.email, ...)
    
    # Update state
    order.confirmation_email_sent = True
    order.save(update_fields=['confirmation_email_sent'])
    
    logger.info(f'Email sent successfully for order {order_id}')
    return {'status': 'success'}
```

## Summary

- Always validate entity exists
- Always verify state before action
- Use exponential backoff for retries
- Log all operations
