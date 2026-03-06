# Advanced Patterns

Advanced patterns for Django applications. This guide covers patterns that are useful beyond the basic Django/DRF workflow.

## Service Layer Pattern

The service layer is used for business logic that goes beyond the basic create/update flow provided by Django and DRF.

### When to Use Service Layer

Use service layer when:

- **Complex business logic** that doesn't fit in serializer/view
- **Multi-step operations** that involve multiple models
- **Transactions** that span multiple database operations
- **Logic reused across multiple views or APIs**
- **External service integrations** (payment gateways, APIs, etc.)

### When NOT to Use Service Layer

**Don't use service layer for:**

- Basic CRUD operations (DRF handles this)
- Simple form validation (Forms handle this)
- Simple model operations that can be done in the model manager

### Correct DRF Flow

```
Request
    │
    ▼
ViewSet (only connects to serializer and returns response)
    │
    ▼
Serializer (data representation and validation)
    │
    ▼
Service Layer (complex business logic only)
    │
    ▼
QuerySet (database operations)
    │
    ▼
Model (data structure only, no logic)
```

### Correct Django Template Flow

```
Request
    │
    ▼
View (only connects to form and returns response)
    │
    ▼
Form (form validation)
    │
    ▼
Service Layer (complex business logic only)
    │
    ▼
QuerySet (database operations)
    │
    ▼
Model (data structure only, no logic)
```

### Example: Service Layer for Complex Logic

```python
# services.py
from dataclasses import dataclass
from decimal import Decimal
from django.db import transaction
import logging

logger = logging.getLogger(__name__)


@dataclass
class OrderProcessData:
    order_id: int
    action: str  # 'cancel', 'refund', 'complete'
    reason: str = ''


class OrderService:
    """
    Service for complex order operations.
    Use only for logic beyond basic CRUD.
    """
    
    @staticmethod
    @transaction.atomic
    def cancel_order(order_id: int, reason: str) -> dict:
        """
        Cancel order and handle inventory, payments, etc.
        Complex operation that goes beyond simple model update.
        """
        logger.info(f'Cancelling order {order_id}')
        
        order = Order.objects.select_for_update().get(pk=order_id)
        
        if order.status not in [OrderStatus.PENDING, OrderStatus.PROCESSING]:
            raise ValueError(f'Cannot cancel order with status {order.status}')
        
        # Restore inventory
        for item in order.items.all():
            item.product.inventory.quantity += item.quantity
            item.product.inventory.save()
        
        # Process refund if paid
        if order.payment and order.payment.status == PaymentStatus.COMPLETED:
            PaymentService.refund_payment(order.payment.id)
        
        order.status = OrderStatus.CANCELLED
        order.cancel_reason = reason
        order.save()
        
        logger.info(f'Order {order_id} cancelled successfully')
        
        return {'success': True, 'order_id': order_id}
    
    @staticmethod
    @transaction.atomic
    def process_payment_and_create_order(order_data: dict, payment_token: str) -> Order:
        """
        Complex operation: create order + process payment + update inventory.
        This is a good case for service layer.
        """
        logger.info(f'Processing order with payment')
        
        order = Order.objects.create(
            customer_id=order_data['customer_id'],
            status=OrderStatus.PENDING
        )
        
        # Create items and update inventory
        for item_data in order_data['items']:
            product = Product.objects.get(pk=item_data['product_id'])
            
            if product.inventory.quantity < item_data['quantity']:
                raise ValueError(f'Insufficient inventory for {product.name}')
            
            product.inventory.quantity -= item_data['quantity']
            product.inventory.save()
            
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=item_data['quantity'],
                unit_price=product.price
            )
        
        # Process payment
        PaymentService.charge(order, payment_token)
        
        order.status = OrderStatus.PROCESSING
        order.save()
        
        return order
```

### DO NOT Use Service Layer for This:

```python
# ❌ BAD: Service layer for basic CRUD
class PostService:
    def create_post(self, data):
        return Post.objects.create(**data)
    
    def update_post(self, pk, data):
        post = Post.objects.get(pk=pk)
        post.title = data['title']
        post.save()
        return post

# ✅ GOOD: Use DRF serializer for basic CRUD
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']
```

## QuerySets and Managers

Organize database queries in dedicated modules or custom managers.

### When to Use Custom QuerySets

Use custom QuerySets/Managers when:
- Complex query logic that is reused across multiple views
- Multiple chained filters
- Complex prefetch/select_related combinations
- Query logic that would clutter the view

**Don't use custom QuerySets for:**
- Simple queries (e.g., `Post.objects.all()` or `Post.objects.filter(status='published')`)
- One-off queries that are only used in one place

### QuerySets in Separate Modules

```python
# querysets/orders.py
from django.db import models
from django.db.models import QuerySet


class OrderQuerySet(QuerySet):
    """Custom queryset for Order model."""
    
    def for_customer(self, customer_id: int):
        """Get orders for a specific customer."""
        return self.filter(customer_id=customer_id)
    
    def pending(self):
        """Get pending orders."""
        return self.filter(status=OrderStatus.PENDING)
    
    def completed(self):
        """Get completed orders."""
        return self.filter(status=OrderStatus.COMPLETED)
    
    def by_date_range(self, start_date, end_date):
        """Get orders within date range."""
        return self.filter(created_at__range=[start_date, end_date])
    
    def with_items(self):
        """Prefetch order items."""
        return self.prefetch_related('items', 'items__product')


class OrderManager(models.Manager):
    """Manager for Order model."""
    
    def get_queryset(self):
        return OrderQuerySet(self.model, using=self._db)
    
    def for_customer(self, customer_id):
        return self.get_queryset().for_customer(customer_id)
    
    def pending(self):
        return self.get_queryset().pending()
    
    def completed(self):
        return self.get_queryset().completed()
    
    def with_items(self):
        return self.get_queryset().with_items()


# models.py
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=OrderStatus.choices)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    
    objects = OrderManager()
    
    def __str__(self):
        return f'Order {self.id}'
```

