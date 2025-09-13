# Advanced Plugin Features

This tutorial explores advanced features and techniques for Ritiko plugins. You'll learn about signals, lifecycle hooks, custom admin integrations, and other advanced capabilities that can enhance your plugins.

## Django Signals

Django signals allow decoupled applications to get notified when certain actions occur. Ritiko plugins can use signals to respond to system events without modifying core code.

### Using Existing Signals

```python
# plugins/audit_plugin/signals.py
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from patients.models.people import Patient
from appointments.models import Appointment
from .models import AuditLog

@receiver(post_save, sender=Patient)
def log_patient_changes(sender, instance, created, **kwargs):
    """Log patient creations and updates."""
    action = "created" if created else "updated"
    AuditLog.objects.create(
        content_object=instance,
        action=action,
        user=kwargs.get('user', instance.updated_by),
        organization=instance.organization
    )

@receiver(post_delete, sender=Patient)
def log_patient_deletion(sender, instance, **kwargs):
    """Log patient deletions."""
    AuditLog.objects.create(
        content_type=ContentType.objects.get_for_model(sender),
        object_id=instance.id,
        action="deleted",
        user=kwargs.get('user'),
        organization=instance.organization,
        object_repr=str(instance)
    )
```

### Creating Custom Signals

You can define custom signals for plugin-specific events:

```python
# plugins/inventory_plugin/signals.py
from django.dispatch import Signal, receiver

# Define custom signals
low_stock_detected = Signal()  # Provides ['item', 'organization']
item_restocked = Signal()     # Provides ['item', 'old_quantity', 'new_quantity']

# Connect to custom signals
@receiver(low_stock_detected)
def handle_low_stock(sender, item, organization, **kwargs):
    """Handle low stock notification."""
    from .models import StockAlert

    # Create alert
    StockAlert.objects.create(
        item=item,
        organization=organization,
        quantity=item.quantity,
        threshold=item.reorder_threshold
    )

    # Send notification
    from .notifications import send_low_stock_notification
    send_low_stock_notification(item)

# Example of sending a signal
def check_stock_levels(organization):
    """Check for low stock items and trigger signals."""
    from .models import InventoryItem
    from .settings import settings

    # Get threshold from settings
    threshold = settings.get_for_organization(
        organization,
        "low_stock_threshold"
    )

    # Find low stock items
    low_stock_items = InventoryItem.objects.filter(
        organization=organization,
        quantity__lt=threshold
    )

    # Send signal for each low stock item
    for item in low_stock_items:
        low_stock_detected.send(
            sender=check_stock_levels,
            item=item,
            organization=organization
        )
```

## Plugin Lifecycle Hooks

Ritiko provides hooks for different stages of the plugin lifecycle, allowing you to execute code at specific times.

### Initialization Hook

The `ready()` method in your plugin's `AppConfig` is called when Django initializes:

```python
# plugins/inventory_plugin/apps.py
from core.plugin_app_config import PluginAppConfig

class InventoryPluginConfig(PluginAppConfig):
    name = "plugins.inventory_plugin"
    verbose_name = "Inventory Plugin"

    def ready(self):
        """
        Called when Django starts. Use this to:
        - Initialize plugin components
        - Register signals
        - Set up background tasks
        """
        super().ready()

        # Import signals module to register signal handlers
        from . import signals

        # Initialize integrations
        from . import integrations
        integrations.initialize()

        # Set up scheduled tasks
        from . import tasks
        tasks.schedule_recurring_tasks()

        print(f"✅ {self.verbose_name} is ready")
```

### Organization Creation Hook

You can respond when a new organization is created:

```python
# plugins/inventory_plugin/hooks.py
from core.signals import organization_created

@receiver(organization_created)
def setup_new_organization(sender, organization, **kwargs):
    """Set up inventory system for new organizations."""
    from .models import InventoryCategory

    # Create default categories
    default_categories = [
        {"name": "Medical Supplies", "slug": "medical-supplies"},
        {"name": "Office Supplies", "slug": "office-supplies"},
        {"name": "Medications", "slug": "medications"},
        {"name": "Equipment", "slug": "equipment"}
    ]

    for category in default_categories:
        InventoryCategory.objects.create(
            name=category["name"],
            slug=category["slug"],
            organization=organization
        )

    # Initialize settings with organization-specific values
    from .settings import settings
    settings.initialize_for_organization(organization)
```

### Plugin Activation/Deactivation

Respond when your plugin is enabled or disabled:

```python
# plugins/inventory_plugin/apps.py
class InventoryPluginConfig(PluginAppConfig):
    # ... other methods ...

    def on_plugin_enabled(self, organization):
        """Called when the plugin is enabled for an organization."""
        from .tasks import import_initial_inventory
        import_initial_inventory.delay(organization.id)

    def on_plugin_disabled(self, organization):
        """Called when the plugin is disabled for an organization."""
        from .tasks import cancel_all_scheduled_tasks
        cancel_all_scheduled_tasks(organization.id)
```

