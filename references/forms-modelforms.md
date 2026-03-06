# Forms & ModelForms Guide

Guide to creating and handling forms in Django for both template-based and API-based applications.

## Basic Forms

### Simple Form

```python
# forms.py
from django import forms


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)
    
    def clean_name(self):
        name = self.cleaned_data['name']
        if len(name) < 2:
            raise forms.ValidationError('Name too short')
        return name
    
    def clean(self):
        cleaned_data = super().clean()
        email = cleaned_data.get('email')
        name = cleaned_data.get('name')
        
        if email and name:
            # Cross-field validation
            pass
        
        return cleaned_data
```

### Form with Validation

```python
class OrderForm(forms.Form):
    PRODUCT_CHOICES = [('', 'Select a product')] + [
        (p.id, p.name) for p in Product.objects.all()
    ]
    
    product = forms.ChoiceField(choices=PRODUCT_CHOICES)
    quantity = forms.IntegerField(min_value=1, max_value=100)
    shipping_method = forms.ChoiceField(
        choices=[('standard', 'Standard'), ('express', 'Express')],
        widget=forms.RadioSelect
    )
    notes = forms.CharField(
        widget=forms.Textarea(attrs={'rows': 3}),
        required=False
    )
    
    def clean_quantity(self):
        quantity = self.cleaned_data['quantity']
        
        # Check inventory
        product_id = self.cleaned_data.get('product')
        if product_id:
            product = Product.objects.get(pk=product_id)
            if quantity > product.inventory:
                raise forms.ValidationError('Insufficient inventory')
        
        return quantity
    
    def clean(self):
        cleaned_data = super().clean()
        
        shipping_method = cleaned_data.get('shipping_method')
        quantity = cleaned_data.get('quantity')
        
        if shipping_method == 'express' and quantity and quantity > 10:
            raise forms.ValidationError(
                'Express shipping not available for orders over 10 items'
            )
        
        return cleaned_data
```

## ModelForms

### Basic ModelForm

```python
# forms.py
from django import forms
from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category', 'status']
        widgets = {
            'content': forms.Textarea(attrs={'rows': 10}),
            'category': forms.Select(attrs={'class': 'form-control'}),
        }
    
    def clean_title(self):
        title = self.cleaned_data['title']
        
        # Check for spam keywords
        spam_words = ['buy now', 'click here', 'free money']
        if any(word in title.lower() for word in spam_words):
            raise forms.ValidationError('Title contains spam keywords')
        
        return title
```

### ModelForm with Custom Fields

```python
class PostForm(forms.ModelForm):
    # Add custom field not in model
    notify_subscribers = forms.BooleanField(
        required=False,
        initial=True,
        help_text='Send notification to subscribers'
    )
    
    class Meta:
        model = Post
        fields = ['title', 'slug', 'content', 'category', 'status', 'tags']
        exclude = ['author', 'created_at', 'updated_at']
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        # Customize form
        self.fields['slug'].required = False
        self.fields['tags'].help_text = 'Hold Ctrl to select multiple'
        
        # Add CSS classes
        for field in self.fields.values():
            field.widget.attrs['class'] = 'form-control'
    
    def clean_slug(self):
        slug = self.cleaned_data.get('slug')
        if not slug:
            # Auto-generate from title
            from django.utils.text import slugify
            slug = slugify(self.cleaned_data.get('title', ''))
        
        # Check uniqueness
        queryset = Post.objects.filter(slug=slug)
        
        if self.instance.pk:
            queryset = queryset.exclude(pk=self.instance.pk)
        
        if queryset.exists():
            raise forms.ValidationError('Slug already exists')
        
        return slug
```

### ModelForm for Editing

```python
class PostEditForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'status']
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        # Make some fields readonly
        if self.instance.pk:
            self.fields['title'].disabled = True
    
    def clean(self):
        cleaned_data = super().clean()
        
        # Only admin can publish
        if (cleaned_data.get('status') == 'published' and 
            not self.request.user.is_staff):
            raise forms.ValidationError('Only admins can publish posts')
        
        return cleaned_data
    
    def save(self, commit=True):
        instance = super().save(commit=False)
        
        if commit:
            instance.save()
        
        return instance
```

