# Plugin App Configuration Guide

The `PluginAppConfig` class provides a flexible and powerful way to register menus for Django plugins using WordPress-style aliases and supports both static and dynamic menu creation.

## Table of Contents

- [Quick Start](#quick-start)
- [Menu Registration Methods](#menu-registration-methods)
- [Menu Parameters](#menu-parameters)
- [Advanced Features](#advanced-features)
- [Real-World Examples](#real-world-examples)
- [Migration Guide](#migration-guide)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Quick Start

The simplest way to create plugin menus:

```python
from core.plugin_app_config import PluginAppConfig

class MyPluginConfig(PluginAppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "plugins.my_plugin"
    verbose_name = "My Plugin"

    def setup_menu(self):
        # Register main menu
        self.register_menu(
            alias="my_menu",
            name="My Menu",
            url_name="my_plugin:dashboard",
            icon="fas fa-home",
            order=50,
        )

        # Register submenu items
        self.register_submenu(
            parent_alias="my_menu",
            name="Sub Item",
            url_name="my_plugin:sub_item",
        )
```

## Menu Registration Methods

### 1. Method-Based Registration (Recommended)

Use `setup_menu()` with register methods for maximum flexibility:

```python
class BlogPluginConfig(PluginAppConfig):
    def setup_menu(self):
        # Register main menus
        self.register_menu(
            alias="content",
            name="Content",
            url_name="content:index",
            icon="fas fa-edit",
            order=20,
            section="main",
        )

        # Register submenus
        self.register_submenu(
            parent_alias="content",
            name="Blog Posts",
            url_name="blog_plugin:post_list",
            icon="fas fa-blog",
            order=10,
            permission="can_view_blog_posts",
        )

        # Conditional menu registration
        if self.is_feature_enabled('advanced_tools'):
            self.register_submenu(
                parent_alias="content",
                name="Advanced Tools",
                url_name="blog_plugin:advanced_tools",
                permission="can_use_advanced_tools",
            )
```

### 2. List-Based Registration

Define menus as lists in `__init__()` for static configurations:

```python
class MyPluginConfig(PluginAppConfig):
    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)

        self.main_menus = [
            {
                "alias": "dashboard",
                "name": "Dashboard",
                "url_name": "my_plugin:dashboard",
                "icon": "fas fa-tachometer-alt",
                "order": 10,
                "section": "main",
            },
        ]

        self.sub_menus = [
            {
                "parent_alias": "dashboard",
                "name": "Overview",
                "url_name": "my_plugin:overview",
                "icon": "fas fa-chart-bar",
                "order": 10,
            },
        ]
```

### 3. Hybrid Approach

Combine both approaches for maximum flexibility:

```python
class MyPluginConfig(PluginAppConfig):
    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)
        # Static menus
        self.main_menus = [
            {"alias": "main_menu", "name": "Main", "url_name": "my_plugin:main", "order": 10}
        ]

    def setup_menu(self):
        # Dynamic menus based on conditions
        if self.has_permission('admin'):
            self.register_menu(
                alias="admin",
                name="Admin",
                url_name="my_plugin:admin",
                permission="is_staff",
            )

        # Add conditional submenus
        enabled_features = getattr(settings, 'MY_PLUGIN_FEATURES', [])
        for feature in enabled_features:
            self.register_submenu(
                parent_alias="main_menu",
                name=feature.title(),
                url_name=f"my_plugin:{feature}",
                order=50 + len(enabled_features),
            )
```

## Menu Parameters

### Main Menu Parameters (`register_menu`)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `alias` | str | ✅ | - | Unique identifier for the menu |
| `name` | str | ✅ | - | Display name |
| `url_name` | str | | - | Django URL name |
| `url` | str | | - | Direct URL (alternative to url_name) |
| `icon` | str | | "fas fa-link" | CSS icon class |
| `order` | int | | 100 | Display order (lower = first) |
| `section` | str | | "main" | Menu section |
| `permission` | str | | - | Required permission to view |

### Submenu Parameters (`register_submenu`)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `parent_alias` | str | ✅ | - | Alias of parent menu |
| `name` | str | ✅ | - | Display name |
| `url_name` | str | | - | Django URL name |
| `url` | str | | - | Direct URL (alternative to url_name) |
| `icon` | str | | "fas fa-circle" | CSS icon class |
| `order` | int | | 100 | Display order within parent |
| `permission` | str | | - | Required permission to view |

## Advanced Features

### Conditional Menu Registration

```python
def setup_menu(self):
    # Feature-based menus
    if getattr(settings, 'BLOG_ENABLED', False):
        self.register_menu(
            alias="blog",
            name="Blog",
            url_name="blog:dashboard",
            icon="fas fa-blog",
        )

    # Permission-based menus
    if self.user_has_permission('can_manage_users'):
        self.register_submenu(
            parent_alias="admin",
            name="User Management",
            url_name="admin:users",
            permission="can_manage_users",
        )

    # Environment-based menus
    if settings.DEBUG:
        self.register_submenu(
            parent_alias="admin",
            name="Debug Tools",
            url_name="debug:tools",
            icon="fas fa-bug",
        )
```

### Menu Sections

Organize menus into logical sections:

```python
def setup_menu(self):
    # Main navigation section
    self.register_menu(
        alias="dashboard",
        name="Dashboard",
        url_name="dashboard:index",
        section="main",
        order=10,
    )

    # Admin section
    self.register_menu(
        alias="settings",
        name="Settings",
        url_name="settings:index",
        section="admin",
        order=90,
        permission="is_staff",
    )

    # Plugin-specific section
    self.register_menu(
        alias="tools",
        name="Tools",
        url_name="tools:index",
        section="plugins",
        order=50,
    )
```

### Dynamic Menu Generation

```python
def setup_menu(self):
    # Register base menu
    self.register_menu(
        alias="reports",
        name="Reports",
        url_name="reports:index",
        icon="fas fa-chart-bar",
    )

    # Generate submenus from database
    from myapp.models import Report
    for report in Report.objects.filter(is_active=True):
        self.register_submenu(
            parent_alias="reports",
            name=report.name,
            url_name="reports:view",
            url=f"/reports/{report.slug}/",
            order=report.order,
            permission=f"can_view_{report.slug}",
        )
```

### Helper Methods

Available utility methods in `PluginAppConfig`:

```python
def setup_menu(self):
    # Get URL prefix for this plugin
    prefix = self.get_menu_prefix()  # Returns "my_plugin" from "plugins.my_plugin"

    # Add menus using helper methods
    self.add_main_menu(
        alias="test",
        name="Test Menu",
        url_name=f"{prefix}:test",
    )
    self.add_submenu(
        parent_alias="test",
        name="Test Sub",
        url_name=f"{prefix}:test_sub",
    )
```

## Real-World Examples

### E-commerce Plugin

```python
class ShopPluginConfig(PluginAppConfig):
    name = "plugins.shop_plugin"
    verbose_name = "Shop Plugin"

    def setup_menu(self):
        # Main shop menu
        self.register_menu(
            alias="shop",
            name="Shop",
            url_name="shop_plugin:dashboard",
            icon="fas fa-shopping-cart",
            order=25,
            section="main",
        )

        # Core shop submenus
        shop_menus = [
            ("Products", "shop_plugin:product_list", "fas fa-box", "can_manage_products"),
            ("Orders", "shop_plugin:order_list", "fas fa-receipt", "can_view_orders"),
            ("Customers", "shop_plugin:customer_list", "fas fa-users", "can_view_customers"),
        ]

        for name, url_name, icon, permission in shop_menus:
            self.register_submenu(
                parent_alias="shop",
                name=name,
                url_name=url_name,
                icon=icon,
                permission=permission,
                order=shop_menus.index((name, url_name, icon, permission)) * 10,
            )

        # Conditional features
        if hasattr(settings, 'INVENTORY_TRACKING') and settings.INVENTORY_TRACKING:
            self.register_submenu(
                parent_alias="shop",
                name="Inventory",
                url_name="shop_plugin:inventory_list",
                icon="fas fa-warehouse",
                permission="can_manage_inventory",
                order=30,
            )

        # Add dashboard quick actions
        self.register_submenu(
            parent_alias="dashboard",
            name="Sales Overview",
            url_name="shop_plugin:sales_dashboard",
            icon="fas fa-chart-pie",
            permission="can_view_sales",
            order=20,
        )
```

### Content Management Plugin

```python
class CMSPluginConfig(PluginAppConfig):
    name = "plugins.cms_plugin"
    verbose_name = "CMS Plugin"

    def __init__(self, app_name, app_module):
        super().__init__(app_name, app_module)

        # Static core menus
        self.main_menus = [
            {
                "alias": "content",
                "name": "Content",
                "url_name": "cms_plugin:content_index",
                "icon": "fas fa-edit",
                "order": 20,
                "section": "main",
            }
        ]

        # Static submenus
        self.sub_menus = [
            {
                "parent_alias": "content",
                "name": "Pages",
                "url_name": "cms_plugin:page_list",
                "icon": "fas fa-file-alt",
                "order": 10,
                "permission": "can_manage_pages",
            },
            {
                "parent_alias": "content",
                "name": "Media Library",
                "url_name": "cms_plugin:media_library",
                "icon": "fas fa-photo-video",
                "order": 20,
                "permission": "can_manage_media",
            }
        ]

    def setup_menu(self):
        # Dynamic menus based on enabled features
        enabled_features = getattr(settings, 'CMS_FEATURES', [])

        feature_menus = {
            'blog': ("Blog", "cms_plugin:blog_list", "fas fa-blog"),
            'events': ("Events", "cms_plugin:event_list", "fas fa-calendar"),
            'newsletter': ("Newsletter", "cms_plugin:newsletter_list", "fas fa-envelope"),
            'seo': ("SEO Tools", "cms_plugin:seo_tools", "fas fa-search"),
        }

        order = 30  # Start after static menus
        for feature, (name, url_name, icon) in feature_menus.items():
            if feature in enabled_features:
                self.register_submenu(
                    parent_alias="content",
                    name=name,
                    url_name=url_name,
                    icon=icon,
                    order=order,
                    permission=f"can_manage_{feature}",
                )
                order += 10

        # Admin tools for superusers
        if settings.DEBUG or getattr(settings, 'SHOW_CMS_ADMIN_TOOLS', False):
            self.register_submenu(
                parent_alias="content",
                name="Admin Tools",
                url_name="cms_plugin:admin_tools",
                icon="fas fa-tools",
                order=90,
                permission="is_superuser",
            )
```

### Multi-tenant Plugin

```python
class TenantPluginConfig(PluginAppConfig):
    name = "plugins.tenant_plugin"
    verbose_name = "Multi-Tenant Plugin"

    def setup_menu(self):
        # Base tenant menu
        self.register_menu(
            alias="tenants",
            name="Tenants",
            url_name="tenant_plugin:tenant_list",
            icon="fas fa-building",
            order=15,
            permission="can_manage_tenants",
        )

        # Dynamic tenant-specific menus
        from django.contrib.auth import get_user_model
        from tenant_plugin.models import Tenant

        current_user = getattr(self, '_current_user', None)  # Set during request
        if current_user and hasattr(current_user, 'tenant'):
            tenant = current_user.tenant

            # Tenant-specific submenus
            self.register_submenu(
                parent_alias="tenants",
                name=f"{tenant.name} Dashboard",
                url_name="tenant_plugin:tenant_dashboard",
                url=f"/tenants/{tenant.slug}/dashboard/",
                icon="fas fa-chart-line",
                order=10,
            )

            # Dynamic features based on tenant plan
            if tenant.plan.name in ['premium', 'enterprise']:
                self.register_submenu(
                    parent_alias="tenants",
                    name="Advanced Analytics",
                    url_name="tenant_plugin:advanced_analytics",
                    icon="fas fa-chart-pie",
                    order=20,
                    permission="can_view_analytics",
                )

            if tenant.plan.name == 'enterprise':
                self.register_submenu(
                    parent_alias="tenants",
                    name="API Management",
                    url_name="tenant_plugin:api_management",
                    icon="fas fa-cogs",
                    order=30,
                    permission="can_manage_api",
                )
```

## Migration Guide

### From Legacy AppConfig

**Before:**
```python
from django.apps import AppConfig
from core.plugin_menus import register_menu, register_submenu

class OldPluginConfig(AppConfig):
    name = "plugins.old_plugin"
    verbose_name = "Old Plugin"

    def ready(self):
        print(f"✅ {self.verbose_name} is ready")

        try:
            register_menu(
                alias="old_menu",
                name="Old Menu",
                url_name="old_plugin:index",
                icon="fas fa-home",
                order=50,
            )

            register_submenu(
                parent_alias="old_menu",
                name="Sub Item",
                url_name="old_plugin:sub_item",
                order=10,
            )

            print("🔗 Old plugin menus registered")
        except Exception as e:
            print(f"⚠️ Error registering menus: {e}")
```

**After:**
```python
from core.plugin_app_config import PluginAppConfig

class NewPluginConfig(PluginAppConfig):
    name = "plugins.new_plugin"
    verbose_name = "New Plugin"

    def setup_menu(self):
        self.register_menu(
            alias="new_menu",
            name="New Menu",
            url_name="new_plugin:index",
            icon="fas fa-home",
            order=50,
        )

        self.register_submenu(
            parent_alias="new_menu",
            name="Sub Item",
            url_name="new_plugin:sub_item",
            order=10,
        )
```

### Migration Checklist

- [ ] Change base class from `AppConfig` to `PluginAppConfig`
- [ ] Move menu registration from `ready()` to `setup_menu()`
- [ ] Replace direct imports with `self.register_*()` methods
- [ ] Remove manual error handling (handled by base class)
- [ ] Update import statements
- [ ] Test menu registration
- [ ] Update documentation

## Best Practices

### 1. Naming Conventions

```python
# Good: Descriptive, unique aliases
self.register_menu(alias="user_management", name="User Management")
self.register_menu(alias="blog_posts", name="Blog Posts")
self.register_menu(alias="shop_orders", name="Orders")

# Bad: Generic, potential conflicts
self.register_menu(alias="admin", name="Admin")  # Too generic
self.register_menu(alias="list", name="List")    # Too vague
```

### 2. Logical Ordering

```python
# Good: Consistent 10-increment ordering
self.register_menu(alias="dashboard", order=10)  # First
self.register_menu(alias="content", order=20)    # Second
self.register_menu(alias="users", order=30)      # Third
self.register_menu(alias="settings", order=90)   # Last

# Within submenus
self.register_submenu(parent_alias="content", name="Posts", order=10)
self.register_submenu(parent_alias="content", name="Pages", order=20)
self.register_submenu(parent_alias="content", name="Media", order=30)
```

### 3. Permission Management

```python
# Good: Specific permissions
self.register_menu(
    alias="user_management",
    permission="can_manage_users",  # Specific
)

self.register_submenu(
    parent_alias="shop",
    name="Orders",
    permission="can_view_orders",   # Granular
)

# Good: Django built-in permissions
self.register_menu(
    alias="admin_tools",
    permission="is_staff",          # Django built-in
)
```

### 4. Icon Consistency

```python
# Good: Consistent icon families and semantics
self.register_menu(alias="dashboard", icon="fas fa-tachometer-alt")  # FontAwesome
self.register_menu(alias="content", icon="fas fa-edit")
self.register_menu(alias="users", icon="fas fa-users")
self.register_menu(alias="settings", icon="fas fa-cogs")

# Submenu icons should be related but distinct
self.register_submenu(parent_alias="content", name="Posts", icon="fas fa-file-alt")
self.register_submenu(parent_alias="content", name="Media", icon="fas fa-images")
```

### 5. Conditional Logic

```python
# Good: Clear, testable conditions
def setup_menu(self):
    # Feature flags
    if getattr(settings, 'BLOG_ENABLED', False):
        self._register_blog_menus()

    if getattr(settings, 'SHOP_ENABLED', False):
        self._register_shop_menus()

    # Environment-based
    if settings.DEBUG:
        self._register_debug_menus()

def _register_blog_menus(self):
    """Separate method for blog menu registration."""
    self.register_menu(alias="blog", name="Blog", ...)

# Bad: Complex nested conditions
def setup_menu(self):
    if settings.BLOG_ENABLED and settings.SHOP_ENABLED and not settings.MAINTENANCE_MODE:
        if user.has_perm('blog.view') or user.has_perm('shop.view'):
            # Complex logic is hard to test and debug
            pass
```

### 6. Documentation

```python
class MyPluginConfig(PluginAppConfig):
    """
    Configuration for MyPlugin.

    Registers the following menus:
    - Dashboard: Main plugin dashboard
    - Content: Content management section
      - Posts: Blog post management (requires 'can_manage_posts')
      - Media: Media library (requires 'can_manage_media')
    - Settings: Plugin configuration (requires 'is_staff')

    Features controlled by settings:
    - MYPLUGIN_ENABLE_BLOG: Enables blog functionality
    - MYPLUGIN_ENABLE_MEDIA: Enables media management
    """

    def setup_menu(self):
        # Register core dashboard menu
        self.register_menu(
            alias="myplugin_dashboard",
            name="My Plugin",
            url_name="myplugin:dashboard",
            icon="fas fa-home",
            order=50,
        )

        # Add conditional features
        if getattr(settings, 'MYPLUGIN_ENABLE_BLOG', True):
            self._register_blog_menus()

    def _register_blog_menus(self):
        """Register blog-related menu items."""
        # Implementation here...
```

## Troubleshooting

### Common Issues

#### Menu Not Appearing

**Problem:** Menu items don't show up in the navigation.

**Solutions:**
1. Check permissions:
   ```python
   # Debug permission issues
   def setup_menu(self):
       print(f"Current user: {getattr(self, '_current_user', 'Unknown')}")

       self.register_menu(
           alias="debug_menu",
           name="Debug Menu",
           # Temporarily remove permission to test
           # permission="can_debug",
       )
   ```

2. Verify conditional logic:
   ```python
   def setup_menu(self):
       enabled = getattr(settings, 'FEATURE_ENABLED', False)
       print(f"Feature enabled: {enabled}")  # Debug output

       if enabled:
           self.register_menu(...)
   ```

3. Check menu registration order:
   ```python
   # Ensure parent menu is registered before submenu
   def setup_menu(self):
       # Register parent first
       self.register_menu(alias="parent", name="Parent")

       # Then register children
       self.register_submenu(parent_alias="parent", name="Child")
   ```

#### Alias Conflicts

**Problem:** `ValueError: Menu alias 'dashboard' already exists`

**Solutions:**
1. Use plugin-specific prefixes:
   ```python
   # Good: Plugin-specific aliases
   self.register_menu(alias="blog_dashboard", name="Blog Dashboard")
   self.register_menu(alias="shop_dashboard", name="Shop Dashboard")

   # Bad: Generic aliases
   self.register_menu(alias="dashboard", name="Dashboard")  # Conflict risk
   ```

2. Check existing registrations:
   ```python
   from core.plugin_menus import plugin_menu_registry

   def setup_menu(self):
       # Debug: Check existing aliases
       existing = plugin_menu_registry._menu_aliases
       print(f"Existing aliases: {list(existing.keys())}")
   ```

#### URL Errors

**Problem:** Menu links result in 404 errors.

**Solutions:**
1. Verify URL patterns exist:
   ```python
   # In your plugin's urls.py
   from django.urls import path
   from . import views

   app_name = 'my_plugin'
   urlpatterns = [
       path('dashboard/', views.dashboard, name='dashboard'),
       path('posts/', views.post_list, name='post_list'),
   ]
   ```

2. Check URL name format:
   ```python
   # Correct format: app_name:url_name
   self.register_menu(
       alias="my_menu",
       url_name="my_plugin:dashboard",  # Correct
   )

   # Wrong formats:
   # url_name="dashboard",           # Missing app_name
   # url_name="my_plugin.dashboard", # Wrong separator
   ```

#### Permission Errors

**Problem:** Menu appears but user gets permission denied.

**Solutions:**
1. Check permission strings:
   ```python
   # Match your permission system
   from django.contrib.auth.models import Permission

   # Check if permission exists
   try:
       perm = Permission.objects.get(codename='can_manage_posts')
       print(f"Permission exists: {perm}")
   except Permission.DoesNotExist:
       print("Permission does not exist")
   ```

2. Use Django's built-in permissions:
   ```python
   # Safe built-in permissions
   self.register_menu(permission="is_staff")      # User is staff
   self.register_menu(permission="is_superuser")  # User is superuser
   ```

#### Order Not Working

**Problem:** Menu items appear in wrong order.

**Solutions:**
1. Check order values:
   ```python
   # Lower numbers appear first
   self.register_menu(alias="first", order=10)   # Appears first
   self.register_menu(alias="second", order=20)  # Appears second
   self.register_menu(alias="third", order=30)   # Appears third
   ```

2. Debug registration order:
   ```python
   def setup_menu(self):
       menus = [
           ("dashboard", "Dashboard", 10),
           ("content", "Content", 20),
           ("users", "Users", 30),
       ]

       for alias, name, order in menus:
           print(f"Registering {alias} with order {order}")
           self.register_menu(alias=alias, name=name, order=order)
   ```

### Debug Utilities

#### Menu Registry Inspector

```python
# Add to your Django shell or management command
from core.plugin_menus import plugin_menu_registry

def debug_menu_registry():
    """Print current menu registry state."""
    print("=== Menu Registry Debug ===")

    print(f"Registered aliases: {list(plugin_menu_registry._menu_aliases.keys())}")

    for menu_id, menu_data in plugin_menu_registry._main_menu_items.items():
        print(f"\nMenu ID: {menu_id}")
        print(f"  Alias: {menu_data.get('alias')}")
        print(f"  Name: {menu_data.get('name')}")
        print(f"  Order: {menu_data.get('order')}")
        print(f"  Children: {len(menu_data.get('children', []))}")

        for child in menu_data.get('children', []):
            print(f"    - {child.get('name')} (order: {child.get('order')})")

# Usage in Django shell:
# python manage.py shell
# >>> exec(open('debug_menus.py').read())
# >>> debug_menu_registry()
```

#### Permission Checker

```python
def check_user_menu_permissions(user):
    """Check what menus a user can see."""
    from core.plugin_menus import plugin_menu_registry
    from core.plugin_permissions import user_has_plugin_permission

    print(f"=== Menu Permissions for {user} ===")

    for menu_id, menu_data in plugin_menu_registry._main_menu_items.items():
        permission = menu_data.get('permission')
        if permission:
            has_perm = user_has_plugin_permission(user, permission)
            status = "✅" if has_perm else "❌"
            print(f"{status} {menu_data['name']} (requires: {permission})")
        else:
            print(f"✅ {menu_data['name']} (no permission required)")

# Usage:
# from django.contrib.auth import get_user_model
# User = get_user_model()
# user = User.objects.get(username='testuser')
# check_user_menu_permissions(user)
```

---

## Summary

The `PluginAppConfig` system provides a robust, flexible way to manage plugin menus with:

- **Multiple registration methods** (method-based, list-based, hybrid)
- **WordPress-style aliases** for easy menu relationships
- **Conditional menu registration** for dynamic UIs
- **Permission integration** for secure menu access
- **Comprehensive error handling** and debugging support

For most use cases, the **method-based approach** using `setup_menu()` is recommended as it provides the most flexibility and readability.

Remember to follow the [best practices](#best-practices) and use the [troubleshooting guide](#troubleshooting) when encountering issues.
