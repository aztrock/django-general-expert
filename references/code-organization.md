# Code Organization & Clean Code Patterns

This guide covers clean code principles specifically for Django projects, with emphasis on maintainability, readability, and testability.

## Early Return Pattern

The early return pattern eliminates nested conditionals by checking for failure conditions first and returning immediately. This makes code more readable and reduces cognitive complexity.

### The Problem: Nested Conditionals

```python
# ❌ BAD: Deeply nested conditionals (arrow code)
def process_payment(order, payment_data):
    if order is not None:
        if order.status == 'pending':
            if payment_data.get('amount') >= order.total:
                if validate_card(payment_data.get('card_number')):
                    result = charge_card(payment_data)
                    if result.success:
                        order.status = 'paid'
                        order.save()
                        send_confirmation_email(order)
                        return {'success': True, 'order': order}
                    else:
                        return {'success': False, 'error': 'Card charge failed'}
                else:
                    return {'success': False, 'error': 'Invalid card'}
            else:
                return {'success': False, 'error': 'Insufficient amount'}
        else:
            return {'success': False, 'error': 'Order not pending'}
    return {'success': False, 'error': 'Order not found'}
```

**Problems:**
- Hard to read (must track multiple levels of indentation)
- Hard to test (many branches)
- Business logic buried in nested blocks
- Difficult to understand the "happy path"

### The Solution: Early Return

```python
# ✅ GOOD: Early return pattern
def process_payment(order, payment_data):
    # Guard clauses - fail fast
    if not order:
        return {'success': False, 'error': 'Order not found'}
    
    if order.status != 'pending':
        return {'success': False, 'error': 'Order not pending'}
    
    if payment_data.get('amount', 0) < order.total:
        return {'success': False, 'error': 'Insufficient amount'}
    
    if not validate_card(payment_data.get('card_number')):
        return {'success': False, 'error': 'Invalid card'}
    
    # Main logic - happy path
    result = charge_card(payment_data)
    if not result.success:
        return {'success': False, 'error': 'Card charge failed'}
    
    order.status = 'paid'
    order.save()
    send_confirmation_email(order)
    
    return {'success': True, 'order': order}
```

**Benefits:**
- Clear guard clauses at the top
- Happy path is easy to follow
- Each condition is tested independently
- Less cognitive load

### Early Return in Views

```python
# ❌ BAD: Nested conditions in view
def update_post(request, pk):
    if request.method == 'POST':
        post = get_object_or_404(Post, pk=pk)
        if request.user == post.author:
            form = PostForm(request.POST, instance=post)
            if form.is_valid():
                post = form.save()
                if post.is_published:
                    notify_subscribers(post)
                return redirect('post_detail', pk=post.pk)
            else:
                return render(request, 'post/edit.html', {'form': form})
        else:
            return HttpResponseForbidden()
    else:
        post = get_object_or_404(Post, pk=pk)
        form = PostForm(instance=post)
        return render(request, 'post/edit.html', {'form': form})

# ✅ GOOD: Class-Based View with early return
from django.views.generic import UpdateView
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.shortcuts import get_object_or_404
from django.http import HttpResponseForbidden


class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'post/edit.html'
    
    def test_func(self):
        post = self.get_object()
        return self.request.user == post.author
    
    def get_success_url(self):
        return self.object.get_absolute_url()
    
    def form_valid(self, form):
        response = super().form_valid(form)
        if self.object.is_published:
            notify_subscribers(self.object)
        return response
    
    def handle_no_permission(self):
        return HttpResponseForbidden("You can only edit your own posts")
```

### Early Return in DRF Views

```python
# ✅ GOOD: Early return in DRF with raise ValidationError
# Note: With drf-standardized-errors, ValidationError is automatically converted to proper HTTP responses
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.exceptions import ValidationError


class OrderViewSet(viewsets.ModelViewSet):
    # Permissions should be configured globally in settings.py
    # Only add specific permissions here when needed
    
    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        order = self.get_object()
        
        # Check ownership
        if order.user != request.user:
            raise ValidationError({'detail': 'Not your order'})
        
        # Check status
        if order.status not in ['pending', 'paid']:
            raise ValidationError({'detail': 'Cannot cancel this order'})
        
        # Business logic
        order.status = 'cancelled'
        order.save()
        
        refund_if_needed(order)
        notify_merchant(order)
        
        return Response({'status': 'cancelled'}, status=status.HTTP_200_OK)
```

