# Plugin Settings and Configuration

This tutorial explores how to create configurable settings for your plugins. You'll learn how to define organization-specific settings, validate user input, and access settings throughout your plugin code.

## Introduction to Plugin Settings

The Ritiko plugin system includes a powerful settings framework that allows:

- **Organization-specific configuration**: Each organization can have different settings
- **Type validation**: Automatically validate settings based on their types
- **User interface integration**: Settings automatically appear in the admin interface
- **Change tracking**: Monitor and respond to setting changes
- **Default values**: Provide sensible defaults for all settings

## Creating Plugin Settings

To create settings for your plugin, create a `settings.py` file in your plugin directory:

```python
# plugins/inventory_plugin/settings.py
from core.plugin_organization_settings import create_settings

# Create a settings object for your plugin
settings = create_settings("inventory_plugin")

# Add boolean settings
settings.add_boolean(
    "enable_low_stock_alerts",
    default=True,
    verbose_name="Enable Low Stock Alerts",
    help_text="Send alerts when inventory items reach low stock levels"
)

# Add number settings
settings.add_number(
    "low_stock_threshold",
    default=10,
    verbose_name="Low Stock Threshold",
    help_text="Items with quantities below this threshold will trigger alerts"
)

settings.add_number(
    "reorder_lead_time_days",
    default=14,
    verbose_name="Reorder Lead Time (Days)",
    help_text="Typical number of days to receive items after ordering"
)

# Add text settings
settings.add_text(
    "default_supplier",
    default="",
    verbose_name="Default Supplier",
    help_text="Default supplier to use for new inventory items"
)

# Add choice settings
settings.add_choice(
    "stock_display_format",
    choices=[
        ("quantity", "Show Quantity Only"),
        ("quantity_units", "Show Quantity and Units"),
        ("detailed", "Show Detailed Information")
    ],
    default="quantity_units",
    verbose_name="Stock Display Format",
    help_text="How to display stock information in lists and reports"
)

# Register the settings
settings.register()
```

### Available Setting Types

The settings framework supports several data types:

| Method | Data Type | Description |
|--------|-----------|-------------|
| `add_boolean` | Boolean | True/False values |
| `add_number` | Integer | Whole number values |
| `add_decimal` | Decimal | Precise decimal values |
| `add_text` | String | Text values |
| `add_choice` | Selection | Predefined options |
| `add_json` | JSON | Complex structured data |
| `add_date` | Date | Calendar dates |
| `add_datetime` | DateTime | Date and time values |

### Setting Validation

You can add custom validation to your settings:

```python
# plugins/inventory_plugin/settings.py
def validate_settings(setting_name, value, organization):
    """Validate settings before they're saved."""
    if setting_name == "low_stock_threshold":
        # Must be between 1 and 1000
        if value < 1 or value > 1000:
            return False, "Low stock threshold must be between 1 and 1000"

    elif setting_name == "reorder_lead_time_days":
        # Must be positive
        if value < 0:
            return False, "Reorder lead time cannot be negative"

    # If we get here, validation passed
    return True, None

# Register the validation function
settings.on_validate(validate_settings)
```

### Setting Change Handlers

You can respond to setting changes with a change handler:

```python
# plugins/inventory_plugin/settings.py
def handle_setting_change(setting_name, old_value, new_value, organization):
    """React to setting changes."""
    if setting_name == "enable_low_stock_alerts" and new_value:
        # If alerts were just enabled, check for current low stock
        from .tasks import check_low_stock_items
        check_low_stock_items.delay(organization.id)

    elif setting_name == "low_stock_threshold":
        # If threshold changed, reevaluate alerts
        from .tasks import update_stock_status
        update_stock_status.delay(organization.id)

# Register the change handler
settings.on_change(handle_setting_change)
```

## Accessing Settings in Your Code

You can access your plugin's settings throughout your code:

### In Views

```python
# plugins/inventory_plugin/views.py
from django.shortcuts import render
from .settings import settings

def inventory_list(request):
    # Get organization-specific settings
    display_format = settings.get_for_organization(
        request.user.organization,
        "stock_display_format"
    )

    # Use settings to control view behavior
    context = {
        'items': get_inventory_items(request.user.organization),
        'display_format': display_format
    }

    return render(request, 'inventory_plugin/inventory_list.html', context)
```

### In Models

```python
# plugins/inventory_plugin/models.py
from django.db import models
from .settings import settings

class InventoryItem(models.Model):
    # ... fields ...

    def is_low_stock(self):
        """Check if this item has low stock."""
        # Get threshold from settings
        threshold = settings.get_for_organization(
            self.organization,
            "low_stock_threshold"
        )

        return self.quantity < threshold

    def should_reorder(self):
        """Check if this item should be reordered."""
        if not self.is_low_stock():
            return False

        # Only suggest reordering if alerts are enabled
        alerts_enabled = settings.get_for_organization(
            self.organization,
            "enable_low_stock_alerts"
        )

        return alerts_enabled
```

