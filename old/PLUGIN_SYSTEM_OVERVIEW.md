# Plugin System Overview

## Table of Contents
- [Introduction](#introduction)
- [Plugin Architecture](#plugin-architecture)
- [Sample Plugins](#sample-plugins)
- [Creating Plugins](#creating-plugins)
- [Extending Models](#extending-models)
- [Extending Views and Pages](#extending-views-and-pages)
- [Template Hooks](#template-hooks)
- [API Reference](#api-reference)

## Introduction

The Ritiko plugin system allows developers to extend and override functionality without modifying core application files. This system supports:

- **Model Extensions**: Add fields and methods to existing models
- **View Overrides**: Create custom views or extend existing ones
- **Page Extensions**: Inject content into existing pages via template hooks
- **URL Overrides**: Override specific system URLs while preserving existing functionality
- **Settings Management**: File-based configuration with validation
- **Permissions System**: Granular permission control with groups
- **Menu Integration**: Hierarchical menu system with automatic discovery

## Plugin Architecture

### Core Components

1. **Plugin Registry** (`plugin_registry.py`)
   - Discovers and manages plugin metadata
   - Handles plugin lifecycle and dependencies
   - Provides plugin information and status

2. **Plugin App Config** (`core/plugin_app_config.py`)
   - Base Django app configuration for plugins
   - Provides helper methods for registering components
   - Handles plugin initialization and setup

3. **Configuration Systems**
   - **Settings**: Organization-specific configuration with validation
   - **Permissions**: Role-based access control with groups
   - **Menus**: Hierarchical navigation with icons and badges

4. **Extension Systems**
   - **Model Extensions**: Monkey-patching and related models
   - **Template Hooks**: Injection points for custom content
   - **View Mixins**: Reusable view functionality
   - **URL Overrides**: System for overriding specific URL patterns

### Plugin Structure

```
plugins/plugin_name/
├── __init__.py              # Package marker
├── apps.py                  # Django app configuration
├── plugin.json              # Plugin metadata
├── settings.py              # Organization settings (optional)
├── permissions.py           # Plugin permissions (optional)
├── menus.py                # Plugin menus (optional)
├── models.py               # Plugin models (optional)
├── views.py                # Plugin views (optional)
├── urls.py                 # Plugin URL patterns (optional)
├── forms.py                # Plugin forms (optional)
├── model_extensions.py     # Model extensions (optional)
├── template_hooks.py       # Template hooks (optional)
├── templates/              # Plugin templates
└── static/                 # Plugin static files
```

## Sample Plugins

### 1. Patient Plugin - Enhanced List Management

**Purpose**: Demonstrates view enhancement with advanced filtering and table functionality.

**Features**:
- Advanced patient filtering (name, MRN, gender, location, etc.)
- Multiple view formats (full, compact, statistics)
- Data export capabilities (CSV, Excel, JSON)
- Bootstrap-styled responsive tables

**Key Files**:
- `tables.py` - Custom table definitions
- `filters.py` - Advanced filtering logic
- `views.py` - Enhanced list views with export
- `urls.py` - URL overrides for `/patients/` route
- `templates/` - Responsive HTML templates

**Usage**:
```python
# Override system's /patients/ with enhanced list
/patients/                  # Now shows plugin view

# Plugin-specific URLs (also available)
/patient_plugin/patients/   # Direct plugin access
/patients/compact/          # Compact view
/patients/stats/           # Statistics view

# Export functionality
/patients/?export=csv
/patients/?format=compact
```

**URL Override Example**:
```python
# In plugins/patient_plugin/urls.py
URL_OVERRIDES = {
    "patients/": [
        path("", views.PatientListView.as_view(), name="list"),
        path("compact/", views.CompactPatientListView.as_view(),
             name="compact_patient_list"),
        path("stats/", views.patient_stats_view, name="patient_stats"),
    ]
}
```

### 2. Patient Email Plugin - Model Extension

**Purpose**: Demonstrates model extension and page injection without modifying core files.

**Features**:
- Extends Patient model with email functionality
- Adds email profile management
- Injects email sections into existing patient pages
- Email template system for notifications

**Key Files**:
- `models.py` - PatientEmail and EmailTemplate models
- `model_extensions.py` - Patient model extension methods
- `template_hooks.py` - Page injection hooks
- `views.py` - Email management views

**Usage**:
```python
# Use extended Patient model
patient = Patient.objects.get(pk=1)
patient.send_email("Subject", "Message")
patient.send_appointment_reminder(date, time)

# Check email capability
if patient.has_email():
    email = patient.get_email()
```

## Creating Plugins

### 1. Plugin Metadata (`plugin.json`)

```json
{
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "Plugin description",
  "author": "Developer Name",
  "email": "dev@example.com",
  "categories": ["category1", "category2"],
  "tags": ["tag1", "tag2"],
  "dependencies": [],
  "django_apps": ["plugins.my_plugin"],
  "enabled": true
}
```

### 2. Django App Configuration (`apps.py`)

```python
from core.plugin_app_config import PluginAppConfig

class MyPluginConfig(PluginAppConfig):
    name = "plugins.my_plugin"
    verbose_name = "My Plugin"

    def ready(self):
        super().ready()
        # Import extensions and hooks
        from . import model_extensions
        from . import template_hooks
```

### 3. Settings Configuration (`settings.py`)

```python
from core.plugin_organization_settings import create_settings

settings = create_settings("my_plugin")

# Basic settings
settings.add_boolean("enable_feature", default=True,
                    verbose_name="Enable Feature")

settings.add_number("page_size", default=25,
                   verbose_name="Items Per Page")

settings.add_choice("theme",
                   choices=[("light", "Light"), ("dark", "Dark")],
                   default="light")

# Custom validation
def validate_settings(setting_name, value, organization):
    if setting_name == "page_size" and (value < 1 or value > 100):
        return False
    return True

settings.on_validate(validate_settings)
settings.register()
```

### 4. Permissions Configuration (`permissions.py`)

```python
from core.plugin_permissions import create_permissions

permissions = create_permissions("my_plugin")

# Add permissions
permissions.add_permission("view_data", "Can view data",
                         category="data_access")
permissions.add_permission("manage_settings", "Can manage settings",
                         category="administration")

# Add permission groups
permissions.add_group("staff", ["can_view_data"])
permissions.add_group("admin", ["can_view_data", "can_manage_settings"])

permissions.register()
```

### 5. Menu Configuration (`menus.py`)

```python
from core.plugin_menus import create_menu

menu = create_menu("my_plugin", section="main")

# Main menu item
menu.add_item("My Plugin", "my_plugin:index",
              icon="fas fa-home", order=10)

# Submenu with items
submenu = menu.add_submenu("Management", icon="fas fa-cogs", order=20)
submenu.add_item("Settings", "my_plugin:settings", order=10)
submenu.add_item("Reports", "my_plugin:reports", order=20)

menu.register()
```

## Extending Models

### Method 1: Related Models (Recommended)

```python
# models.py
from django.db import models
from patients.models.people import Patient

class PatientExtension(models.Model):
    patient = models.OneToOneField(Patient, on_delete=models.CASCADE,
                                  related_name="extension")
    additional_field = models.CharField(max_length=100)

    def custom_method(self):
        return f"Extended: {self.additional_field}"
```

### Method 2: Model Monkey Patching

```python
# model_extensions.py
from patients.models.people import Patient

def new_method(self):
    """Add new method to Patient model."""
    return f"Extended functionality for {self.get_full_name()}"

def has_extension(self):
    """Check if patient has extension."""
    return hasattr(self, 'extension')

# Add methods to Patient model
Patient.add_to_class("new_method", new_method)
Patient.add_to_class("has_extension", has_extension)
```

## URL Overrides

The plugin system allows overriding specific system URLs while preserving existing functionality. This enables plugins to replace specific views without affecting the entire URL structure.

### How URL Overrides Work

1. **Plugin defines overrides** in `urls.py` using `URL_OVERRIDES`
2. **System prepends plugin URLs** to original URL patterns
3. **Django checks plugin patterns first** due to ordering
4. **Original URLs remain accessible** if not overridden
5. **Namespace compatibility** is maintained for templates

### Basic URL Override

```python
# plugins/my_plugin/urls.py
from django.urls import path
from . import views

# Regular plugin URLs (accessible at /my_plugin/)
urlpatterns = [
    path("dashboard/", views.DashboardView.as_view(), name="dashboard"),
]

# URL Overrides (replace system URLs)
URL_OVERRIDES = {
    "patients/": [
        # Override the main patients list view
        path("", views.CustomPatientListView.as_view(), name="list"),
        # Add new functionality
        path("export/", views.PatientExportView.as_view(), name="export"),
    ]
}
```

### Advanced URL Override Example

```python
# Override multiple URL patterns with preserved functionality
URL_OVERRIDES = {
    "patients/": [
        # Override main list with enhanced view
        path("", views.EnhancedPatientListView.as_view(), name="list"),

        # Add new views while preserving original functionality
        path("analytics/", views.PatientAnalyticsView.as_view(), name="analytics"),
        path("bulk-import/", views.BulkImportView.as_view(), name="bulk_import"),

        # Override specific detail functionality
        path("<int:pk>/enhanced/", views.EnhancedPatientDetailView.as_view(),
             name="enhanced_detail"),
    ],

    "reports/": [
        # Add custom reports to existing reports app
        path("custom/", views.CustomReportListView.as_view(), name="custom_reports"),
        path("custom/<slug:report_type>/", views.CustomReportView.as_view(),
             name="custom_report"),
    ]
}
```

### URL Override Best Practices

1. **Override Selectively**: Only override the specific URLs you need to change
2. **Preserve Namespaces**: Existing templates will continue to work with `app_name:view_name`
3. **Document Changes**: Clearly document which URLs are being overridden
4. **Test Thoroughly**: Ensure all existing functionality still works
5. **Handle Conflicts**: Be aware of URL pattern precedence and conflicts

### Template Compatibility

URL overrides maintain full template compatibility:

```django
<!-- Templates continue to work as before -->
<a href="{% url 'patients:list' %}">Patient List</a>           <!-- Plugin view -->
<a href="{% url 'patients:referrals-list' %}">Referrals</a>    <!-- Original view -->
<a href="{% url 'patients:analytics' %}">Analytics</a>         <!-- New plugin view -->
```

### System Behavior

- **Plugin URLs checked first**: `/patients/` → Plugin's PatientListView
- **Original URLs preserved**: `/patients/referrals/` → Original referrals view
- **New URLs available**: `/patients/analytics/` → New plugin functionality
- **Plugin URLs accessible**: `/my_plugin/dashboard/` → Direct plugin access

## Extending Views and Pages

### Method 1: View Inheritance

```python
# views.py
from patients.views import PatientListView

class CustomPatientListView(PatientListView):
    template_name = "my_plugin/custom_patient_list.html"
    paginate_by = 50

    def get_queryset(self):
        qs = super().get_queryset()
        # Add custom filtering
        return qs.filter(custom_condition=True)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['custom_data'] = "Additional data"
        return context
```

### Method 2: Function-Based View Override

```python
# views.py
from patients.models.people import Patient
from django.shortcuts import render

def custom_patient_list(request):
    patients = Patient.objects.filter(organization=request.user.organization)

    # Custom filtering logic
    name_filter = request.GET.get('name')
    if name_filter:
        patients = patients.filter(
            Q(first_name__icontains=name_filter) |
            Q(last_name__icontains=name_filter)
        )

    context = {
        'patients': patients,
        'total_count': patients.count(),
    }

    return render(request, 'my_plugin/patient_list.html', context)
```

## Template Hooks

Template hooks allow injecting content into existing templates without modifying them.

### 1. Register Template Hooks

```python
# template_hooks.py
from django.template.loader import render_to_string

def patient_detail_extension(context, request):
    """Add custom section to patient detail page."""
    patient = context.get('patient')

    hook_context = {
        'patient': patient,
        'custom_data': get_custom_data(patient),
        'request': request,
    }

    return render_to_string(
        'my_plugin/patient_detail_extension.html',
        hook_context,
        request=request
    )

# Register hook
from core.template_hooks import template_hook_registry

template_hook_registry.register_hook(
    "patient_detail_additional_sections",
    patient_detail_extension,
    plugin_id="my_plugin",
    priority=10
)
```

### 2. Hook Points in Templates

```django
<!-- In existing patient detail template -->
<div class="patient-details">
    <!-- Existing content -->
</div>

{% load template_hooks %}
{% hook "patient_detail_additional_sections" %}

<div class="patient-actions">
    <!-- More existing content -->
</div>
```

## API Reference

### Plugin App Config Methods

```python
class MyPluginConfig(PluginAppConfig):
    def setup_menu(self):
        """Override to setup menus programmatically."""
        pass

    def setup_permission(self):
        """Override to setup permissions programmatically."""
        pass

    def setup_setting(self):
        """Override to setup settings programmatically."""
        pass
```

### Settings Builder API

```python
settings = create_settings("plugin_id")

# Field types
settings.add_boolean(name, default=False, **kwargs)
settings.add_number(name, default=0, **kwargs)
settings.add_text(name, default="", **kwargs)
settings.add_decimal(name, default=0.0, **kwargs)
settings.add_choice(name, choices=[], default="", **kwargs)
settings.add(name, field_type, default_value=None, **kwargs)

# Validation and change handling
settings.on_validate(validator_function)
settings.on_change(change_handler_function)

# Registration
settings.register()
```

### Permissions Builder API

```python
permissions = create_permissions("plugin_id")

# Add permissions
permissions.add_permission(codename, name, description="", category="")

# Add permission groups
permissions.add_group(group_name, permission_list)

# Callbacks
permissions.on_created(callback_function)

# Registration
permissions.register()
```

### Menu Builder API

```python
menu = create_menu("plugin_id", section="main")

# Add menu items
menu.add_item(name, url_name, icon="", order=0, **kwargs)

# Add submenus
submenu = menu.add_submenu(name, icon="", order=0, **kwargs)
submenu.add_item(name, url_name, **kwargs)
submenu.add_separator()

# Registration
menu.register()
```

### Template Hooks API

```python
from core.template_hooks import template_hook_registry

# Register hooks
template_hook_registry.register_hook(
    hook_name,
    callback_function,
    plugin_id="plugin_id",
    priority=10
)

# Hook callback signature
def hook_callback(context, request):
    return rendered_html_string
```

## Best Practices

1. **Use Related Models**: Prefer creating related models over monkey-patching when extending functionality.

2. **Follow Naming Conventions**: Use consistent naming for plugin files and functions.

3. **Handle Permissions**: Always check permissions in views and templates.

4. **Validate Settings**: Implement proper validation for plugin settings.

5. **Document Extensions**: Clearly document any model extensions or view overrides.

6. **Test Plugin Integration**: Test plugins with different configurations and user permissions.

7. **Handle Dependencies**: Specify plugin dependencies in metadata and check them at runtime.

8. **Use Template Hooks**: Prefer template hooks over template overrides for better maintainability.

9. **URL Override Guidelines**:
   - Only override URLs when necessary to replace core functionality
   - Preserve existing URL patterns that don't need changes
   - Document all URL overrides in plugin documentation
   - Test that all existing templates and links continue to work
   - Consider backward compatibility when overriding system URLs

This plugin system provides a robust foundation for extending Ritiko while maintaining code quality and system stability.