## Form Handling in Views

### FBV with Form

```python
# views.py
from django.shortcuts import render, redirect
from .forms import PostForm


def create_post(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()  # If using M2M fields
            
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm()
    
    return render(request, 'posts/create.html', {'form': form})


def edit_post(request, pk):
    post = get_object_or_404(Post, pk=pk)
    
    if request.method == 'POST':
        form = PostForm(request.POST, instance=post)
        
        if form.is_valid():
            post = form.save()
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm(instance=post)
    
    return render(request, 'posts/edit.html', {'form': form, 'post': post})
```

### CBV with FormMixin

```python
from django.views.generic import CreateView, UpdateView
from django.contrib.auth.mixins import LoginRequiredMixin
from django.core.urlresolvers import reverse_lazy


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = 'posts/create.html'
    success_url = reverse_lazy('post_list')
    
    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'posts/edit.html'
    
    def get_queryset(self):
        return Post.objects.filter(author=self.request.user)
```

## DRF Serializers as Forms

```python
# serializers.py
from rest_framework import serializers
from .models import Order


class OrderCreateSerializer(serializers.Serializer):
    items = serializers.ListField(
        child=serializers.DictField(
            child=serializers.IntegerField()
        ),
        min_length=1
    )
    shipping_address_id = serializers.IntegerField()
    notes = serializers.CharField(required=False, allow_blank=True)
    
    def validate_items(self, value):
        if len(value) > 50:
            raise serializers.ValidationError('Maximum 50 items per order')
        
        for item in value:
            if 'product_id' not in item or 'quantity' not in item:
                raise serializers.ValidationError('Invalid item format')
            
            if item['quantity'] < 1:
                raise serializers.ValidationError('Quantity must be at least 1')
        
        return value
    
    def validate(self, data):
        # Cross-field validation
        if not data.get('items'):
            raise serializers.ValidationError('Order must have items')
        
        return data
    
    def create(self, validated_data):
        # Custom create logic
        return Order.objects.create(**validated_data)
```

## Form Widgets

```python
class ProductForm(forms.ModelForm):
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'category', 'image']
        widgets = {
            'name': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Enter product name'
            }),
            'description': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 4
            }),
            'price': forms.NumberInput(attrs={
                'class': 'form-control',
                'step': '0.01'
            }),
            'category': forms.Select(attrs={'class': 'form-control'}),
            'image': forms.FileInput(attrs={'class': 'form-control'}),
        }
```

## Form Sets

```python
# forms.py
from django.forms import formset_factory


class ItemForm(forms.Form):
    product = forms.ModelChoiceField(queryset=Product.objects.all())
    quantity = forms.IntegerField(min_value=1)


ItemFormSet = formset_factory(ItemForm, extra=2)


# views.py
def create_order_items(request):
    if request.method == 'POST':
        formset = ItemFormSet(request.POST)
        
        if formset.is_valid():
            # Process items
            for form in formset:
                if form.cleaned_data:
                    item = form.cleaned_data
                    # Add to order
    else:
        formset = ItemFormSet()
    
    return render(request, 'order/items_form.html', {'formset': formset})


# ModelFormSet
from django.forms import modelformset_factory
from .models import OrderItem


OrderItemFormSet = modelformset_factory(
    OrderItem,
    fields=['product', 'quantity', 'unit_price'],
    extra=1
)
```

## Form Media (CSS/JS)

```python
# forms.py
class RichTextForm(forms.Form):
    content = forms.CharField(
        widget=forms.Textarea(attrs={'class': 'rich-editor'})
    )
    
    class Media:
        css = {
            'all': ('css/editor.css',)
        }
        js = ('js/editor.js',)


# templates
{{ form.media }}
```

## Summary

1. **Use ModelForms** for model-backed forms
2. **Customize widgets** for better UX
3. **Use clean methods** for validation
4. **Handle M2M** with save_m2m()
5. **Use formset_factory** for multiple forms
6. **Add CSS classes** for styling
7. **Use cross-field validation** in clean()
8. **Return meaningful errors** to users
