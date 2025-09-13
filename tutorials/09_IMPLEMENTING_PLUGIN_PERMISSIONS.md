# Implementing Plugin Permissions

This tutorial explores how to implement a permission system for your plugins. You'll learn how to create custom permissions, organize them into groups, and enforce permission checks in your code.

## Introduction to Plugin Permissions

The Ritiko plugin system includes a flexible permissions framework that allows:

- **Fine-grained access control**: Control access to specific plugin features
- **Role-based permissions**: Organize permissions into logical groups
- **Integration with Django's permission system**: Build on Django's authentication system
- **Permission checking helpers**: Easily check permissions in views and templates
- **Organization-specific permissions**: Permissions scoped to specific organizations

## Creating Plugin Permissions

To define permissions for your plugin, create a `permissions.py` file in your plugin directory:

```python
# plugins/inventory_plugin/permissions.py
from core.plugin_permissions import create_permissions

# Create a permissions object for your plugin
permissions = create_permissions("inventory_plugin")

# Define permissions
permissions.add_permission(
    "view_inventory",
    "Can view inventory items",
    category="inventory"
)

permissions.add_permission(
    "add_inventory",
    "Can add inventory items",
    category="inventory"
)

permissions.add_permission(
    "change_inventory",
    "Can modify inventory items",
    category="inventory"
)

permissions.add_permission(
    "delete_inventory",
    "Can delete inventory items",
    category="inventory"
)

permissions.add_permission(
    "view_reports",
    "Can view inventory reports",
    category="reports"
)

permissions.add_permission(
    "export_data",
    "Can export inventory data",
    category="reports"
)

permissions.add_permission(
    "manage_settings",
    "Can manage inventory settings",
    category="administration"
)

# Define permission groups
permissions.add_group(
    "inventory_viewer",
    ["view_inventory", "view_reports"]
)

permissions.add_group(
    "inventory_manager",
    ["view_inventory", "add_inventory", "change_inventory", "view_reports", "export_data"]
)

permissions.add_group(
    "inventory_admin",
    ["view_inventory", "add_inventory", "change_inventory", "delete_inventory",
     "view_reports", "export_data", "manage_settings"]
)

# Register the permissions
permissions.register()
```

## Accessing Permissions in Your Code

You can check permissions in various parts of your code:

### In Views

#### Function-Based Views

```python
# plugins/inventory_plugin/views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from .permissions import permissions

@login_required
def inventory_list(request):
    """View to list inventory items."""
    # Check if user has permission to view inventory
    if not permissions.has_perm(request.user, "view_inventory"):
        messages.error(request, "You don't have permission to view inventory.")
        return redirect('dashboard')

    # Continue with view logic...
    context = {
        'items': get_inventory_items(request.user.organization),
    }

    return render(request, 'inventory_plugin/inventory_list.html', context)

@login_required
def delete_item(request, pk):
    """View to delete an inventory item."""
    # Check if user has permission to delete inventory
    if not permissions.has_perm(request.user, "delete_inventory"):
        messages.error(request, "You don't have permission to delete inventory items.")
        return redirect('inventory_plugin:inventory_list')

    # Continue with deletion logic...
```

#### Using the Permission Decorator

The plugin system provides a convenient decorator for permission checks:

```python
# plugins/inventory_plugin/views.py
from core.plugin_permission_decorators import plugin_permission_required

@login_required
@plugin_permission_required("inventory_plugin.view_inventory")
def inventory_list(request):
    """View to list inventory items."""
    # Permission is already checked by the decorator
    # Continue with view logic...
    context = {
        'items': get_inventory_items(request.user.organization),
    }

    return render(request, 'inventory_plugin/inventory_list.html', context)

@login_required
@plugin_permission_required("inventory_plugin.delete_inventory")
def delete_item(request, pk):
    """View to delete an inventory item."""
    # Permission is already checked by the decorator
    # Continue with deletion logic...
```

#### Class-Based Views

For class-based views, you can use the `PluginPermissionRequiredMixin`:

