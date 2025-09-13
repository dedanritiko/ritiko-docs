# Plugin Menu Registration

This tutorial explores how to integrate your plugin with the Ritiko navigation system by registering custom menu items. You'll learn how to create menu entries, organize them into hierarchical structures, and control their visibility based on permissions.

## Introduction to Plugin Menus

The Ritiko plugin system includes a flexible menu framework that allows:

- **Seamless integration**: Add your plugin's pages to the main navigation
- **Hierarchical organization**: Create nested menu structures with submenus
- **Permission-based visibility**: Show menu items only to users with specific permissions
- **Icon support**: Include icons for visual recognition
- **Badge support**: Add notification badges to menu items
- **Dynamic content**: Update menu items based on application state

## Creating Plugin Menus

To define menus for your plugin, create a `menus.py` file in your plugin directory:

```python
# plugins/inventory_plugin/menus.py
from core.plugin_menus import create_menu

# Create a menu object for your plugin
menu = create_menu("inventory_plugin")

# Add a main menu item
menu.add_item(
    name="Inventory",
    url_name="inventory_plugin:dashboard",
    icon="fas fa-boxes",
    order=100
)

# Add a submenu
inventory_submenu = menu.add_submenu(
    name="Inventory",
    icon="fas fa-boxes",
    order=100
)

# Add items to the submenu
inventory_submenu.add_item(
    name="Dashboard",
    url_name="inventory_plugin:dashboard",
    order=10
)

inventory_submenu.add_item(
    name="All Items",
    url_name="inventory_plugin:item_list",
    order=20
)

inventory_submenu.add_item(
    name="Low Stock",
    url_name="inventory_plugin:low_stock",
    order=30,
    badge_function=get_low_stock_count
)

inventory_submenu.add_item(
    name="Categories",
    url_name="inventory_plugin:category_list",
    order=40
)

# Add a separator
inventory_submenu.add_separator(order=50)

# Add more items after the separator
inventory_submenu.add_item(
    name="Reports",
    url_name="inventory_plugin:reports",
    icon="fas fa-chart-bar",
    order=60,
    permission="inventory_plugin.view_reports"
)

inventory_submenu.add_item(
    name="Settings",
    url_name="inventory_plugin:settings",
    icon="fas fa-cog",
    order=70,
    permission="inventory_plugin.manage_settings"
)

# Define the function to get the badge count
def get_low_stock_count(request):
    """Get the count of low stock items for the badge."""
    if not request.user.is_authenticated:
        return None

    from .models import InventoryItem
    from .settings import settings

    # Get the threshold from settings
    threshold = settings.get_for_organization(
        request.user.organization,
        "low_stock_threshold"
    )

    # Count items below threshold
    count = InventoryItem.objects.filter(
        organization=request.user.organization,
        quantity__lt=threshold
    ).count()

    # Return None if no low stock items (no badge will be shown)
    if count == 0:
        return None

    return count

# Register the menu
menu.register()
```

## Menu Item Properties

Menu items support various properties:

| Property | Description | Example |
|----------|-------------|---------|
| `name` | Display name of the menu item | `"Inventory"` |
| `url_name` | URL name for the link | `"inventory_plugin:dashboard"` |
| `icon` | FontAwesome icon class | `"fas fa-boxes"` |
| `order` | Sorting order (lower numbers appear first) | `100` |
| `permission` | Required permission to see this item | `"inventory_plugin.view_inventory"` |
| `badge_function` | Function that returns badge content | `get_low_stock_count` |
| `badge_class` | CSS class for the badge | `"badge-danger"` |
| `children` | Submenu items (added automatically) | |
| `active_function` | Function to determine if item is active | `custom_active_check` |

## Menu Sections

The Ritiko system supports different menu sections:

```python
# Create menu for main sidebar
menu = create_menu("inventory_plugin", section="main")

# Create menu for user dropdown
user_menu = create_menu("inventory_plugin", section="user")
user_menu.add_item(
    name="My Inventory Profile",
    url_name="inventory_plugin:user_profile",
    icon="fas fa-user-cog",
    order=10
)

# Create menu for admin section
admin_menu = create_menu("inventory_plugin", section="admin")
admin_menu.add_item(
    name="Inventory Administration",
    url_name="inventory_plugin:admin",
    icon="fas fa-toolbox",
    order=10
)
```

## Permission-Based Visibility

You can control menu visibility using permissions:

```python
# Only users with the view_reports permission can see this item
menu.add_item(
    name="Reports",
    url_name="inventory_plugin:reports",
    icon="fas fa-chart-bar",
    permission="inventory_plugin.view_reports",
    order=20
)

# Complex permission check with a function
def can_see_advanced_reports(request):
    """Check if user can see advanced reports."""
    if not request.user.is_authenticated:
        return False

    from .permissions import permissions
    has_perm = permissions.has_perm(request.user, "view_advanced_reports")

    # Additional business logic
    is_business_hours = 9 <= datetime.now().hour <= 17

    return has_perm and is_business_hours

menu.add_item(
    name="Advanced Reports",
    url_name="inventory_plugin:advanced_reports",
    icon="fas fa-chart-line",
    permission_function=can_see_advanced_reports,
    order=30
)
```

## Dynamic Badges

Badges can display dynamic information like counts or status indicators:

```python
def get_alerts_badge(request):
    """Get the number of active alerts for the badge."""
    if not request.user.is_authenticated:
        return None

    from .models import InventoryAlert

    # Count unresolved alerts
    count = InventoryAlert.objects.filter(
        organization=request.user.organization,
        resolved=False
    ).count()

    if count == 0:
        return None  # No badge if no alerts

    return count

menu.add_item(
    name="Alerts",
    url_name="inventory_plugin:alerts",
    icon="fas fa-bell",
    badge_function=get_alerts_badge,
    badge_class="badge-warning",  # Custom badge style
    order=40
)
```

## Custom Active State

You can customize when a menu item appears active:

```python
def is_inventory_section_active(request):
    """Check if we're in any inventory section."""
    path = request.path.lstrip('/')
    return path.startswith('inventory') or path.startswith('items')

menu.add_item(
    name="Inventory",
    url_name="inventory_plugin:dashboard",
    icon="fas fa-boxes",
    active_function=is_inventory_section_active,
    order=10
)
```

## External Links

You can add external links to your menu:

```python
menu.add_item(
    name="Documentation",
    url="https://docs.example.com/inventory",
    icon="fas fa-book",
    external=True,  # Opens in new tab
    order=200
)
```

## Organization with Separators

Separators help organize menu items visually:

```python
submenu = menu.add_submenu("Administration", icon="fas fa-cogs")

submenu.add_item("Settings", "inventory_plugin:settings")
submenu.add_item("User Access", "inventory_plugin:user_access")

submenu.add_separator()  # Visual divider

submenu.add_item("Import Data", "inventory_plugin:import")
submenu.add_item("Export Data", "inventory_plugin:export")

submenu.add_separator()  # Another divider

submenu.add_item("System Status", "inventory_plugin:status")
```

## Accessing URL Parameters

For dynamic URLs with parameters, use the `args` or `kwargs` properties:

```python
# For a URL pattern like 'item/<int:pk>/'
menu.add_item(
    name="Sample Item",
    url_name="inventory_plugin:item_detail",
    kwargs={"pk": 1},
    order=50
)

# For a URL pattern with multiple parameters
menu.add_item(
    name="Department Items",
    url_name="inventory_plugin:department_items",
    kwargs={"department_id": 5, "category": "medical"},
    order=60
)
```

## Custom Menu Rendering

If you need more control over menu rendering, you can provide a custom template:

```python
menu = create_menu(
    "inventory_plugin",
    template="inventory_plugin/custom_menu.html"
)
```

## Setting Up Menus in the App Configuration

You can also set up menus in your plugin's `apps.py` file:

```python
# plugins/inventory_plugin/apps.py
from core.plugin_app_config import PluginAppConfig

class InventoryPluginConfig(PluginAppConfig):
    name = "plugins.inventory_plugin"
    verbose_name = "Inventory Plugin"

    def ready(self):
        super().ready()
        # Import other modules that need to be initialized
        from . import models, views, signals

    def setup_menu(self):
        """Set up the plugin's menu."""
        from core.plugin_menus import create_menu

        # Create main menu
        menu = create_menu(self.name)

        # Add main item
        menu.add_item(
            name="Inventory",
            url_name=f"{self.name}:dashboard",
            icon="fas fa-boxes",
            order=100
        )

        # Add submenu
        submenu = menu.add_submenu(
            name="Inventory Management",
            icon="fas fa-box-open",
            order=10
        )

        # Add submenu items
        submenu.add_item(
            name="All Items",
            url_name=f"{self.name}:item_list",
            order=10
        )

        submenu.add_item(
            name="Categories",
            url_name=f"{self.name}:category_list",
            order=20
        )

        # Register the menu
        menu.register()
```

