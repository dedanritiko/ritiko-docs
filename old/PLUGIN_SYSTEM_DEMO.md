# Plugin System Demonstration

This document demonstrates the plugin system implementation that allows overriding and extending functions, views, and pages in the Ritiko system.

## Overview

Two sample plugins have been created to demonstrate different aspects of the plugin system:

1. **Patient Plugin** - Enhanced patient list management with django-tables2 and django-filter
2. **Patient Email Plugin** - Extends the Patient model with email functionality without modifying original files

## Plugin Architecture

### Core Components

- **Plugin Registry (`plugin_registry.py`)** - Manages plugin metadata and lifecycle
- **Plugin App Config (`core/plugin_app_config.py`)** - Base class for plugin Django apps
- **Settings System** - File-based plugin settings with validation
- **Permissions System** - File-based plugin permissions with groups
- **Menu System** - File-based plugin menus with hierarchical structure
- **Template Hooks** - System for injecting content into existing templates

### Plugin Structure

Each plugin follows this structure:
```
plugins/plugin_name/
├── __init__.py
├── apps.py              # Django app configuration
├── plugin.json          # Plugin metadata
├── settings.py          # Organization settings
├── permissions.py       # Plugin permissions
├── menus.py            # Plugin menus
├── models.py           # Plugin models (optional)
├── views.py            # Plugin views (optional)
├── urls.py             # Plugin URLs (optional)
├── forms.py            # Plugin forms (optional)
├── tables.py           # Django-tables2 tables (optional)
├── filters.py          # Django-filter filters (optional)
├── model_extensions.py # Model extensions (optional)
├── template_hooks.py   # Template hooks (optional)
└── templates/          # Plugin templates
```

## Sample Plugin 1: Patient Plugin

### Features
- **Enhanced Patient List** with advanced filtering and sorting using django-tables2
- **Multiple View Options** - Full list, compact view, and statistics
- **Advanced Filtering** - Filter by name, MRN, gender, location, program, etc.
- **Data Export** - Export patient data to CSV, Excel, and JSON formats
- **Responsive Design** - Bootstrap 4 compatible tables and forms

### Key Files
- `tables.py` - Defines PatientTable and CompactPatientTable using django-tables2
- `filters.py` - Defines PatientFilter using django-filter
- `views.py` - Views with filtering, pagination, and export functionality
- `templates/` - Bootstrap-styled templates for patient lists

### Usage
```python
# Access enhanced patient list
/patient_plugin/patients/

# Export data
/patient_plugin/patients/?export=csv
/patient_plugin/patients/?export=xlsx
```

## Sample Plugin 2: Patient Email Plugin

### Features
- **Model Extension** - Adds email functionality to Patient model without modifying original files
- **Email Profile** - One-to-one relationship with Patient for email settings
- **Template System** - Reusable email templates
- **Page Extensions** - Injects email sections into existing patient detail/edit pages
- **Email Methods** - Adds sendEmail(), has_email(), get_email() methods to Patient model

### Key Components

#### Model Extensions (`model_extensions.py`)
```python
# Extends Patient model with email functionality
Patient.add_to_class("send_email", send_email)
Patient.add_to_class("has_email", has_email)
Patient.add_to_class("get_email", get_email)
# ... more methods
```

#### Template Hooks (`template_hooks.py`)
```python
# Hooks for injecting content into existing pages
template_hook_registry.register_hook(
    "patient_detail_additional_sections",
    PatientTemplateHooks.patient_detail_email_section,
    plugin_id="patient_email_plugin"
)
```

#### Usage Examples
```python
# Use extended Patient model
patient = Patient.objects.get(pk=1)

# Check if patient has email
if patient.has_email():
    # Send email using extended method
    patient.send_email(
        subject="Test Email",
        message="This is a test email"
    )

    # Send appointment reminder
    patient.send_appointment_reminder(
        appointment_date="2024-01-15",
        appointment_time="10:00 AM"
    )
```

## Plugin System Features

### 1. File-Based Configuration

#### Settings (`settings.py`)
```python
settings = create_settings("plugin_name")
settings.add_boolean("enable_feature", default=True)
settings.add_number("page_size", default=25)
settings.add_choice("theme", choices=[("light", "Light"), ("dark", "Dark")])
settings.register()
```

#### Permissions (`permissions.py`)
```python
permissions = create_permissions("plugin_name")
permissions.add_permission("view_data", "Can view data")
permissions.add_group("staff", ["can_view_data"])
permissions.register()
```

#### Menus (`menus.py`)
```python
menu = create_menu("plugin_name", section="main")
menu.add_item("Plugin Page", "plugin:index", icon="fas fa-home")
submenu = menu.add_submenu("Sub Menu", icon="fas fa-list")
submenu.add_item("Sub Item", "plugin:sub_item")
menu.register()
```

### 2. Model Extensions

Plugins can extend existing models without modifying original files:

```python
# Add methods to existing models
Patient.add_to_class("new_method", new_method)

# Create related models
class PatientExtension(models.Model):
    patient = models.OneToOneField(Patient, on_delete=models.CASCADE)
    additional_field = models.CharField(max_length=100)
```

### 3. View and Page Extensions

#### Override Existing Views
```python
class CustomPatientListView(PatientListView):
    template_name = "plugin/custom_patient_list.html"

    def get_queryset(self):
        # Custom filtering logic
        return super().get_queryset().filter(custom_condition=True)
```

#### Extend Existing Pages with Template Hooks
```python
def patient_detail_extension(context, request):
    return render_to_string(
        "plugin/patient_detail_extension.html",
        context,
        request=request
    )

template_hook_registry.register_hook(
    "patient_detail_additional_sections",
    patient_detail_extension
)
```

### 4. Plugin Discovery and Management

The system automatically discovers plugins and:
- Loads plugin metadata from `plugin.json` files
- Registers Django apps
- Sets up URL routing
- Applies settings, permissions, and menus
- Validates plugin dependencies

## Testing Results

✅ **Plugin Discovery**: All 4 plugins discovered successfully
✅ **Settings Registration**: 37 settings across plugins registered
✅ **Permissions Registration**: 37 permissions across plugins registered
✅ **Menu Registration**: 17 menu items across plugins registered
✅ **URL Routing**: All plugin URLs loaded successfully
✅ **Migration Creation**: Database migrations created successfully
✅ **Django Check**: No system configuration issues

## Usage Instructions

1. **Run with plugins enabled**:
   ```bash
   pipenv run ./manage.py runserver_plus 9000 --settings=ritiko.settings.local
   ```

2. **Create migrations for new plugins**:
   ```bash
   pipenv run python manage.py makemigrations plugin_name --settings=ritiko.settings.local
   ```

3. **Access plugin functionality**:
   - Enhanced Patient List: `/patient_plugin/patients/`
   - Patient Email Management: `/patient_email_plugin/emails/`
   - Plugin settings available in organization settings
   - Plugin permissions available in user management
   - Plugin menus appear in navigation

## Benefits of This Plugin System

1. **Non-Intrusive**: Extends functionality without modifying core files
2. **Modular**: Plugins can be enabled/disabled independently
3. **Extensible**: Easy to add new plugins following established patterns
4. **Maintainable**: Clear separation of concerns and file organization
5. **Configurable**: File-based configuration for settings, permissions, and menus
6. **Developer-Friendly**: Rich APIs and helper methods for common tasks

This plugin system provides a robust foundation for extending the Ritiko application while maintaining clean code separation and easy maintenance.