```python
# plugins/inventory_plugin/views.py
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.contrib.auth.mixins import LoginRequiredMixin
from core.plugin_permissions import PluginPermissionRequiredMixin
from .models import InventoryItem

class InventoryListView(LoginRequiredMixin, PluginPermissionRequiredMixin, ListView):
    """View to list inventory items."""
    model = InventoryItem
    template_name = 'inventory_plugin/inventory_list.html'
    context_object_name = 'items'
    permission_required = "inventory_plugin.view_inventory"

    def get_queryset(self):
        """Filter items by the current user's organization."""
        return InventoryItem.objects.filter(
            organization=self.request.user.organization
        )

class InventoryCreateView(LoginRequiredMixin, PluginPermissionRequiredMixin, CreateView):
    """View to create an inventory item."""
    model = InventoryItem
    template_name = 'inventory_plugin/inventory_form.html'
    fields = ['name', 'description', 'quantity', 'unit', 'price']
    permission_required = "inventory_plugin.add_inventory"

    def form_valid(self, form):
        """Set the organization before saving."""
        form.instance.organization = self.request.user.organization
        return super().form_valid(form)
```

### In Templates

You can check permissions in templates using template tags:

```html
<!-- plugins/inventory_plugin/templates/inventory_plugin/inventory_list.html -->
{% load plugin_permissions %}

<h1>Inventory Items</h1>

<div class="actions">
    {% if request.user|has_plugin_perm:"inventory_plugin.add_inventory" %}
    <a href="{% url 'inventory_plugin:add_item' %}" class="btn btn-primary">
        <i class="fas fa-plus"></i> Add Item
    </a>
    {% endif %}

    {% if request.user|has_plugin_perm:"inventory_plugin.export_data" %}
    <a href="{% url 'inventory_plugin:export_inventory' %}" class="btn btn-secondary">
        <i class="fas fa-file-export"></i> Export
    </a>
    {% endif %}
</div>

<table class="table">
    <thead>
        <tr>
            <th>Name</th>
            <th>Quantity</th>
            <th>Price</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        {% for item in items %}
        <tr>
            <td>{{ item.name }}</td>
            <td>{{ item.quantity }} {{ item.unit }}</td>
            <td>${{ item.price }}</td>
            <td>
                <div class="btn-group">
                    <a href="{% url 'inventory_plugin:view_item' item.pk %}" class="btn btn-sm btn-info">
                        <i class="fas fa-eye"></i>
                    </a>

                    {% if request.user|has_plugin_perm:"inventory_plugin.change_inventory" %}
                    <a href="{% url 'inventory_plugin:edit_item' item.pk %}" class="btn btn-sm btn-primary">
                        <i class="fas fa-edit"></i>
                    </a>
                    {% endif %}

                    {% if request.user|has_plugin_perm:"inventory_plugin.delete_inventory" %}
                    <a href="{% url 'inventory_plugin:delete_item' item.pk %}" class="btn btn-sm btn-danger">
                        <i class="fas fa-trash"></i>
                    </a>
                    {% endif %}
                </div>
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
```

### In Models and Business Logic

You can check permissions in model methods and business logic:

```python
# plugins/inventory_plugin/models.py
from django.db import models

class InventoryItem(models.Model):
    # ... fields ...

    def can_user_modify(self, user):
        """Check if a user can modify this item."""
        from .permissions import permissions
        return permissions.has_perm(user, "change_inventory")

    def can_user_delete(self, user):
        """Check if a user can delete this item."""
        from .permissions import permissions
        return permissions.has_perm(user, "delete_inventory")
```

## Permission Groups

Permission groups allow you to bundle related permissions together for easier assignment:

```python
# plugins/inventory_plugin/permissions.py
# Define permission groups
permissions.add_group(
    "inventory_viewer",
    ["view_inventory", "view_reports"]
)

permissions.add_group(
    "inventory_manager",
    ["view_inventory", "add_inventory", "change_inventory", "view_reports", "export_data"]
)

permissions.add_group(
    "inventory_admin",
    ["view_inventory", "add_inventory", "change_inventory", "delete_inventory",
     "view_reports", "export_data", "manage_settings"]
)
```

These groups will be available in the admin interface, allowing administrators to assign multiple permissions at once by assigning a user to a group.

## Advanced Permission Features

### Custom Permission Checkers

You can create custom permission checking logic:

```python
# plugins/inventory_plugin/permissions.py
def check_export_permission(user, permission_name, **kwargs):
    """Custom permission checker for export permission."""
    # Only allow exports during business hours
    from datetime import datetime
    current_hour = datetime.now().hour

    if permission_name == "export_data" and (current_hour < 8 or current_hour > 18):
        return False

    # Fall back to standard permission check
    return None  # None means "use default checking"

# Register the custom checker
permissions.add_permission_checker(check_export_permission)
```

### Organization-Specific Permissions

The plugin system automatically scopes permissions to organizations. This means a user with the `view_inventory` permission in one organization doesn't automatically have that permission in another organization.

