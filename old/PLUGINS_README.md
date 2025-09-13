# Ritiko Plugins System Documentation

## Overview

The Ritiko Plugins System is a comprehensive, auto-discovery plugin architecture that allows you to extend the Django application with modular functionality. The system provides automatic plugin registration, metadata management, dependency tracking, and seamless Django integration.

## Table of Contents

1. [Architecture](#architecture)
2. [Getting Started](#getting-started)
3. [Creating a Plugin](#creating-a-plugin)
4. [Plugin Metadata](#plugin-metadata)
5. [Plugin Registry](#plugin-registry)
6. [Management Commands](#management-commands)
7. [Auto-Discovery](#auto-discovery)
8. [URL Routing](#url-routing)
9. [Plugin Organization Settings](#plugin-organization-settings)
10. [Plugin Permissions](#plugin-permissions)
11. [Plugin Menus](#plugin-menus)
12. [Plugin Widgets](#plugin-widgets)
13. [Examples](#examples)
14. [Best Practices](#best-practices)
15. [Troubleshooting](#troubleshooting)

## Architecture

The plugins system consists of several key components:

- **Plugin Registry** (`plugin_registry.py`): Central registry for managing plugin metadata and lifecycle
- **Auto-Discovery** (`plugins.py`): Automatic plugin detection and Django integration
- **Plugin Structure**: Standardized directory structure for plugins
- **Management Commands**: CLI tools for plugin administration
- **URL Auto-Routing**: Automatic URL inclusion for enabled plugins

## Getting Started

### Prerequisites

- Django 2.1+
- Python 3.6+

### Installation

The plugins system is already integrated into your Django project. No additional installation is required.

### Quick Start

1. Create a new plugin directory in the `plugins/` folder
2. Add the required files (`__init__.py`, `models.py`, `views.py`, etc.)
3. Define plugin metadata in `__init__.py`
4. Run Django migrations if your plugin has models
5. Access your plugin at `/your_plugin_name/`

## Creating a Plugin

### Directory Structure

Each plugin should follow this structure:

```
plugins/
  your_plugin_name/
    __init__.py          # Plugin metadata and configuration
    apps.py              # Django app configuration
    models.py            # Database models (optional)
    views.py             # View functions/classes (optional)
    urls.py              # URL patterns (optional)
    admin.py             # Django admin configuration (optional)
    templates/           # Template files (optional)
      your_template.html
    static/              # Static files (optional)
      css/
      js/
      images/
    migrations/          # Database migrations (optional)
      __init__.py
    management/          # Management commands (optional)
      commands/
        your_command.py
```

### Required Files

#### `__init__.py`
```python
"""
Your Plugin - Description of what your plugin does.
"""
from plugin_registry import PluginMetadata

# Plugin metadata
PLUGIN_METADATA = PluginMetadata(
    name="Your Plugin Name",
    version="1.0.0",
    description="Detailed description of your plugin functionality",
    author="Your Name",
    email="your.email@example.com",
    url="https://github.com/your-repo/your-plugin",
    dependencies=[],  # List of required plugin IDs
    django_apps=["plugins.your_plugin_name"],
    enabled=True
)

# Plugin configuration
default_app_config = 'plugins.your_plugin_name.apps.YourPluginConfig'
```

#### `apps.py`
```python
from django.apps import AppConfig

class YourPluginConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'plugins.your_plugin_name'
    verbose_name = 'Your Plugin Name'

    def ready(self):
        """Called when the app is ready."""
        print(f"✅ {self.verbose_name} is ready")
```

## Plugin Metadata

Plugin metadata should be defined in a `plugin.json` file in the plugin directory. This JSON format is preferred for marketplace compatibility and easier configuration management.

```json
{
  "name": "Your Plugin Name",
  "version": "1.0.0",
  "description": "Description of what your plugin does",
  "author": "Your Name",
  "email": "your.email@example.com",
  "url": "https://github.com/your/plugin",
  "categories": ["category1", "category2"],
  "tags": ["tag1", "tag2", "tag3"],
  "dependencies": [],
  "django_apps": ["plugins.your_plugin"],
  "enabled": true
}
```

### Metadata Fields

- **name**: Display name for your plugin
- **version**: Follow semantic versioning (MAJOR.MINOR.PATCH)
- **description**: Explain what your plugin does and its key features
- **author**: Your name or organization
- **email**: Contact email for support
- **url**: Link to documentation, repository, or homepage
- **categories**: List of categories for marketplace classification
- **tags**: List of searchable tags for discovery
- **dependencies**: List of plugin IDs that must be installed
- **django_apps**: List of Django app names (usually just your plugin)
- **enabled**: Whether the plugin should be loaded (default: true)

### Legacy Python Metadata Support

For backward compatibility, plugins can still define metadata in their `__init__.py` file:

```python
from plugin_registry import PluginMetadata

PLUGIN_METADATA = PluginMetadata(
    name="Your Plugin Name",
    version="1.0.0",
    description="Description of what your plugin does",
    author="Your Name",
    email="your.email@example.com",
    url="https://github.com/your/plugin",
    dependencies=[],
    django_apps=["plugins.your_plugin"],
    enabled=True
)
```

**Note**: JSON metadata takes precedence over Python metadata if both exist.

## Plugin Registry

The plugin registry (`plugin_registry.py`) manages all plugin operations:

### Key Methods

```python
from plugin_registry import plugin_registry

# Get all plugins
all_plugins = plugin_registry.get_all_plugins()

# Get enabled plugins only
enabled_plugins = plugin_registry.get_enabled_plugins()

# Get specific plugin
plugin = plugin_registry.get_plugin('shop_plugin')

# Enable/disable plugins
plugin_registry.enable_plugin('shop_plugin')
plugin_registry.disable_plugin('blog_plugin')

# Check dependencies
missing_deps = plugin_registry.check_dependencies()

# Get plugin information
info = plugin_registry.get_plugin_info()
```

## Management Commands

### `plugin_info`

Display information about registered plugins.

```bash
# Show all plugins
python manage.py plugin_info

# Show only enabled plugins
python manage.py plugin_info --enabled-only

# Show specific plugin details
python manage.py plugin_info --plugin shop_plugin
```

**Example Output:**
```
All Registered Plugins:

Shop Plugin (shop_plugin)
  Version: 1.0.0
  Status: ✅ Enabled
  Description: E-commerce plugin providing product management...
  Author: Ritiko Development Team

Blog Plugin (blog_plugin)
  Version: 1.0.0
  Status: ✅ Enabled
  Description: Blogging plugin providing post management...
  Author: Ritiko Development Team
```

## Auto-Discovery

The auto-discovery system automatically:

1. Scans the `plugins/` directory
2. Imports plugin modules
3. Registers plugin metadata
4. Adds enabled plugins to `INSTALLED_APPS`
5. Checks for dependency issues
6. Reports plugin status

### How It Works

1. **Discovery**: `plugin_registry.discover_plugins()` scans for plugin directories
2. **Import**: Each plugin's `__init__.py` is imported
3. **Registration**: Plugin metadata is extracted and registered
4. **Integration**: Enabled plugins are added to Django's `INSTALLED_APPS`

## URL Routing

Plugin URLs are automatically included based on the plugin ID:

- `shop_plugin` → `/shop_plugin/`
- `blog_plugin` → `/blog_plugin/`
- `your_plugin` → `/your_plugin/`

### URL Configuration

Create a `urls.py` file in your plugin:

```python
from django.urls import path
from . import views

app_name = 'your_plugin_name'

urlpatterns = [
    path('', views.index_view, name='index'),
    path('detail/<int:id>/', views.detail_view, name='detail'),
]
```

## Examples

### Example 1: Simple Plugin

```python
# plugins/hello_world/__init__.py
from plugin_registry import PluginMetadata

PLUGIN_METADATA = PluginMetadata(
    name="Hello World",
    version="1.0.0",
    description="A simple hello world plugin",
    author="Developer",
    django_apps=["plugins.hello_world"],
)
```

```python
# plugins/hello_world/views.py
from django.http import HttpResponse

def hello_view(request):
    return HttpResponse("Hello, World!")
```

```python
# plugins/hello_world/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.hello_view, name='hello'),
]
```

## Plugin Organization Settings

The Plugin Organization Settings system allows plugins to extend organization settings without modifying the Organization model directly. This provides a clean way to add plugin-specific configuration options that are managed per organization.

### Overview

Instead of adding fields to the Organization model, plugins can define their own settings that are:
- Stored in a JSON field on the organization
- Managed through the Django admin interface
- Validated by the plugin
- Automatically discovered and integrated

### Creating Organization Settings in Your Plugin

#### Step 1: Create Organization Settings File

Create an `organization_settings.py` file in your plugin directory:

```python
# plugins/your_plugin/organization_settings.py
from django.db import models
from plugin_organization_settings import (
    OrganizationSettingsPlugin,
    OrganizationSettingField,
    organization_settings_registry,
)

class YourPluginOrganizationSettings(OrganizationSettingsPlugin):
    """Plugin that adds your custom settings to organizations."""

    def get_plugin_id(self) -> str:
        return "your_plugin"

    def get_settings(self):
        return [
            OrganizationSettingField(
                name="enable_feature",
                field_type=models.BooleanField,
                default_value=False,
                verbose_name="Enable Feature",
                help_text="Enable the main feature for this organization",
                form_section="general",
            ),
            OrganizationSettingField(
                name="api_key",
                field_type=models.CharField,
                default_value="",
                verbose_name="API Key",
                help_text="External API key for integration",
                form_section="api",
                admin_only=True,  # Only superadmins can edit
            ),
            OrganizationSettingField(
                name="items_per_page",
                field_type=models.PositiveIntegerField,
                default_value=20,
                verbose_name="Items Per Page",
                help_text="Number of items to display per page",
                form_section="display",
            ),
            OrganizationSettingField(
                name="theme_color",
                field_type=models.CharField,
                default_value="blue",
                verbose_name="Theme Color",
                help_text="Color theme for the interface",
                choices=[
                    ("blue", "Blue"),
                    ("green", "Green"),
                    ("red", "Red"),
                    ("purple", "Purple"),
                ],
                form_section="appearance",
            ),
        ]

    def validate_setting(self, setting_name: str, value, organization) -> bool:
        """Custom validation for settings."""
        if setting_name == "items_per_page":
            return isinstance(value, int) and 1 <= value <= 100

        if setting_name == "api_key":
            # Validate API key format
            return len(str(value)) >= 10 if value else True

        return True

    def on_setting_changed(self, setting_name: str, old_value, new_value, organization) -> None:
        """Handle setting changes."""
        if setting_name == "enable_feature":
            if new_value and not old_value:
                print(f"Feature enabled for organization: {organization.name}")
                # Trigger feature setup logic here
            elif old_value and not new_value:
                print(f"Feature disabled for organization: {organization.name}")
                # Trigger cleanup logic here

# Register the plugin
your_settings_plugin = YourPluginOrganizationSettings()
organization_settings_registry.register_plugin(your_settings_plugin)
```

#### Step 2: Available Field Types

The system supports all standard Django field types:

```python
# Boolean field
OrganizationSettingField(
    name="enable_notifications",
    field_type=models.BooleanField,
    default_value=True,
)

# Text field with choices
OrganizationSettingField(
    name="notification_frequency",
    field_type=models.CharField,
    default_value="daily",
    choices=[
        ("immediate", "Immediate"),
        ("daily", "Daily"),
        ("weekly", "Weekly"),
    ],
)

# Integer field with validation
OrganizationSettingField(
    name="max_users",
    field_type=models.PositiveIntegerField,
    default_value=50,
    help_text="Maximum number of users allowed",
)

# Decimal field for monetary values
OrganizationSettingField(
    name="service_fee",
    field_type=models.DecimalField,
    default_value=9.99,
    help_text="Service fee in dollars",
)

# Email field
OrganizationSettingField(
    name="support_email",
    field_type=models.EmailField,
    default_value="",
    help_text="Support contact email",
)

# URL field
OrganizationSettingField(
    name="webhook_url",
    field_type=models.URLField,
    default_value="",
    help_text="Webhook endpoint URL",
)

# Text area field
OrganizationSettingField(
    name="custom_message",
    field_type=models.TextField,
    default_value="",
    help_text="Custom message to display to users",
)
```

#### Step 3: Setting Field Properties

Each `OrganizationSettingField` supports these properties:

- **name**: Unique setting name within the plugin
- **field_type**: Django model field type
- **default_value**: Default value when setting is not configured
- **verbose_name**: Human-readable label (auto-generated if not provided)
- **help_text**: Description of the setting
- **required**: Whether the field is required
- **choices**: List of (value, label) tuples for choice fields
- **form_section**: Group settings into sections (e.g., "general", "api", "display")
- **admin_only**: Whether only superadmins can modify this setting

### Using Organization Settings in Your Code

#### Getting Setting Values

```python
from plugin_organization_settings import get_organization_setting

# In your views or business logic
def your_view(request):
    organization = request.user.profile.organization

    # Get a setting value
    feature_enabled = get_organization_setting(
        organization,
        "enable_feature",
        default=False
    )

    items_per_page = get_organization_setting(
        organization,
        "items_per_page",
        default=20
    )

    if feature_enabled:
        # Feature-specific logic here
        pass
```

#### Setting Values Programmatically

```python
from plugin_organization_settings import set_organization_setting

# In your business logic
def enable_feature_for_org(organization):
    success = set_organization_setting(
        organization,
        "enable_feature",
        True
    )

    if success:
        print("Feature enabled successfully")
    else:
        print("Failed to enable feature (validation failed)")
```

#### Getting All Plugin Settings

```python
from plugin_organization_settings import get_all_organization_plugin_settings

def get_plugin_config(organization):
    all_settings = get_all_organization_plugin_settings(organization)

    # Returns: {plugin_id: {setting_name: value}}
    your_plugin_settings = all_settings.get("your_plugin", {})

    return your_plugin_settings
```

### Django Admin Integration

The system automatically integrates with Django admin. To enable this for your Organization admin:

```python
# In your admin.py or core/admin.py
from django.contrib import admin
from plugin_organization_admin import PluginSettingsAdminMixin
from core.models import Organization

class OrganizationAdmin(PluginSettingsAdminMixin, admin.ModelAdmin):
    # Your existing admin configuration
    list_display = ['name', 'is_active', 'program']

    # Plugin settings will be automatically added as collapsible fieldsets

admin.site.register(Organization, OrganizationAdmin)
```

This will add plugin settings as collapsible sections in the organization admin form, grouped by plugin and section.

### Auto-Discovery

Plugin organization settings are automatically discovered when the application starts. The system:

1. Scans all plugin directories for `organization_settings.py` files
2. Imports and registers any settings plugins found
3. Makes settings available in the Django admin interface
4. Reports discovery results in the console

### Real-World Examples

#### Blog Plugin Settings

```python
# plugins/blog_plugin/organization_settings.py
class BlogOrganizationSettingsPlugin(OrganizationSettingsPlugin):
    def get_plugin_id(self) -> str:
        return "blog_plugin"

    def get_settings(self):
        return [
            OrganizationSettingField(
                name="posts_per_page",
                field_type=models.PositiveIntegerField,
                default_value=10,
                verbose_name="Posts Per Page",
                help_text="Number of blog posts to display per page",
                form_section="display",
            ),
            OrganizationSettingField(
                name="allow_comments",
                field_type=models.BooleanField,
                default_value=True,
                verbose_name="Allow Comments",
                help_text="Allow users to comment on blog posts",
                form_section="interaction",
            ),
            OrganizationSettingField(
                name="blog_theme",
                field_type=models.CharField,
                default_value="default",
                choices=[
                    ("default", "Default"),
                    ("modern", "Modern"),
                    ("classic", "Classic"),
                ],
                form_section="appearance",
            ),
        ]
```

#### E-commerce Plugin Settings

```python
# plugins/shop_plugin/organization_settings.py
class ShopOrganizationSettingsPlugin(OrganizationSettingsPlugin):
    def get_plugin_id(self) -> str:
        return "shop_plugin"

    def get_settings(self):
        return [
            OrganizationSettingField(
                name="shop_currency",
                field_type=models.CharField,
                default_value="USD",
                choices=[
                    ("USD", "US Dollar"),
                    ("EUR", "Euro"),
                    ("GBP", "British Pound"),
                ],
                form_section="general",
            ),
            OrganizationSettingField(
                name="tax_rate",
                field_type=models.DecimalField,
                default_value=0.0875,
                verbose_name="Tax Rate",
                help_text="Tax rate as decimal (0.0875 = 8.75%)",
                form_section="pricing",
            ),
            OrganizationSettingField(
                name="free_shipping_threshold",
                field_type=models.DecimalField,
                default_value=50.00,
                verbose_name="Free Shipping Threshold",
                help_text="Minimum order total for free shipping",
                form_section="pricing",
            ),
        ]
```

### Testing Plugin Organization Settings

```python
# In your tests
def test_plugin_organization_settings(self):
    from plugin_organization_settings import (
        get_organization_setting,
        set_organization_setting
    )

    # Test getting default value
    default_value = get_organization_setting(
        self.organization,
        "enable_feature",
        default=False
    )
    self.assertEqual(default_value, False)

    # Test setting value
    success = set_organization_setting(
        self.organization,
        "enable_feature",
        True
    )
    self.assertTrue(success)

    # Test getting updated value
    updated_value = get_organization_setting(
        self.organization,
        "enable_feature"
    )
    self.assertEqual(updated_value, True)
```

### Best Practices for Organization Settings

1. **Use descriptive setting names**: Choose clear, specific names like `enable_email_notifications` instead of `email_on`

2. **Provide sensible defaults**: Always set appropriate default values that work for most organizations

3. **Group related settings**: Use `form_section` to organize settings logically (e.g., "general", "api", "display", "security")

4. **Add helpful descriptions**: Use `help_text` to explain what each setting does and any format requirements

5. **Implement validation**: Use the `validate_setting` method to ensure data integrity

6. **Handle setting changes**: Use `on_setting_changed` to trigger actions when important settings change

7. **Use admin_only sparingly**: Only mark truly sensitive settings as `admin_only`

8. **Consider performance**: Cache frequently accessed settings in your views

### Migration Path

If you have existing organization settings in the Organization model that you want to move to the plugin system:

1. Create migration to copy existing data to the plugin_settings JSON field
2. Update your code to use the new plugin settings system
3. Create migration to remove old fields from Organization model

```python
# Migration example
def migrate_existing_settings(apps, schema_editor):
    Organization = apps.get_model('core', 'Organization')

    for org in Organization.objects.all():
        plugin_settings = org.plugin_settings or {}
        if 'your_plugin' not in plugin_settings:
            plugin_settings['your_plugin'] = {}

        # Migrate existing field
        plugin_settings['your_plugin']['old_setting'] = org.old_field_name
        org.plugin_settings = plugin_settings
        org.save()
```

## Plugin Permissions

The Plugin Permissions system allows plugins to define custom permissions that integrate seamlessly with Django's authentication and authorization framework. This enables plugins to create their own permission-based access control without modifying core models.

### Overview

Instead of manually creating permissions in Django admin or database, plugins can:
- Define permissions programmatically in code
- Automatically create permissions in the database
- Organize permissions by category and plugin
- Create default permission groups
- Use decorators and mixins for view protection

### Creating Plugin Permissions

#### Step 1: Define Permission Provider

Create a `permissions.py` file in your plugin directory:

```python
# plugins/your_plugin/permissions.py
from plugin_permissions import (
    PluginPermissionProvider,
    PluginPermission,
    plugin_permission_registry,
)

class YourPluginPermissionProvider(PluginPermissionProvider):
    """Permission provider for your plugin."""

    def get_plugin_id(self) -> str:
        return "your_plugin"

    def get_permissions(self):
        return [
            # Basic CRUD permissions
            PluginPermission(
                codename="can_create_items",
                name="Can create items",
                description="Allows user to create new items",
                category="item_management",
            ),
            PluginPermission(
                codename="can_edit_items",
                name="Can edit items",
                description="Allows user to modify existing items",
                category="item_management",
            ),
            PluginPermission(
                codename="can_delete_items",
                name="Can delete items",
                description="Allows user to delete items",
                category="item_management",
            ),

            # Administrative permissions
            PluginPermission(
                codename="can_configure_plugin",
                name="Can configure plugin settings",
                description="Allows user to modify plugin configuration",
                category="administration",
            ),

            # Reporting permissions
            PluginPermission(
                codename="can_view_reports",
                name="Can view reports",
                description="Allows user to view plugin analytics and reports",
                category="reporting",
            ),
        ]

    def get_default_permission_groups(self):
        """Define default permission groups."""
        return {
            "users": [
                "can_create_items",
                "can_edit_items",
            ],
            "moderators": [
                "can_create_items",
                "can_edit_items",
                "can_delete_items",
            ],
            "administrators": [
                "can_create_items",
                "can_edit_items",
                "can_delete_items",
                "can_configure_plugin",
                "can_view_reports",
            ],
        }

    def on_permissions_created(self, permissions):
        """Handle post-creation actions."""
        print(f"Created {len(permissions)} permissions for your_plugin")

        # Auto-assign permissions to specific user types
        from core.models.auth import User
        superadmins = User.objects.filter(user_type=User.SUPERADMIN_USER)

        for user in superadmins:
            for perm in permissions:
                if perm.codename == "can_configure_plugin":
                    user.user_permissions.add(perm)

# Register the permission provider
your_permission_provider = YourPluginPermissionProvider()
plugin_permission_registry.register_plugin(your_permission_provider)
```

#### Step 2: Permission Field Properties

Each `PluginPermission` supports these properties:

- **codename**: Unique permission identifier (auto-prefixed with `can_` if needed)
- **name**: Human-readable permission name
- **content_type_model**: Django model to attach permission to (default: "organization")
- **content_type_app**: Django app containing the model (default: "core")
- **description**: Detailed explanation of what the permission allows
- **category**: Group related permissions (e.g., "management", "reporting", "administration")

### Using Plugin Permissions in Views

#### Function-Based Views with Decorators

```python
# In your plugin views.py
from plugin_permission_decorators import (
    plugin_permission_required,
    any_plugin_permission_required,
)

@plugin_permission_required('can_create_items')
def create_item_view(request):
    """View that requires can_create_items permission."""
    # Your view logic here
    pass

@plugin_permission_required(['can_edit_items', 'can_delete_items'])
def manage_items_view(request):
    """View that requires BOTH edit AND delete permissions."""
    # Your view logic here
    pass

@any_plugin_permission_required(['can_edit_items', 'can_view_reports'])
def dashboard_view(request):
    """View that requires EITHER edit items OR view reports permission."""
    # Your view logic here
    pass
```

#### Class-Based Views with Mixins

```python
# In your plugin views.py
from django.views.generic import CreateView, ListView, UpdateView
from plugin_permission_decorators import (
    PluginPermissionRequiredMixin,
    AnyPluginPermissionRequiredMixin,
)

class CreateItemView(PluginPermissionRequiredMixin, CreateView):
    """View for creating items - requires specific permission."""
    plugin_permission_required = 'can_create_items'
    plugin_permission_denied_message = "You need permission to create items."
    # ... rest of your view

class ManageItemsView(PluginPermissionRequiredMixin, ListView):
    """View that requires multiple permissions."""
    plugin_permission_required = ['can_edit_items', 'can_delete_items']
    # ... rest of your view

class DashboardView(AnyPluginPermissionRequiredMixin, TemplateView):
    """View that accepts any of several permissions."""
    plugin_permissions_required = ['can_edit_items', 'can_view_reports']
    # ... rest of your view
```

### Using Plugin Permissions in Templates

#### Template Tags

```html
<!-- In your templates -->
{% load plugin_permission_decorators %}

<!-- Check single permission -->
{% has_plugin_permission 'can_create_items' as can_create %}
{% if can_create %}
    <a href="{% url 'create_item' %}" class="btn btn-primary">Create Item</a>
{% endif %}

<!-- Check multiple permissions (user needs ANY) -->
{% has_any_plugin_permission 'can_edit_items' 'can_delete_items' as can_manage %}
{% if can_manage %}
    <a href="{% url 'manage_items' %}" class="btn btn-secondary">Manage Items</a>
{% endif %}
```

#### Context Processor (Global Access)

Add to your Django settings:

```python
# settings.py
TEMPLATES = [
    {
        # ... other settings
        'OPTIONS': {
            'context_processors': [
                # ... other context processors
                'plugin_permission_decorators.plugin_permissions_context_processor',
            ],
        },
    },
]
```

Then use in templates:

```html
<!-- Direct access to user permissions -->
{% if plugin_perms.can_create_items %}
    <a href="{% url 'create_item' %}">Create Item</a>
{% endif %}

{% if plugin_perms.can_configure_plugin %}
    <a href="{% url 'plugin_settings' %}">Plugin Settings</a>
{% endif %}
```

### Management Commands

#### Sync Permissions to Database

```bash
# Create/update all plugin permissions in database
python manage.py plugin_permissions --sync

# Force recreation of existing permissions
python manage.py plugin_permissions --sync --force
```

#### View Permissions

```bash
# Show all plugin permissions
python manage.py plugin_permissions

# Show permissions for specific plugin
python manage.py plugin_permissions --plugin blog_plugin
```

#### Manage User Permissions

```bash
# Assign permission to user
python manage.py plugin_permissions --assign user@example.com can_create_items

# Remove permission from user
python manage.py plugin_permissions --remove user@example.com can_create_items

# Show user's plugin permissions
python manage.py plugin_permissions --user-permissions user@example.com
```

#### Create Permission Groups

```bash
# Create default permission groups for all plugins
python manage.py plugin_permissions --create-groups
```

### Programmatic Permission Management

#### Check Permissions in Code

```python
from plugin_permissions import user_has_plugin_permission, PluginPermissionManager

def some_business_logic(user):
    # Check single permission
    if user_has_plugin_permission(user, 'can_create_items'):
        # User can create items
        pass

    # Get all user's plugin permissions
    user_perms = PluginPermissionManager.get_user_plugin_permissions(user)
    print(f"User has permissions: {user_perms}")
```

#### Assign/Remove Permissions

```python
from plugin_permissions import PluginPermissionManager

# Assign permission to user
success = PluginPermissionManager.assign_plugin_permission(user, 'can_create_items')

# Remove permission from user
success = PluginPermissionManager.remove_plugin_permission(user, 'can_create_items')
```

### Real-World Examples

#### Blog Plugin Permissions

```python
# plugins/blog_plugin/permissions.py
class BlogPermissionProvider(PluginPermissionProvider):
    def get_plugin_id(self) -> str:
        return "blog_plugin"

    def get_permissions(self):
        return [
            # Content management
            PluginPermission(
                codename="can_create_blog_posts",
                name="Can create blog posts",
                description="Allows creating new blog posts",
                category="content_management",
            ),
            PluginPermission(
                codename="can_publish_blog_posts",
                name="Can publish blog posts",
                description="Allows publishing/unpublishing posts",
                category="content_management",
            ),

            # Comment moderation
            PluginPermission(
                codename="can_moderate_comments",
                name="Can moderate blog comments",
                description="Allows approving, editing, or deleting comments",
                category="moderation",
            ),

            # Analytics
            PluginPermission(
                codename="can_view_blog_analytics",
                name="Can view blog analytics",
                description="Allows viewing post statistics and analytics",
                category="analytics",
            ),
        ]

    def get_default_permission_groups(self):
        return {
            "blog_writers": ["can_create_blog_posts"],
            "blog_editors": [
                "can_create_blog_posts",
                "can_publish_blog_posts",
                "can_moderate_comments",
            ],
            "blog_admins": [
                "can_create_blog_posts",
                "can_publish_blog_posts",
                "can_moderate_comments",
                "can_view_blog_analytics",
            ],
        }
```

#### E-commerce Plugin Permissions

```python
# plugins/shop_plugin/permissions.py
class ShopPermissionProvider(PluginPermissionProvider):
    def get_plugin_id(self) -> str:
        return "shop_plugin"

    def get_permissions(self):
        return [
            # Product management
            PluginPermission(
                codename="can_manage_products",
                name="Can manage products",
                description="Create, edit, and delete products",
                category="product_management",
            ),
            PluginPermission(
                codename="can_set_prices",
                name="Can set product prices",
                description="Modify product pricing and discounts",
                category="product_management",
            ),

            # Order processing
            PluginPermission(
                codename="can_process_orders",
                name="Can process orders",
                description="View and update order status",
                category="order_management",
            ),
            PluginPermission(
                codename="can_refund_orders",
                name="Can refund orders",
                description="Process refunds and returns",
                category="order_management",
            ),

            # Financial access
            PluginPermission(
                codename="can_view_financial_reports",
                name="Can view financial reports",
                description="Access sales and revenue reports",
                category="financial",
            ),
        ]
```

### Auto-Discovery

Plugin permissions are automatically discovered when the application starts. The system:

1. Scans all plugin directories for `permissions.py` files
2. Imports and registers permission providers
3. Creates permissions in the database
4. Reports discovery results in the console

You'll see output like:

```
🔐 Registered permissions plugin: blog_plugin with 4 permissions
🔐 Registered permissions plugin: shop_plugin with 5 permissions
🔐 Discovered 2 permission plugins
🔐 Created 9 plugin permissions in database
```

### Testing Plugin Permissions

```python
# In your plugin tests
from django.test import TestCase
from django.contrib.auth.models import Permission
from core.models.auth import User
from plugin_permissions import PluginPermissionManager

class PluginPermissionTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            email='test@example.com',
            password='testpass123'
        )

    def test_permission_creation(self):
        """Test that plugin permissions are created."""
        permission = Permission.objects.get(codename='can_create_items')
        self.assertEqual(permission.name, 'Can create items')

    def test_permission_assignment(self):
        """Test assigning plugin permission to user."""
        success = PluginPermissionManager.assign_plugin_permission(
            self.user, 'can_create_items'
        )
        self.assertTrue(success)

        # Verify user has permission
        has_perm = PluginPermissionManager.has_plugin_permission(
            self.user, 'can_create_items'
        )
        self.assertTrue(has_perm)

    def test_view_permission_required(self):
        """Test view permission decorator."""
        # Test without permission - should be denied
        response = self.client.get('/your_plugin/create/')
        self.assertEqual(response.status_code, 302)  # Redirect

        # Assign permission and test again
        PluginPermissionManager.assign_plugin_permission(
            self.user, 'can_create_items'
        )
        self.client.login(email='test@example.com', password='testpass123')

        response = self.client.get('/your_plugin/create/')
        self.assertEqual(response.status_code, 200)  # Success
```

### Integration with Django Admin

Plugin permissions automatically appear in Django admin:

1. **User Admin**: Plugin permissions show up in the user permissions list
2. **Group Admin**: Can assign plugin permissions to groups
3. **Permission Admin**: All plugin permissions are listed with clear names

### Best Practices for Plugin Permissions

1. **Use descriptive names**: Choose clear permission names like `can_create_blog_posts` instead of `blog_create`

2. **Organize by category**: Group related permissions using the `category` field

3. **Follow Django conventions**: Use `can_` prefix for permission codenames

4. **Create logical groups**: Define default permission groups that make sense for your plugin

5. **Document permissions**: Use the `description` field to explain what each permission allows

6. **Test thoroughly**: Write tests for both permission creation and enforcement

7. **Handle edge cases**: Consider what happens when permissions are missing or database is out of sync

### Security Considerations

1. **Least privilege**: Only grant minimum necessary permissions
2. **Regular audits**: Periodically review who has what permissions
3. **Group management**: Use permission groups rather than individual assignments
4. **Sensitive operations**: Mark critical permissions clearly in descriptions
5. **Logging**: Consider logging permission-sensitive operations

### Migration and Deployment

When deploying plugins with permissions:

1. **Run sync command**: Always run `python manage.py plugin_permissions --sync`
2. **Create groups**: Use `--create-groups` to set up default permission groups
3. **Assign permissions**: Set up initial permission assignments for existing users
4. **Test access**: Verify that permissions work as expected in production

## Plugin Menus

The Plugin Menu system allows plugins to define custom menu items and submenus that integrate seamlessly into the application's sidebar navigation. This enables plugins to provide intuitive navigation to their features without modifying core templates.

### Overview

Instead of manually adding menu items to templates, plugins can:
- Define menu items and submenus programmatically
- Organize menu items by sections and categories
- Control visibility based on user permissions
- Add dynamic badges and tooltips
- Create hierarchical navigation structures
- Automatically integrate with existing UI components

### Creating Plugin Menus

#### Step 1: Define Menu Provider

Create a `menus.py` file in your plugin directory:

```python
# plugins/your_plugin/menus.py
from core.plugin_menus import (
    PluginMenuProvider,
    PluginMenuItem,
    PluginSubmenu,
    plugin_menu_registry,
    PluginMenuUtils,
)

class YourPluginMenuProvider(PluginMenuProvider):
    """Menu provider for your plugin."""

    def get_plugin_id(self) -> str:
        return "your_plugin"

    def get_menu_items(self):
        """Return standalone menu items (not in submenus)."""
        return [
            PluginMenuItem(
                name="Dashboard",
                url_name="your_plugin:dashboard",
                icon="fas fa-tachometer-alt",
                order=10,
                tooltip="View plugin dashboard"
            ),
            PluginMenuItem(
                name="Quick Action",
                url_name="your_plugin:quick_action",
                icon="fas fa-bolt",
                permission="can_use_quick_actions",
                order=20,
                badge_text="New",
                badge_color="info"
            ),
        ]

    def get_submenus(self):
        """Return submenus with nested items."""
        management_submenu = PluginSubmenu(
            name="Management",
            icon="fas fa-cog",
            order=30,
            tooltip="Manage plugin settings and data"
        )

        management_submenu.add_item(PluginMenuItem(
            name="All Items",
            url_name="your_plugin:item_list",
            icon="fas fa-list",
            order=10,
        ))

        management_submenu.add_item(PluginMenuItem(
            name="Create Item",
            url_name="your_plugin:item_create",
            icon="fas fa-plus",
            permission="can_create_items",
            order=20,
        ))

        # Add separator
        management_submenu.add_item(PluginMenuUtils.create_separator())

        management_submenu.add_item(PluginMenuItem(
            name="Settings",
            url_name="your_plugin:settings",
            icon="fas fa-sliders-h",
            permission="can_configure_plugin",
            order=40,
        ))

        return [management_submenu]

    def get_menu_section(self) -> str:
        """Return the menu section where items should be placed."""
        return "plugins"  # or "content", "commerce", "reports", etc.

    def get_menu_order(self) -> int:
        """Return the order for this plugin in its section."""
        return 100

# Register the menu provider
your_menu_provider = YourPluginMenuProvider()
plugin_menu_registry.register_plugin(your_menu_provider)
```

#### Step 2: Menu Item Properties

Each `PluginMenuItem` supports these properties:

- **name**: Display name for the menu item
- **url**: Direct URL (alternative to url_name)
- **url_name**: Django URL name (preferred over url)
- **icon**: CSS icon class (e.g., "fas fa-home")
- **permission**: Required permission to view the item
- **order**: Display order (lower numbers first)
- **is_separator**: Whether this is a visual separator
- **css_class**: Additional CSS classes
- **badge_text**: Badge text to display
- **badge_color**: Bootstrap badge color
- **tooltip**: Tooltip text
- **target**: Link target (_blank, _self, etc.)
- **condition**: Custom condition function for visibility

#### Step 3: Submenu Properties

Each `PluginSubmenu` supports these properties:

- **name**: Display name for the submenu
- **icon**: CSS icon class
- **permission**: Required permission to view submenu
- **order**: Display order
- **css_class**: Additional CSS classes
- **items**: List of menu items in the submenu
- **collapsed**: Whether submenu starts collapsed
- **tooltip**: Tooltip text
- **condition**: Custom condition function

### Using Plugin Menus in Templates

#### Template Tags

Load the menu tags in your templates:

```html
{% load plugin_menu_tags %}

<!-- Get all plugin menus -->
{% get_plugin_menus as plugin_menus %}

<!-- Get menus for specific section -->
{% get_plugin_menus section="plugins" as plugin_menus %}

<!-- Render menu items -->
{% render_plugin_menu_items plugin_menus.items %}

<!-- Render a submenu -->
{% for plugin_id, submenus in plugin_menus.submenus.items %}
    {% for submenu in submenus %}
        {% render_plugin_submenu submenu %}
    {% endfor %}
{% endfor %}

<!-- Get breadcrumb navigation -->
{% get_plugin_breadcrumb as breadcrumb %}
{% render_plugin_breadcrumb breadcrumb %}
```

#### Context Processor (Global Access)

Add to your Django settings to make menus available in all templates:

```python
# settings.py
TEMPLATES = [
    {
        # ... other settings
        'OPTIONS': {
            'context_processors': [
                # ... other context processors
                'core.plugin_menu_context_processor.plugin_menu_context_processor',
            ],
        },
    },
]
```

Then use in templates:

```html
<!-- Direct access to plugin menus -->
{{ plugin_menus.items }}
{{ plugin_menus.submenus }}
{{ plugin_breadcrumb }}

<!-- Check if user has menu access -->
{% if plugin_menus.items.blog_plugin %}
    <!-- User can see blog menu items -->
{% endif %}
```

#### Manual Template Integration

For custom sidebar integration:

```html
<!-- In your sidebar template -->
{% load plugin_menu_tags %}
{% get_plugin_menus section="content" as content_menus %}

<!-- Render content section menus -->
{% if content_menus.items or content_menus.submenus %}
    <li class="nav-header">CONTENT</li>

    <!-- Render standalone menu items -->
    {% for plugin_id, items in content_menus.items.items %}
        {% for item in items %}
            <li class="nav-item">
                <a href="{{ item.get_url }}" class="nav-link">
                    <i class="{{ item.icon }}"></i>
                    <p>{{ item.name }}</p>
                </a>
            </li>
        {% endfor %}
    {% endfor %}

    <!-- Render submenus -->
    {% for plugin_id, submenus in content_menus.submenus.items %}
        {% for submenu in submenus %}
            {% render_plugin_submenu submenu %}
        {% endfor %}
    {% endfor %}
{% endif %}
```

### Menu Sections and Organization

Organize menus by logical sections:

```python
def get_menu_section(self) -> str:
    # Common sections:
    return "dashboard"    # Dashboard/overview items
    return "content"      # Content management
    return "commerce"     # E-commerce related
    return "reports"      # Analytics and reports
    return "settings"     # Configuration
    return "plugins"      # Plugin-specific items (default)
```

### Advanced Menu Features

#### Dynamic Badge Counts

```python
class BlogMenuProvider(PluginMenuProvider):
    def get_submenus(self):
        comments_submenu = PluginSubmenu(
            name="Comments",
            icon="fas fa-comments"
        )

        # Dynamic count from database
        pending_count = self._get_pending_comments_count()

        comments_submenu.add_item(PluginMenuItem(
            name="Pending Comments",
            url_name="blog:comments_pending",
            badge_text=str(pending_count) if pending_count > 0 else "",
            badge_color="warning" if pending_count > 0 else ""
        ))

        return [comments_submenu]

    def _get_pending_comments_count(self):
        # Query your models here
        from blog.models import Comment
        return Comment.objects.filter(status='pending').count()
```

#### Custom Visibility Conditions

```python
def user_has_admin_access(user, request=None):
    """Custom condition function."""
    return user.is_superuser or user.groups.filter(name='Administrators').exists()

class AdminMenuProvider(PluginMenuProvider):
    def get_menu_items(self):
        return [
            PluginMenuItem(
                name="Admin Panel",
                url_name="admin:index",
                icon="fas fa-tools",
                condition=user_has_admin_access,  # Custom condition
                order=10
            )
        ]
```

#### Menu Item Separators

```python
def get_submenus(self):
    submenu = PluginSubmenu(name="Tools", icon="fas fa-wrench")

    submenu.add_item(PluginMenuItem(
        name="Import Data",
        url_name="plugin:import"
    ))

    # Add visual separator
    submenu.add_item(PluginMenuUtils.create_separator())

    submenu.add_item(PluginMenuItem(
        name="Export Data",
        url_name="plugin:export"
    ))

    return [submenu]
```

### Real-World Examples

#### Blog Plugin Menu

```python
# plugins/blog_plugin/menus.py
class BlogMenuProvider(PluginMenuProvider):
    def get_plugin_id(self) -> str:
        return "blog_plugin"

    def get_menu_items(self):
        return [
            PluginMenuItem(
                name="Blog Dashboard",
                url_name="blog:dashboard",
                icon="fas fa-blog",
                permission="can_view_blog_analytics"
            )
        ]

    def get_submenus(self):
        # Content submenu
        content = PluginSubmenu(name="Content", icon="fas fa-edit")
        content.add_item(PluginMenuItem(
            name="All Posts", url_name="blog:posts", icon="fas fa-list"
        ))
        content.add_item(PluginMenuItem(
            name="Create Post", url_name="blog:create",
            icon="fas fa-plus", permission="can_create_blog_posts"
        ))

        # Comments submenu with dynamic badges
        comments = PluginSubmenu(name="Comments", icon="fas fa-comments")
        pending_count = Comment.objects.filter(status='pending').count()
        comments.add_item(PluginMenuItem(
            name="Pending", url_name="blog:comments_pending",
            badge_text=str(pending_count) if pending_count else "",
            badge_color="warning"
        ))

        return [content, comments]

    def get_menu_section(self) -> str:
        return "content"
```

#### E-commerce Plugin Menu

```python
# plugins/shop_plugin/menus.py
class ShopMenuProvider(PluginMenuProvider):
    def get_plugin_id(self) -> str:
        return "shop_plugin"

    def get_menu_items(self):
        return [
            PluginMenuItem(
                name="Shop Dashboard", url_name="shop:dashboard",
                icon="fas fa-store", permission="can_view_shop_reports"
            ),
            PluginMenuItem(
                name="Quick Sale", url_name="shop:pos",
                icon="fas fa-cash-register", permission="can_process_orders",
                badge_text="POS", badge_color="success"
            )
        ]

    def get_submenus(self):
        # Products submenu
        products = PluginSubmenu(name="Products", icon="fas fa-box")
        products.add_item(PluginMenuItem(
            name="All Products", url_name="shop:products"
        ))
        products.add_item(PluginMenuItem(
            name="Add Product", url_name="shop:product_create",
            permission="can_manage_products"
        ))
        products.add_item(PluginMenuUtils.create_separator())
        products.add_item(PluginMenuItem(
            name="Inventory", url_name="shop:inventory",
            badge_text="Low Stock", badge_color="warning"
        ))

        # Orders submenu
        orders = PluginSubmenu(name="Orders", icon="fas fa-shopping-cart")
        orders.add_item(PluginMenuItem(
            name="Pending", url_name="shop:orders_pending",
            badge_text="12", badge_color="warning"
        ))
        orders.add_item(PluginMenuItem(
            name="Processing", url_name="shop:orders_processing"
        ))

        return [products, orders]

    def get_menu_section(self) -> str:
        return "commerce"
```

### Auto-Discovery

Plugin menus are automatically discovered when the application starts. The system:

1. Scans all plugin directories for `menus.py` files
2. Imports and registers menu providers
3. Makes menus available in templates
4. Reports discovery results in console

You'll see output like:

```
🔗 Registered menu plugin: blog_plugin with 1 items and 3 submenus
🔗 Registered menu plugin: shop_plugin with 2 items and 4 submenus
🔗 Discovered 2 menu plugins
```

### Template Integration Patterns

#### AdminLTE Sidebar Integration

```html
<!-- For AdminLTE theme -->
{% load plugin_menu_tags %}
<nav class="mt-2">
    <ul class="nav nav-pills nav-sidebar flex-column">
        {% get_plugin_menus section="dashboard" as dashboard_menus %}

        <!-- Dashboard section -->
        {% if dashboard_menus.items or dashboard_menus.submenus %}
            <li class="nav-header">DASHBOARD</li>
            {% render_plugin_menu_items dashboard_menus.items %}

            {% for plugin_id, submenus in dashboard_menus.submenus.items %}
                {% for submenu in submenus %}
                    {% render_plugin_submenu submenu %}
                {% endfor %}
            {% endfor %}
        {% endif %}

        <!-- Repeat for other sections -->
        {% get_plugin_menus section="content" as content_menus %}
        {% if content_menus.items or content_menus.submenus %}
            <li class="nav-header">CONTENT</li>
            <!-- ... render content menus ... -->
        {% endif %}
    </ul>
</nav>
```

#### Bootstrap Navbar Integration

```html
<!-- For Bootstrap navbar -->
{% load plugin_menu_tags %}
<nav class="navbar navbar-expand-lg">
    <div class="navbar-nav">
        {% get_plugin_menu_items section="main" as main_items %}
        {% for plugin_id, items in main_items.items %}
            {% for item in items %}
                <a class="nav-link" href="{{ item.get_url }}">
                    <i class="{{ item.icon }}"></i> {{ item.name }}
                </a>
            {% endfor %}
        {% endfor %}

        <!-- Dropdown for submenus -->
        {% get_plugin_submenus section="main" as main_submenus %}
        {% for plugin_id, submenus in main_submenus.items %}
            {% for submenu in submenus %}
                <div class="nav-item dropdown">
                    <a class="nav-link dropdown-toggle" data-toggle="dropdown">
                        <i class="{{ submenu.icon }}"></i> {{ submenu.name }}
                    </a>
                    <div class="dropdown-menu">
                        {% for item in submenu.items %}
                            <a class="dropdown-item" href="{{ item.get_url }}">
                                {{ item.name }}
                            </a>
                        {% endfor %}
                    </div>
                </div>
            {% endfor %}
        {% endfor %}
    </div>
</nav>
```

### Best Practices for Plugin Menus

1. **Use descriptive names**: Choose clear, concise menu item names
2. **Organize logically**: Group related items in submenus
3. **Set appropriate permissions**: Control access based on user capabilities
4. **Use meaningful icons**: Choose icons that represent the functionality
5. **Provide tooltips**: Add helpful tooltips for complex features
6. **Order thoughtfully**: Set logical order values for menu flow
7. **Use badges wisely**: Show important counts or status indicators
8. **Consider mobile**: Ensure menu items work on mobile devices

### Performance Considerations

1. **Cache dynamic counts**: Cache badge counts to avoid repeated database queries
2. **Lazy load submenus**: Use collapsed submenus for better initial load times
3. **Optimize permissions**: Cache permission checks where possible
4. **Minimize database queries**: Batch queries for menu data

### Testing Plugin Menus

```python
# In your plugin tests
from django.test import TestCase, RequestFactory
from core.plugin_menus import plugin_menu_registry
from core.models.auth import User

class PluginMenuTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            email='test@example.com',
            password='testpass'
        )
        self.factory = RequestFactory()

    def test_menu_items_visibility(self):
        """Test menu items are visible to authorized users."""
        request = self.factory.get('/')
        request.user = self.user

        # Get menus for user
        menu_items = plugin_menu_registry.get_all_menu_items(self.user, request)

        # Check expected menus are present
        self.assertIn('your_plugin', menu_items)
        self.assertTrue(len(menu_items['your_plugin']) > 0)

    def test_permission_based_visibility(self):
        """Test menu items respect permission requirements."""
        from core.plugin_permissions import PluginPermissionManager

        request = self.factory.get('/')
        request.user = self.user

        # Initially user shouldn't see restricted items
        menu_items = plugin_menu_registry.get_all_menu_items(self.user, request)
        restricted_items = [item for item in menu_items.get('your_plugin', [])
                          if item.permission]
        self.assertEqual(len(restricted_items), 0)

        # Grant permission
        PluginPermissionManager.assign_plugin_permission(
            self.user, 'can_create_items'
        )

        # Now user should see restricted items
        menu_items = plugin_menu_registry.get_all_menu_items(self.user, request)
        restricted_items = [item for item in menu_items.get('your_plugin', [])
                          if item.permission == 'can_create_items']
        self.assertTrue(len(restricted_items) > 0)
```

## Plugin Widgets

The Plugin Widget system allows plugins to create reusable UI components that can be rendered in predefined zones throughout the application. This enables plugins to display their content on dashboards, CRUD pages, and other areas without modifying core templates.

### Overview

Instead of hardcoding widget content into templates, plugins can:
- Register widgets for specific zones (dashboard, list pages, form pages)
- Control widget visibility based on permissions and conditions
- Create responsive layouts with Bootstrap grid classes
- Process dynamic context data for each widget
- Organize widgets by priority and layout constraints

### Widget Zone System

The system provides predefined zones for different page types:

#### Dashboard Zones
- **dashboard_top**: Full width section at top of dashboard
- **dashboard_medium**: Two-column layout (40%/60% split, max 2 widgets)
- **dashboard_bottom**: Full width section below main content
- **dashboard_footer**: Full width section at bottom

#### CRUD Page Zones
For any app and model combination, these zones are available:
- **{app_name}_{model_name}_list_top**: Top of list pages
- **{app_name}_{model_name}_list_bottom**: Bottom of list pages
- **{app_name}_{model_name}_list_footer**: Footer of list pages
- **{app_name}_{model_name}_form_top**: Top of form pages
- **{app_name}_{model_name}_form_bottom**: Bottom of form pages
- **{app_name}_{model_name}_form_footer**: Footer of form pages

### Creating Plugin Widgets

#### Step 1: Define Widget Components

Create a `widgets.py` file in your plugin directory:

```python
# plugins/your_plugin/widgets.py
from core.plugin_widgets import plugin_widget_registry, Widget

def get_stats_context(request):
    """Context processor for statistics widget."""
    return {
        'total_items': 42,
        'active_items': 35,
        'last_updated': '2 hours ago'
    }

def check_admin_access(request):
    """Condition function to check if widget should show."""
    return request.user.is_superuser if request and request.user else False

# Register widgets
plugin_widget_registry.register_widget(
    widget=Widget(
        name="item_statistics",
        plugin_id="your_plugin",
        template="your_plugin/widgets/stats.html",
        context_processor=get_stats_context,
        zone_compatible=["dashboard_top", "dashboard_medium"],
        priority=5,  # Lower numbers render first
        permissions=["can_view_statistics"],
        conditions=check_admin_access,
        css_classes="stats-widget",
        width_classes="col-md-6"  # Bootstrap grid classes
    ),
    zones=["dashboard_top"]
)

print("🎛️ Your plugin widgets registered successfully")
```

#### Step 2: Widget Properties

Each `Widget` supports these properties:

- **name**: Unique widget identifier within the plugin
- **plugin_id**: ID of the plugin registering the widget
- **template**: Path to the widget template
- **context_processor**: Function to provide dynamic context data
- **zone_compatible**: List of zones this widget can be placed in
- **priority**: Rendering order (lower numbers first)
- **permissions**: Required permissions to view the widget
- **conditions**: Function to check if widget should be shown
- **css_classes**: Additional CSS classes for styling
- **width_classes**: Bootstrap grid classes for responsive layout

#### Step 3: Widget Templates

Create templates in your plugin's templates directory:

```html
<!-- plugins/your_plugin/templates/your_plugin/widgets/stats.html -->
{% load i18n %}
<div class="card shadow-sm">
    <div class="card-header bg-primary text-white">
        <h6 class="card-title mb-0">
            <i class="fas fa-chart-bar mr-2"></i>
            {% trans "Statistics" %}
        </h6>
    </div>
    <div class="card-body">
        <div class="row text-center">
            <div class="col-6">
                <h3 class="text-success">{{ total_items }}</h3>
                <small class="text-muted">{% trans "Total Items" %}</small>
            </div>
            <div class="col-6">
                <h3 class="text-primary">{{ active_items }}</h3>
                <small class="text-muted">{% trans "Active Items" %}</small>
            </div>
        </div>
        <hr>
        <small class="text-muted">
            <i class="fas fa-clock mr-1"></i>
            {% trans "Updated" %} {{ last_updated }}
        </small>
    </div>
</div>
```

### Using Widgets in Templates

#### Template Tags

Load the widget tags in your templates:

```html
{% load plugin_widget_tags %}

<!-- Render widgets in a zone -->
{% render_widget_zone 'dashboard_top' %}

<!-- Render with app/model context -->
{% render_widget_zone 'app_model_list_top' app_name='patients' model_name='patient' %}

<!-- Use wrapper template for additional styling -->
{% widget_zone 'dashboard_medium' css_class='custom-zone' %}

<!-- Check if zone has widgets -->
{% has_zone_widgets 'dashboard_bottom' as has_widgets %}
{% if has_widgets %}
    <div class="widget-section">
        {% render_widget_zone 'dashboard_bottom' %}
    </div>
{% endif %}
```

#### Context Integration

For CRUD pages, add widget zones to your templates:

```html
<!-- In your list template (e.g., patient_list.html) -->
{% extends "base.html" %}
{% load plugin_widget_tags %}

{% block content %}
{# List Top Zone #}
{% render_widget_zone 'patients_patient_list_top' app_name='patients' model_name='patient' %}

<div class="row">
    <div class="col-12">
        <h1>Patient List</h1>
        <!-- Your list content here -->
    </div>
</div>

{# List Bottom Zone #}
{% render_widget_zone 'patients_patient_list_bottom' app_name='patients' model_name='patient' %}
{% endblock %}

{# List Footer Zone #}
{% render_widget_zone 'patients_patient_list_footer' app_name='patients' model_name='patient' %}
```

#### Dashboard Integration

The dashboard zones are automatically included in the main dashboard template:

```html
<!-- Dashboard layout with widget zones -->
<div class="dashboard">
    <!-- dashboard_top: Full width widgets -->

    <!-- Existing dashboard content -->
    <div class="row">
        <div class="col-lg-10">
            <!-- dashboard_medium: 40%/60% split widgets -->

            <!-- Original dashboard cards -->
            <div class="card-columns">
                <!-- Existing cards -->
            </div>
        </div>
    </div>

    <!-- dashboard_bottom: Full width widgets -->
    <!-- dashboard_footer: Full width widgets -->
</div>
```

### Zone Layouts and Responsive Design

#### Layout Types

Different zone types support different layout patterns:

```python
# Full width zones
"dashboard_top", "dashboard_bottom", "dashboard_footer"
# Single widgets span full width

# Split zones
"dashboard_medium"
# First widget: col-md-4 (40%)
# Second widget: col-md-8 (60%)
# Max 2 widgets

# Custom width zones
# Use width_classes to control individual widget sizing
Widget(width_classes="col-md-3")  # 25% width
Widget(width_classes="col-lg-6 col-md-12")  # Responsive sizing
```

#### Responsive Considerations

```python
# Define responsive widgets
Widget(
    name="mobile_friendly_stats",
    template="plugin/widgets/stats_responsive.html",
    width_classes="col-xl-3 col-lg-4 col-md-6 col-sm-12",
    css_classes="mb-3"
)
```

```html
<!-- Responsive widget template -->
<div class="card h-100">
    <div class="card-body">
        <div class="d-flex justify-content-between">
            <div class="d-none d-md-block">
                <h5>{{ title }}</h5>
            </div>
            <div class="d-md-none">
                <h6>{{ title }}</h6>
            </div>
            <i class="{{ icon }} fa-2x text-primary"></i>
        </div>
    </div>
</div>
```

### Advanced Widget Features

#### Dynamic Context Processing

```python
def get_dynamic_stats_context(request):
    """Dynamic context based on user and organization."""
    if not request or not request.user:
        return {}

    # Get user's organization
    org = getattr(request.user, 'organization', None)
    if not org:
        return {}

    # Query data specific to this organization
    stats = {
        'total_users': org.users.count(),
        'active_sessions': get_active_sessions_count(org),
        'recent_activity': get_recent_activity(org, days=7)
    }

    # Add time-sensitive data
    from django.utils import timezone
    stats['current_time'] = timezone.now()
    stats['business_hours'] = is_business_hours()

    return stats

def get_recent_activity(org, days=7):
    """Get recent activity for organization."""
    from django.utils import timezone
    from datetime import timedelta

    cutoff = timezone.now() - timedelta(days=days)
    # Query your activity models here
    return []
```

#### Conditional Widget Rendering

```python
def show_admin_widget(request):
    """Show widget only to administrators during business hours."""
    if not request or not request.user:
        return False

    # Check user permissions
    if not request.user.is_staff:
        return False

    # Check time-based conditions
    from datetime import datetime
    current_hour = datetime.now().hour
    if current_hour < 9 or current_hour > 17:  # Outside 9-5
        return False

    return True

# Register conditional widget
plugin_widget_registry.register_widget(
    widget=Widget(
        name="admin_alerts",
        template="plugin/widgets/admin_alerts.html",
        conditions=show_admin_widget,
        priority=1  # High priority
    ),
    zones=["dashboard_top"]
)
```

#### Multi-Zone Widget Registration

```python
# Register the same widget in multiple zones
zones_for_stats = [
    "dashboard_top",
    "patients_patient_list_top",
    "staff_staff_list_top"
]

plugin_widget_registry.register_widget(
    widget=Widget(
        name="universal_stats",
        template="plugin/widgets/universal_stats.html",
        context_processor=get_universal_stats,
        zone_compatible=zones_for_stats,
        priority=10
    ),
    zones=zones_for_stats
)
```

### Real-World Examples

#### Organization Statistics Widget

```python
# Blog plugin providing org stats for dashboard
def get_org_stats_context(request):
    """Get organization-wide statistics."""
    if not request or not hasattr(request, 'user'):
        return {}

    org = getattr(request.user, 'organization', None)
    if not org:
        return {}

    return {
        'total_users': org.users.count(),
        'active_sessions': 12,  # Would query session store
        'departments': org.departments.count(),
        'pending_tasks': 23,   # Would query task system
        'overdue_items': 5     # Would query your models
    }

plugin_widget_registry.register_widget(
    widget=Widget(
        name="organization_statistics",
        plugin_id="blog_plugin",
        template="blog_plugin/widgets/organization_stats.html",
        context_processor=get_org_stats_context,
        priority=1,  # Show first
        permissions=["core.view_organization_stats"],
        css_classes="org-stats-widget"
    ),
    zones=["dashboard_top"]
)
```

#### Visit Statistics Widget

```python
# Shop plugin providing visit/sales stats
def get_visit_stats_context(request):
    """Get visit statistics for last week."""
    return {
        'total_visits': 1234,
        'unique_visitors': 856,
        'bounce_rate': 23.4,
        'avg_session_time': '4m 32s',
        'top_pages': [
            {'url': '/shop/', 'views': 234},
            {'url': '/products/', 'views': 187},
            {'url': '/about/', 'views': 98}
        ]
    }

plugin_widget_registry.register_widget(
    widget=Widget(
        name="visit_statistics",
        plugin_id="shop_plugin",
        template="shop_plugin/widgets/visit_stats.html",
        context_processor=get_visit_stats_context,
        width_classes="col-md-8",  # Takes 60% in medium zone
        priority=6,
        css_classes="visit-stats-widget"
    ),
    zones=["dashboard_medium"]
)
```

#### Caregiver Log Statistics Widget

```python
# Healthcare plugin for caregiver statistics
def get_caregiver_log_stats_context(request):
    """Get caregiver log statistics for last month."""
    from datetime import datetime, timedelta

    last_month = datetime.now() - timedelta(days=30)

    return {
        'total_logs': 145,
        'completed_visits': 138,
        'missed_visits': 7,
        'average_duration': '2.5 hours',
        'top_caregivers': [
            {'name': 'Jane Smith', 'logs': 28},
            {'name': 'John Doe', 'logs': 24},
            {'name': 'Mary Johnson', 'logs': 22}
        ],
        'period': 'Last 30 days'
    }

plugin_widget_registry.register_widget(
    widget=Widget(
        name="caregiver_log_statistics",
        plugin_id="healthcare_plugin",
        template="healthcare/widgets/caregiver_stats.html",
        context_processor=get_caregiver_log_stats_context,
        priority=3,
        permissions=["healthcare.view_caregiver_logs"],
        css_classes="caregiver-stats-widget"
    ),
    zones=["dashboard_footer"]
)
```

### Auto-Discovery

Plugin widgets are automatically discovered when the application starts. The system:

1. Scans all plugin directories for `widgets.py` files
2. Imports widget registration code
3. Makes widgets available in template zones
4. Reports discovery results in console

You'll see output like:

```
🎛️ Blog plugin widgets registered successfully
🎛️ Shop plugin widgets registered successfully
🎛️ Discovered 2 widget plugins
```

### Template Tag Reference

#### Core Tags

```html
{% load plugin_widget_tags %}

<!-- Basic zone rendering -->
{% render_widget_zone 'zone_name' %}

<!-- Zone with app/model context -->
{% render_widget_zone 'zone_pattern' app_name='app' model_name='model' %}

<!-- Zone with additional context -->
{% render_widget_zone 'zone_name' extra_var=value another_var=value2 %}

<!-- Wrapper template with styling -->
{% widget_zone 'zone_name' css_class='custom-styling' %}
```

#### Utility Tags

```html
<!-- Check if zone has widgets -->
{% has_zone_widgets 'zone_name' as has_widgets %}
{% if has_widgets %}
    <div class="widget-section">
        {% render_widget_zone 'zone_name' %}
    </div>
{% endif %}

<!-- Get widget count -->
{{ 'dashboard_top'|widget_count }}

<!-- Get zone information -->
{% get_zone_info 'dashboard_top' as zone_info %}
{{ zone_info.description }}

<!-- Render all CRUD zones at once -->
{% render_crud_zones 'patients' 'patient' 'list' %}

<!-- Debug widget information (DEBUG mode only) -->
{% debug_zone_widgets 'dashboard_top' %}
```

### Testing Plugin Widgets

```python
# In your plugin tests
from django.test import TestCase, RequestFactory
from core.plugin_widgets import plugin_widget_registry
from core.models.auth import User

class PluginWidgetTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            email='test@example.com',
            password='testpass'
        )
        self.factory = RequestFactory()

    def test_widget_registration(self):
        """Test that widgets are registered correctly."""
        widgets = plugin_widget_registry.get_zone_widgets('dashboard_top')
        self.assertTrue(len(widgets) > 0)

        # Check specific widget
        widget_names = [w.name for w in widgets]
        self.assertIn('organization_statistics', widget_names)

    def test_widget_rendering(self):
        """Test widget rendering with context."""
        request = self.factory.get('/')
        request.user = self.user

        rendered = plugin_widget_registry.render_zone(
            'dashboard_top',
            request=request
        )

        self.assertIn('Organization Statistics', rendered)
        self.assertIn('card', rendered)  # Should contain card HTML

    def test_permission_filtering(self):
        """Test widgets respect permission requirements."""
        # Create request without permissions
        request = self.factory.get('/')
        request.user = self.user

        widgets = plugin_widget_registry.get_zone_widgets(
            'dashboard_top',
            request=request
        )

        # Should filter out widgets requiring permissions
        restricted_widgets = [w for w in widgets if w.permissions]
        # Verify filtering logic based on your implementation

    def test_context_processor(self):
        """Test widget context processors."""
        from plugins.blog_plugin.widgets import get_org_stats_context

        request = self.factory.get('/')
        request.user = self.user

        context = get_org_stats_context(request)
        self.assertIn('total_users', context)
        self.assertIsInstance(context['total_users'], int)
```

### Performance Considerations

1. **Cache context data**: Use Django's cache framework for expensive queries
2. **Lazy loading**: Only load widget data when zones are rendered
3. **Permission caching**: Cache permission checks where possible
4. **Template caching**: Use template fragment caching for static widgets
5. **Database optimization**: Optimize context processor queries

```python
# Example of cached context processor
from django.core.cache import cache

def get_cached_stats_context(request):
    """Context processor with caching."""
    if not request or not request.user:
        return {}

    org_id = getattr(request.user, 'organization_id', None)
    if not org_id:
        return {}

    cache_key = f'org_stats_{org_id}'
    stats = cache.get(cache_key)

    if stats is None:
        # Expensive database queries here
        stats = {
            'total_users': calculate_user_count(org_id),
            'active_sessions': get_session_count(org_id),
            # ... other expensive calculations
        }
        # Cache for 5 minutes
        cache.set(cache_key, stats, 300)

    return stats
```

### Best Practices for Plugin Widgets

1. **Responsive design**: Use Bootstrap grid classes for mobile compatibility
2. **Performance**: Cache expensive data and optimize database queries
3. **Permissions**: Always check user permissions before showing sensitive data
4. **Error handling**: Handle missing data gracefully in templates
5. **Accessibility**: Use proper ARIA labels and semantic HTML
6. **Consistent styling**: Follow the application's design system
7. **Documentation**: Document widget purpose and context requirements
8. **Testing**: Write tests for widget registration, rendering, and permissions

### Example 2: Plugin with Dependencies

```python
# plugins/advanced_shop/__init__.py
from plugin_registry import PluginMetadata

PLUGIN_METADATA = PluginMetadata(
    name="Advanced Shop",
    version="2.0.0",
    description="Advanced e-commerce features",
    dependencies=["shop_plugin"],  # Requires shop_plugin
    django_apps=["plugins.advanced_shop"],
)
```

## Best Practices

### 1. Plugin Naming
- Use descriptive, lowercase names with underscores
- Avoid conflicts with existing Django apps
- Example: `user_management`, `payment_gateway`, `analytics_dashboard`

### 2. Versioning
- Follow semantic versioning (MAJOR.MINOR.PATCH)
- Update version when making changes
- Document breaking changes in MAJOR versions

### 3. Dependencies
- Keep dependencies minimal
- Document required plugins clearly
- Test with and without optional dependencies

### 4. Database Models
- Use appropriate field types and constraints
- Add helpful `__str__` methods
- Include proper Meta options (ordering, verbose names)

### 5. Templates and Static Files
- Organize templates in plugin-specific directories
- Use namespaced static file paths
- Follow Django template best practices

### 6. Error Handling
- Handle missing dependencies gracefully
- Provide meaningful error messages
- Log important events and errors

### 7. Testing
- Write tests for your plugin functionality
- Test plugin loading and unloading
- Test with different dependency configurations

## Troubleshooting

### Common Issues

#### Plugin Not Loading
1. Check that `__init__.py` exists in plugin directory
2. Verify plugin metadata is correctly defined
3. Check for import errors in plugin code
4. Review Django logs for error messages

#### Missing Dependencies
```bash
python manage.py plugin_info
```
Look for "Dependency Issues" section in output.

#### URL Not Working
1. Ensure `urls.py` exists in plugin directory
2. Check URL patterns are correctly defined
3. Verify views are properly imported
4. Check that plugin is enabled

#### Database Errors
1. Run migrations: `python manage.py makemigrations your_plugin`
2. Apply migrations: `python manage.py migrate`
3. Check model definitions for errors

### Debug Mode

Enable debug output by checking Django logs or adding print statements in plugin code.

### Getting Help

1. Check plugin metadata with `python manage.py plugin_info --plugin your_plugin`
2. Review Django error logs
3. Verify plugin structure matches documentation
4. Test with minimal plugin configuration

## Advanced Topics

### Custom Plugin Loaders

You can extend the plugin system by creating custom loaders:

```python
from plugin_registry import plugin_registry

def load_external_plugins():
    """Load plugins from external sources."""
    # Custom loading logic here
    pass
```

### Plugin Hooks

Plugins can define hooks for other plugins to extend:

```python
# In your plugin
def register_hooks():
    """Register plugin hooks."""
    from django.dispatch import Signal

    plugin_loaded = Signal()
    plugin_loaded.send(sender=__name__)
```

### Configuration Management

Store plugin configuration in Django settings:

```python
# settings.py
PLUGIN_SETTINGS = {
    'shop_plugin': {
        'currency': 'USD',
        'tax_rate': 0.08,
    }
}
```

## Contributing

When contributing to the plugins system:

1. Follow existing code style and patterns
2. Add tests for new functionality
3. Update documentation
4. Ensure backward compatibility
5. Test with existing plugins

## License

This plugins system is part of the Ritiko project and follows the same license terms.