### Global Permissions Configuration

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Django settings.py
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/dashboard/'
```

### Early Return in Serializers

```python
# ✅ GOOD: Validation with early return
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'items', 'shipping_address', 'payment_method']
    
    def validate(self, data):
        items = data.get('items', [])
        
        if not items:
            raise serializers.ValidationError({'items': 'Order must have items'})
        
        if not data.get('shipping_address'):
            raise serializers.ValidationError({'shipping_address': 'Required'})
        
        if not data.get('payment_method'):
            raise serializers.ValidationError({'payment_method': 'Required'})
        
        # All validations passed, check business rules
        if not all_items_in_stock(items):
            raise serializers.ValidationError({'items': 'Some items out of stock'})
        
        return data
```

## Bool Trap

The "bool trap" occurs when multiple boolean flags create confusing, hard-to-maintain code. Each combination of flags can represent a different state, leading to complexity.

### The Problem: Multiple Booleans

```python
# ❌ BAD: Bool trap
class Order(models.Model):
    is_pending = models.BooleanField(default=True)
    is_paid = models.BooleanField(default=False)
    is_shipped = models.BooleanField(default=False)
    is_delivered = models.BooleanField(default=False)
    is_cancelled = models.BooleanField(default=False)
    is_refunded = models.BooleanField(default=False)

# What does this mean?
order = Order.objects.get(pk=1)
if order.is_paid and not order.is_shipped and not order.is_cancelled:
    # Is this valid? What about is_delivered?
    prepare_shipment(order)

# Invalid states are possible!
# is_paid=True, is_shipped=True, is_delivered=False, is_cancelled=True ???
```

**Problems:**
- Invalid state combinations possible
- Unclear what the "current" state is
- Complex conditional logic
- Database inconsistencies
- Hard to enforce transitions

### The Solution: Status Field with Choices

```python
# constants.py
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'
    SHIPPED = 'shipped', 'Shipped'
    DELIVERED = 'delivered', 'Delivered'
    CANCELLED = 'cancelled', 'Cancelled'
    REFUNDED = 'refunded', 'Refunded'


# models.py
from .constants import OrderStatus


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )
    
    # Optional: Helper properties for common checks
    @property
    def is_paid(self):
        return self.status == OrderStatus.PAID
    
    @property
    def can_ship(self):
        return self.status == OrderStatus.PAID
    
    @property
    def can_cancel(self):
        return self.status in [OrderStatus.PENDING, OrderStatus.PAID]


# Clear state checks
# ❌ BAD: Using get() can raise DoesNotExist
order = Order.objects.get(pk=1)
if order.can_ship:
    prepare_shipment(order)

# ✅ GOOD: Using filter().first() avoids unnecessary exceptions
order = Order.objects.filter(pk=1).first()
if not order:
    raise ValidationError('Order not found')
    
if order.can_ship:
    prepare_shipment(order)

# State transitions are explicit
def ship_order(order_id):
    order = Order.objects.filter(pk=order_id).first()
    if not order:
        raise ValidationError('Order not found')
    
    if order.status != OrderStatus.PAID:
        raise ValidationError("Only paid orders can be shipped")
    
    order.status = OrderStatus.SHIPPED
    order.shipped_at = timezone.now()
    order.save()
```

### Status Transitions

```python
# constants.py
ORDER_STATUS_TRANSITIONS = {
    OrderStatus.PENDING: [OrderStatus.PAID, OrderStatus.CANCELLED],
    OrderStatus.PAID: [OrderStatus.SHIPPED, OrderStatus.CANCELLED, OrderStatus.REFUNDED],
    OrderStatus.SHIPPED: [OrderStatus.DELIVERED, OrderStatus.REFUNDED],
    OrderStatus.DELIVERED: [OrderStatus.REFUNDED],
    OrderStatus.CANCELLED: [],  # Terminal state
    OrderStatus.REFUNDED: [],   # Terminal state
}


