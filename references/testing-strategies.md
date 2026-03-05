# Testing Strategies

Comprehensive guide to testing Django applications using Django's built-in TestCase.

## Test Organization

### Directory Structure

```
myapp/
├── models.py
├── views.py
├── serializers.py
├── services.py
└── tests/
    ├── __init__.py
    ├── test_models.py
    ├── test_views.py
    ├── test_serializers.py
    ├── test_services.py
    └── factories.py
```

### Running Tests

```bash
# Run all tests
python manage.py test

# Run specific app
python manage.py test myapp

# Run specific test file
python manage.py test myapp.tests.test_views

# Run specific test class
python manage.py test myapp.tests.test_views.PostViewTest

# Run specific test method
python manage.py test myapp.tests.test_views.PostViewTest.test_get_queryset
```

## TestCase Basics

### TestCase vs TransactionTestCase

```python
from django.test import TestCase, TransactionTestCase

# ✅ GOOD: Use TestCase for most tests (faster, uses transactions)
class PostModelTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='test', password='pass')
        self.post = Post.objects.create(title='Test', author=self.user)
    
    def test_post_creation(self):
        self.assertEqual(self.post.title, 'Test')
        self.assertEqual(Post.objects.count(), 1)


# Use TransactionTestCase only when testing transaction behavior
class PaymentTest(TransactionTestCase):
    def test_payment_transaction(self):
        # Test actual database commits/rollbacks
        pass
```

**Rule**: Use `TestCase` unless you need to test transaction behavior.

### setUp vs setUpTestData

```python
# ❌ BAD: setUp() runs before each test method - slow!
class PostModelTest(TestCase):
    def setUp(self):
        # This runs before EACH test method
        self.user = User.objects.create_user(username='test', password='pass')
        self.post = Post.objects.create(title='Test', author=self.user)
    
    def test_one(self):
        # setUp() runs here
        pass
    
    def test_two(self):
        # setUp() runs here too
        pass


# ✅ GOOD: setUpTestData() runs once for the whole class - fast!
class PostModelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        # This runs ONCE for the whole class
        cls.user = User.objects.create_user(username='test', password='pass')
        cls.post = Post.objects.create(title='Test', author=cls.user)
    
    def test_one(self):
        # Use self.user and self.post
        pass
    
    def test_two(self):
        # Same self.user and self.post
        pass
```

**Rule**: Use `@classmethod setUpTestData` for data shared across all tests in a class.

## Testing Models

```python
from django.test import TestCase
from django.core.exceptions import ValidationError
from myapp.models import Post, User


class PostModelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
    
    def setUp(self):
        # Test-specific data
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_create_post(self):
        post = Post.objects.create(
            title='New Post',
            content='New content',
            author=self.user
        )
        
        self.assertEqual(post.title, 'New Post')
        self.assertEqual(post.author, self.user)
        self.assertEqual(Post.objects.count(), 2)
    
    def test_post_str(self):
        self.assertEqual(str(self.post), 'Test Post')
    
    def test_post_ordering(self):
        post2 = Post.objects.create(
            title='Second Post',
            content='Content',
            author=self.user
        )
        
        posts = Post.objects.all().order_by('-created_at')
        self.assertEqual(posts[0], post2)
        self.assertEqual(posts[1], self.post)
    
    def test_cascade_delete(self):
        user = User.objects.create_user(username='delete', password='pass')
        post = Post.objects.create(title='Delete', author=user)
        
        user_id = user.id
        user.delete()
        
        self.assertEqual(Post.objects.filter(author_id=user_id).count(), 0)
```

## Testing Views

### Testing FBVs

```python
from django.test import TestCase
from django.urls import reverse
from myapp.models import Post, User


class PostViewsTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(username='test', password='pass123')
        cls.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=cls.user
        )
    
    def test_post_list_view(self):
        response = self.client.get(reverse('post_list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed('blog/post_list.html')
        self.assertContains(response, 'Test Post')
    
    def test_post_detail_view(self):
        response = self.client.get(
            reverse('post_detail', kwargs={'pk': self.post.pk})
        )
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed('blog/post_detail.html')
        self.assertContains(response, 'Test Post')
        self.assertContains(response, 'Test content')
    
    def test_post_detail_not_found(self):
        response = self.client.get(
            reverse('post_detail', kwargs={'pk': 99999})
        )
        
        self.assertEqual(response.status_code, 404)
    
    def test_post_create_requires_login(self):
        response = self.client.get(reverse('post_create'))
        
        self.assertNotEqual(response.status_code, 200)
        self.assertRedirects(response, '/login/')
    
    def test_post_create_authenticated(self):
        self.client.login(username='test', password='pass123')
        
        response = self.client.get(reverse('post_create'))
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed('blog/post_form.html')
    
    def test_post_create_post(self):
        self.client.login(username='test', password='pass123')
        
        response = self.client.post(
            reverse('post_create'),
            {'title': 'New Post', 'content': 'New content'}
        )
        
        self.assertEqual(response.status_code, 302)
        self.assertRedirects(response, reverse('post_detail', kwargs={'pk': 2}))
        
        self.assertEqual(Post.objects.count(), 2)
```

