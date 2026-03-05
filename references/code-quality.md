# Code Quality Guide

Comprehensive guide to maintaining high code quality in Django projects.

## Principles

### Boy Scout Rule

**Always leave code better than you found it.**

```python
# ❌ BAD: Ignoring mess
def process_order(order):
    x = calculate(order)  # What is x?
    if x > 100:
        x = x * 1.1  # Magic number!
    return x


# ✅ GOOD: Clean as you go
def process_order(order):
    subtotal = calculate_subtotal(order)
    
    if subtotal > LARGE_ORDER_THRESHOLD:
        subtotal *= BULK_DISCOUNT_MULTIPLIER
    
    return subtotal
```

---

### DRY (Don't Repeat Yourself)

```python
# ❌ BAD: Repeated logic
def calculate_order_total(order):
    subtotal = sum(item.price * item.quantity for item in order.items.all())
    tax = subtotal * 0.21
    return subtotal + tax


def calculate_invoice_total(invoice):
    subtotal = sum(item.price * item.quantity for item in invoice.items.all())
    tax = subtotal * 0.21
    return subtotal + tax


# ✅ GOOD: Extract common logic
def calculate_items_total(items, tax_rate=TAX_RATE):
    subtotal = sum(item.price * item.quantity for item in items)
    tax = subtotal * tax_rate
    return subtotal + tax


def calculate_order_total(order):
    return calculate_items_total(order.items.all())


def calculate_invoice_total(invoice):
    return calculate_items_total(invoice.items.all())
```

---

### KISS (Keep It Simple, Stupid)

```python
# ❌ BAD: Over-engineered
class OrderProcessor:
    def __init__(self, order):
        self._order = order
        self._strategies = {
            'total': TotalCalculationStrategy(),
            'tax': TaxCalculationStrategy(),
            'shipping': ShippingCalculationStrategy(),
        }
    
    def process(self):
        return sum(
            strategy.calculate(self._order) 
            for strategy in self._strategies.values()
        )


# ✅ GOOD: Simple and clear
def process_order(order):
    subtotal = calculate_subtotal(order.items)
    tax = calculate_tax(subtotal)
    shipping = calculate_shipping(subtotal)
    return subtotal + tax + shipping
```

---

### YAGNI (You Aren't Gonna Need It)

```python
# ❌ BAD: Premature optimization
class Order(models.Model):
    # Fields we might need someday...
    estimated_delivery = models.DateField(null=True)
    gift_wrap = models.BooleanField(default=False)
    gift_message = models.TextField(blank=True)
    customer_notes = models.TextField(blank=True)
    internal_notes = models.TextField(blank=True)
    priority_level = models.IntegerField(default=0)
    # ... 20 more fields


# ✅ GOOD: Only what you need now
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

### SOLID Principles

#### Single Responsibility

```python
# ❌ BAD: Multiple responsibilities
class OrderService:
    def create_order(self, data):
        order = Order.objects.create(**data)
        self.send_confirmation_email(order)
        self.update_inventory(order)
        self.charge_payment(order)
        return order
    
    def send_confirmation_email(self, order):
        ...
    
    def update_inventory(self, order):
        ...
    
    def charge_payment(self, order):
        ...


# ✅ GOOD: Single responsibility
class OrderService:
    def create_order(self, data):
        return Order.objects.create(**data)


class EmailService:
    def send_order_confirmation(self, order):
        ...


class InventoryService:
    def update_for_order(self, order):
        ...


class PaymentService:
    def charge_order(self, order):
        ...
```

#### Open/Closed

```python
# ❌ BAD: Modifying existing code for new payment types
def process_payment(order, payment_type):
    if payment_type == 'credit_card':
        ...
    elif payment_type == 'paypal':
        ...
    elif payment_type == 'stripe':  # Adding new type modifies function
        ...


# ✅ GOOD: Open for extension, closed for modification
from abc import ABC, abstractmethod


class PaymentProcessor(ABC):
    @abstractmethod
    def charge(self, order):
        pass


class CreditCardProcessor(PaymentProcessor):
    def charge(self, order):
        ...


class PayPalProcessor(PaymentProcessor):
    def charge(self, order):
        ...


class StripeProcessor(PaymentProcessor):  # New type = new class
    def charge(self, order):
        ...
```

#### Liskov Substitution

```python
# ❌ BAD: Violates LSP
class Order:
    def calculate_total(self):
        return sum(item.price for item in self.items)


class DigitalOrder(Order):
    def calculate_total(self):
        return 0  # Free! Different behavior


# ✅ GOOD: Consistent behavior
class Order:
    def calculate_total(self):
        return sum(item.price for item in self.items)


class DigitalOrder(Order):
    def calculate_total(self):
        # Same behavior, different implementation
        return sum(item.price for item in self.items)
```

#### Interface Segregation

```python
# ❌ BAD: Fat interface
class PaymentGateway:
    def charge(self, amount):
        ...
    
    def refund(self, amount):
        ...
    
    def subscribe(self, plan):
        ...
    
    def cancel_subscription(self):
        ...


# ✅ GOOD: Segregated interfaces
class PaymentGateway:
    def charge(self, amount):
        ...


class RefundGateway:
    def refund(self, amount):
        ...


class SubscriptionGateway:
    def subscribe(self, plan):
        ...
    
    def cancel_subscription(self):
        ...
```

#### Dependency Inversion

```python
# ❌ BAD: High-level depends on low-level
class OrderService:
    def __init__(self):
        self.payment = StripePaymentGateway()  # Direct dependency
    
    def process_payment(self, order):
        return self.payment.charge(order.total)


