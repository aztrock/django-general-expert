# Examples

Complete, working examples demonstrating Django patterns and best practices. These examples integrate multiple skills covered in the references.

## Example 1: Blog API with DRF

### Project Structure

```
myblog/
├── myblog/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── blog/
│   ├── __init__.py
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   ├── urls.py
│   ├── services.py
│   ├── selectors.py
│   ├── constants.py
│   └── tests/
│       ├── __init__.py
│       ├── test_models.py
│       ├── test_views.py
│       └── test_services.py
├── templates/
│   └── blog/
├── static/
└── manage.py
```

### Constants

```python
# blog/constants.py
from django.db import models


class PublicationStatus(models.TextChoices):
    DRAFT = 'draft', 'Draft'
    PUBLISHED = 'published', 'Published'
    ARCHIVED = 'archived', 'Archived'


class CommentStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    APPROVED = 'approved', 'Approved'
    REJECTED = 'rejected', 'Rejected'


MAX_TITLE_LENGTH = 200
MAX_EXCERPT_LENGTH = 300
POSTS_PER_PAGE = 10
```

### Models

```python
# blog/models.py
import uuid
from django.db import models
from django.contrib.auth.models import AbstractUser
from django.conf import settings
from .constants import PublicationStatus, CommentStatus


class User(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    is_verified = models.BooleanField(default=False)
    
    class Meta:
        db_table = 'users'


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        verbose_name_plural = 'categories'
        ordering = ['name']
    
    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(max_length=50, unique=True)
    
    class Meta:
        ordering = ['name']
    
    def __str__(self):
        return self.name


class Post(models.Model):
    id = models.BigAutoField(primary_key=True)
    public_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    excerpt = models.TextField(max_length=300, blank=True)
    content = models.TextField()
    
    status = models.CharField(
        max_length=20,
        choices=PublicationStatus.choices,
        default=PublicationStatus.DRAFT
    )
    
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='posts'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        related_name='posts'
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name='posts')
    
    featured_image = models.ImageField(upload_to='posts/', blank=True)
    view_count = models.PositiveIntegerField(default=0)
    
    published_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'blog_posts'
        ordering = ['-published_at', '-created_at']
        indexes = [
            models.Index(fields=['status', 'published_at']),
            models.Index(fields=['author', 'status']),
            models.Index(fields=['slug']),
            models.Index(fields=['public_id']),
        ]
    
    def __str__(self):
        return self.title
    
    def get_absolute_url(self):
        return f'/blog/{self.slug}/'


class Comment(models.Model):
    id = models.BigAutoField(primary_key=True)
    public_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='comments')
    parent = models.ForeignKey('self', on_delete=models.CASCADE, null=True, blank=True, related_name='replies')
    
    content = models.TextField(max_length=2000)
    status = models.CharField(
        max_length=20,
        choices=CommentStatus.choices,
        default=CommentStatus.PENDING
    )
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'blog_comments'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['post', 'status']),
            models.Index(fields=['author']),
        ]
    
    def __str__(self):
        return f'Comment by {self.author.username} on {self.post.title}'
```

### Services

