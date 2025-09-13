# Plugin System Quick Reference

## Creating a New Plugin

### 1. Create Plugin Directory
```bash
mkdir plugins/my_plugin
cd plugins/my_plugin
```

### 2. Required Files

#### `__init__.py`
```python
from plugin_registry import PluginMetadata

PLUGIN_METADATA = PluginMetadata(
    name="My Plugin",
    version="1.0.0",
    description="What my plugin does",
    author="Your Name",
    email="your@email.com",
    django_apps=["plugins.my_plugin"],
)

default_app_config = 'plugins.my_plugin.apps.MyPluginConfig'
```

#### `apps.py`
```python
from django.apps import AppConfig

class MyPluginConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'plugins.my_plugin'
    verbose_name = 'My Plugin'

    def ready(self):
        print(f"✅ {self.verbose_name} is ready")
```

#### `migrations/__init__.py`
```python
# Empty file
```

### 3. Optional Files

#### `models.py`
```python
from django.db import models

class MyModel(models.Model):
    name = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
```

#### `views.py`
```python
from django.shortcuts import render
from django.http import HttpResponse

def index_view(request):
    return HttpResponse("Hello from My Plugin!")
```

#### `urls.py`
```python
from django.urls import path
from . import views

app_name = 'my_plugin'

urlpatterns = [
    path('', views.index_view, name='index'),
]
```

#### `templates/my_template.html`
```html
<!DOCTYPE html>
<html>
<head>
    <title>My Plugin</title>
</head>
<body>
    <h1>Welcome to My Plugin</h1>
</body>
</html>
```

## Management Commands

```bash
# List all plugins
python manage.py plugin_info

# List enabled plugins only
python manage.py plugin_info --enabled-only

# Show specific plugin details
python manage.py plugin_info --plugin my_plugin

# Run migrations for plugins
python manage.py makemigrations
python manage.py migrate
```

## Plugin Registry API

```python
from plugin_registry import plugin_registry

# Get all plugins
plugins = plugin_registry.get_all_plugins()

# Get enabled plugins
enabled = plugin_registry.get_enabled_plugins()

# Get specific plugin
plugin = plugin_registry.get_plugin('my_plugin')

# Enable/disable plugin
plugin_registry.enable_plugin('my_plugin')
plugin_registry.disable_plugin('my_plugin')

# Check dependencies
missing = plugin_registry.check_dependencies()
```

## Common Patterns

### Model with Admin
```python
# models.py
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

# admin.py
from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ['title', 'published', 'created_at']
    list_filter = ['published', 'created_at']
    search_fields = ['title', 'content']
```

### Views with Templates
```python
# views.py
from django.shortcuts import render, get_object_or_404
from .models import Article

def article_list(request):
    articles = Article.objects.filter(published=True)
    return render(request, 'articles/list.html', {'articles': articles})

def article_detail(request, pk):
    article = get_object_or_404(Article, pk=pk, published=True)
    return render(request, 'articles/detail.html', {'article': article})

# urls.py
from django.urls import path
from . import views

app_name = 'articles'

urlpatterns = [
    path('', views.article_list, name='list'),
    path('<int:pk>/', views.article_detail, name='detail'),
]
```

### Plugin with Dependencies
```python
# __init__.py
PLUGIN_METADATA = PluginMetadata(
    name="Advanced Articles",
    version="1.0.0",
    description="Advanced article features",
    dependencies=["articles_plugin"],  # Requires articles_plugin
    django_apps=["plugins.advanced_articles"],
)
```

## Troubleshooting Checklist

- [ ] `__init__.py` exists in plugin directory
- [ ] `PLUGIN_METADATA` is defined correctly
- [ ] Plugin directory name matches metadata
- [ ] No import errors in plugin code
- [ ] Migrations created and applied
- [ ] URLs properly configured
- [ ] Templates in correct location
- [ ] Static files properly organized

## File Structure Template

```
plugins/
  my_plugin/
    __init__.py              ✓ Required
    apps.py                  ✓ Required
    migrations/
      __init__.py            ✓ Required
    models.py                ○ Optional
    views.py                 ○ Optional
    urls.py                  ○ Optional
    admin.py                 ○ Optional
    forms.py                 ○ Optional
    templates/
      my_plugin/
        base.html            ○ Optional
        list.html            ○ Optional
    static/
      my_plugin/
        css/
          style.css          ○ Optional
        js/
          script.js          ○ Optional
    management/
      commands/
        my_command.py        ○ Optional
    tests/
      __init__.py            ○ Optional
      test_models.py         ○ Optional
      test_views.py          ○ Optional
```

## Testing Your Plugin

```python
# tests/test_models.py
from django.test import TestCase
from ..models import MyModel

class MyModelTest(TestCase):
    def test_model_creation(self):
        obj = MyModel.objects.create(name="Test")
        self.assertEqual(obj.name, "Test")

# tests/test_views.py
from django.test import TestCase, Client
from django.urls import reverse

class ViewTest(TestCase):
    def setUp(self):
        self.client = Client()

    def test_index_view(self):
        response = self.client.get('/my_plugin/')
        self.assertEqual(response.status_code, 200)
```

Run tests:
```bash
python manage.py test plugins.my_plugin
