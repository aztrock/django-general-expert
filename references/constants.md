# Constants Guide

Guide to organizing constants and choices in Django projects.

## Why Separate Constants?

### Problem: Inline Constants

```python
# ❌ BAD: Constants scattered in code
class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        PAID = 'paid', 'Paid'
    
    # Magic numbers
    def calculate_fee(self):
        return self.total * 0.21  # What is 0.21?


def process_payment(order):
    for attempt in range(3):  # Magic number!
        try:
            charge(order)
            break
        except:
            time.sleep(60)  # Magic number!
```

**Problems:**
- Hard to find constants
- Hard to update values
- No single source of truth
- Magic numbers everywhere
- Circular import issues

### Solution: Constants File

```python
# constants.py
from django.db import models
from decimal import Decimal


class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'
    SHIPPED = 'shipped', 'Shipped'
    DELIVERED = 'delivered', 'Delivered'
    CANCELLED = 'cancelled', 'Cancelled'


class PaymentStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    COMPLETED = 'completed', 'Completed'
    FAILED = 'failed', 'Failed'
    REFUNDED = 'refunded', 'Refunded'


# Numeric constants
TAX_RATE = Decimal('0.21')
MAX_RETRY_ATTEMPTS = 3
RETRY_DELAY_SECONDS = 60
LARGE_ORDER_THRESHOLD = Decimal('100.00')
BULK_DISCOUNT_MULTIPLIER = Decimal('0.9')


# String constants
DEFAULT_FROM_EMAIL = 'noreply@example.com'
ORDER_CONFIRMATION_SUBJECT = 'Order Confirmation #{order_id}'
```

---

## TextChoices Organization

### Basic TextChoices

```python
# constants.py
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'
    SHIPPED = 'shipped', 'Shipped'


# models.py
from django.db import models
from .constants import OrderStatus


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )
```

### Using TextChoices

```python
# views.py
from .constants import OrderStatus


def get_orders(request):
    # Using constants
    orders = Order.objects.filter(status=OrderStatus.PAID)
    
    # ❌ BAD: String literals
    orders = Order.objects.filter(status='paid')


# serializers.py
from .constants import OrderStatus


class OrderSerializer(serializers.ModelSerializer):
    status = serializers.ChoiceField(choices=OrderStatus.choices)


# services.py
from .constants import OrderStatus


class OrderService:
    @staticmethod
    def can_cancel(order):
        return order.status in [OrderStatus.PENDING, OrderStatus.PAID]
```

---

## Avoiding Circular Imports

### Problem: Circular Import

```python
# apps/orders/models.py
from apps.payments.constants import PaymentStatus  # Import from payments


class Order(models.Model):
    payment_status = models.CharField(
        max_length=20,
        choices=PaymentStatus.choices
    )


# apps/payments/models.py
from apps.orders.constants import OrderStatus  # Import from orders


class Payment(models.Model):
    order_status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices
    )  # CIRCULAR!
```

### Solution: Shared Constants

```python
# apps/core/constants.py
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'
    SHIPPED = 'shipped', 'Shipped'


class PaymentStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    COMPLETED = 'completed', 'Completed'
    FAILED = 'failed', 'Failed'


# apps/orders/models.py
from apps.core.constants import OrderStatus, PaymentStatus


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices
    )
    payment_status = models.CharField(
        max_length=20,
        choices=PaymentStatus.choices
    )


# apps/payments/models.py
from apps.core.constants import OrderStatus, PaymentStatus


class Payment(models.Model):
    order_status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices
    )
    status = models.CharField(
        max_length=20,
        choices=PaymentStatus.choices
    )
```

---

## Constants Organization

### Directory Structure

```
apps/
├── core/
│   ├── __init__.py
│   └── constants.py          # Shared constants
├── orders/
│   ├── __init__.py
│   ├── constants.py          # Order-specific constants
│   ├── models.py
│   ├── services.py
│   └── ...
└── payments/
    ├── __init__.py
    ├── constants.py          # Payment-specific constants
    ├── models.py
    ├── services.py
    └── ...
```

### Shared vs App-Specific

```python
# apps/core/constants.py
# Shared across multiple apps
class OrderStatus(models.TextChoices):
    ...

class PaymentStatus(models.TextChoices):
    ...

# Shared numeric constants
MAX_RETRY_ATTEMPTS = 3
DEFAULT_TIMEOUT = 30


# apps/orders/constants.py
from apps.core.constants import OrderStatus  # Re-export if needed

# Order-specific constants
MAX_ITEMS_PER_ORDER = 100
ORDER_EXPIRATION_DAYS = 7


# apps/payments/constants.py
from apps.core.constants import PaymentStatus  # Re-export if needed

# Payment-specific constants
STRIPE_API_VERSION = '2023-10-16'
MAX_PAYMENT_AMOUNT = Decimal('999999.99')
```

