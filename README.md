# Django General Expert

A comprehensive skill for expert Django development, focusing on clean code, best practices, and production-ready patterns.

## Overview

This skill provides expert guidance for Django backend development with emphasis on:

- **Clean Code Patterns**: Early return, avoiding bool trap, meaningful naming, SOLID principles
- **Architecture**: Service layer, decoupled business logic, modular design
- **Django REST Framework**: Serializers, viewsets, permissions, performance, drf-standardized-errors
- **Testing**: TestCase-based testing strategies with FactoryBoy (no pytest)
- **Performance**: Query optimization, caching, database patterns
- **Security**: OWASP Top 10, authentication, authorization, password hashers, rate limiting
- **Advanced Patterns**: Service layer, repository pattern, domain events
- **Task Processing**: Celery patterns with validation, state management, django-celery-beat
- **Signals**: When to use and avoid Django signals

**Target Audience**: Advanced Django developers working on production applications.

**Django Version**: 4.2 LTS and later

## Installation

```bash
npx skills add aztrock/skills@django-general-expert
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
- Class-based views (CBV) over function-based views (FBV)
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
- Use `raise ValidationError` instead of returning error responses
- Use `filter().first()` instead of `get()` to avoid exceptions

### Filters
- **django-filters** integration
- Custom FilterSets organized by endpoint
- **Filter configuration in settings**, NOT in views
- Performance considerations

### Testing
- TestCase and APITestCase
- **FactoryBoy** for test data
- **Mock external services** (requests.get/post)
- **DON'T mock the service itself**
- Modular test organization (tests/factories/, tests/views/, etc.)

### Security
- OWASP Top 10 coverage
- Authentication patterns (JWT, session management)
- Password hashers (Argon2, PBKDF2)
- Rate limiting for DRF
- **Minimum permissions**
- **Always use ORM** (sanitize if RawSQL needed)
- Input validation
- Production security checklist

### Performance
- Query optimization (select_related/prefetch_related)
- Use `.first()` instead of `exists()` when you need the object
- Use values()/values_list() for flat data
- Use annotate() for complex filters and aggregations
- **Database indexing** strategies
- Caching with Redis
- Bulk operations
- Profiling with django-silk

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
- **django-celery-beat** for database-backed periodic tasks

### Signals
- **When to use**: Cross-app communication, audit logging
- **When to avoid**: Simple business logic, complex operations
- **Signal + Celery** for critical processes
- **Avoid modifying multiple models** in signals

### Additional Topics
- Django Admin customization
- Forms and ModelForms
- Middleware development
- Database migrations
- Logging configuration
- Static and media files
- Project structure
- Third-party packages recommendations
- Docker and deployment

## Structure

```
django-general-expert/
├── SKILL.md                          # Main skill documentation
├── README.md                         # This file
└── references/
    ├── code-organization.md          # Early return, bool trap, imports
    ├── code-quality.md               # SOLID, DRY, KISS, YAGNI
    ├── constants.md                  # Constants and TextChoices
    ├── models-and-orm.md             # Models, ORM, UUID, indexes
    ├── views-and-urls.md             # Views, URLs, CBV
    ├── drf-guidelines.md            # Django REST Framework
    ├── filters.md                    # django-filters
    ├── testing-strategies.md         # Testing strategies
    ├── security-checklist.md        # Security checklist
    ├── performance-optimization.md   # Performance optimization
    ├── advanced-patterns.md          # Service layer, repository
    ├── celery-patterns.md            # Celery task patterns
    ├── signals-guide.md              # Django signals guide
    ├── examples.md                   # Complete working examples
    ├── admin-guide.md                # Django Admin customization
    ├── forms-modelforms.md          # Forms and ModelForms
    ├── authentication.md             # Authentication patterns
    ├── middleware.md                 # Custom middleware
    ├── migrations.md                 # Database migrations
    ├── logging.md                    # Logging configuration
    ├── static-media-files.md         # Static and media files
    ├── project-structure.md          # Project organization
    ├── third-party-packages.md      # Recommended packages
    └── docker-deployment.md         # Docker and deployment
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
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}

# views.py - filter_backends NOT needed here
class OrderViewSet(viewsets.ModelViewSet):
    filterset_class = OrderFilter
    search_fields = ['customer__name', 'id']
    ordering_fields = ['created_at', 'total']
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

### Permissions in Settings

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# views.py - NOT here
class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    # permission_classes NOT needed - already in settings
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
- Admin customization
- Forms and ModelForms
- Middleware development
- Database migrations
- Logging
- Deployment

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

MIT