### Multiple QuerySets per Domain

```python
# querysets/payments.py
class PaymentQuerySet(QuerySet):
    def successful(self):
        return self.filter(status=PaymentStatus.COMPLETED)
    
    def failed(self):
        return self.filter(status=PaymentStatus.FAILED)
    
    def for_order(self, order_id):
        return self.filter(order_id=order_id)


class PaymentManager(models.Manager):
    def get_queryset(self):
        return PaymentQuerySet(self.model, using=self._db)
    
    def successful(self):
        return self.get_queryset().successful()


# querysets/posts.py
class PostQuerySet(QuerySet):
    def published(self):
        return self.filter(status=PublicationStatus.PUBLISHED)
    
    def draft(self):
        return self.filter(status=PublicationStatus.DRAFT)
    
    def by_author(self, author_id):
        return self.filter(author_id=author_id)
    
    def with_relations(self):
        return self.select_related('author', 'category').prefetch_related('tags')


class PostManager(models.Manager):
    def get_queryset(self):
        return PostQuerySet(self.model, using=self._db)
    
    def published(self):
        return self.get_queryset().published()
    
    def with_relations(self):
        return self.get_queryset().with_relations()
```

### Usage in Views

```python
# views.py
class OrderViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Order.objects.with_items().filter(
            customer=self.request.user
        )


# views.py - Django template
def order_list(request):
    orders = Order.objects.pending().for_customer(request.user.id)
    return render(request, 'orders/list.html', {'orders': orders})


def order_detail(request, order_id):
    order = Order.objects.with_items().get(
        pk=order_id,
        customer=request.user
    )
    return render(request, 'orders/detail.html', {'order': order})
```

### Selectors (Alternative to QuerySets in modules)

```python
# selectors.py
class OrderSelector:
    """Alternative: selectors for complex queries."""
    
    @staticmethod
    def get_customer_orders(customer_id: int):
        return Order.objects.filter(
            customer_id=customer_id
        ).select_related(
            'shipping_address', 'billing_address'
        ).prefetch_related('items__product')
    
    @staticmethod
    def get_order_detail(order_id: int):
        return Order.objects.select_related(
            'customer__profile',
            'payment'
        ).prefetch_related(
            'items__product__category'
        ).get(pk=order_id)
    
    @staticmethod
    def get_pending_orders():
        return Order.objects.filter(
            status__in=[OrderStatus.PENDING, OrderStatus.PROCESSING]
        ).order_by('created_at')
```

## Fat Model Problem

Avoid putting business logic in models. Use services instead.

```python
# ❌ BAD: Fat Model
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    
    def calculate_total(self):
        """Business logic in model - AVOID"""
        subtotal = sum(item.total for item in self.items.all())
        tax = subtotal * Decimal('0.21')
        return subtotal + tax
    
    def cancel(self, reason):
        """Business logic in model - AVOID"""
        self.status = OrderStatus.CANCELLED
        self.cancel_reason = reason
        self.save()
        # Inventory, payments, etc.
        for item in self.items.all():
            item.product.inventory.quantity += item.quantity
            item.product.inventory.save()


# ✅ GOOD: Slim Model
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=OrderStatus.choices)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    cancel_reason = models.TextField(blank=True)
    
    def __str__(self):
        return f'Order {self.id}'


# ✅ GOOD: Service Layer
class OrderService:
    @staticmethod
    @transaction.atomic
    def cancel_order(order_id: int, reason: str):
        order = Order.objects.select_for_update().get(pk=order_id)
        order.status = OrderStatus.CANCELLED
        order.cancel_reason = reason
        order.save()
        
        # Restore inventory
        for item in order.items.all():
            item.product.inventory.quantity += item.quantity
            item.product.inventory.save()
```

## Fat View Problem

Avoid putting business logic in views. Use services.

```python
# ❌ BAD: Fat View
class OrderViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        # Business logic in view - AVOID
        order = serializer.save()
        
        # Calculate totals
        subtotal = sum(item.total for item in order.items.all())
        order.total = subtotal * Decimal('1.21')
        order.save()
        
        # Update inventory
        for item in order.items.all():
            item.product.inventory.quantity -= item.quantity
            item.product.inventory.save()
        
        # Send notification
        send_order_notification(order.id)


# ✅ GOOD: Slim View
class OrderViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        order = serializer.save()
        OrderService.calculate_totals(order.id)
        OrderService.update_inventory(order.id)
        send_order_notification.delay(order.id)


# ✅ GOOD: Even better - handle in serializer
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'items', 'shipping_address']
    
    def create(self, validated_data):
        items_data = validated_data.pop('items')
        order = Order.objects.create(**validated_data)
        
        for item_data in items_data:
            OrderItem.objects.create(order=order, **item_data)
        
        OrderService.calculate_totals(order.id)
        
        return order
```

## Summary

1. **Service Layer**: Use only for complex business logic beyond basic CRUD
2. **QuerySets**: Organize in custom managers or modules
3. **Avoid Fat Models**: Keep models slim, use services
4. **Avoid Fat Views**: Keep views thin, delegate to services
5. **DRF Flow**: ViewSet → Serializer → Service → QuerySet → Model
6. **Django Flow**: View → Form → Service → QuerySet → Model

Key principles:
- DRF serializers handle basic CRUD validation
- Django forms handle basic form validation
- Services are for complex business rules
- QuerySets/MManagers handle database queries
- Models are just data structure
