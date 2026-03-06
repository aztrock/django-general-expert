# Migrations Guide

Guide to managing Django database migrations effectively.

## Basic Commands

```bash
# Create migrations
python manage.py makemigrations

# Name the migration
python manage.py makemigrations myapp --name add_field

# Apply migrations
python manage.py migrate

# Check status
python manage.py showmigrations

# Rollback
python manage.py migrate myapp 0001  # To specific version

# Fake initial
python manage.py migrate --fake-initial
```

## Creating Migrations

### New Model

```python
# models.py
class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField(blank=True)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['name']),
            models.Index(fields=['category', 'is_active']),
        ]


# Migration is auto-generated
# migrations/0002_product.py
```

### Adding Field

```bash
python manage.py makemigrations myapp --name add_sku
```

### Renaming Model

```python
# Old
class Author(models.Model):
    name = models.CharField(max_length=100)


# New (use migration to rename)
class Writer(models.Model):
    name = models.CharField(max_length=100)
    
    class Meta:
        db_table = 'author'  # Keep same table
```

### Data Migration

```python
# migrations/0003_update_status_values.py
from django.db import migrations


def update_status_values(apps, schema_editor):
    Order = apps.get_model('orders', 'Order')
    Order.objects.filter(status='new').update(status='pending')


def reverse_status_values(apps, schema_editor):
    Order = apps.get_model('orders', 'Order')
    Order.objects.filter(status='pending').update(status='new')


class Migration(migrations.Migration):
    dependencies = [
        ('orders', '0002_add_field'),
    ]
    
    operations = [
        migrations.RunPython(
            update_status_values,
            reverse_status_values
        ),
    ]
```

### Custom Migration Operations

```python
# migrations/0004_create_indexes.py
from django.db import migrations, models


class Migration(migrations.Migration):
    operations = [
        # Create table
        migrations.CreateModel(
            name='Tag',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('name', models.CharField(max_length=50)),
            ],
        ),
        
        # Add field
        migrations.AddField(
            model_name='product',
            name='tags',
            field=models.ManyToManyField(to='products.Tag'),
        ),
        
        # Alter field
        migrations.AlterField(
            model_name='product',
            name='price',
            field=models.DecimalField(decimal_places=2, max_digits=12),
        ),
        
        # Rename field
        migrations.RenameField(
            model_name='product',
            old_name='description',
            new_name='desc',
        ),
        
        # Remove field
        migrations.RemoveField(
            model_name='product',
            name='old_field',
        ),
        
        # Delete model
        migrations.DeleteModel(
            name='OldModel',
        ),
        
        # Run SQL
        migrations.RunSQL(
            sql="CREATE INDEX ...",
            reverse_sql="DROP INDEX ...",
        ),
        
        # Run Python
        migrations.RunPython(
            forward_func,
            backward_func,
        ),
    ]
```

## Best Practices

### 1. One Migration Per Change

```bash
# ✅ GOOD: Separate migrations
python manage.py makemigrations --name add_status_field
python manage.py makemigrations --name add_created_at_field

# ❌ BAD: Multiple changes in one
# Don't add multiple fields at once if you can avoid it
```

### 2. Meaningful Names

```bash
# ✅ GOOD
python manage.py makemigrations --name add_status_to_order

# ❌ BAD
python manage.py makemigrations --name initial
python manage.py makemigrations --name changes
```

### 3. Test Migrations

```bash
# In development
python manage.py migrate --fake

# In CI/CD
python manage.py test --keepdb

# Or use django-test-migrations
```

### 4. Handle Large Data

```python
# For large tables, use RunSQL for better performance
class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL(
            sql="""
                ALTER TABLE orders 
                ADD COLUMN status VARCHAR(20) DEFAULT 'pending';
            """,
            reverse_sql="""
                ALTER TABLE orders DROP COLUMN status;
            """
        ),
    ]


# For data migration with large datasets, chunk it
def migrate_data(apps, schema_editor):
    Order = apps.get_model('orders', 'Order')
    
    # Process in chunks
    CHUNK_SIZE = 1000
    queryset = Order.objects.filter(status='new')
    
    last_id = 0
    while True:
        chunk = list(queryset.filter(id__gt=last_id).order_by('id')[:CHUNK_SIZE])
        if not chunk:
            break
        
        for order in chunk:
            order.status = 'pending'
            order.save(update_fields=['status'])
        
        last_id = chunk[-1].id
```

## Troubleshooting

### Circular Dependencies

```python
# ❌ BAD: Model depends on itself
class Book(models.Model):
    related = models.ForeignKey('self', on_delete=models.CASCADE)


# ✅ GOOD: String reference
class Book(models.Model):
    related = models.ForeignKey('Book', on_delete=models.CASCADE)
```

### Migration Conflicts

```bash
# Show conflicts
python manage.py migrate --check

# Resolve by squashing
python manage.py squashmigrations myapp 0001_initial
```

### Fake Migrations

```bash
# Mark as applied without running
python manage.py migrate myapp 0003 --fake

# Fake initial if table already exists
python manage.py migrate --fake-initial
```

## Summary

1. **Run makemigrations** after every model change
2. **Review migrations** before applying
3. **Use meaningful names** for migrations
4. **Test migrations** in development first
5. **Handle large data** with RunSQL or chunking
6. **Use --fake-initial** for existing tables
7. **Squash old migrations** to reduce files
8. **Keep migrations in version control**