### In Templates

```html
<!-- plugins/inventory_plugin/templates/inventory_plugin/item_detail.html -->
{% load plugin_settings %}

<div class="stock-info">
    <h4>Stock Information</h4>

    <p>Current Quantity: {{ item.quantity }} {{ item.unit }}</p>

    {% get_plugin_setting "inventory_plugin" "low_stock_threshold" as threshold %}

    {% if item.quantity < threshold %}
        <div class="alert alert-warning">
            <i class="fas fa-exclamation-triangle"></i>
            This item is below the low stock threshold ({{ threshold }}).
        </div>

        {% get_plugin_setting "inventory_plugin" "enable_low_stock_alerts" as alerts_enabled %}

        {% if alerts_enabled %}
            <p>Automatic alerts are enabled for low stock items.</p>
        {% else %}
            <p>Note: Automatic alerts are currently disabled.</p>
        {% endif %}
    {% endif %}
</div>
```

## Settings in the Admin Interface

The settings framework automatically integrates with the Ritiko admin interface, providing a user-friendly way to configure your plugin.

### Settings Interface Organization

Settings are displayed in the Organization Settings section of the admin interface, grouped by plugin:

```
Organization Settings
├── General
├── Users & Permissions
├── Plugins
│   ├── Inventory Plugin
│   │   ├── Enable Low Stock Alerts
│   │   ├── Low Stock Threshold
│   │   ├── Reorder Lead Time (Days)
│   │   ├── Default Supplier
│   │   └── Stock Display Format
│   └── Other Plugins...
└── Integrations
```

### Accessing Settings Programmatically

You can also provide a settings UI within your plugin's pages:

```python
# plugins/inventory_plugin/views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from .settings import settings
from .forms import InventorySettingsForm

def plugin_settings(request):
    """View to manage plugin settings."""
    organization = request.user.organization

    # Get current settings
    current_settings = {
        'enable_low_stock_alerts': settings.get_for_organization(
            organization, 'enable_low_stock_alerts'),
        'low_stock_threshold': settings.get_for_organization(
            organization, 'low_stock_threshold'),
        'reorder_lead_time_days': settings.get_for_organization(
            organization, 'reorder_lead_time_days'),
        'default_supplier': settings.get_for_organization(
            organization, 'default_supplier'),
        'stock_display_format': settings.get_for_organization(
            organization, 'stock_display_format'),
    }

    if request.method == 'POST':
        form = InventorySettingsForm(request.POST)
        if form.is_valid():
            # Update settings
            for key, value in form.cleaned_data.items():
                settings.set_for_organization(organization, key, value)

            messages.success(request, "Settings updated successfully")
            return redirect('inventory_plugin:dashboard')
    else:
        form = InventorySettingsForm(initial=current_settings)

    return render(request, 'inventory_plugin/settings.html', {
        'form': form,
        'title': 'Inventory Plugin Settings'
    })
```

## Default Settings

When a plugin is first installed, the default values you specify are used for all organizations. If you need to customize the defaults based on the organization, you can use an initialization function:

```python
# plugins/inventory_plugin/settings.py
def initialize_settings(organization):
    """Initialize settings for a new organization."""
    # You can customize defaults based on organization properties
    if organization.is_large_organization():
        settings.set_for_organization(organization, "low_stock_threshold", 50)
    else:
        settings.set_for_organization(organization, "low_stock_threshold", 10)

    # Set organization-specific supplier
    if organization.default_supplier:
        settings.set_for_organization(
            organization, "default_supplier", organization.default_supplier)

# Register the initialization function
settings.on_initialize(initialize_settings)
```

## Advanced Settings Features

### Dependent Settings

Some settings may only be relevant if another setting is enabled:

```python
# plugins/inventory_plugin/settings.py
settings.add_boolean(
    "enable_low_stock_alerts",
    default=True,
    verbose_name="Enable Low Stock Alerts"
)

# These settings depend on enable_low_stock_alerts
settings.add_number(
    "low_stock_threshold",
    default=10,
    verbose_name="Low Stock Threshold",
    depends_on="enable_low_stock_alerts"
)

settings.add_boolean(
    "send_email_alerts",
    default=True,
    verbose_name="Send Email Alerts",
    depends_on="enable_low_stock_alerts"
)
```

### Setting Categories

For plugins with many settings, you can organize them into categories:

```python
# plugins/inventory_plugin/settings.py
settings.add_boolean(
    "enable_low_stock_alerts",
    default=True,
    verbose_name="Enable Low Stock Alerts",
    category="Alerts"
)

settings.add_boolean(
    "enable_expiration_alerts",
    default=True,
    verbose_name="Enable Expiration Alerts",
    category="Alerts"
)

settings.add_choice(
    "stock_display_format",
    choices=[
        ("quantity", "Show Quantity Only"),
        ("quantity_units", "Show Quantity and Units"),
        ("detailed", "Show Detailed Information")
    ],
    default="quantity_units",
    verbose_name="Stock Display Format",
    category="Display"
)
```