```python
# blog/services.py
import logging
from dataclasses import dataclass
from typing import Optional
from django.db import transaction
from django.utils import timezone
from .models import Post, Comment, Category, Tag
from .constants import PublicationStatus, CommentStatus

logger = logging.getLogger(__name__)


@dataclass
class PostData:
    title: str
    content: str
    category_id: Optional[int]
    tag_ids: list[int]
    author_id: int
    status: str = PublicationStatus.DRAFT


@dataclass
class CommentData:
    post_id: int
    author_id: int
    content: str
    parent_id: Optional[int] = None


class PostService:
    @staticmethod
    @transaction.atomic
    def create_post(data: PostData) -> Post:
        logger.info(f'Creating post: {data.title}')
        
        category = None
        if data.category_id:
            category = Category.objects.get(pk=data.category_id)
        
        post = Post.objects.create(
            title=data.title,
            slug=PostService._generate_slug(data.title),
            content=data.content,
            status=data.status,
            author_id=data.author_id,
            category=category,
        )
        
        if data.tag_ids:
            post.tags.set(Tag.objects.filter(pk__in=data.tag_ids))
        
        if data.status == PublicationStatus.PUBLISHED:
            post.published_at = timezone.now()
            post.save()
        
        logger.info(f'Post created with id: {post.id}')
        return post
    
    @staticmethod
    @transaction.atomic
    def publish_post(post_id: int) -> Post:
        post = Post.objects.get(pk=post_id)
        
        if post.status == PublicationStatus.PUBLISHED:
            logger.warning(f'Post {post_id} already published')
            return post
        
        post.status = PublicationStatus.PUBLISHED
        post.published_at = timezone.now()
        post.save()
        
        logger.info(f'Post {post_id} published')
        return post
    
    @staticmethod
    def _generate_slug(title: str) -> str:
        from django.utils.text import slugify
        import uuid
        
        base_slug = slugify(title)
        return f'{base_slug}-{uuid.uuid4().hex[:8]}'


class CommentService:
    @staticmethod
    @transaction.atomic
    def create_comment(data: CommentData) -> Comment:
        logger.info(f'Creating comment on post {data.post_id}')
        
        post = Post.objects.get(pk=data.post_id)
        
        parent = None
        if data.parent_id:
            parent = Comment.objects.get(pk=data.parent_id)
        
        comment = Comment.objects.create(
            post=post,
            author_id=data.author_id,
            content=data.content,
            parent=parent,
            status=CommentStatus.APPROVED if post.author_id == data.author_id else CommentStatus.PENDING,
        )
        
        logger.info(f'Comment created with id: {comment.id}')
        return comment
    
    @staticmethod
    @transaction.atomic
    def approve_comment(comment_id: int) -> Comment:
        comment = Comment.objects.get(pk=comment_id)
        comment.status = CommentStatus.APPROVED
        comment.save()
        
        logger.info(f'Comment {comment_id} approved')
        return comment
```

### Selectors

```python
# blog/selectors.py
from django.db.models import Count, Q
from .models import Post, Comment
from .constants import PublicationStatus, CommentStatus


class PostSelector:
    @staticmethod
    def get_published():
        return Post.objects.filter(
            status=PublicationStatus.PUBLISHED
        ).select_related('author', 'category').prefetch_related('tags')
    
    @staticmethod
    def get_by_author(author_id):
        return Post.objects.filter(
            author_id=author_id
        ).select_related('category').prefetch_related('tags')
    
    @staticmethod
    def get_by_slug(slug: str):
        return Post.objects.select_related(
            'author', 'category'
        ).prefetch_related('tags', 'comments__author').get(slug=slug)
    
    @staticmethod
    def get_featured(limit: int = 5):
        return PostSelector.get_published().filter(
            featured_image__isnull=False
        )[:limit]
    
    @staticmethod
    def search(query: str):
        return PostSelector.get_published().filter(
            Q(title__icontains=query) |
            Q(content__icontains=query)
        )
    
    @staticmethod
    def get_by_category(category_id: int):
        return PostSelector.get_published().filter(category_id=category_id)
    
    @staticmethod
    def get_by_tag(tag_id: int):
        return PostSelector.get_published().filter(tags__id=tag_id)


class CommentSelector:
    @staticmethod
    def get_approved_for_post(post_id: int):
        return Comment.objects.filter(
            post_id=post_id,
            status=CommentStatus.APPROVED
        ).select_related('author', 'parent').order_by('created_at')
    
    @staticmethod
    def get_pending():
        return Comment.objects.filter(
            status=CommentStatus.PENDING
        ).select_related('post', 'author')
```

### Serializers

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post, Comment, Category, Tag, User


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'bio', 'avatar', 'is_verified']
        read_only_fields = ['id', 'is_verified']


class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']
        read_only_fields = ['id']


class CategorySerializer(serializers.ModelSerializer):
    post_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description', 'post_count']
        read_only_fields = ['id']
    
    def get_post_count(self, obj):
        return obj.posts.count()


