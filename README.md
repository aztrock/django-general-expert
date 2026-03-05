# Django General Expert

A comprehensive skill for expert Django development, focusing on clean code, best practices, and production-ready patterns.

## Overview

This skill provides expert guidance for Django backend development with emphasis on:

- **Clean Code Patterns**: Early return, avoiding bool trap, meaningful naming, SOLID principles
- **Architecture**: Service layer, decoupled business logic, modular design
- **Django REST Framework**: Serializers, viewsets, permissions, performance, drf-standardized-errors
- **Testing**: TestCase-based testing strategies with FactoryBoy (no pytest)
- **Performance**: Query optimization, caching, database patterns
- **Security**: OWASP Top 10, authentication, authorization
- **Advanced Patterns**: Service layer, repository pattern, domain events
- **Task Processing**: Celery patterns with validation and state management
- **Signals**: When to use and avoid Django signals

**Target Audience**: Advanced Django developers working on production applications.

**Django Version**: 4.2 LTS and later

## Installation

```bash
npx skills add jorge-c/skills@django-general-expert
```

Or manually install by adding to your agent's skills directory.

## Features

### Clean Code & Organization
- **Early return pattern** for readable control flow
- **Bool trap avoidance** with status enums in constants.py
- **Descriptive naming conventions** (no comments needed)
- **File organization** by feature and scope
- **SOLID principles** applied throughout
- **No imports inside functions/classes**

### Models & ORM
- Model design best practices
- **UUID for public IDs**, id for internal (security)
- Relationship configurations
- Custom managers and querysets
- **Database indexes** on search fields
- Query optimization (N+1 prevention)
- Migration strategies

### Views & APIs
- Function-based vs class-based views
- Service layer integration
- **Keep views thin**, delegate to services
- URL patterns and routing
- Middleware patterns
- Error handling

### Django REST Framework
- Serializer patterns (read vs write, explicit fields)
- Viewsets and APIViews (**avoid `@api_view`**)
- **Permissions in settings**, NOT in every view
- **drf-standardized-errors** for consistent error handling
- Pagination and filtering
- API versioning

### Filters
- **django-filters** integration
- Custom FilterSets organized by endpoint
- Filter configuration in settings
- Performance considerations

### Testing
- TestCase and APITestCase
- **FactoryBoy** for test data
- **Mock external services** (requests.get/post)
- **DON'T mock the service itself**
- Test organization

### Security
- OWASP Top 10 coverage
- Authentication patterns
- **Minimum permissions**
- **Always use ORM** (sanitize if RawSQL needed)
- Input validation
- Production security checklist

### Performance
- Query optimization (select_related/prefetch_related)
- **Database indexing** strategies
- Caching with Redis
- Bulk operations
- Profiling

### Advanced Patterns
- **Service layer** with dataclasses and typing
- Repository pattern
- Domain events
- **Business logic in services**, NOT in views

### Constants
- **All constants in separate file** (constants.py)
- **Status choices as TextChoices** in constants.py
- Avoid circular imports
- **No magic numbers** in code

### Celery Tasks
- **Validation before creation** (check if entity exists)
- **State verification** before actions (check if email already sent)
- Error handling with exponential backoff
- Proper logging

### Signals
- **When to use**: Cross-app communication, audit logging
- **When to avoid**: Simple business logic, complex operations
- **Signal + Celery** for critical processes
- **Avoid modifying multiple models** in signals

## Structure

```
django-general-expert/
├── SKILL.md                          # Main skill documentation
└── references/
    ├── code-organization.md          # Early return, bool trap, imports
    ├── code-quality.md               # SOLID, DRY, KISS, YAGNI
    ├── constants.md                  # Constants and TextChoices
    ├── models-and-orm.md             # Models, ORM, UUID, indexes
    ├── views-and-urls.md             # Views, URLs
    ├── drf-guidelines.md             # Django REST Framework
    ├── filters.md                    # django-filters
    ├── testing-strategies.md         # Testing strategies
    ├── security-checklist.md         # Security checklist
    ├── performance-optimization.md   # Performance optimization
    ├── advanced-patterns.md          # Service layer, repository
    ├── celery-patterns.md            # Celery task patterns
    ├── signals-guide.md              # Django signals guide
    └── examples.md                   # Complete working examples
```

## Quick Examples

### Early Return Pattern

```python
def process_order(order):
    if not order:
        return None
    if not order.is_paid:
        return None
    if not order.items.exists():
        return None
    
    return process_items(order.items)
```

### Service Layer

```python
# constants.py
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'


# services.py
@dataclass
class OrderData:
    customer_id: int
    items: list[dict]


class OrderService:
    @staticmethod
    def create_order(data: OrderData) -> Order:
        ...
```

### UUID for Security

```python
class Order(models.Model):
    id = models.BigAutoField(primary_key=True)  # Internal
    public_id = models.UUIDField(default=uuid.uuid4, editable=False)  # Public
```

### Filters

```python
# apps/orders/api/v1/order_list/filters.py
class OrderFilter(django_filters.FilterSet):
    status = django_filters.CharFilter(field_name='status')
    min_total = django_filters.NumberFilter(field_name='total', lookup_expr='gte')
```

### Celery Validation

```python
@shared_task
def send_order_email_task(order_id: int):
    try:
        order = Order.objects.get(pk=order_id)
    except Order.DoesNotExist:
        logger.warning(f'Order {order_id} not found')
        return {'status': 'skipped', 'reason': 'not_found'}
    
    if order.confirmation_email_sent:
        return {'status': 'skipped', 'reason': 'already_sent'}
    
    send_email(order.customer.email, ...)
```

## Usage

Once installed, this skill will be automatically available when you work with Django code. The skill will be triggered by:

- Creating Django models or views
- Working with Django REST Framework
- Optimizing queries
- Implementing authentication
- Writing tests
- Security concerns
- Refactoring code
- Setting up Celery tasks
- Using Django signals

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

MIT
