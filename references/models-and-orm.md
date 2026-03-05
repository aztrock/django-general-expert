# Models & ORM Best Practices

Comprehensive guide to Django models, ORM patterns, and database optimization.

## Model Design Principles

### Use Descriptive Field Names

```python
# ❌ BAD: Unclear field names
class Order(models.Model):
    d = models.DateTimeField(auto_now_add=True)
    amt = models.DecimalField(max_digits=10, decimal_places=2)
    uid = models.ForeignKey(User, on_delete=models.CASCADE)

# ✅ GOOD: Clear field names
class Order(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    customer = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
```

### Use TextChoices for Status Fields

```python
# constants.py
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PAID = 'paid', 'Paid'
    SHIPPED = 'shipped', 'Shipped'
    DELIVERED = 'delivered', 'Delivered'
    CANCELLED = 'cancelled', 'Cancelled'


# models.py
from .constants import OrderStatus


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING,
        db_index=True
    )
    
    def can_cancel(self):
        return self.status in [OrderStatus.PENDING, OrderStatus.PAID]
```

### Always Add related_name

```python
# ❌ BAD: Default related_name causes conflicts
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    # Access: user.order_set.all()

class Review(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    # ERROR: user.order_set conflicts with user.review_set!

# ✅ GOOD: Explicit related_name
class Order(models.Model):
    customer = models.ForeignKey(
        User, 
        on_delete=models.CASCADE,
        related_name='orders'
    )

class Review(models.Model):
    customer = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='reviews'
    )

# Usage
user.orders.all()
user.reviews.all()
```

### Add Indexes for Common Queries

```python
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    class Meta:
        indexes = [
            # Single field index
            models.Index(fields=['created_at']),
            
            # Composite index (order matters!)
            models.Index(fields=['status', 'created_at']),
            
            # Descending order
            models.Index(fields=['-created_at']),
            
            # For filtering + ordering
            models.Index(fields=['customer', '-created_at']),
        ]
        
        # Database constraints
        constraints = [
            models.CheckConstraint(
                check=models.Q(total_amount__gte=0),
                name='total_amount_positive'
            ),
        ]
```

## UUID for Security

### Use UUID for Public Identifiers

```python
import uuid

class Order(models.Model):
    # Internal use (auto-increment, fast for joins)
    id = models.BigAutoField(primary_key=True)
    
    # Public use (UUID prevents enumeration attacks)
    public_id = models.UUIDField(
        default=uuid.uuid4,
        editable=False,
        unique=True,
        db_index=True
    )
    
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'orders'
        ordering = ['-created_at']
    
    def __str__(self):
        return f'Order {self.public_id}'


# Usage in views
def order_detail(request, public_id):
    order = get_object_or_404(Order, public_id=public_id)
    # User cannot guess other order IDs
```

