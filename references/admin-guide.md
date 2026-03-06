# Django Admin Guide

Guide to customizing the Django admin interface for content management.

## Basic Admin Setup

```python
# admin.py
from django.contrib import admin
from .models import Post, Category, Comment


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'status', 'created_at']
    list_filter = ['status', 'created_at', 'category']
    search_fields = ['title', 'content']
    prepopulated_fields = {'slug': ('title',)}
    date_hierarchy = 'created_at'
    ordering = ['-created_at']


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'post_count']
    prepopulated_fields = {'slug': ('name',)}
    
    def post_count(self, obj):
        return obj.posts.count()


admin.site.register(Comment)
```

## List Display Options

```python
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    # Columns in list view
    list_display = [
        'name',
        'sku',
        'price',
        'stock',
        'is_active',
        'created_at',
    ]
    
    # Show boolean as icon
    list_display_links = ['name', 'sku']
    
    # Clickable fields
    list_editable = ['price', 'stock', 'is_active']
    
    # Fields that can be edited directly in list
    list_filter = ['category', 'is_active', 'created_at']
    
    # Filter sidebar
    search_fields = ['name', 'sku', 'description']
    
    # Search box
    date_hierarchy = 'created_at'
    
    # Date-based navigation
    ordering = ['-created_at']
    
    # Default ordering
    list_per_page = 25
```

## Forms in Admin

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'customer', 'status', 'total', 'created_at']
    
    fields = [
        'customer',
        ('status', 'priority'),
        'shipping_address',
        'billing_address',
        'notes',
    ]
    
    # Fields to display when creating
    readonly_fields = ['created_at', 'updated_at']
    
    # Fields that can't be edited
    radio_fields = {
        'status': admin.VERTICAL,
    }
    
    raw_id_fields = ['customer']
    
    # Use raw ID for ForeignKey with many objects
```

## Inline Models

```python
class OrderItemInline(admin.TabularInline):
    model = OrderItem
    extra = 1
    fields = ['product', 'quantity', 'unit_price', 'total_price']
    readonly_fields = ['total_price']
    
    def total_price(self, obj):
        return obj.quantity * obj.unit_price


class OrderItemStackedInline(admin.StackedInline):
    model = OrderItem
    extra = 1


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'customer', 'status', 'total']
    inlines = [OrderItemInline]
```

## Custom Admin Views

```python
@admin.register(Report)
class ReportAdmin(admin.ModelAdmin):
    list_display = ['name', 'created_at']
    
    def get_urls(self):
        urls = super().get_urls()
        custom_urls = [
            path(
                '<int:report_id>/generate/',
                self.admin_site.admin_view(self.generate_report),
                name='report_generate',
            ),
        ]
        return custom_urls + urls
    
    def generate_report(self, request, report_id):
        report = self.get_object(request, report_id)
        # Generate report logic
        self.message_user(request, 'Report generated successfully')
        return HttpResponseRedirect('..')
```

## Admin Actions

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'status', 'created_at']
    list_filter = ['status']
    actions = ['publish_posts', 'unpublish_posts', 'send_to_analysis']
    
    @admin.action(description='Publish selected posts')
    def publish_posts(self, request, queryset):
        queryset.update(status=PublicationStatus.PUBLISHED)
        self.message_user(request, f'Published {queryset.count()} posts')
    
    @admin.action(description='Unpublish selected posts')
    def unpublish_posts(self, request, queryset):
        queryset.update(status=PublicationStatus.DRAFT)
        self.message_user(request, f'Unpublished {queryset.count()} posts')
    
    def send_to_analysis(self, request, queryset):
        for post in queryset:
            analyze_post.delay(post.id)
        self.message_user(request, f'Sent {queryset.count()} posts to analysis')
```

## Search and Filters

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'customer', 'status', 'total', 'created_at']
    
    search_fields = [
        'id',
        'customer__username',
        'customer__email',
        'shipping_address__address1',
    ]
    
    list_filter = [
        ('created_at', DateFieldListFilter),
        'status',
        ('total', admin.RangeFilter),
    ]
    
    @admin.display(description='Customer Name', ordering='customer__username')
    def customer_name(self, obj):
        return obj.customer.get_full_name()
```

## Admin Theme Customization

```python
# admin.py
from django.contrib import admin
from django.contrib.admin import AdminSite


class MyAdminSite(AdminSite):
    site_title = 'My Site Admin'
    site_header = 'My Site Administration'
    index_title = 'Welcome to the Admin'


admin_site = MyAdminSite(name='myadmin')


# Register with custom site
@admin.register(Post, site=admin_site)
class PostAdmin(admin.ModelAdmin):
    ...
```

## Admin Permissions

```python
@admin.register(Document)
class DocumentAdmin(admin.ModelAdmin):
    list_display = ['title', 'status', 'created_by', 'created_at']
    
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(created_by=request.user)
    
    def has_view_permission(self, request, obj=None):
        if obj is None:
            return True
        return obj.created_by == request.user or request.user.is_staff
    
    def has_change_permission(self, request, obj=None):
        if obj is None:
            return True
        return obj.created_by == request.user
    
    def has_delete_permission(self, request, obj=None):
        if obj is None:
            return True
        return obj.created_by == request.user or request.user.is_superuser
```

## Admin Actions with Forms

```python
from django import forms
from django.contrib import admin


@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ['username', 'email', 'is_active', 'date_joined']
    actions = ['activate_users', 'send_welcome_email']
    
    @admin.action(description='Activate selected users')
    def activate_users(self, request, queryset):
        queryset.update(is_active=True)
        self.message_user(request, f'Activated {queryset.count()} users')


# Custom action with form
@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    actions = ['export_selected']
    
    def export_selected(self, request, queryset):
        # Custom export logic
        pass
    
    export_selected.short_description = 'Export selected articles'
```

## Summary

1. **Use list_display** to show important fields
2. **Use list_filter** for sidebar filtering
3. **Use search_fields** for search functionality
4. **Use inlines** for related models
5. **Use actions** for bulk operations
6. **Customize permissions** for security
7. **Use readonly_fields** for audit fields
8. **Organize with fieldsets** for complex forms