# ✅ GOOD: Depend on abstractions
class OrderService:
    def __init__(self, payment_gateway: PaymentGateway):
        self.payment = payment_gateway  # Injected dependency
    
    def process_payment(self, order):
        return self.payment.charge(order.total)
```

---

## Function Guidelines

### Length

**Keep functions short: 50-70 lines of code maximum.**

```python
# ❌ BAD: 200 lines function
def process_order(request):
    # 50 lines of validation
    ...
    # 50 lines of business logic
    ...
    # 50 lines of payment processing
    ...
    # 50 lines of email sending
    ...


# ✅ GOOD: Small, focused functions
def process_order(request):
    data = validate_order_data(request)
    order = create_order(data)
    process_payment(order)
    send_confirmation_email(order)
    return order


def validate_order_data(request):
    ...


def create_order(data):
    ...


def process_payment(order):
    ...


def send_confirmation_email(order):
    ...
```

---

### Descriptive Names

```python
# ❌ BAD: Cryptic names
def calc(o):
    x = 0
    for i in o.stuff:
        x += i.p * i.q
    return x


# ✅ GOOD: Self-documenting names
def calculate_order_total(order):
    subtotal = 0
    for item in order.items:
        subtotal += item.price * item.quantity
    return subtotal
```

**Rules:**
- Use descriptive names (no comments needed)
- Name should explain WHAT, not HOW
- Avoid abbreviations
- Be consistent

---

### Magic Numbers

```python
# ❌ BAD: Magic numbers
def calculate_discount(order):
    if order.total > 100:
        return order.total * 0.1
    elif order.total > 50:
        return order.total * 0.05
    return 0


# constants.py
LARGE_ORDER_THRESHOLD = 100
MEDIUM_ORDER_THRESHOLD = 50
LARGE_ORDER_DISCOUNT = 0.1
MEDIUM_ORDER_DISCOUNT = 0.05


# services.py
from .constants import (
    LARGE_ORDER_THRESHOLD,
    MEDIUM_ORDER_THRESHOLD,
    LARGE_ORDER_DISCOUNT,
    MEDIUM_ORDER_DISCOUNT
)


def calculate_discount(order):
    if order.total > LARGE_ORDER_THRESHOLD:
        return order.total * LARGE_ORDER_DISCOUNT
    elif order.total > MEDIUM_ORDER_THRESHOLD:
        return order.total * MEDIUM_ORDER_DISCOUNT
    return 0
```

---

### Dead Code

```python
# ❌ BAD: Dead code
def process_order(order):
    # Old validation (not used anymore)
    # if order.total < 0:
    #     raise ValueError("Invalid total")
    
    # New validation
    validate_order(order)
    
    # TODO: implement payment later
    # payment_result = process_payment(order)
    
    return order


# ✅ GOOD: Remove dead code
def process_order(order):
    validate_order(order)
    return order
```

---

## Documentation

### Type Hints

```python
# ❌ BAD: No type hints
def calculate_total(items):
    return sum(item.price * item.quantity for item in items)


# ✅ GOOD: With type hints
from typing import List
from decimal import Decimal


def calculate_total(items: List[OrderItem]) -> Decimal:
    return sum(item.price * item.quantity for item in items)
```

---

### Comments

```python
# ❌ BAD: Comments explain WHAT (obvious from code)
def calculate_total(items):
    # Calculate total
    subtotal = 0
    # Loop through items
    for item in items:
        # Add item price * quantity
        subtotal += item.price * item.quantity
    # Return total
    return subtotal


# ✅ GOOD: Comments explain WHY
def calculate_total(items):
    # Use Decimal for precision (avoid floating point errors with money)
    subtotal = Decimal('0')
    for item in items:
        subtotal += item.price * item.quantity
    return subtotal
```

---

### No Useless Docstrings

```python
# ❌ BAD: Useless docstring
def calculate_total(items):
    """Calculate total."""
    return sum(item.price * item.quantity for item in items)


def send_email(user):
    """Send email to user."""
    send_mail(to=user.email, ...)


# ✅ GOOD: Self-documenting code (function name is clear)
def calculate_total(items):
    return sum(item.price * item.quantity for item in items)


def send_welcome_email(user):
    send_mail(to=user.email, subject='Welcome!', ...)
```

**Rule**: If function name is self-explanatory, no docstring needed.

---

## Clarity Over Tricks

### Avoid Clever Code

```python
# ❌ BAD: Clever but unclear
total = sum(map(lambda x: x.price * x.quantity, items))


# ✅ GOOD: Clear and readable
total = 0
for item in items:
    total += item.price * item.quantity


# ✅ GOOD: Even better (list comprehension)
total = sum(item.price * item.quantity for item in items)
```

---

### Explicit is Better Than Implicit

```python
# ❌ BAD: Implicit behavior
def get_user(**kwargs):
    return User.objects.filter(**kwargs).first()


# ✅ GOOD: Explicit behavior
def get_user_by_id(user_id: int) -> User:
    return User.objects.get(pk=user_id)


def get_user_by_email(email: str) -> User:
    return User.objects.get(email=email)
```

---

## Summary

1. **Boy Scout Rule**: Leave code better than you found it
2. **DRY**: Don't repeat yourself
3. **KISS**: Keep it simple, stupid
4. **YAGNI**: You aren't gonna need it
5. **SOLID**: Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion
6. **Function length**: 50-70 lines maximum
7. **Descriptive names**: Self-documenting code
8. **No magic numbers**: Use constants
9. **Remove dead code**: Clean up as you go
10. **Type hints**: Use them for clarity
11. **Comments explain WHY**: Not WHAT
12. **No useless docstrings**: Function name should be clear
13. **Clarity over tricks**: Readable code wins