**Why UUID for Public?**
- Prevents enumeration attacks (user can't guess order_id=1, order_id=2, etc.)
- Safe to expose in URLs and APIs
- Internal id remains fast for database joins

## Relationships

### ForeignKey (One-to-Many)

```python
class Author(models.Model):
    name = models.CharField(max_length=100)


class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,  # Delete books when author is deleted
        related_name='books'
    )
    
    class Meta:
        indexes = [
            models.Index(fields=['author']),
        ]


# Usage
author = Author.objects.get(pk=1)
books = author.books.all()  # Reverse relation

book = Book.objects.get(pk=1)
author = book.author  # Forward relation
```

**on_delete options:**
- `CASCADE`: Delete related objects
- `PROTECT`: Prevent deletion if related objects exist
- `SET_NULL`: Set to NULL (field must be nullable)
- `SET_DEFAULT`: Set to default value
- `DO_NOTHING`: No action (dangerous, can cause integrity errors)

### OneToOneField (One-to-One)

```python
class User(models.Model):
    username = models.CharField(max_length=100, unique=True)


class Profile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField()
    avatar = models.ImageField(upload_to='avatars/')


# Usage
user = User.objects.get(pk=1)
profile = user.profile  # Access profile

profile = Profile.objects.get(pk=1)
user = profile.user  # Access user
```

### ManyToManyField (Many-to-Many)

```python
class Student(models.Model):
    name = models.CharField(max_length=100)


class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField(
        Student,
        related_name='courses',
        through='Enrollment'  # Optional: custom through model
    )


class Enrollment(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    course = models.ForeignKey(Course, on_delete=models.CASCADE)
    enrolled_at = models.DateTimeField(auto_now_add=True)
    grade = models.CharField(max_length=2, null=True)
    
    class Meta:
        unique_together = ['student', 'course']


# Usage
course = Course.objects.get(pk=1)
students = course.students.all()

student = Student.objects.get(pk=1)
courses = student.courses.all()

# Add student to course
course.students.add(student)

# Remove student from course
course.students.remove(student)

# With through model
Enrollment.objects.create(student=student, course=course)
```

## The N+1 Query Problem

The most common performance issue in Django applications.

### Problem: N+1 Queries

```python
# ❌ BAD: 1 + N queries
books = Book.objects.all()  # 1 query
for book in books:
    print(book.author.name)  # N additional queries!

# SQL executed:
# SELECT * FROM books;
# SELECT * FROM authors WHERE id = 1;
# SELECT * FROM authors WHERE id = 2;
# SELECT * FROM authors WHERE id = 3;
# ... (N times)
```

### Solution: select_related() for ForeignKey

```python
# ✅ GOOD: 1 query with JOIN
books = Book.objects.select_related('author').all()  # 1 query
for book in books:
    print(book.author.name)  # No additional queries!

# SQL executed:
# SELECT books.*, authors.*
# FROM books
# INNER JOIN authors ON books.author_id = authors.id;
```

### Solution: prefetch_related() for ManyToMany

```python
# ❌ BAD: N+1 queries
authors = Author.objects.all()  # 1 query
for author in authors:
    for book in author.books.all():  # N queries!
        print(book.title)

# ✅ GOOD: 2 queries total
authors = Author.objects.prefetch_related('books').all()  # 2 queries
for author in authors:
    for book in author.books.all():  # No additional queries!
        print(book.title)

# SQL executed:
# SELECT * FROM authors;
# SELECT * FROM books WHERE author_id IN (1, 2, 3, ...);
```

### Combining select_related and prefetch_related

```python
# Get orders with customer (FK) and items (reverse FK)
orders = Order.objects.select_related(
    'customer'
).prefetch_related(
    'items',
    'items__product'  # Nested prefetch
).all()

for order in orders:
    print(order.customer.username)  # No query
    for item in order.items.all():  # No query
        print(item.product.name)  # No query
```

### Advanced Prefetching

```python
from django.db.models import Prefetch

# Prefetch with filtering
orders = Order.objects.prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.filter(quantity__gt=1),
        to_attr='bulk_items'  # Store in custom attribute
    )
).all()

for order in orders:
    # order.bulk_items contains only items with quantity > 1
    for item in order.bulk_items:
        print(item)

# Nested prefetch with filtering
authors = Author.objects.prefetch_related(
    Prefetch(
        'books',
        queryset=Book.objects.filter(published=True).select_related('publisher')
    )
).all()
```

## Custom Managers

Encapsulate common queries in managers.

```python
class OrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().select_related('customer')
    
    def pending(self):
        return self.filter(status=OrderStatus.PENDING)
    
    def paid(self):
        return self.filter(status=OrderStatus.PAID)
    
    def recent(self, days=30):
        from django.utils import timezone
        from datetime import timedelta
        cutoff = timezone.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff)
    
    def for_customer(self, customer):
        return self.filter(customer=customer)
    
    def with_totals(self):
        from django.db.models import Sum
        return self.annotate(
            items_total=Sum('items__price')
        )


class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=OrderStatus.choices)
    created_at = models.DateTimeField(auto_now_add=True)
    
    objects = OrderManager()


# Usage
pending_orders = Order.objects.pending()
recent_paid = Order.objects.paid().recent(7)
customer_orders = Order.objects.for_customer(user).with_totals()
```

## Query Optimization

### Use only() and defer()

```python
# ❌ BAD: Load all fields (including large text fields)
books = Book.objects.all()
for book in books:
    print(book.title, book.author.name)

# ✅ GOOD: Load only needed fields
books = Book.objects.only('title', 'author__name')
for book in books:
    print(book.title, book.author.name)  # No extra queries

# ✅ GOOD: Defer large fields
books = Book.objects.defer('content')  # Don't load content
for book in books:
    print(book.title)
    # book.content will load on demand
```

### Use count() instead of len()

```python
# ❌ BAD: Loads all records into memory
count = len(Book.objects.all())

# ✅ GOOD: COUNT query at database
count = Book.objects.count()

# ✅ GOOD: exists() instead of count() when you only need to know if records exist
if Book.objects.filter(author=author).exists():
    print("Author has books")
```

### Bulk Operations

```python
# ❌ BAD: N queries
for i in range(1000):
    Book.objects.create(title=f'Book {i}', author=author)

# ✅ GOOD: 1 query (or batch_size queries)
books = [
    Book(title=f'Book {i}', author=author)
    for i in range(1000)
]
Book.objects.bulk_create(books, batch_size=100)

# ❌ BAD: N queries
for book in books:
    book.price = 19.99
    book.save()

# ✅ GOOD: 1 query
Book.objects.filter(id__in=book_ids).update(price=19.99)

# ✅ GOOD: bulk_update (Django 2.2+)
books = Book.objects.filter(author=author)
for book in books:
    book.price = book.price * 1.1  # 10% increase

Book.objects.bulk_update(books, ['price'], batch_size=100)
```

### Annotate and Aggregate

```python
from django.db.models import Count, Sum, Avg, Max, Min

# Annotate: Add calculated field to each object
authors = Author.objects.annotate(
    book_count=Count('books'),
    total_sales=Sum('books__sales')
)

for author in authors:
    print(f"{author.name}: {author.book_count} books, ${author.total_sales}")

# Aggregate: Calculate single value
total_books = Book.objects.aggregate(
    total=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price')
)
# Returns: {'total': 100, 'avg_price': 25.50, 'max_price': 99.99}

# Conditional aggregation
from django.db.models import Q, Case, When, Value

authors = Author.objects.annotate(
    published_books=Count('books', filter=Q(books__published=True)),
    draft_books=Count('books', filter=Q(books__published=False))
)
```

### F() Expressions

```python
from django.db.models import F

# ❌ BAD: Two queries
product = Product.objects.get(pk=1)
product.stock = product.stock - 1
product.save()

# ✅ GOOD: One query, atomic update
Product.objects.filter(pk=1).update(stock=F('stock') - 1)

# ✅ GOOD: Avoid race conditions
Product.objects.filter(pk=1, stock__gt=0).update(stock=F('stock') - 1)

# Use F() in queries
books = Book.objects.filter(price__lt=F('author__avg_book_price'))
```

## Migrations Best Practices

### Create Migrations Properly

```bash
# After model changes
python manage.py makemigrations

# With descriptive name
python manage.py makemigrations --name add_user_email_index

# Preview SQL
python manage.py sqlmigrate myapp 0001

# Check for issues
python manage.py check
```

### Migration Files

```python
# Generated migration
from django.db import migrations, models


class Migration(migrations.Migration):
    dependencies = [
        ('myapp', '0001_initial'),
    ]
    
    operations = [
        migrations.AddField(
            model_name='user',
            name='email',
            field=models.EmailField(max_length=254, unique=True),
        ),
        migrations.AddIndex(
            model_name='user',
            index=models.Index(fields=['email'], name='user_email_idx'),
        ),
    ]
```

### Data Migrations

```python
# migrations/0002_populate_user_emails.py
from django.db import migrations


def populate_emails(apps, schema_editor):
    User = apps.get_model('myapp', 'User')
    for user in User.objects.all():
        user.email = f"{user.username}@example.com"
        user.save()


class Migration(migrations.Migration):
    dependencies = [
        ('myapp', '0001_initial'),
    ]
    
    operations = [
        migrations.RunPython(populate_emails),
    ]
```

### Reversible Migrations

```python
def add_email_forward(apps, schema_editor):
    User = apps.get_model('myapp', 'User')
    for user in User.objects.all():
        user.email = f"{user.username}@example.com"
        user.save()


def add_email_reverse(apps, schema_editor):
    User = apps.get_model('myapp', 'User')
    User.objects.update(email='')


class Migration(migrations.Migration):
    dependencies = [
        ('myapp', '0001_initial'),
    ]
    
    operations = [
        migrations.AddField(
            model_name='user',
            name='email',
            field=models.EmailField(max_length=254, unique=True),
        ),
        migrations.RunPython(add_email_forward, add_email_reverse),
    ]
```

### Never Edit Applied Migrations

```python
# ❌ BAD: Editing applied migration
# This will cause inconsistencies between environments

# ✅ GOOD: Create new migration
python manage.py makemigrations
```

## Common Model Patterns

### Soft Delete

```python
from django.db import models
from django.utils import timezone


class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)


class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    objects = SoftDeleteManager()
    all_objects = models.Manager()  # Include deleted
    
    def delete(self, using=None, keep_parents=False):
        self.deleted_at = timezone.now()
        self.save()
    
    def hard_delete(self):
        super().delete()
    
    class Meta:
        abstract = True


class Order(SoftDeleteModel):
    # ... fields
    pass
```

### Timestamped Model

```python
class TimestampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True


class Order(TimestampedModel):
    # ... fields
    pass
```

### JSON Field (Django 3.1+)

```python
class Product(models.Model):
    name = models.CharField(max_length=200)
    metadata = models.JSONField(default=dict)
    
    def get_attribute(self, key):
        return self.metadata.get(key)
    
    def set_attribute(self, key, value):
        self.metadata[key] = value
        self.save()


# Query JSON fields
Product.objects.filter(metadata__color='red')
Product.objects.filter(metadata__price__gt=100)
```

## Summary

1. **Use descriptive names** for fields and related_name
2. **Always add indexes** for frequently queried fields
3. **Use TextChoices** for status fields (avoid bool trap)
4. **Prevent N+1 queries** with select_related and prefetch_related
5. **Encapsulate queries** in custom managers
6. **Use bulk operations** for creating/updating many records
7. **Create reversible migrations** for data changes
8. **Never edit applied migrations** - create new ones
9. **Use UUID for public IDs** for security
10. **Add indexes** for search and filter fields
