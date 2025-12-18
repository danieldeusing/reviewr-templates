# Django 5 Code Review Skill

## Overview

This skill provides code review guidelines for Django 5 applications, focusing on async views, Django REST Framework, security best practices, and modern patterns.

## Key Review Areas

### 1. Async Views and ORM

**Check for proper async usage:**
- Use async views for I/O-bound operations
- Use `sync_to_async` for ORM operations in async views
- Consider `aiterator()` for large querysets
- Be aware of connection pool exhaustion

```python
# Good: Async view with proper ORM handling
from asgiref.sync import sync_to_async

async def user_list(request):
    users = await sync_to_async(list)(
        User.objects.filter(is_active=True).select_related('profile')
    )
    return JsonResponse({'users': [u.to_dict() for u in users]})

# Good: Async iteration for large datasets
async def export_data(request):
    async for item in Item.objects.filter(active=True).aiterator():
        yield process_item(item)

# Avoid: Direct ORM calls in async context
async def bad_view(request):
    users = User.objects.all()  # SynchronousOnlyOperation error
```

### 2. Security Best Practices

**Verify security patterns:**
- Check for proper CSRF protection
- Verify user input validation
- Ensure SQL injection prevention
- Review authentication and authorization

```python
# Good: Parameterized queries
User.objects.filter(email=user_input)  # Safe
User.objects.raw('SELECT * FROM users WHERE email = %s', [user_input])  # Safe

# Avoid: String interpolation in queries
User.objects.raw(f'SELECT * FROM users WHERE email = "{user_input}"')  # SQL injection!

# Good: Permission checks
from django.contrib.auth.decorators import permission_required

@permission_required('app.change_item', raise_exception=True)
def edit_item(request, pk):
    item = get_object_or_404(Item, pk=pk)
    # ...

# Good: Object-level permissions
def update_item(request, pk):
    item = get_object_or_404(Item, pk=pk)
    if item.owner != request.user:
        raise PermissionDenied
```

### 3. Django REST Framework

**Review DRF patterns:**
- Use proper serializers with validation
- Implement viewsets for CRUD operations
- Configure proper permissions and throttling
- Use pagination for list endpoints

```python
# Good: Serializer with validation
class UserSerializer(serializers.ModelSerializer):
    email = serializers.EmailField(validators=[UniqueValidator(queryset=User.objects.all())])

    class Meta:
        model = User
        fields = ['id', 'email', 'first_name', 'last_name']
        read_only_fields = ['id']

    def validate_email(self, value):
        if 'spam' in value:
            raise serializers.ValidationError("Invalid email domain")
        return value.lower()

# Good: ViewSet with permissions
class ItemViewSet(viewsets.ModelViewSet):
    queryset = Item.objects.all()
    serializer_class = ItemSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    pagination_class = StandardResultsSetPagination
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['status', 'category']
    search_fields = ['name', 'description']
    ordering_fields = ['created_at', 'name']
```

### 4. Database Optimization

**Check for database performance:**
- Use `select_related` for ForeignKey fields
- Use `prefetch_related` for reverse relations and M2M
- Avoid N+1 queries
- Use `only()` or `defer()` for large models

```python
# Good: Optimized queries
articles = Article.objects.select_related('author').prefetch_related(
    'tags',
    Prefetch('comments', queryset=Comment.objects.select_related('user'))
).filter(published=True)

# Avoid: N+1 queries
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # Separate query for each article!

# Good: Efficient bulk operations
Item.objects.bulk_create([Item(name=n) for n in names])
Item.objects.filter(status='draft').update(status='published')
```

### 5. Model Design

**Review model patterns:**
- Use appropriate field types
- Define proper indexes
- Implement clean() for validation
- Use model managers for query encapsulation

```python
# Good: Well-designed model
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.PROTECT, related_name='articles')
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    objects = ArticleManager()
    published = PublishedManager()

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['status', '-created_at']),
        ]

    def clean(self):
        if self.status == Status.PUBLISHED and not self.content:
            raise ValidationError("Published articles must have content")

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

### 6. Forms and Validation

**Verify form handling:**
- Use Django forms for data validation
- Implement custom validators
- Handle form errors properly
- Use ModelForm for model-backed forms

```python
# Good: Form with validation
class RegistrationForm(forms.ModelForm):
    password = forms.CharField(widget=forms.PasswordInput)
    password_confirm = forms.CharField(widget=forms.PasswordInput)

    class Meta:
        model = User
        fields = ['email', 'first_name', 'last_name']

    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        password_confirm = cleaned_data.get('password_confirm')

        if password and password_confirm and password != password_confirm:
            raise forms.ValidationError("Passwords don't match")

        return cleaned_data
```

### 7. Testing

**Verify testing practices:**
- Use Django's TestCase for database tests
- Use pytest-django for better testing experience
- Mock external services
- Test permissions and edge cases

```python
# Good: Comprehensive test
import pytest
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APIClient

@pytest.mark.django_db
class TestArticleAPI:
    def test_list_articles_authenticated(self, api_client, user):
        api_client.force_authenticate(user=user)
        response = api_client.get(reverse('article-list'))
        assert response.status_code == status.HTTP_200_OK

    def test_create_article_requires_auth(self, api_client):
        response = api_client.post(reverse('article-list'), {'title': 'Test'})
        assert response.status_code == status.HTTP_401_UNAUTHORIZED

    def test_user_can_only_edit_own_articles(self, api_client, user, other_user_article):
        api_client.force_authenticate(user=user)
        response = api_client.patch(
            reverse('article-detail', args=[other_user_article.id]),
            {'title': 'Hacked'}
        )
        assert response.status_code == status.HTTP_403_FORBIDDEN
```

## Common Issues to Flag

1. **N+1 queries** from missing select_related/prefetch_related
2. **Raw SQL injection** from string interpolation
3. **Missing CSRF protection** on forms
4. **Improper permission checks** or missing authorization
5. **Synchronous ORM calls** in async views
6. **Missing database indexes** for filtered/ordered fields
7. **Hardcoded secrets** in settings
8. **Missing input validation** in forms/serializers

## Review Checklist

- [ ] Queries are optimized (select_related, prefetch_related)
- [ ] No SQL injection vulnerabilities
- [ ] Proper authentication and authorization
- [ ] CSRF protection enabled
- [ ] Input validation implemented
- [ ] Async code uses sync_to_async properly
- [ ] Database migrations are reversible
- [ ] Tests cover critical functionality