class CommentSerializer(serializers.ModelSerializer):
    author_username = serializers.CharField(source='author.username', read_only=True)
    replies = serializers.SerializerMethodField()
    
    class Meta:
        model = Comment
        fields = [
            'id', 'public_id', 'content', 'author_username',
            'parent', 'replies', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'public_id', 'created_at', 'updated_at']
    
    def get_replies(self, obj):
        if obj.replies.exists():
            return CommentSerializer(obj.replies.filter(status='approved'), many=True).data
        return []


class PostListSerializer(serializers.ModelSerializer):
    author_username = serializers.CharField(source='author.username', read_only=True)
    category_name = serializers.CharField(source='category.name', read_only=True)
    comment_count = serializers.SerializerMethodField()
    tags = TagSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = [
            'id', 'public_id', 'title', 'slug', 'excerpt',
            'status', 'author_username', 'category_name',
            'featured_image', 'tags', 'comment_count',
            'published_at', 'created_at'
        ]
        read_only_fields = ['id', 'public_id', 'published_at', 'created_at']
    
    def get_comment_count(self, obj):
        return obj.comments.filter(status='approved').count()


class PostDetailSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)
    category = CategorySerializer(read_only=True)
    tags = TagSerializer(many=True, read_only=True)
    comments = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = [
            'id', 'public_id', 'title', 'slug', 'excerpt', 'content',
            'status', 'author', 'category', 'tags',
            'featured_image', 'view_count', 'comments',
            'published_at', 'created_at', 'updated_at'
        ]
        read_only_fields = [
            'id', 'public_id', 'view_count', 'published_at',
            'created_at', 'updated_at'
        ]
    
    def get_comments(self, obj):
        comments = obj.comments.filter(
            status=CommentStatus.APPROVED,
            parent__isnull=True
        ).select_related('author')
        return CommentSerializer(comments, many=True).data
```

### Views

```python
# blog/views.py
from rest_framework import viewsets, status, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly
from django_filters.rest_framework import DjangoFilterBackend
from .models import Post, Comment
from .serializers import (
    PostListSerializer, PostDetailSerializer,
    CommentSerializer
)
from .selectors import PostSelector, CommentSelector
from .services import PostService, CommentService


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostListSerializer
    detail_serializer_class = PostDetailSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'status', 'tags']
    search_fields = ['title', 'content']
    ordering_fields = ['published_at', 'created_at', 'view_count']
    lookup_field = 'slug'
    
    def get_queryset(self):
        if self.action == 'list':
            if self.request.user.is_authenticated:
                return Post.objects.filter(author=self.request.user).select_related(
                    'author', 'category'
                ).prefetch_related('tags')
            return PostSelector.get_published()
        
        return Post.objects.select_related(
            'author', 'category'
        ).prefetch_related('tags', 'comments__author')
    
    def get_serializer_class(self):
        if self.action == 'retrieve':
            return self.detail_serializer_class
        return super().get_serializer_class()
    
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        instance.view_count += 1
        instance.save(update_fields=['view_count'])
        
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
    
    def perform_create(self, serializer):
        from .services import PostService
        from .constants import PublicationStatus
        
        data = PostData(
            title=serializer.validated_data['title'],
            content=serializer.validated_data['content'],
            category_id=serializer.validated_data.get('category_id'),
            tag_ids=serializer.validated_data.get('tags', []),
            author_id=request.user.id,
            status=serializer.validated_data.get('status', PublicationStatus.DRAFT),
        )
        
        post = PostService.create_post(data)
        serializer.instance = post
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, slug=None):
        post = self.get_object()
        post = PostService.publish_post(post.id)
        return Response(PostDetailSerializer(post).data)


class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        post_slug = self.request.query_params.get('post')
        if post_slug:
            try:
                post = Post.objects.get(slug=post_slug)
                return CommentSelector.get_approved_for_post(post.id)
            except Post.DoesNotExist:
                return Comment.objects.none()
        return Comment.objects.none()
    
    def perform_create(self, serializer):
        from .services import CommentService
        from dataclasses import dataclass
        
        data = CommentData(
            post_id=serializer.validated_data['post'].id,
            author_id=self.request.user.id,
            content=serializer.validated_data['content'],
            parent_id=serializer.validated_data.get('parent'),
        )
        
        comment = CommentService.create_comment(data)
        serializer.instance = comment