### Permission for Specific Objects

Sometimes you need to check permissions for specific objects, not just general actions:

```python
# plugins/inventory_plugin/views.py
@login_required
def edit_item(request, pk):
    """View to edit an inventory item."""
    item = get_object_or_404(InventoryItem, pk=pk, organization=request.user.organization)

    # Basic permission check
    if not permissions.has_perm(request.user, "change_inventory"):
        messages.error(request, "You don't have permission to edit inventory items.")
        return redirect('inventory_plugin:inventory_list')

    # Additional object-level check
    if item.locked and not permissions.has_perm(request.user, "edit_locked_inventory"):
        messages.error(request, "This item is locked and cannot be edited.")
        return redirect('inventory_plugin:view_item', pk=pk)

    # Continue with edit logic...
```

## Permission Management in Admin Interface

The plugin permissions are automatically integrated with the Ritiko admin interface, allowing administrators to:

- View all available permissions
- Assign permissions to users
- Assign permissions to groups
- View permissions by category
- View permissions by plugin

## Best Practices for Plugin Permissions

1. **Use Descriptive Names**: Choose clear permission names that reflect what they allow
2. **Organize with Categories**: Group related permissions using categories
3. **Create Logical Groups**: Bundle permissions into meaningful groups
4. **Check Permissions Everywhere**: Validate permissions in views, templates, and business logic
5. **Default to Denial**: Start with restricted permissions and only grant what's needed
6. **Document Permissions**: Clearly document what each permission allows
7. **Provide Gradual Access**: Create a hierarchy of permission groups with increasing access

## Practical Example: Patient Records Plugin

Let's look at a more complex permission system for a patient records plugin:

```python
# plugins/patient_records/permissions.py
from core.plugin_permissions import create_permissions

permissions = create_permissions("patient_records")

# Basic record permissions
permissions.add_permission(
    "view_patient_records",
    "Can view patient records",
    category="records"
)

permissions.add_permission(
    "add_patient_records",
    "Can add patient records",
    category="records"
)

permissions.add_permission(
    "change_patient_records",
    "Can modify patient records",
    category="records"
)

# Specialized permissions
permissions.add_permission(
    "view_sensitive_data",
    "Can view sensitive patient data",
    category="sensitive"
)

permissions.add_permission(
    "view_billing_data",
    "Can view patient billing data",
    category="financial"
)

permissions.add_permission(
    "approve_record_changes",
    "Can approve changes to patient records",
    category="records"
)

# Administrative permissions
permissions.add_permission(
    "manage_record_templates",
    "Can manage record templates",
    category="administration"
)

permissions.add_permission(
    "view_audit_logs",
    "Can view record audit logs",
    category="administration"
)

# Define permission groups
permissions.add_group(
    "records_viewer",
    ["view_patient_records"]
)

permissions.add_group(
    "records_editor",
    ["view_patient_records", "add_patient_records", "change_patient_records"]
)

permissions.add_group(
    "clinical_staff",
    ["view_patient_records", "add_patient_records", "change_patient_records",
     "view_sensitive_data"]
)

permissions.add_group(
    "billing_staff",
    ["view_patient_records", "view_billing_data"]
)

permissions.add_group(
    "records_admin",
    ["view_patient_records", "add_patient_records", "change_patient_records",
     "view_sensitive_data", "view_billing_data", "approve_record_changes",
     "manage_record_templates", "view_audit_logs"]
)

# Custom permission checker
def check_record_permissions(user, permission_name, **kwargs):
    """Custom permission checker that considers record type."""
    if not kwargs.get('record'):
        return None  # No record provided, use default checking

    record = kwargs['record']

    # Special handling for mental health records
    if record.is_mental_health_record:
        # Only allow mental health staff to access these records
        if permission_name.startswith('view_') and not user.is_mental_health_staff:
            return False

    # Special handling for VIP patients
    if record.patient.is_vip:
        # Only senior staff can modify VIP patient records
        if permission_name.startswith('change_') and not user.is_senior_staff:
            return False

    # Use default checking for other cases
    return None

# Register the custom checker
permissions.add_permission_checker(check_record_permissions)

# Register all permissions
permissions.register()
```

This example demonstrates a more complex permission system with:
- Multiple permission categories
- Specialized permission groups for different staff roles
- Custom permission checking logic that considers record types

## Next Steps

In the next tutorial, we'll explore plugin menu registration, which allows you to integrate your plugin with the Ritiko navigation system.
