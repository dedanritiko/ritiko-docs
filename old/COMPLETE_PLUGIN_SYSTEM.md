# Complete Plugin System Guide

The `PluginAppConfig` class provides a unified, easy-to-use system for registering **menus**, **permissions**, and **organization settings** for Django plugins. This comprehensive guide covers all three registration systems with practical examples.

## Table of Contents

- [Quick Start](#quick-start)
- [Menu Registration](#menu-registration)
- [Permission Registration](#permission-registration)
- [Organization Settings Registration](#organization-settings-registration)
- [Registration Approaches](#registration-approaches)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)
- [Migration Guide](#migration-guide)
- [Troubleshooting](#troubleshooting)

## Quick Start

The simplest way to create a comprehensive plugin:

```python
from django.db import models
from core.plugin_app_config import PluginAppConfig

class MyPluginConfig(PluginAppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "plugins.my_plugin"
    verbose_name = "My Plugin"

    def setup_menu(self):
        # Register menu
        self.register_menu(
            alias="my_dashboard",
            name="My Dashboard",
            url_name="my_plugin:dashboard",
            icon="fas fa-home",
        )

        self.register_submenu(
            parent_alias="my_dashboard",
            name="Items",
            url_name="my_plugin:items",
        )

    def setup_permission(self):
        # Register permissions
        self.register_permission(
            codename="manage_items",
            name="Can manage items",
            description="Permission to create, edit, and delete items",
        )

    def setup_setting(self):
        # Register organization settings
        self.register_organization_setting(
            name="enable_feature",
            field_type=models.BooleanField,
            default_value=True,
            verbose_name="Enable Feature",
            help_text="Enable the main feature for this organization",
        )
```

## Menu Registration

### Method-Based Menu Registration (Recommended)

```python
def setup_menu(self):
    # Main menu
    self.register_menu(
        alias="content",
        name="Content",
        url_name="my_plugin:content_index",
        icon="fas fa-edit",
        order=20,
        section="main",
    )

    # Submenus
    self.register_submenu(
        parent_alias="content",
        name="Posts",
        url_name="my_plugin:posts",
        icon="fas fa-file-alt",
        order=10,
        permission="can_manage_posts",
    )
```

### List-Based Menu Registration

```python
def __init__(self, app_name, app_module):
    super().__init__(app_name, app_module)

    self.main_menus = [
        {
            "alias": "content",
            "name": "Content",
            "url_name": "my_plugin:content_index",
            "icon": "fas fa-edit",
            "order": 20,
        }
    ]

    self.sub_menus = [
        {
            "parent_alias": "content",
            "name": "Posts",
            "url_name": "my_plugin:posts",
            "order": 10,
        }
    ]
```

### Menu Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `alias` | str | Unique menu identifier | Required |
| `name` | str | Display name | Required |
| `url_name` | str | Django URL name | - |
| `url` | str | Direct URL | - |
| `icon` | str | CSS icon class | "fas fa-link" |
| `order` | int | Display order | 100 |
| `section` | str | Menu section | "main" |
| `permission` | str | Required permission | - |

## Permission Registration

### Method-Based Permission Registration (Recommended)

```python
def setup_permission(self):
    # Content permissions
    self.register_permission(
        codename="manage_posts",
        name="Can manage posts",
        description="Permission to create, edit, and delete posts",
        category="content",
    )

    # Admin permissions
    self.register_permission(
        codename="admin_settings",
        name="Can access admin settings",
        description="Permission to modify plugin administrative settings",
        category="admin",
    )
```

### List-Based Permission Registration

```python
def __init__(self, app_name, app_module):
    super().__init__(app_name, app_module)

    self.permissions = [
        {
            "codename": "manage_posts",
            "name": "Can manage posts",
            "description": "Permission to create, edit, and delete posts",
            "category": "content",
        },
        {
            "codename": "view_analytics",
            "name": "Can view analytics",
            "description": "Permission to view reports and analytics",
            "category": "analytics",
        }
    ]
```

### Permission Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `codename` | str | Permission code (auto-prefixed with "can_") | Required |
| `name` | str | Display name | Required |
| `description` | str | Permission description | "" |
| `category` | str | Permission category | "general" |
| `content_type_model` | str | Django model for permission | "organization" |
| `content_type_app` | str | Django app for permission | "core" |

## Organization Settings Registration

### Method-Based Settings Registration (Recommended)

```python
def setup_setting(self):
    # Boolean setting
    self.register_organization_setting(
        name="enable_notifications",
        field_type=models.BooleanField,
        default_value=True,
        verbose_name="Enable Notifications",
        help_text="Send email notifications for important events",
        form_section="notifications",
    )

    # Choice setting
    self.register_organization_setting(
        name="theme",
        field_type=models.CharField,
        default_value="light",
        verbose_name="Theme",
        help_text="Visual theme for the interface",
        choices=[
            ("light", "Light"),
            ("dark", "Dark"),
            ("auto", "Auto"),
        ],
        form_section="appearance",
    )

    # Numeric setting
    self.register_organization_setting(
        name="max_items",
        field_type=models.PositiveIntegerField,
        default_value=100,
        verbose_name="Maximum Items",
        help_text="Maximum number of items allowed",
        form_section="limits",
    )
```

### List-Based Settings Registration

```python
def __init__(self, app_name, app_module):
    super().__init__(app_name, app_module)

    self.organization_settings = [
        {
            "name": "enable_feature",
            "field_type": models.BooleanField,
            "default_value": False,
            "verbose_name": "Enable Feature",
            "help_text": "Enable the main feature",
            "form_section": "general",
        },
        {
            "name": "api_key",
            "field_type": models.CharField,
            "default_value": "",
            "verbose_name": "API Key",
            "help_text": "Third-party service API key",
            "form_section": "integrations",
            "admin_only": True,
        }
    ]
```

### Setting Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `name` | str | Setting name | Required |
| `field_type` | Type[models.Field] | Django field type | Required |
| `default_value` | Any | Default value | None |
| `verbose_name` | str | Display name | Auto-generated |
| `help_text` | str | Help description | "" |
| `required` | bool | Whether required | False |
| `choices` | List[tuple] | Choice options | None |
| `form_section` | str | Form section grouping | "general" |
| `admin_only` | bool | Superuser only | False |

## Registration Approaches

### 1. Method-Based Approach (Recommended)

**Best for:** Dynamic registration, conditional logic, complex setups.

```python
class MyPluginConfig(PluginAppConfig):
    def setup_menu(self):
        self.register_menu(alias="main", name="Main")

    def setup_permission(self):
        self.register_permission(codename="access", name="Can access")

    def setup_setting(self):
        self.register_organization_setting(
            name="enabled",
            field_type=models.BooleanField
        )
```

### 2. List-Based Approach

**Best for:** Static configurations, simple setups, declarative style.

```python
class MyPluginConfig(PluginAppConfig):
    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)

        self.main_menus = [{"alias": "main", "name": "Main"}]
        self.permissions = [{"codename": "access", "name": "Can access"}]
        self.organization_settings = [{
            "name": "enabled",
            "field_type": models.BooleanField
        }]
```

### 3. Hybrid Approach

**Best for:** Mixed static and dynamic requirements.

```python
class MyPluginConfig(PluginAppConfig):
    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)

        # Static configurations
        self.main_menus = [{"alias": "main", "name": "Main"}]

    def setup_permission(self):
        # Dynamic permissions based on features
        if getattr(settings, 'ADVANCED_FEATURES', False):
            self.register_permission(
                codename="advanced_access",
                name="Can access advanced features"
            )
```

## Real-World Examples

### Blog Plugin (Method-Based)

```python
class BlogPluginConfig(PluginAppConfig):
    name = "plugins.blog_plugin"
    verbose_name = "Blog Plugin"

    def setup_menu(self):
        # Main content menu
        self.register_menu(
            alias="content",
            name="Content",
            url_name="content:index",
            icon="fas fa-edit",
            order=20,
        )

        # Blog submenus
        self.register_submenu(
            parent_alias="content",
            name="Blog Posts",
            url_name="blog_plugin:post_list",
            icon="fas fa-blog",
            permission="can_manage_blog_posts",
        )

        self.register_submenu(
            parent_alias="content",
            name="Categories",
            url_name="blog_plugin:category_list",
            icon="fas fa-tags",
            permission="can_manage_blog_categories",
        )

    def setup_permission(self):
        # Content management
        self.register_permission(
            codename="manage_blog_posts",
            name="Can manage blog posts",
            description="Create, edit, and delete blog posts",
            category="content",
        )

        self.register_permission(
            codename="manage_blog_categories",
            name="Can manage blog categories",
            description="Manage blog post categories",
            category="content",
        )

        # Moderation
        self.register_permission(
            codename="moderate_blog_comments",
            name="Can moderate blog comments",
            description="Approve and manage blog comments",
            category="moderation",
        )

    def setup_setting(self):
        # Core settings
        self.register_organization_setting(
            name="enable_blog",
            field_type=models.BooleanField,
            default_value=False,
            verbose_name="Enable Blog System",
            form_section="blog",
        )

        self.register_organization_setting(
            name="posts_per_page",
            field_type=models.PositiveIntegerField,
            default_value=10,
            verbose_name="Posts Per Page",
            help_text="Number of posts to display per page",
            form_section="blog",
        )

        self.register_organization_setting(
            name="blog_theme",
            field_type=models.CharField,
            default_value="default",
            verbose_name="Blog Theme",
            choices=[
                ("default", "Default"),
                ("modern", "Modern"),
                ("classic", "Classic"),
            ],
            form_section="blog",
        )
```

### E-commerce Plugin (List-Based)

```python
class ShopPluginConfig(PluginAppConfig):
    name = "plugins.shop_plugin"
    verbose_name = "Shop Plugin"

    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)

        # Define permissions
        self.permissions = [
            {
                "codename": "manage_products",
                "name": "Can manage products",
                "description": "Create, edit, and delete products",
                "category": "inventory",
            },
            {
                "codename": "view_orders",
                "name": "Can view orders",
                "description": "View customer orders",
                "category": "sales",
            },
            {
                "codename": "manage_inventory",
                "name": "Can manage inventory",
                "description": "Manage stock and inventory",
                "category": "inventory",
            },
        ]

        # Define settings
        self.organization_settings = [
            {
                "name": "enable_shop",
                "field_type": models.BooleanField,
                "default_value": False,
                "verbose_name": "Enable Shop",
                "form_section": "shop",
            },
            {
                "name": "currency",
                "field_type": models.CharField,
                "default_value": "USD",
                "verbose_name": "Currency",
                "choices": [
                    ("USD", "US Dollar"),
                    ("EUR", "Euro"),
                    ("GBP", "British Pound"),
                ],
                "form_section": "shop",
            },
            {
                "name": "tax_rate",
                "field_type": models.DecimalField,
                "default_value": 0.08,
                "verbose_name": "Tax Rate",
                "help_text": "Tax rate as decimal (0.08 = 8%)",
                "form_section": "shop",
            },
        ]

    def setup_menu(self):
        # Dynamic menu registration
        self.register_menu(
            alias="shop",
            name="Shop",
            url_name="shop_plugin:dashboard",
            icon="fas fa-shopping-cart",
            order=25,
        )

        # Shop submenus
        shop_menus = [
            ("Products", "shop_plugin:product_list", "fas fa-box", "can_manage_products"),
            ("Orders", "shop_plugin:order_list", "fas fa-receipt", "can_view_orders"),
        ]

        for name, url_name, icon, permission in shop_menus:
            self.register_submenu(
                parent_alias="shop",
                name=name,
                url_name=url_name,
                icon=icon,
                permission=permission,
            )
```

### Multi-Feature Plugin (Hybrid)

```python
class AdvancedPluginConfig(PluginAppConfig):
    name = "plugins.advanced_plugin"
    verbose_name = "Advanced Plugin"

    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)

        # Static core permissions
        self.permissions = [
            {
                "codename": "basic_access",
                "name": "Can access basic features",
                "category": "basic",
            }
        ]

        # Static core settings
        self.organization_settings = [
            {
                "name": "plugin_enabled",
                "field_type": models.BooleanField,
                "default_value": True,
                "verbose_name": "Enable Plugin",
            }
        ]

    def setup_menu(self):
        # Always register main menu
        self.register_menu(
            alias="advanced",
            name="Advanced",
            url_name="advanced_plugin:dashboard",
            icon="fas fa-cogs",
        )

        # Conditional submenus based on enabled features
        enabled_features = getattr(settings, 'ADVANCED_FEATURES', [])

        if 'analytics' in enabled_features:
            self.register_submenu(
                parent_alias="advanced",
                name="Analytics",
                url_name="advanced_plugin:analytics",
                icon="fas fa-chart-bar",
            )

        if 'reporting' in enabled_features:
            self.register_submenu(
                parent_alias="advanced",
                name="Reports",
                url_name="advanced_plugin:reports",
                icon="fas fa-file-chart",
            )

    def setup_permission(self):
        # Dynamic permissions based on enabled features
        enabled_features = getattr(settings, 'ADVANCED_FEATURES', [])

        if 'analytics' in enabled_features:
            self.register_permission(
                codename="view_analytics",
                name="Can view analytics",
                category="analytics",
            )

        if 'reporting' in enabled_features:
            self.register_permission(
                codename="generate_reports",
                name="Can generate reports",
                category="reporting",
            )

    def setup_setting(self):
        # Dynamic settings based on environment
        if settings.DEBUG:
            self.register_organization_setting(
                name="debug_mode",
                field_type=models.BooleanField,
                default_value=False,
                verbose_name="Enable Debug Mode",
                admin_only=True,
            )

        # Feature-specific settings
        enabled_features = getattr(settings, 'ADVANCED_FEATURES', [])

        if 'analytics' in enabled_features:
            self.register_organization_setting(
                name="analytics_retention_days",
                field_type=models.PositiveIntegerField,
                default_value=90,
                verbose_name="Analytics Retention (Days)",
                help_text="How long to keep analytics data",
                form_section="analytics",
            )
```

## Best Practices

### 1. Consistent Naming

```python
# Good: Clear, descriptive names
self.register_menu(alias="user_management", name="User Management")
self.register_permission(codename="manage_users", name="Can manage users")
self.register_organization_setting(name="max_users", ...)

# Bad: Vague, generic names
self.register_menu(alias="admin", name="Admin")  # Too generic
self.register_permission(codename="access", name="Access")  # Too vague
```

### 2. Logical Organization

```python
def setup_menu(self):
    # Group related functionality
    self.register_menu(alias="content", name="Content", order=20)

    # Order submenus logically
    self.register_submenu(parent_alias="content", name="Posts", order=10)
    self.register_submenu(parent_alias="content", name="Categories", order=20)
    self.register_submenu(parent_alias="content", name="Settings", order=90)

def setup_permission(self):
    # Group by category
    # Content permissions
    self.register_permission(codename="view_posts", category="content")
    self.register_permission(codename="manage_posts", category="content")

    # Admin permissions
    self.register_permission(codename="admin_settings", category="admin")

def setup_setting(self):
    # Group by form section
    self.register_organization_setting(name="enable_blog", form_section="blog")
    self.register_organization_setting(name="posts_per_page", form_section="blog")
    self.register_organization_setting(name="blog_theme", form_section="blog")
```

### 3. Proper Defaults and Validation

```python
def setup_setting(self):
    # Good: Sensible defaults
    self.register_organization_setting(
        name="posts_per_page",
        field_type=models.PositiveIntegerField,
        default_value=10,  # Reasonable default
        help_text="Number of posts per page (1-100)",  # Clear guidance
    )

    # Good: Clear choices
    self.register_organization_setting(
        name="notification_level",
        field_type=models.CharField,
        default_value="normal",
        choices=[
            ("minimal", "Minimal"),
            ("normal", "Normal"),
            ("verbose", "Verbose"),
        ],
    )
```

### 4. Permission Categories

```python
def setup_permission(self):
    # Organize permissions by logical categories
    categories = {
        "content": ["view_posts", "manage_posts", "delete_posts"],
        "moderation": ["moderate_comments", "ban_users"],
        "analytics": ["view_reports", "export_data"],
        "admin": ["change_settings", "manage_users"],
    }

    for category, permissions in categories.items():
        for perm in permissions:
            self.register_permission(
                codename=perm,
                name=f"Can {perm.replace('_', ' ')}",
                category=category,
            )
```

### 5. Form Sections

```python
def setup_setting(self):
    # Group settings by logical sections
    sections = {
        "general": ["enable_feature", "plugin_name"],
        "display": ["theme", "items_per_page"],
        "notifications": ["email_alerts", "push_notifications"],
        "integrations": ["api_key", "webhook_url"],
        "advanced": ["debug_mode", "cache_timeout"],
    }

    for section, settings in sections.items():
        for setting in settings:
            self.register_organization_setting(
                name=setting,
                field_type=models.BooleanField,  # Example
                form_section=section,
                admin_only=(section == "advanced"),
            )
```

## Migration Guide

### From Legacy Separate Systems

**Before (separate files):**
```python
# apps.py
class OldPluginConfig(AppConfig):
    def ready(self):
        from core.plugin_menus import register_menu
        register_menu(alias="old", name="Old")

# permissions.py
class OldPermissionsPlugin(PluginPermissionProvider):
    def get_permissions(self):
        return [PluginPermission(codename="can_access", name="Can access")]

# organization_settings.py
class OldSettingsPlugin(OrganizationSettingsPlugin):
    def get_settings(self):
        return [OrganizationSettingField(name="enabled", field_type=models.BooleanField)]
```

**After (unified):**
```python
# apps.py
class NewPluginConfig(PluginAppConfig):
    def setup_menu(self):
        self.register_menu(alias="new", name="New")

    def setup_permission(self):
        self.register_permission(codename="access", name="Can access")

    def setup_setting(self):
        self.register_organization_setting(
            name="enabled",
            field_type=models.BooleanField
        )
```

### Migration Steps

1. **Update inheritance:** Change from `AppConfig` to `PluginAppConfig`
2. **Consolidate registration:** Move all registrations to single config class
3. **Update method names:** Use new `setup_*()` method names
4. **Update parameters:** Use new simplified parameter format
5. **Test thoroughly:** Verify all registrations work correctly

## Troubleshooting

### Common Issues

#### 1. Registration Not Working

**Problem:** Menus, permissions, or settings don't appear.

**Solutions:**
- Ensure app is in `INSTALLED_APPS`
- Check for errors in setup methods
- Verify method names (`setup_menu`, `setup_permission`, `setup_setting`)
- Check plugin ID conflicts

#### 2. Permission Denied Errors

**Problem:** Permission-based menus don't appear.

**Solutions:**
- Verify permission codenames match exactly
- Check user permissions in Django admin
- Ensure permissions are created in database:
  ```python
  from core.plugin_permissions import create_plugin_permissions
  create_plugin_permissions()
  ```

#### 3. Settings Not Saving

**Problem:** Organization settings don't persist.

**Solutions:**
- Ensure `Organization` model has `plugin_settings` JSONField
- Check form section names match
- Verify setting names are unique

#### 4. Import Errors

**Problem:** Import errors when using field types.

**Solutions:**
```python
# Correct imports
from django.db import models
from core.plugin_app_config import PluginAppConfig

# Use models.BooleanField not BooleanField
self.register_organization_setting(
    name="enabled",
    field_type=models.BooleanField,  # Correct
)
```

### Debug Utilities

#### Check Plugin Registrations

```python
# Django shell
from core.plugin_menus import plugin_menu_registry
from core.plugin_permissions import plugin_permission_registry
from core.plugin_organization_settings import organization_settings_registry

# Check menu registrations
print("Menus:", list(plugin_menu_registry._menu_aliases.keys()))

# Check permission registrations
print("Permissions:", plugin_permission_registry.get_all_permissions())

# Check settings registrations
print("Settings:", organization_settings_registry.get_all_settings())
```

#### Test Permission Creation

```python
# Create all plugin permissions
from core.plugin_permissions import create_plugin_permissions
created = create_plugin_permissions(force=True)
print("Created permissions:", created)
```

#### Validate Settings

```python
# Check setting validation
from core.plugin_organization_settings import organization_settings_registry
from myapp.models import Organization

org = Organization.objects.first()
is_valid = organization_settings_registry.validate_setting(
    "my_setting", "test_value", org
)
print("Setting valid:", is_valid)
```

---

## Summary

The enhanced `PluginAppConfig` system provides:

- **Unified Registration**: Single class for menus, permissions, and settings
- **Multiple Approaches**: Method-based, list-based, or hybrid registration
- **Easy Developer Experience**: Simple methods with sensible defaults
- **Flexible Configuration**: Static and dynamic registration options
- **WordPress-Style**: Familiar alias-based menu system
- **Comprehensive**: Full coverage of plugin functionality needs

### Key Benefits

1. **Simplicity**: One class handles all plugin configuration
2. **Flexibility**: Multiple registration approaches for different use cases
3. **Consistency**: Uniform API across all registration types
4. **Power**: Support for dynamic, conditional registration
5. **Maintainability**: Clear separation of concerns with dedicated methods

For most plugins, the **method-based approach** using `setup_menu()`, `setup_permission()`, and `setup_setting()` provides the best balance of simplicity and flexibility.