```

### URLs

```python
# blog/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import PostViewSet, CommentViewSet

router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')
router.register(r'comments', CommentViewSet, basename='comment')

urlpatterns = [
    path('', include(router.urls)),
]
```

### Tests

```python
# blog/tests/test_services.py
from django.test import TestCase
from unittest.mock import patch, MagicMock
from blog.models import Post, Comment, User, Category, Tag
from blog.services import PostService, CommentService
from blog.services import PostData, CommentData
from blog.constants import PublicationStatus, CommentStatus
from blog.factories import UserFactory, PostFactory, CategoryFactory


class PostServiceTest(TestCase):
    def setUp(self):
        self.user = UserFactory()
        self.category = CategoryFactory()
        self.tags = [Tag.objects.create(name='Python'), Tag.objects.create(name='Django')]
    
    def test_create_draft_post(self):
        data = PostData(
            title='Test Post',
            content='Test content',
            category_id=self.category.id,
            tag_ids=[t.id for t in self.tags],
            author_id=self.user.id,
            status=PublicationStatus.DRAFT,
        )
        
        post = PostService.create_post(data)
        
        self.assertEqual(post.title, 'Test Post')
        self.assertEqual(post.status, PublicationStatus.DRAFT)
        self.assertEqual(post.author, self.user)
        self.assertEqual(post.category, self.category)
        self.assertEqual(list(post.tags.all()), self.tags)
        self.assertIsNone(post.published_at)
    
    def test_create_published_post(self):
        data = PostData(
            title='Published Post',
            content='Published content',
            category_id=self.category.id,
            tag_ids=[],
            author_id=self.user.id,
            status=PublicationStatus.PUBLISHED,
        )
        
        post = PostService.create_post(data)
        
        self.assertEqual(post.status, PublicationStatus.PUBLISHED)
        self.assertIsNotNone(post.published_at)
    
    @patch('blog.services.send_notification')
    def test_publish_post(self, mock_notify):
        post = PostFactory(author=self.user, status=PublicationStatus.DRAFT)
        
        published_post = PostService.publish_post(post.id)
        
        self.assertEqual(published_post.status, PublicationStatus.PUBLISHED)
        self.assertIsNotNone(published_post.published_at)
        mock_notify.assert_called_once()


class CommentServiceTest(TestCase):
    def setUp(self):
        self.user = UserFactory()
        self.post = PostFactory(author=self.user)
    
    def test_create_comment_as_author(self):
        data = CommentData(
            post_id=self.post.id,
            author_id=self.user.id,
            content='Test comment',
        )
        
        comment = CommentService.create_comment(data)
        
        self.assertEqual(comment.content, 'Test comment')
        self.assertEqual(comment.status, CommentStatus.APPROVED)
        self.assertEqual(comment.post, self.post)
        self.assertEqual(comment.author, self.user)
    
    def test_create_comment_as_non_author(self):
        other_user = UserFactory()
        
        data = CommentData(
            post_id=self.post.id,
            author_id=other_user.id,
            content='Test comment',
        )
        
        comment = CommentService.create_comment(data)
        
        self.assertEqual(comment.status, CommentStatus.PENDING)
```

## Example 2: E-commerce Order Processing

### Models

```python
# orders/models.py
import uuid
from django.db import models
from django.conf import settings
from products.models import Product, Inventory
from .constants import OrderStatus, PaymentStatus, ShippingStatus


class Order(models.Model):
    id = models.BigAutoField(primary_key=True)
    public_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    
    customer = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.PROTECT,
        related_name='orders'
    )
    
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )
    
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
    tax_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    shipping_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    discount_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    
    shipping_address = models.ForeignKey(
        'Address',
        on_delete=models.PROTECT,
        related_name='shipping_orders'
    )
    billing_address = models.ForeignKey(
        'Address',
        on_delete=models.PROTECT,
        related_name='billing_orders'
    )
    
    notes = models.TextField(blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        db_table = 'orders'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['customer', 'status']),
            models.Index(fields=['public_id']),
            models.Index(fields=['created_at']),
        ]
    
    def __str__(self):
        return f'Order {self.public_id}'


