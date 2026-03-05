# Advanced Patterns

Enterprise patterns for complex Django applications.

## Service Layer Pattern

### Why Use Service Layer?

- **Separation of concerns**: Business logic separate from HTTP layer
- **Testability**: Services are easy to test in isolation
- **Reusability**: Logic can be reused across different views
- **Maintainability**: Changes to business logic don't affect views

### Basic Service Layer

```python
# services.py
from dataclasses import dataclass
from django.db import transaction
import logging

logger = logging.getLogger(__name__)


@dataclass
class OrderData:
    customer_id: int
    items: list[dict]
    shipping_address: str


class OrderService:
    @staticmethod
    @transaction.atomic
    def create_order(user, cart, shipping_address):
        # Validate
        OrderService.validate_cart(cart)
        
        # Calculate totals
        totals = OrderService.calculate_totals(cart)
        
        # Create order
        order = Order.objects.create(
            customer=user,
            subtotal=totals['subtotal'],
            tax=totals['tax'],
            shipping=totals['shipping'],
            total=totals['total'],
            shipping_address=shipping_address,
            status=OrderStatus.PENDING
        )
        
        # Create order items
        order_items = []
        for item in cart.items.select_related('product'):
            order_items.append(OrderItem(
                order=order,
                product=item.product,
                quantity=item.quantity,
                price=item.product.price
            ))
        
        OrderItem.objects.bulk_create(order_items)
        
        return order
```

## Repository Pattern

### Selector Pattern

```python
# selectors.py
class PostSelector:
    @staticmethod
    def get_published():
        return Post.objects.filter(status='published')
    
    @staticmethod
    def get_by_author(author):
        return Post.objects.filter(author=author)
    
    @staticmethod
    def search(query):
        return Post.objects.filter(
            Q(title__icontains=query) | 
            Q(content__icontains=query)
        )
```

## Summary

1. **Service Layer**: Encapsulate business logic separately from views
2. **Repository/Selector**: Abstract data access for flexibility
3. **Testing**: Services are easy to test in isolation
4. **Keep Views Thin**: Delegate to services, return HTTP responses