---

## Using Constants

### In Models

```python
# models.py
from django.db import models
from .constants import OrderStatus, MAX_ITEMS_PER_ORDER


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )
    
    def can_add_items(self):
        return self.items.count() < MAX_ITEMS_PER_ORDER
```

### In Services

```python
# services.py
from django.db import transaction
from .constants import (
    OrderStatus,
    MAX_RETRY_ATTEMPTS,
    RETRY_DELAY_SECONDS,
    TAX_RATE
)


class OrderService:
    @staticmethod
    def calculate_total(order):
        subtotal = sum(item.price * item.quantity for item in order.items.all())
        tax = subtotal * TAX_RATE
        return subtotal + tax
    
    @staticmethod
    def process_payment(order):
        for attempt in range(MAX_RETRY_ATTEMPTS):
            try:
                result = PaymentGateway.charge(order.total)
                order.status = OrderStatus.PAID
                order.save()
                return True
            except PaymentError:
                time.sleep(RETRY_DELAY_SECONDS)
        return False
```

### In Views

```python
# views.py
from rest_framework import status
from rest_framework.response import Response
from .constants import OrderStatus


class OrderViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Order.objects.all()
        
        status_filter = self.request.query_params.get('status')
        if status_filter:
            # Use constant, not string
            queryset = queryset.filter(status=status_filter)
        
        return queryset
    
    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        order = self.get_object()
        
        if order.status not in [OrderStatus.PENDING, OrderStatus.PAID]:
            return Response(
                {'error': 'Cannot cancel'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        order.status = OrderStatus.CANCELLED
        order.save()
        
        return Response({'status': 'cancelled'})
```

---

## Best Practices

### 1. Group Related Constants

```python
# ✅ GOOD: Grouped by domain
# constants.py

# Order-related
class OrderStatus(models.TextChoices):
    ...

MAX_ITEMS_PER_ORDER = 100
ORDER_EXPIRATION_DAYS = 7


# Payment-related
class PaymentStatus(models.TextChoices):
    ...

MAX_PAYMENT_AMOUNT = Decimal('999999.99')
STRIPE_API_VERSION = '2023-10-16'


# Email-related
DEFAULT_FROM_EMAIL = 'noreply@example.com'
ORDER_CONFIRMATION_SUBJECT = 'Order Confirmation #{order_id}'


# ❌ BAD: Random order
TAX_RATE = Decimal('0.21')
DEFAULT_FROM_EMAIL = 'noreply@example.com'
MAX_ITEMS_PER_ORDER = 100
STRIPE_API_VERSION = '2023-10-16'
ORDER_EXPIRATION_DAYS = 7
```

### 2. Use Descriptive Names

```python
# ❌ BAD: Unclear names
MAX = 100
TIMEOUT = 60
RATE = 0.21


# ✅ GOOD: Descriptive names
MAX_ITEMS_PER_ORDER = 100
PAYMENT_TIMEOUT_SECONDS = 60
TAX_RATE = Decimal('0.21')
```

### 3. Type Your Constants

```python
# ✅ GOOD: Typed constants
from decimal import Decimal
from datetime import timedelta


TAX_RATE: Decimal = Decimal('0.21')
MAX_ITEMS_PER_ORDER: int = 100
PAYMENT_TIMEOUT: timedelta = timedelta(seconds=60)
DEFAULT_CURRENCY: str = 'USD'


# ❌ BAD: No types
TAX_RATE = 0.21
MAX_ITEMS = 100
TIMEOUT = 60
CURRENCY = 'USD'
```

### 4. Document Complex Constants

```python
# ✅ GOOD: Documented complex constants
# Retry configuration for payment processing
# After MAX_RETRY_ATTEMPTS failures, the system will give up
MAX_RETRY_ATTEMPTS: int = 3

# Exponential backoff base delay
# Each retry waits: base_delay * (2 ** retry_number)
RETRY_BASE_DELAY_SECONDS: int = 60


# ❌ BAD: No documentation
MAX_RETRIES = 3
RETRY_DELAY = 60
```

---

## Summary

1. **Separate constants** into dedicated file(s)
2. **Use TextChoices** for status/enum fields
3. **Avoid magic numbers** by defining constants
4. **Group related constants** together
5. **Use descriptive names** for clarity
6. **Type your constants** for better IDE support
7. **Document complex constants** when needed
8. **Avoid circular imports** by using shared constants file
9. **Import from constants** in all modules
10. **No inline choices** in models