# models.py
class Order(models.Model):
    def can_transition_to(self, new_status):
        return new_status in ORDER_STATUS_TRANSITIONS.get(self.status, [])
    
    def transition_to(self, new_status):
        if not self.can_transition_to(new_status):
            raise ValueError(
                f"Cannot transition from {self.status} to {new_status}"
            )
        self.status = new_status
        self.save()


# Usage
# ❌ BAD: Using get() can raise DoesNotExist
order = Order.objects.get(pk=1)
if order.can_transition_to(OrderStatus.SHIPPED):
    order.transition_to(OrderStatus.SHIPPED)

# ✅ GOOD: Using filter().first() avoids unnecessary exceptions
order = Order.objects.filter(pk=1).first()
if not order:
    raise ValidationError('Order not found')
    
if order.can_transition_to(OrderStatus.SHIPPED):
    order.transition_to(OrderStatus.SHIPPED)
```

### Common Bool Trap Examples

```python
# ❌ BAD: Bool trap for user state
class User(models.Model):
    is_active = models.BooleanField(default=True)
    is_verified = models.BooleanField(default=False)
    is_premium = models.BooleanField(default=False)
    is_staff = models.BooleanField(default=False)
    is_banned = models.BooleanField(default=False)

# ✅ GOOD: Account status enum
class AccountStatus(models.TextChoices):
    ACTIVE = 'active', 'Active'
    UNVERIFIED = 'unverified', 'Unverified'
    SUSPENDED = 'suspended', 'Suspended'
    BANNED = 'banned', 'Banned'


class User(models.Model):
    account_status = models.CharField(
        max_length=20,
        choices=AccountStatus.choices,
        default=AccountStatus.UNVERIFIED
    )
    account_type = models.CharField(
        max_length=20,
        choices=[('free', 'Free'), ('premium', 'Premium'), ('staff', 'Staff')],
        default='free'
    )
```

```python
# ❌ BAD: Bool trap for content
class Article(models.Model):
    is_draft = models.BooleanField(default=True)
    is_published = models.BooleanField(default=False)
    is_featured = models.BooleanField(default=False)
    is_archived = models.BooleanField(default=False)

# ✅ GOOD: Publication status
class PublicationStatus(models.TextChoices):
    DRAFT = 'draft', 'Draft'
    PUBLISHED = 'published', 'Published'
    FEATURED = 'featured', 'Featured'
    ARCHIVED = 'archived', 'Archived'


class Article(models.Model):
    publication_status = models.CharField(
        max_length=20,
        choices=PublicationStatus.choices,
        default=PublicationStatus.DRAFT
    )
```

### When Booleans Are OK

Booleans are appropriate for:
- Simple on/off flags (no state transitions)
- Independent features (not mutually exclusive)
- Settings/preferences

```python
# ✅ GOOD: Boolean for simple flags
class User(models.Model):
    email_notifications_enabled = models.BooleanField(default=True)
    sms_notifications_enabled = models.BooleanField(default=False)
    dark_mode_enabled = models.BooleanField(default=False)
    marketing_consent = models.BooleanField(default=False)