## Custom Admin Integrations

You can extend the Ritiko admin interface with custom admin pages and components.

### Custom Admin Views

```python
# plugins/inventory_plugin/admin_views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from django.contrib.auth.decorators import user_passes_test
from core.decorators import staff_member_required
from .forms import InventoryImportForm

@staff_member_required
def import_inventory(request):
    """Admin view for importing inventory from CSV."""
    if request.method == 'POST':
        form = InventoryImportForm(request.POST, request.FILES)
        if form.is_valid():
            # Process the import
            from .admin_utils import process_inventory_import
            result = process_inventory_import(
                request.FILES['csv_file'],
                request.user.organization,
                form.cleaned_data['update_existing']
            )

            messages.success(
                request,
                f"Import complete: {result['created']} items created, "
                f"{result['updated']} items updated, {result['errors']} errors."
            )
            return redirect('admin:inventory_plugin_inventoryitem_changelist')
    else:
        form = InventoryImportForm()

    return render(request, 'admin/inventory_plugin/import.html', {
        'form': form,
        'title': 'Import Inventory'
    })
```

### Admin URL Integration

Register your admin views in your plugin's URLs:

```python
# plugins/inventory_plugin/urls.py
from django.urls import path
from . import views, admin_views

app_name = 'inventory_plugin'

urlpatterns = [
    # Regular plugin views
    path('', views.dashboard, name='dashboard'),
    # ...
]

# Admin-specific URLs
admin_urlpatterns = [
    path('import/', admin_views.import_inventory, name='import_inventory'),
    path('export/', admin_views.export_inventory, name='export_inventory'),
    path('reports/', admin_views.inventory_reports, name='inventory_reports'),
]
```

### Custom Admin Actions

Add custom actions to the admin list view:

```python
# plugins/inventory_plugin/admin.py
from django.contrib import admin
from .models import InventoryItem

@admin.register(InventoryItem)
class InventoryItemAdmin(admin.ModelAdmin):
    list_display = ['name', 'category', 'quantity', 'unit', 'price', 'last_updated']
    list_filter = ['category', 'in_stock']
    search_fields = ['name', 'sku']

    actions = ['mark_as_in_stock', 'mark_as_out_of_stock', 'generate_order']

    def mark_as_in_stock(self, request, queryset):
        """Mark selected items as in stock."""
        updated = queryset.update(in_stock=True)
        self.message_user(
            request,
            f"{updated} items marked as in stock."
        )
    mark_as_in_stock.short_description = "Mark selected items as in stock"

    def mark_as_out_of_stock(self, request, queryset):
        """Mark selected items as out of stock."""
        updated = queryset.update(in_stock=False)
        self.message_user(
            request,
            f"{updated} items marked as out of stock."
        )
    mark_as_out_of_stock.short_description = "Mark selected items as out of stock"

    def generate_order(self, request, queryset):
        """Generate purchase order for selected items."""
        from .utils import create_purchase_order

        # Filter to items below reorder threshold
        items_to_reorder = [
            item for item in queryset
            if item.quantity < item.reorder_threshold
        ]

        if not items_to_reorder:
            self.message_user(
                request,
                "No items below reorder threshold selected.",
                level=messages.WARNING
            )
            return

        # Create purchase order
        order = create_purchase_order(items_to_reorder, request.user)

        # Redirect to the order detail page
        return redirect('inventory_plugin:purchase_order_detail', pk=order.pk)
    generate_order.short_description = "Generate purchase order for selected items"
```

## Background Tasks

For long-running or scheduled operations, you can use background tasks.

### Using Celery Tasks