class OrderItem(models.Model):
    id = models.BigAutoField(primary_key=True)
    
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product = models.ForeignKey(Product, on_delete=models.PROTECT)
    inventory = models.ForeignKey(Inventory, on_delete=models.PROTECT, null=True)
    
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
    total_price = models.DecimalField(max_digits=10, decimal_places=2)
    
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'order_items'
    
    def __str__(self):
        return f'{self.product.name} x {self.quantity}'


class Payment(models.Model):
    id = models.BigAutoField(primary_key=True)
    public_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    
    order = models.OneToOneField(Order, on_delete=models.CASCADE, related_name='payment')
    
    status = models.CharField(
        max_length=20,
        choices=PaymentStatus.choices,
        default=PaymentStatus.PENDING
    )
    
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_method = models.CharField(max_length=50)
    transaction_id = models.CharField(max_length=100, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    processed_at = models.DateTimeField(null=True, blank=True)
    failed_at = models.DateTimeField(null=True, blank=True)
    error_message = models.TextField(blank=True)
    
    class Meta:
        db_table = 'payments'
        indexes = [
            models.Index(fields=['order']),
            models.Index(fields=['transaction_id']),
        ]
```

### Service Layer

```python
# orders/services.py
import logging
from dataclasses import dataclass
from decimal import Decimal
from typing import Optional
from django.db import transaction
from django.utils import timezone
from .models import Order, OrderItem, Payment
from .constants import OrderStatus, PaymentStatus
from .exceptions import (
    OrderValidationError,
    PaymentError,
    InventoryError
)
from orders.selectors import OrderSelector

logger = logging.getLogger(__name__)


@dataclass
class OrderItemData:
    product_id: int
    quantity: int


@dataclass
class CreateOrderData:
    customer_id: int
    items: list[OrderItemData]
    shipping_address_id: int
    billing_address_id: int
    notes: str = ''
    coupon_code: Optional[str] = None


class OrderService:
    @staticmethod
    @transaction.atomic
    def create_order(data: CreateOrderData) -> Order:
        logger.info(f'Creating order for customer {data.customer_id}')
        
        OrderService._validate_order_data(data)
        
        order = Order.objects.create(
            customer_id=data.customer_id,
            shipping_address_id=data.shipping_address_id,
            billing_address_id=data.billing_address_id,
            notes=data.notes,
            status=OrderStatus.PENDING,
        )
        
        subtotal = Decimal('0')
        
        for item_data in data.items:
            product = Product.objects.get(pk=item_data.product_id)
            
            OrderService._validate_inventory(product, item_data.quantity)
            
            item_total = product.price * item_data.quantity
            subtotal += item_total
            
            OrderItem.objects.create(
                order=order,
                product=product,
                inventory=product.inventory,
                quantity=item_data.quantity,
                unit_price=product.price,
                total_price=item_total,
            )
        
        tax_amount = subtotal * Decimal('0.21')
        shipping_amount = OrderService._calculate_shipping(order)
        discount_amount = OrderService._apply_coupon(data.coupon_code, subtotal)
        
        order.subtotal = subtotal
        order.tax_amount = tax_amount
        order.shipping_amount = shipping_amount
        order.discount_amount = discount_amount
        order.total = subtotal + tax_amount + shipping_amount - discount_amount
        order.save()
        
        logger.info(f'Order {order.id} created with total {order.total}')
        return order
    
    @staticmethod
    def _validate_order_data(data: CreateOrderData):
        if not data.items:
            raise OrderValidationError('Order must have at least one item')
        
        if data.shipping_address_id != data.billing_address_id:
            from addresses.models import Address
            shipping = Address.objects.get(pk=data.shipping_address_id)
            if shipping.customer_id != data.customer_id:
                raise OrderValidationError('Shipping address must belong to customer')
    
    @staticmethod
    def _validate_inventory(product: Product, quantity: int):
        if not product.is_active:
            raise InventoryError(f'Product {product.name} is not available')
        
        if product.inventory.quantity < quantity:
            raise InventoryError(f'Insufficient inventory for {product.name}')
    
    @staticmethod
    def _calculate_shipping(order: Order) -> Decimal:
        if order.subtotal >= Decimal('100.00'):
            return Decimal('0')
        return Decimal('10.00')
    
    @staticmethod
    def _apply_coupon(code: Optional[str], subtotal: Decimal) -> Decimal:
        if not code:
            return Decimal('0')
        
        try:
            coupon = Coupon.objects.get(code=code, is_active=True)
            if coupon.is_valid():
                return subtotal * (coupon.discount_percent / Decimal('100'))
        except Coupon.DoesNotExist:
            pass
        
        return Decimal('0')
    
    @staticmethod
    @transaction.atomic
    def cancel_order(order_id: int, reason: str) -> Order:
        order = Order.objects.select_for_update().get(pk=order_id)
        
        if order.status not in [OrderStatus.PENDING, OrderStatus.PROCESSING]:
            raise OrderValidationError(f'Cannot cancel order with status {order.status}')
        
        for item in order.items.all():
            inventory = item.inventory
            inventory.quantity += item.quantity
            inventory.save()
        
        order.status = OrderStatus.CANCELLED
        order.notes = f'{order.notes}\nCancellation reason: {reason}'
        order.save()
        
        logger.info(f'Order {order_id} cancelled')
        return order


class PaymentService:
    @staticmethod
    @transaction.atomic
    def process_payment(order_id: int, payment_method: str, payment_token: str) -> Payment:
        logger.info(f'Processing payment for order {order_id}')
        
        order = Order.objects.select_for_update().get(pk=order_id)
        
        if order.status != OrderStatus.PENDING:
            raise PaymentError('Order is not pending')
        
        payment = Payment.objects.create(
            order=order,
            amount=order.total,
            payment_method=payment_method,
            status=PaymentStatus.PROCESSING,
        )
        
        try:
            result = PaymentGateway.charge(
                amount=order.total,
                token=payment_token,
                metadata={'order_id': order.id}
            )
            
            payment.transaction_id = result.transaction_id
            payment.status = PaymentStatus.COMPLETED
            payment.processed_at = timezone.now()
            payment.save()
            
            order.status = OrderStatus.PROCESSING
            order.save()
            
            logger.info(f'Payment successful for order {order_id}')
            return payment
            
        except PaymentGatewayError as e:
            payment.status = PaymentStatus.FAILED
            payment.failed_at = timezone.now()
            payment.error_message = str(e)
            payment.save()
            
            logger.error(f'Payment failed for order {order_id}: {e}')
            raise PaymentError(f'Payment failed: {e}')
    
    @staticmethod
    @transaction.atomic
    def refund_payment(payment_id: int) -> Payment:
        payment = Payment.objects.select_for_update().get(pk=payment_id)
        
        if payment.status != PaymentStatus.COMPLETED:
            raise PaymentError('Can only refund completed payments')
        
        result = PaymentGateway.refund(
            transaction_id=payment.transaction_id,
            amount=payment.amount
        )
        
        payment.status = PaymentStatus.REFUNDED
        payment.save()
        
        order = payment.order
        order.status = OrderStatus.REFUNDED
        order.save()
        
        logger.info(f'Payment {payment_id} refunded')
        return payment
```

## Summary

These examples demonstrate:

1. **Clean separation** - Models, services, selectors, serializers, views
2. **Service layer** - Business logic encapsulated in services
3. **Early return pattern** - In views and validation
4. **Optimized queries** - select_related, prefetch_related
5. **UUID for public IDs** - Security through obscurity
6. **TextChoices in constants.py** - Avoid circular imports
7. **Transaction management** - @transaction.atomic
8. **Comprehensive tests** - Service layer testing
9. **Proper error handling** - Custom exceptions
10. **Dataclasses for data transfer** - Type safety
