# Django REST Framework Best Practices

Comprehensive guide to building production-ready APIs with Django REST Framework.

## Serializers

### Never Use __all__

```python
# ❌ BAD: Exposes all fields including sensitive ones
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # Includes password hash!

# ✅ GOOD: Explicit field list
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name']
        read_only_fields = ['id']
```

### Read vs Write Serializers

```python
# ✅ GOOD: Separate serializers for different operations
class UserReadSerializer(serializers.ModelSerializer):
    """Serializer for reading user data."""
    full_name = serializers.CharField(source='get_full_name', read_only=True)
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'full_name', 'created_at']
        read_only_fields = ['id', 'created_at']


class UserWriteSerializer(serializers.ModelSerializer):
    """Serializer for creating/updating users."""
    password = serializers.CharField(write_only=True, min_length=8)
    confirm_password = serializers.CharField(write_only=True)
    
    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'confirm_password', 'first_name', 'last_name']
    
    def validate(self, data):
        if data['password'] != data['confirm_password']:
            raise serializers.ValidationError({'confirm_password': 'Passwords must match'})
        return data
    
    def create(self, validated_data):
        validated_data.pop('confirm_password')
        return User.objects.create_user(**validated_data)
```

### Nested Serializers

```python
# ✅ GOOD: Nested read-only serializer
class CommentSerializer(serializers.ModelSerializer):
    author = UserReadSerializer(read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'content', 'author', 'created_at']


class PostSerializer(serializers.ModelSerializer):
    author = UserReadSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    comment_count = serializers.IntegerField(source='comments.count', read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'comment_count', 'created_at']
```

### Serializer Validation

```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'customer', 'items', 'shipping_address', 'status']
    
    # Field-level validation
    def validate_shipping_address(self, value):
        if not value or len(value.strip()) < 10:
            raise serializers.ValidationError("Address must be at least 10 characters")
        return value
    
    # Object-level validation
    def validate(self, data):
        items = data.get('items', [])
        
        if not items:
            raise serializers.ValidationError({'items': 'Order must have at least one item'})
        
        # Check stock
        for item in items:
            if item.quantity > item.product.stock:
                raise serializers.ValidationError({
                    'items': f'{item.product.name} is out of stock'
                })
        
        return data
```

### Custom Fields

```python
class TrimmedCharField(serializers.CharField):
    """CharField that trims whitespace."""
    
    def to_internal_value(self, data):
        data = super().to_internal_value(data)
        return data.strip()


class EnumField(serializers.ChoiceField):
    """Field for enum types."""
    
    def __init__(self, enum_class, **kwargs):
        self.enum_class = enum_class
        choices = [(e.value, e.label) for e in enum_class]
        super().__init__(choices, **kwargs)
    
    def to_internal_value(self, data):
        try:
            return self.enum_class(data)
        except ValueError:
            self.fail('invalid_choice', input=data)
    
    def to_representation(self, value):
        return value.value


class OrderSerializer(serializers.ModelSerializer):
    status = EnumField(OrderStatus)
    notes = TrimmedCharField(required=False, allow_blank=True)
    
    class Meta:
        model = Order
        fields = ['id', 'status', 'notes']
```

## ViewSets

### ModelViewSet

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    
    def get_queryset(self):
        # Filter by user
        queryset = Order.objects.filter(customer=self.request.user)
        
        # Filter by status
        status = self.request.query_params.get('status')
        if status:
            queryset = queryset.filter(status=status)
        
        # Optimize queries
        return queryset.select_related('customer').prefetch_related('items__product')
    
    def get_serializer_class(self):
        if self.action == 'create':
            return OrderCreateSerializer
        return OrderSerializer
    
    def perform_create(self, serializer):
        serializer.save(customer=self.request.user)
    
    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        order = self.get_object()
        
        if not order.can_cancel():
            return Response(
                {'error': 'Cannot cancel this order'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        order.status = OrderStatus.CANCELLED
        order.save()
        
        return Response({'status': 'cancelled'})
```

### ReadOnlyModelViewSet

```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    
    @action(detail=True, methods=['get'])
    def products(self, request, pk=None):
        category = self.get_object()
        products = category.products.filter(
            status=ProductStatus.ACTIVE
        )
        
        page = self.paginate_queryset(products)
        serializer = ProductSerializer(page, many=True)
        return self.get_paginated_response(serializer.data)
```

## Permissions

### Custom Permissions

```python
from rest_framework import permissions


class IsAuthorOrReadOnly(permissions.BasePermission):
    """Allow read access to all, write access only to author."""
    
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        
        return obj.author == request.user


class IsOwner(permissions.BasePermission):
    """Allow access only to owner."""
    
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user
```

### Using Permissions in ViewSets

```python
class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsAuthorOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

## Error Handling

### Use drf-standardized-errors

```bash
pip install drf-standardized-errors
```

```python
# settings.py
INSTALLED_APPS = [
    'drf_standardized_errors',
]

REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'drf_standardized_errors.handler.exception_handler',
}
```

### Raising Errors

```python
from rest_framework.exceptions import ValidationError, NotFound


class OrderViewSet(viewsets.ModelViewSet):
    def perform_update(self, serializer):
        order = self.get_object()
        
        if order.status == OrderStatus.SHIPPED:
            raise ValidationError({
                'status': 'Cannot update shipped orders'
            })
        
        serializer.save()
```

## Configuration

### Permissions in Settings

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
    'EXCEPTION_HANDLER': 'drf_standardized_errors.handler.exception_handler',
}
```

### ViewSet Without Redundant Config

```python
# ✅ GOOD: Uses settings
class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    # permission_classes NOT needed (from settings)
    # filter_backends NOT needed (from settings)

# ❌ BAD: Redundant configuration
class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]  # Already in settings!
    filter_backends = [DjangoFilterBackend]  # Already in settings!
    serializer_class = OrderSerializer
```

## Performance

### N+1 in Serializers

```python
# ❌ BAD: N+1 in serializer
class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True, read_only=True)
    
    class Meta:
        model = Order
        fields = ['id', 'items']


# View
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()  # N+1 problem!


# ✅ GOOD: Optimize queryset
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.prefetch_related('items__product')
```

### Query Count Optimization

```python
class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True, read_only=True)
    item_count = serializers.IntegerField(read_only=True)  # From annotation
    
    class Meta:
        model = Order
        fields = ['id', 'items', 'item_count']


class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.annotate(
        item_count=Count('items')
    ).prefetch_related(
        'items',
        'items__product'
    ).select_related(
        'customer'
    )
```

## Summary

1. **Never use `__all__`** in serializers - explicitly list fields
2. **Separate read/write serializers** for different operations
3. **Use nested serializers** for related data
4. **Validate thoroughly** at field and object level
5. **Use custom permissions** for fine-grained access control
6. **Configure permissions in settings**, not in every view
7. **Use drf-standardized-errors** for consistent error responses
8. **Optimize queries** with select_related/prefetch_related
9. **Use ViewSets** for standard CRUD operations
10. **Avoid `@api_view`** - use ViewSets or APIViews instead