```

## Decoupling Business Logic

Keep views thin. Move business logic to services, keeping views focused on HTTP concerns.

### The Problem: Fat Views

```python
# ❌ BAD: Business logic in view
def checkout(request):
    if request.method == 'POST':
        cart = Cart.objects.get(user=request.user)
        
        # Validate cart has items
        if not cart.items.exists():
            return JsonResponse({'error': 'Cart is empty'}, status=400)
        
        # Check inventory
        for item in cart.items.all():
            if item.quantity > item.product.stock:
                return JsonResponse(
                    {'error': f'{item.product.name} out of stock'}, 
                    status=400
                )
        
        # Calculate totals
        subtotal = sum(item.total_price for item in cart.items.all())
        tax = subtotal * 0.21
        shipping = 10 if subtotal < 100 else 0
        total = subtotal + tax + shipping
        
        # Create order
        order = Order.objects.create(
            user=request.user,
            subtotal=subtotal,
            tax=tax,
            shipping=shipping,
            total=total,
            status='pending'
        )
        
        # Create order items and update inventory
        for item in cart.items.all():
            OrderItem.objects.create(
                order=order,
                product=item.product,
                quantity=item.quantity,
                price=item.product.price
            )
            item.product.stock -= item.quantity
            item.product.save()
        
        # Process payment
        payment_result = stripe.Charge.create(
            amount=int(total * 100),
            currency='usd',
            source=request.POST.get('stripe_token')
        )
        
        if payment_result['status'] == 'succeeded':
            order.status = 'paid'
            order.payment_id = payment_result['id']
            order.save()
            
            # Clear cart
            cart.items.all().delete()
            
            # Send emails
            send_order_confirmation_email(order)
            send_merchant_notification(order)
            
            return JsonResponse({'order_id': order.id}, status=201)
        else:
            order.status = 'failed'
            order.save()
            return JsonResponse({'error': 'Payment failed'}, status=400)
```

**Problems:**
- View does too much (validation, calculation, payment, email)
- Hard to test (requires request object)
- Hard to reuse logic
- Hard to modify (changes affect multiple concerns)

### The Solution: Service Layer

```python
# services.py
from django.core.exceptions import ValidationError
from django.db import transaction