### JSON Settings

For complex settings that don't fit into simple types, you can use JSON settings:

```python
# plugins/inventory_plugin/settings.py
default_alert_recipients = {
    "low_stock": ["inventory_manager", "purchasing_manager"],
    "expiration": ["inventory_manager", "clinical_manager"],
    "stockout": ["inventory_manager", "purchasing_manager", "admin"]
}

settings.add_json(
    "alert_recipients",
    default=default_alert_recipients,
    verbose_name="Alert Recipients",
    help_text="User roles that should receive different types of alerts"
)
```

Accessing JSON settings:

```python
# Get the entire JSON structure
recipients = settings.get_for_organization(organization, "alert_recipients")

# Access a specific part
low_stock_recipients = recipients.get("low_stock", [])

# Loop through recipients
for role in low_stock_recipients:
    users = get_users_with_role(organization, role)
    for user in users:
        send_alert(user, item, "low_stock")
```

## Best Practices for Plugin Settings

1. **Provide Sensible Defaults**: Choose default values that work for most users
2. **Clear Naming**: Use descriptive names and helpful text
3. **Validate Input**: Add validation to prevent invalid settings
4. **Group Related Settings**: Use categories to organize settings
5. **Document Settings**: Clearly document what each setting does
6. **Performance Considerations**: Cache frequently accessed settings
7. **Handle Upgrades**: Consider how to handle settings during plugin upgrades

## Practical Example: Report Generator Plugin

Let's look at a more complex example for a report generator plugin:

```python
# plugins/report_generator/settings.py
from core.plugin_organization_settings import create_settings

settings = create_settings("report_generator")

# General settings
settings.add_boolean(
    "enabled",
    default=True,
    verbose_name="Enable Report Generator",
    category="General"
)

settings.add_choice(
    "default_format",
    choices=[
        ("pdf", "PDF"),
        ("xlsx", "Excel"),
        ("csv", "CSV"),
        ("html", "HTML")
    ],
    default="pdf",
    verbose_name="Default Report Format",
    category="General"
)

# PDF settings
settings.add_text(
    "pdf_header_text",
    default="{organization_name} - {report_name}",
    verbose_name="PDF Header Text",
    help_text="Available variables: {organization_name}, {report_name}, {date}",
    category="PDF Options",
    depends_on="default_format",
    depends_value="pdf"
)

settings.add_boolean(
    "pdf_include_logo",
    default=True,
    verbose_name="Include Organization Logo",
    category="PDF Options",
    depends_on="default_format",
    depends_value="pdf"
)

# Excel settings
settings.add_boolean(
    "excel_freeze_header",
    default=True,
    verbose_name="Freeze Header Row",
    category="Excel Options",
    depends_on="default_format",
    depends_value="xlsx"
)

settings.add_boolean(
    "excel_autofilter",
    default=True,
    verbose_name="Enable Auto-Filters",
    category="Excel Options",
    depends_on="default_format",
    depends_value="xlsx"
)

# Scheduling settings
settings.add_boolean(
    "enable_scheduled_reports",
    default=False,
    verbose_name="Enable Scheduled Reports",
    category="Scheduling"
)

default_schedules = {
    "patient_list": {
        "frequency": "weekly",
        "day": "monday",
        "time": "06:00",
        "format": "pdf",
        "recipients": ["admin"]
    },
    "activity_summary": {
        "frequency": "daily",
        "time": "23:00",
        "format": "xlsx",
        "recipients": ["admin", "clinical_manager"]
    }
}

settings.add_json(
    "report_schedules",
    default=default_schedules,
    verbose_name="Report Schedules",
    help_text="Configuration for scheduled reports",
    category="Scheduling",
    depends_on="enable_scheduled_reports"
)

def validate_settings(setting_name, value, organization):
    """Validate settings before they're saved."""
    if setting_name == "report_schedules" and isinstance(value, dict):
        # Validate each schedule
        for report_type, schedule in value.items():
            if "frequency" not in schedule:
                return False, f"Missing frequency for report '{report_type}'"

            if schedule["frequency"] not in ["daily", "weekly", "monthly"]:
                return False, f"Invalid frequency for report '{report_type}'"

            if schedule["frequency"] == "weekly" and "day" not in schedule:
                return False, f"Missing day for weekly report '{report_type}'"

    return True, None

settings.on_validate(validate_settings)
settings.register()
```

This example demonstrates a plugin with multiple categories of settings, dependent settings, and complex JSON settings with custom validation.

## Next Steps

In the next tutorial, we'll explore implementing plugin permissions, which allow you to control access to your plugin's features based on user roles.