## Best Practices for Plugin Menus

1. **Use Clear Names**: Choose concise, clear menu item names
2. **Add Icons**: Include relevant icons for visual recognition
3. **Organize Logically**: Group related functionality in submenus
4. **Control Visibility**: Show menu items only to users who have permission to use them
5. **Prioritize Order**: Place commonly used items higher in the menu
6. **Use Badges Sparingly**: Only add badges for important notifications
7. **Consistent Naming**: Use consistent terminology across your plugin
8. **Limit Menu Depth**: Avoid deeply nested menus (no more than 2 levels)

## Practical Example: Comprehensive Healthcare Plugin

Let's look at a more complex menu structure for a healthcare plugin:

```python
# plugins/healthcare_plugin/menus.py
from core.plugin_menus import create_menu
from datetime import datetime, timedelta

# Main sidebar menu
menu = create_menu("healthcare_plugin")

# Patient section
patient_menu = menu.add_submenu(
    name="Patients",
    icon="fas fa-user-injured",
    order=10
)

patient_menu.add_item(
    name="Patient List",
    url_name="healthcare_plugin:patient_list",
    order=10
)

patient_menu.add_item(
    name="Add Patient",
    url_name="healthcare_plugin:patient_create",
    permission="healthcare_plugin.add_patient",
    order=20
)

patient_menu.add_item(
    name="Today's Appointments",
    url_name="healthcare_plugin:todays_appointments",
    badge_function=get_todays_appointment_count,
    order=30
)

# Clinical section
clinical_menu = menu.add_submenu(
    name="Clinical",
    icon="fas fa-stethoscope",
    order=20,
    permission="healthcare_plugin.access_clinical"
)

clinical_menu.add_item(
    name="Encounter Notes",
    url_name="healthcare_plugin:encounter_list",
    order=10
)

clinical_menu.add_item(
    name="Prescriptions",
    url_name="healthcare_plugin:prescription_list",
    order=20
)

clinical_menu.add_item(
    name="Lab Orders",
    url_name="healthcare_plugin:lab_order_list",
    badge_function=get_pending_lab_count,
    badge_class="badge-info",
    order=30
)

clinical_menu.add_separator(order=40)

clinical_menu.add_item(
    name="Clinical Guidelines",
    url_name="healthcare_plugin:guidelines",
    icon="fas fa-book-medical",
    order=50
)

# Administrative section
admin_menu = menu.add_submenu(
    name="Administration",
    icon="fas fa-cogs",
    order=30,
    permission="healthcare_plugin.access_admin"
)

admin_menu.add_item(
    name="Staff Management",
    url_name="healthcare_plugin:staff_list",
    order=10
)

admin_menu.add_item(
    name="Scheduling",
    url_name="healthcare_plugin:scheduling",
    order=20
)

admin_menu.add_item(
    name="Reports",
    url_name="healthcare_plugin:reports",
    order=30
)

admin_menu.add_separator(order=40)

admin_menu.add_item(
    name="Settings",
    url_name="healthcare_plugin:settings",
    icon="fas fa-wrench",
    order=50
)

# Badge functions
def get_todays_appointment_count(request):
    """Get count of today's appointments for badge."""
    if not request.user.is_authenticated:
        return None

    from .models import Appointment

    today = datetime.now().date()

    count = Appointment.objects.filter(
        organization=request.user.organization,
        date=today,
        status='scheduled'
    ).count()

    return count if count > 0 else None

def get_pending_lab_count(request):
    """Get count of pending lab results for badge."""
    if not request.user.is_authenticated:
        return None

    from .models import LabOrder

    count = LabOrder.objects.filter(
        organization=request.user.organization,
        status='pending',
        ordered_date__gte=datetime.now().date() - timedelta(days=7)
    ).count()

    return count if count > 0 else None

# Register main menu
menu.register()

# User dropdown menu
user_menu = create_menu("healthcare_plugin", section="user")

user_menu.add_item(
    name="My Schedule",
    url_name="healthcare_plugin:my_schedule",
    icon="fas fa-calendar-day",
    order=10
)

user_menu.add_item(
    name="My Patients",
    url_name="healthcare_plugin:my_patients",
    icon="fas fa-users",
    order=20
)

# Register user menu
user_menu.register()
```

This example demonstrates a comprehensive menu structure with multiple sections, permission-based visibility, badges, and separators for a healthcare plugin.

## Next Steps

In the next tutorial, we'll explore advanced plugin features, including signals, hooks, and custom admin integrations.