class OrderService:
    @staticmethod
    def validate_cart(cart):
        if not cart.items.exists():
            raise ValidationError('Cart is empty')
        
        for item in cart.items.select_related('product'):
            if item.quantity > item.product.stock:
                raise ValidationError(
                    f'{item.product.name} out of stock'
                )
        
        return True
    
    @staticmethod
    def calculate_totals(cart):
        subtotal = sum(item.total_price for item in cart.items.all())
        tax = subtotal * 0.21
        shipping = 10 if subtotal < 100 else 0
        total = subtotal + tax + shipping
        
        return {
            'subtotal': subtotal,
            'tax': tax,
            'shipping': shipping,
            'total': total
        }
    
    @staticmethod
    @transaction.atomic
    def create_order(user, cart, shipping_address):
        # Validate
        OrderService.validate_cart(cart)
        
        # Calculate totals
        totals = OrderService.calculate_totals(cart)
        
        # Create order
        order = Order.objects.create(
            user=user,
            subtotal=totals['subtotal'],
            tax=totals['tax'],
            shipping=totals['shipping'],
            total=totals['total'],
            shipping_address=shipping_address,
            status='pending'
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
        
        # Update inventory
        for item in cart.items.select_related('product'):
            item.product.stock -= item.quantity
            item.product.save()
        
        return order
    
    @staticmethod
    @transaction.atomic
    def complete_checkout(user, cart, shipping_address, stripe_token):
        order = OrderService.create_order(user, cart, shipping_address)
        
        result = PaymentService.process(order, stripe_token)
        
        if not result.success:
            raise ValidationError(f'Payment failed: {result.error}')
        
        # Clear cart
        cart.items.all().delete()
        
        # Send notifications
        NotificationService.send_order_confirmation(order)
        
        return order


# views.py
from .services import OrderService


@require_POST
def checkout(request):
    try:
        cart = Cart.objects.get(user=request.user)
        stripe_token = request.POST.get('stripe_token')
        
        order = OrderService.complete_checkout(
            user=request.user,
            cart=cart,
            shipping_address=request.POST['shipping_address'],
            stripe_token=stripe_token
        )
        
        return JsonResponse({'order_id': order.id}, status=201)
    
    except ValidationError as e:
        return JsonResponse({'error': str(e)}, status=400)
    except Cart.DoesNotExist:
        return JsonResponse({'error': 'Cart not found'}, status=404)
```

**Benefits:**
- View is thin (only handles HTTP)
- Service is testable (no request object needed)
- Logic is reusable (can be called from API, CLI, etc.)
- Easy to modify (changes are isolated)

### Service Layer Testing

```python
# tests/test_services.py
from django.test import TestCase
from myapp.services import OrderService
from myapp.factories import UserFactory, CartFactory, ProductFactory


class OrderServiceTest(TestCase):
    def setUp(self):
        self.user = UserFactory()
        self.cart = CartFactory(user=self.user)
        self.product = ProductFactory(stock=10, price=100)
        self.cart.items.create(product=self.product, quantity=2)
    
    def test_validate_cart_empty(self):
        self.cart.items.all().delete()
        
        with self.assertRaises(ValidationError):
            OrderService.validate_cart(self.cart)
    
    def test_validate_cart_out_of_stock(self):
        self.product.stock = 1
        self.product.save()
        
        with self.assertRaises(ValidationError):
            OrderService.validate_cart(self.cart)
    
    def test_calculate_totals(self):
        totals = OrderService.calculate_totals(self.cart)
        
        self.assertEqual(totals['subtotal'], 200)
        self.assertEqual(totals['tax'], 42)
        self.assertEqual(totals['shipping'], 0)
        self.assertEqual(totals['total'], 242)
    
    def test_create_order(self):
        totals = OrderService.calculate_totals(self.cart)
        order = OrderService.create_order(
            user=self.user,
            cart=self.cart,
            shipping_address='123 Test St'
        )
        
        self.assertEqual(order.user, self.user)
        self.assertEqual(order.total, 242)
        self.assertEqual(order.items.count(), 1)
        
        # Verify inventory updated
        self.product.refresh_from_db()
        self.assertEqual(self.product.stock, 8)
```

## Naming Conventions

### Variables and Functions

```python
# ❌ BAD: Unclear names
def process(d):
    x = d.get('items')
    r = []
    for i in x:
        r.append(i['value'])
    return sum(r)

# ✅ GOOD: Descriptive names
def calculate_order_total(order_data):
    items = order_data.get('items', [])
    item_totals = [item['value'] for item in items]
    return sum(item_totals)
```

### QuerySets: Use Descriptive Manager Methods

```python
# ❌ BAD: Complex filters scattered in code
active_paid_orders = Order.objects.filter(
    status='paid',
    created_at__gte=timezone.now() - timedelta(days=30),
    user__is_active=True
)

# ✅ GOOD: Encapsulate in manager
class OrderManager(models.Manager):
    def recent_paid(self):
        return self.filter(
            status=OrderStatus.PAID,
            created_at__gte=timezone.now() - timedelta(days=30)
        )
    
    def for_active_users(self):
        return self.filter(user__is_active=True)


class Order(models.Model):
    objects = OrderManager()


# Usage
active_paid_orders = Order.objects.recent_paid().for_active_users()
```

### Booleans: Use is_, has_, can_ Prefixes

```python
# ❌ BAD: Unclear boolean names
user.active
user.admin
order.paid

# ✅ GOOD: Clear boolean names
user.is_active
user.is_admin
order.is_paid

# ✅ GOOD: has_ for possession
user.has_verified_email
product.has_discount
order.has_items

# ✅ GOOD: can_ for permissions/abilities
user.can_delete_post
order.can_be_cancelled
product.can_be_purchased
```

## File Organization

### Feature-Based Structure

For small projects, a flat structure works fine. For medium to large projects, separate into modules:

```
myproject/
├── manage.py
├── myproject/
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── users/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── constants.py
│   │   ├── managers.py        # Custom managers
│   │   ├── services.py        # Business logic
│   │   ├── views/
│   │   │   ├── __init__.py
│   │   │   ├── orders.py      # Class-based views
│   │   │   └── payments.py
│   │   ├── forms/
│   │   │   ├── __init__.py
│   │   │   └── orders.py
│   │   ├── integrations/     # External service integrations
│   │   │   ├── __init__.py
│   │   │   ├── sms.py         # SMS providers
│   │   │   ├── email.py       # Email providers
│   │   │   └── payments.py    # Payment gateways
│   │   ├── tests/
│   │   └── migrations/
│   ├── orders/
│   │   ├── models.py
│   │   ├── constants.py
│   │   ├── services.py
│   │   ├── views/
│   │   ├── forms/
│   │   ├── queryset.py      # Complex query methods
│   │   └── tests/
│   └── products/
│       └── ...
├── core/
│   ├── __init__.py
│   ├── constants.py
│   ├── mixins.py
│   ├── utils.py
│   └── validators.py
└── requirements/
    ├── base.txt
    ├── development.txt
    └── production.txt
```

### DRF Project Structure with Versioning

For DRF APIs in medium/large projects, organize by version:

```
apps/
├── orders/
│   ├── __init__.py
│   ├── models.py
│   ├── constants.py
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── urls.py
│   │       ├── serializers.py
│   │       ├── viewsets.py
│   │       └── permissions.py  # Specific permissions only
│   └── services.py
├── products/
│   └── api/
│       └── v1/
│           ├── urls.py
│           ├── serializers.py
│           └── viewsets.py
```

### Module Responsibilities

Each module has a specific purpose:

| Module | Responsibility |
|--------|---------------|
| `constants.py` | Status enums, TextChoices, configuration values |
| `models.py` | Data models, field definitions, simple properties |
| `managers.py` | Custom QuerySets and Managers |
| `services.py` | Business logic, complex operations, transactions |
| `views/` or `viewsets.py` | HTTP handling, connect to services/forms |
| `forms/` | Form validation for template views |
| `serializers.py` | Serialization/deserialization for DRF |
| `queryset.py` | Complex query composition |
| `integrations/` | External service integrations (SMS, email, payments) |

### External Integrations

Create separate modules for third-party integrations. Use abstract base classes for multiple providers:

```python
# integrations/sms.py
from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass
class SMSResult:
    success: bool
    message_id: str = ''
    error: str = ''


class BaseSMSProvider(ABC):
    """Abstract base class for SMS providers."""
    
    @abstractmethod
    def send(self, to: str, message: str) -> SMSResult:
        pass


class TwilioSMSProvider(BaseSMSProvider):
    def __init__(self, account_sid: str, auth_token: str):
        self.client = TwilioClient(account_sid, auth_token)
    
    def send(self, to: str, message: str) -> SMSResult:
        try:
            result = self.client.messages.create(
                body=message,
                from_=self.from_number,
                to=to
            )
            return SMSResult(success=True, message_id=result.sid)
        except Exception as e:
            return SMSResult(success=False, error=str(e))


class AWS SNSProvider(BaseSMSProvider):
    def __init__(self, region: str, access_key: str, secret_key: str):
        self.client = boto3.client('sns', region_name=region)
    
    def send(self, to: str, message: str) -> SMSResult:
        try:
            result = self.client.publish(
                PhoneNumber=to,
                Message=message
            )
            return SMSResult(success=True, message_id=result['MessageId'])
        except Exception as e:
            return SMSResult(success=False, error=str(e))


# Usage in services
class NotificationService:
    def __init__(self, sms_provider: BaseSMSProvider):
        self.sms_provider = sms_provider
    
    def send_sms(self, to: str, message: str):
        result = self.sms_provider.send(to, message)
        if not result.success:
            logger.error(f'SMS failed: {result.error}')
        return result
```

## Imports

### All Imports at Module Level

```python
# ✅ GOOD: All imports at top
from django.db import models
from .constants import OrderStatus
from .services import OrderService


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )


# ❌ BAD: Import inside function
class Order(models.Model):
    def calculate_total(self):
        from .services import OrderService  # WRONG!
        return OrderService.calculate(self)


# ❌ BAD: Import inside class
class Order(models.Model):
    from .constants import OrderStatus  # WRONG!
    
    status = models.CharField(...)
```

**Rule: ALL imports at module level, NEVER inside functions/classes.**

## Summary

1. **Early Return**: Check guard conditions first, return early. Avoid nested conditionals.
2. **Avoid Bool Trap**: Use status enums instead of multiple booleans for state.
3. **Decouple Logic**: Keep views thin, move business logic to services.
4. **Descriptive Names**: Use clear names for variables, functions, and querysets.
5. **Feature-Based Organization**: Group by feature, not by file type.
6. **All Imports at Top**: Never import inside functions or classes.