## Testing DRF APIs

### APITestCase

```python
from rest_framework.test import APITestCase
from rest_framework import status
from django.urls import reverse
from myapp.models import Post, User


class PostAPITest(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(
            username='test',
            email='test@example.com',
            password='testpass123'
        )
        cls.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=cls.user
        )
    
    def setUp(self):
        self.client.force_authenticate(user=self.user)
    
    def test_list_posts(self):
        response = self.client.get('/api/posts/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
    
    def test_create_post(self):
        data = {
            'title': 'New Post',
            'content': 'New content'
        }
        
        response = self.client.post('/api/posts/', data, format='json')
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'New Post')
        self.assertEqual(Post.objects.count(), 2)
    
    def test_create_post_invalid(self):
        data = {
            'title': '',  # Invalid: empty title
            'content': 'Content'
        }
        
        response = self.client.post('/api/posts/', data, format='json')
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('title', response.data)
    
    def test_retrieve_post(self):
        response = self.client.get(f'/api/posts/{self.post.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Post')
    
    def test_update_own_post(self):
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        
        response = self.client.put(
            f'/api/posts/{self.post.id}/',
            data,
            format='json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Updated Title')
    
    def test_delete_own_post(self):
        response = self.client.delete(f'/api/posts/{self.post.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Post.objects.count(), 0)
    
    def test_unauthenticated_access(self):
        self.client.force_authenticate(user=None)
        
        response = self.client.get('/api/posts/')
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

## Testing Services

```python
from django.test import TestCase
from django.core.exceptions import ValidationError
from myapp.models import Order, User, Product
from myapp.services import OrderService


class OrderServiceTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='test', password='pass')
        self.product = Product.objects.create(
            name='Test Product',
            price=100,
            stock=10
        )
    
    def test_create_order_success(self):
        items_data = [
            {'product_id': self.product.id, 'quantity': 2}
        ]
        
        order = OrderService.create_order(
            customer=self.user,
            items_data=items_data,
            shipping_address='123 Test St'
        )
        
        self.assertEqual(order.customer, self.user)
        self.assertEqual(order.items.count(), 1)
        self.assertEqual(order.total_amount, 200)
    
    def test_create_order_out_of_stock(self):
        # Reduce stock
        self.product.stock = 0
        self.product.save()
        
        items_data = [
            {'product_id': self.product.id, 'quantity': 1}
        ]
        
        with self.assertRaises(ValidationError) as cm:
            OrderService.create_order(
                customer=self.user,
                items_data=items_data,
                shipping_address='123 Test St'
            )
        
        self.assertIn('out of stock', str(cm.exception))
    
    def test_create_order_empty_items(self):
        with self.assertRaises(ValidationError):
            OrderService.create_order(
                customer=self.user,
                items_data=[],
                shipping_address='123 Test St'
            )
```

## Mocking

### Basic Mocking

```python
from unittest.mock import Mock, patch
from django.test import TestCase
from myapp.services import EmailService
from myapp.models import User


class EmailServiceTest(TestCase):
    @patch('myapp.services.send_mail')
    def test_send_welcome_email(self, mock_send):
        user = User.objects.create_user(username='test', email='test@example.com', password='pass')
        
        EmailService.send_welcome_email(user)
        
        mock_send.assert_called_once_with(
            to=user.email,
            subject='Welcome!',
            message=f'Welcome {user.username}!'
        )
```

### Mocking External APIs

```python
from unittest.mock import patch
from django.test import TestCase
from myapp.services import PaymentService
from myapp.models import Order, User


class PaymentServiceTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(username='test', password='pass')
    
    @patch('stripe.Charge.create')
    def test_process_payment_success(self, mock_charge):
        mock_charge.return_value = {
            'id': 'ch_123',
            'status': 'succeeded'
        }
        
        order = Order.objects.create(
            customer=self.user,
            total=100
        )
        
        result = PaymentService.process_payment(order, 'tok_123')
        
        self.assertTrue(result.success)
        self.assertEqual(result.charge_id, 'ch_123')
        
        mock_charge.assert_called_once_with(
            amount=10000,  # 100 in cents
            currency='usd',
            source='tok_123'
        )
    
    @patch('stripe.Charge.create')
    def test_process_payment_failure(self, mock_charge):
        mock_charge.return_value = {
            'id': 'ch_456',
            'status': 'failed'
        }
        
        order = Order.objects.create(
            customer=self.user,
            total=100
        )
        
        result = PaymentService.process_payment(order, 'tok_123')
        
        self.assertFalse(result.success)
```

### Mock External Services, Not the Service Itself

```python
from unittest.mock import patch
from django.test import TestCase
from myapp.services import OrderService
from myapp.factories import UserFactory, ProductFactory


class OrderServiceTest(TestCase):
    def setUp(self):
        self.customer = UserFactory()
        self.product = ProductFactory(price=100)
    
    @patch('requests.post')
    def test_notify_external_service(self, mock_post):
        mock_post.return_value.json.return_value = {'success': True}
        
        items_data = [
            {'product_id': self.product.id, 'quantity': 2}
        ]
        
        order = OrderService.create_order(
            customer=self.customer,
            items_data=items_data,
            shipping_address='123 Test St'
        )
        
        # Verify external service was called
        mock_post.assert_called_once()
        
        self.assertEqual(order.total_amount, 200)


# ❌ BAD: Mocking the service itself
@patch('myapp.services.OrderService.create_order')
def test_bad_mock(self, mock_create):
    # Don't mock the service you're testing!
    mock_create.return_value = Order(total=200)
    
    # This doesn't test the actual service logic!
    order = OrderService.create_order(...)
    self.assertEqual(order.total, 200)
```

## FactoryBoy

### Using FactoryBoy

```python
# factories.py
import factory
from factory.django import DjangoModelFactory
from myapp.models import User, Post


class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    password = factory.PostGenerationMethod('set_password', 'pass123')


class PostFactory(DjangoModelFactory):
    class Meta:
        model = Post
    
    title = factory.Faker('sentence')
    content = factory.Faker('text')
    author = factory.SubFactory(UserFactory)
    status = 'published'


# tests/test_models.py
from django.test import TestCase
from .factories import UserFactory, PostFactory


class PostModelTest(TestCase):
    def test_create_multiple_posts(self):
        # Create multiple users and posts easily
        users = UserFactory.create_batch(5)
        posts = PostFactory.create_batch(10, author=factory.Iterator(users))
        
        self.assertEqual(len(posts), 10)
        self.assertEqual(len(set(p.author for p in posts)), 5)
```

## Best Practices

### 1. Test One Thing

```python
# ❌ BAD: Testing multiple things
def test_post(self):
    post = Post.objects.create(title='Test', author=self.user)
    self.assertEqual(post.title, 'Test')
    response = self.client.get(f'/posts/{post.id}/')
    self.assertEqual(response.status_code, 200)

# ✅ GOOD: One test, one thing
def test_post_creation(self):
    post = Post.objects.create(title='Test', author=self.user)
    self.assertEqual(post.title, 'Test')

def test_post_detail_view(self):
    post = Post.objects.create(title='Test', author=self.user)
    response = self.client.get(f'/posts/{post.id}/')
    self.assertEqual(response.status_code, 200)
```

### 2. Use Descriptive Test Names

```python
# ❌ BAD: Unclear names
def test_1(self):
    pass

def test_view(self):
    pass

# ✅ GOOD: Descriptive names
def test_create_post_with_valid_data(self):
    pass

def test_create_post_with_empty_title_fails(self):
    pass

def test_unauthenticated_user_cannot_create_post(self):
    pass
```

### 3. Test Edge Cases

```python
class PostAPITest(APITestCase):
    def test_create_post_success(self):
        # Happy path
        pass
    
    def test_create_post_empty_title(self):
        # Edge case: empty title
        pass
    
    def test_create_post_very_long_title(self):
        # Edge case: max length
        pass
    
    def test_create_post_special_characters(self):
        # Edge case: special chars
        pass
    
    def test_create_post_unauthenticated(self):
        # Edge case: no auth
        pass
```

## Summary

1. **Use TestCase** for most tests, TransactionTestCase only when needed
2. **Use setUpTestData** for class-level fixtures
3. **Test models, views, serializers, and services separately**
4. **Use APITestCase** for DRF endpoints
5. **Mock external dependencies** (email, APIs, etc.)
6. **Use factories** to create test data
7. **Mock external services** (requests.get/post)
8. **DON'T mock the service itself** - mock its dependencies