```python
# plugins/inventory_plugin/tasks.py
from celery import shared_task
from celery.schedules import crontab
from django.core.mail import send_mail
from celery.task.base import periodic_task

@shared_task
def process_inventory_import(file_path, organization_id, update_existing=False):
    """Process inventory import in the background."""
    from .models import Organization, InventoryItem, InventoryCategory
    from .utils import parse_csv

    organization = Organization.objects.get(id=organization_id)

    # Process the import (simplified example)
    items = parse_csv(file_path)
    result = {'created': 0, 'updated': 0, 'errors': 0}

    for item_data in items:
        try:
            # Create or update item
            item, created = InventoryItem.objects.update_or_create(
                sku=item_data['sku'],
                organization=organization,
                defaults=item_data
            )

            if created:
                result['created'] += 1
            else:
                result['updated'] += 1

        except Exception as e:
            result['errors'] += 1
            print(f"Error importing item {item_data.get('sku')}: {e}")

    return result

@periodic_task(run_every=crontab(hour=0, minute=0))  # Run at midnight
def check_low_stock_daily():
    """Daily check for low stock items."""
    from .models import Organization, InventoryItem
    from .settings import settings
    from django.db.models import F

    # Check each organization
    for org in Organization.objects.filter(is_active=True):
        # Get threshold from settings
        threshold = settings.get_for_organization(org, "low_stock_threshold")

        # Find low stock items
        low_stock_items = InventoryItem.objects.filter(
            organization=org,
            quantity__lt=threshold
        )

        if low_stock_items.exists():
            # Send notification email
            send_low_stock_notification(org, low_stock_items)

def send_low_stock_notification(organization, items):
    """Send notification about low stock items."""
    # Get notification email from settings
    from .settings import settings
    email = settings.get_for_organization(
        organization,
        "notification_email"
    )

    if not email:
        return

    # Build email content
    subject = f"Low Stock Alert - {organization.name}"
    message = "The following items are low in stock:\n\n"

    for item in items:
        message += f"- {item.name}: {item.quantity} {item.unit} remaining\n"

    message += "\nPlease restock these items soon."

    # Send email
    send_mail(
        subject=subject,
        message=message,
        from_email="inventory@example.com",
        recipient_list=[email],
        fail_silently=False
    )
```

## Plugin Settings Migration

For handling settings changes when upgrading your plugin:

```python
# plugins/inventory_plugin/migrations.py
def migrate_settings(organization):
    """Migrate settings from v1.0 to v2.0."""
    from .settings import settings

    # Check if organization has old settings
    if settings.has_for_organization(organization, "email_notifications"):
        # Get old setting
        old_value = settings.get_for_organization(
            organization,
            "email_notifications"
        )

        # Set new settings based on old value
        settings.set_for_organization(
            organization,
            "enable_low_stock_alerts",
            old_value
        )

        settings.set_for_organization(
            organization,
            "enable_expiration_alerts",
            old_value
        )

        # Remove old setting
        settings.delete_for_organization(
            organization,
            "email_notifications"
        )

# In your plugin's apps.py
def ready(self):
    super().ready()

    # Register settings migration
    from .settings import settings
    from .migrations import migrate_settings

    settings.register_migration(
        from_version="1.0",
        to_version="2.0",
        migration_func=migrate_settings
    )
```

## Custom Template Tags and Filters

Create custom template tags and filters for your plugin:

```python
# plugins/inventory_plugin/templatetags/inventory_tags.py
from django import template
from django.utils.safestring import mark_safe

register = template.Library()

@register.simple_tag
def inventory_status(item):
    """Display inventory status with color-coded indicator."""
    if item.quantity <= 0:
        return mark_safe(
            '<span class="badge badge-danger">Out of Stock</span>'
        )
    elif item.quantity < item.reorder_threshold:
        return mark_safe(
            f'<span class="badge badge-warning">Low Stock ({item.quantity})</span>'
        )
    else:
        return mark_safe(
            f'<span class="badge badge-success">In Stock ({item.quantity})</span>'
        )

@register.filter
def currency(value):
    """Format a number as currency."""
    if value is None:
        return "$0.00"
    return f"${value:.2f}"

@register.inclusion_tag('inventory_plugin/tags/stock_chart.html')
def stock_chart(item, days=30):
    """Generate a stock history chart for an item."""
    # Get stock history
    from datetime import timedelta
    from django.utils import timezone

    start_date = timezone.now().date() - timedelta(days=days)

    history = item.stock_history.filter(
        date__gte=start_date
    ).order_by('date')

    # Prepare data for chart
    dates = [h.date.strftime('%Y-%m-%d') for h in history]
    quantities = [h.quantity for h in history]

    return {
        'item': item,
        'dates': dates,
        'quantities': quantities,
        'days': days
    }
```

## REST API Extensions

You can extend the Ritiko REST API with your plugin's endpoints:

