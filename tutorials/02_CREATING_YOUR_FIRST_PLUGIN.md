# Creating Your First Plugin

In this tutorial, we'll walk through the process of creating your first Ritiko plugin - a simple "Hello World" example that demonstrates the basic plugin structure and functionality.

## Prerequisites

Before you begin, ensure you have:

1. A working Ritiko installation
2. Django development environment
3. Basic understanding of Python and Django

## Step 1: Create the Plugin Directory Structure

Start by creating the necessary directories and files for your plugin:

```bash
# Create the plugin directory
mkdir -p plugins/hello_world/migrations
mkdir -p plugins/hello_world/templates/hello_world

# Create required empty files
touch plugins/hello_world/__init__.py
touch plugins/hello_world/migrations/__init__.py
```

## Step 2: Define Plugin Metadata

Edit the `__init__.py` file to include your plugin's metadata:

```python
# plugins/hello_world/__init__.py
from plugin_registry import PluginMetadata

PLUGIN_METADATA = PluginMetadata(
    name="Hello World Plugin",
    version="1.0.0",
    description="A simple hello world plugin for Ritiko",
    author="Your Name",
    email="your.email@example.com",
    django_apps=["plugins.hello_world"],
)

default_app_config = 'plugins.hello_world.apps.HelloWorldConfig'
```

This metadata provides essential information about your plugin and will be used by the plugin registry to manage your plugin within the system.

## Step 3: Create the App Configuration

Next, create the Django app configuration for your plugin:

```python
# plugins/hello_world/apps.py
from core.plugin_app_config import PluginAppConfig

class HelloWorldConfig(PluginAppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'plugins.hello_world'
    verbose_name = 'Hello World Plugin'

    def ready(self):
        super().ready()
        print(f"✅ {self.verbose_name} is ready")
```

The `PluginAppConfig` base class provides hooks for plugin initialization and configuration. The `ready()` method is called when the plugin is loaded.

## Step 4: Create a Simple View

Now, let's create a simple view for your plugin:

```python
# plugins/hello_world/views.py
from django.shortcuts import render
from django.http import HttpResponse

def hello_view(request):
    """A simple view that returns a greeting."""
    return HttpResponse("<h1>Hello, World!</h1><p>My first Ritiko plugin is working!</p>")

def template_view(request):
    """A view that renders a template."""
    context = {
        'title': 'Hello World Plugin',
        'message': 'This is rendered from a plugin template!'
    }
    return render(request, 'hello_world/hello.html', context)
```

## Step 5: Create a Template

Create a simple HTML template for your plugin:

```html
<!-- plugins/hello_world/templates/hello_world/hello.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 40px;
            line-height: 1.6;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        h1 {
            color: #2c3e50;
        }
        .message {
            padding: 15px;
            background-color: #f8f9fa;
            border-left: 4px solid #28a745;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>{{ title }}</h1>
        <div class="message">
            <p>{{ message }}</p>
        </div>
        <p>This is a simple template from our plugin.</p>
        <p>Current time: {% now "jS F Y H:i" %}</p>
    </div>
</body>
</html>
```

## Step 6: Configure URLs

Create a URL configuration file for your plugin:

```python
# plugins/hello_world/urls.py
from django.urls import path
from . import views

app_name = 'hello_world'

urlpatterns = [
    path('', views.hello_view, name='index'),
    path('template/', views.template_view, name='template'),
]
```

This configuration defines two URL patterns:
- `/hello_world/` - Displays a simple HTTP response
- `/hello_world/template/` - Renders the template

## Step 7: Register Your Plugin

Add your plugin to the `INSTALLED_APPS` setting in your Django project's settings file:

```python
# In your project's settings.py

INSTALLED_APPS = [
    # ... existing apps
    'plugins.hello_world',
]
```

## Step 8: Run Migrations (if needed)

If your plugin includes models, you'll need to run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

For our simple Hello World plugin, this step isn't necessary since we haven't defined any models.

## Step 9: Test Your Plugin

Start the development server:

```bash
python manage.py runserver
```

Then, visit the following URLs in your browser:

- `http://localhost:8000/hello_world/` - Should display "Hello, World! My first Ritiko plugin is working!"
- `http://localhost:8000/hello_world/template/` - Should display your template-rendered page

## Step 10: Check Plugin Registration

You can use the management command to verify your plugin is registered correctly:

```bash
python manage.py plugin_info
```

This should list your Hello World plugin among the installed plugins.

## Understanding What Happened

Let's break down what you've accomplished:

1. **Plugin Discovery**: The system discovers your plugin through the `PLUGIN_METADATA` in `__init__.py`
2. **App Registration**: Your plugin is registered as a Django app
3. **URL Integration**: Your plugin's URLs are automatically included in the project's URL patterns
4. **Template Loading**: Django's template system can find and load templates from your plugin's `templates` directory

## Next Steps

Now that you've created your first plugin, you can:

1. Add models to store plugin-specific data
2. Create more complex views and templates
3. Add admin interfaces for your models
4. Extend existing models using the techniques covered in the next tutorial

In the next tutorial, we'll explore the core components of a plugin in more detail, including models, forms, and admin integration.