```python
# plugins/inventory_plugin/api.py
from rest_framework import viewsets, permissions
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import InventoryItem, InventoryCategory
from .serializers import InventoryItemSerializer, InventoryCategorySerializer
from core.permissions import IsOrganizationMember

class InventoryItemViewSet(viewsets.ModelViewSet):
    """API endpoint for inventory items."""
    serializer_class = InventoryItemSerializer
    permission_classes = [permissions.IsAuthenticated, IsOrganizationMember]

    def get_queryset(self):
        """Filter queryset by organization."""
        return InventoryItem.objects.filter(
            organization=self.request.user.organization
        )

    @action(detail=False, methods=['get'])
    def low_stock(self, request):
        """Get low stock items."""
        from .settings import settings

        # Get threshold from settings
        threshold = settings.get_for_organization(
            request.user.organization,
            "low_stock_threshold"
        )

        # Filter items
        items = self.get_queryset().filter(quantity__lt=threshold)
        serializer = self.get_serializer(items, many=True)

        return Response(serializer.data)

    @action(detail=True, methods=['post'])
    def adjust_stock(self, request, pk=None):
        """Adjust stock level for an item."""
        item = self.get_object()

        # Get adjustment amount from request
        amount = request.data.get('amount', 0)

        if not amount:
            return Response({"error": "Amount is required"}, status=400)

        # Adjust stock
        try:
            amount = int(amount)
            old_quantity = item.quantity
            item.quantity += amount
            item.save()

            # Log adjustment
            from .models import StockAdjustment
            StockAdjustment.objects.create(
                item=item,
                quantity=amount,
                previous_quantity=old_quantity,
                adjusted_by=request.user,
                organization=request.user.organization
            )

            return Response({
                "success": True,
                "old_quantity": old_quantity,
                "new_quantity": item.quantity,
                "adjustment": amount
            })

        except ValueError:
            return Response({"error": "Invalid amount"}, status=400)
```

Register your API endpoints:

```python
# plugins/inventory_plugin/urls.py
from rest_framework.routers import DefaultRouter
from . import api

# Create a router for API endpoints
router = DefaultRouter()
router.register(r'items', api.InventoryItemViewSet, basename='inventory-item')
router.register(r'categories', api.InventoryCategoryViewSet, basename='inventory-category')

# Include API URLs
urlpatterns = [
    # Regular views
    # ...

    # API endpoints
    path('api/', include(router.urls)),
]
```

## Integration with External Services

You can integrate your plugin with external services:

```python
# plugins/inventory_plugin/integrations/shopify.py
import requests
from ..settings import settings

def sync_with_shopify(organization):
    """Sync inventory with Shopify."""
    # Get API credentials from settings
    api_key = settings.get_for_organization(organization, "shopify_api_key")
    shop_url = settings.get_for_organization(organization, "shopify_shop_url")

    if not (api_key and shop_url):
        raise ValueError("Shopify API credentials not configured")

    # Get inventory items
    from ..models import InventoryItem
    items = InventoryItem.objects.filter(organization=organization)

    # Update Shopify inventory
    for item in items:
        if not item.shopify_inventory_id:
            continue

        # Send update to Shopify
        response = requests.put(
            f"{shop_url}/admin/api/2022-01/inventory_levels/set.json",
            headers={
                "X-Shopify-Access-Token": api_key,
                "Content-Type": "application/json"
            },
            json={
                "inventory_item_id": item.shopify_inventory_id,
                "location_id": item.shopify_location_id,
                "available": item.quantity
            }
        )

        if response.status_code != 200:
            print(f"Error updating Shopify inventory for {item.name}: {response.text}")
```

## Event-Driven Architecture

You can implement an event-driven architecture using Django signals:

```python
# plugins/inventory_plugin/events.py
from django.dispatch import Signal

# Define events
item_created = Signal()  # provides 'item', 'user', 'organization'
item_updated = Signal()  # provides 'item', 'user', 'organization', 'changes'
item_deleted = Signal()  # provides 'item_id', 'user', 'organization'
stock_adjusted = Signal()  # provides 'item', 'old_quantity', 'new_quantity', 'user'
low_stock_detected = Signal()  # provides 'item', 'threshold'
reorder_suggested = Signal()  # provides 'item', 'organization'

# Example event producer
def adjust_stock(item, quantity, user):
    """Adjust stock level and emit events."""
    old_quantity = item.quantity
    item.quantity += quantity
    item.save()

    # Emit stock adjusted event
    stock_adjusted.send(
        sender=adjust_stock,
        item=item,
        old_quantity=old_quantity,
        new_quantity=item.quantity,
        user=user
    )

    # Check if we've gone below threshold
    if (old_quantity >= item.reorder_threshold and
            item.quantity < item.reorder_threshold):
        low_stock_detected.send(
            sender=adjust_stock,
            item=item,
            threshold=item.reorder_threshold
        )
```

## Best Practices for Advanced Features

1. **Use Signals Carefully**: Signals can make code harder to debug - use them for truly decoupled functionality
2. **Background Task Considerations**: Make background tasks idempotent (safe to run multiple times)
3. **Error Handling**: Implement robust error handling for integrations with external services
4. **API Documentation**: Document your API endpoints for other developers
5. **Performance**: Be mindful of performance implications, especially for background tasks
6. **Testability**: Design your advanced features to be testable
7. **Maintainability**: Document complex features for future maintenance

## Next Steps

In the next tutorial, we'll explore testing and debugging plugins, covering unit tests, integration tests, and debugging techniques.
