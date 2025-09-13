# Testing and Debugging Plugins

This tutorial covers techniques for testing and debugging Ritiko plugins. You'll learn how to write unit tests, integration tests, and how to troubleshoot common issues that arise during plugin development.

## Introduction to Plugin Testing

Testing is a critical part of plugin development that ensures your code works correctly and continues to work as the system evolves. Effective testing:

- **Validates functionality**: Ensures your plugin does what it's supposed to do
- **Prevents regressions**: Catches bugs introduced by new changes
- **Documents behavior**: Tests serve as executable documentation
- **Improves design**: Well-tested code tends to be better designed
- **Facilitates refactoring**: Allows you to improve code with confidence

## Setting Up a Testing Environment

Before writing tests, you need to set up a proper testing environment:

```python
# plugins/inventory_plugin/tests/test_settings.py

# Django test settings
TEST_RUNNER = 'django.test.runner.DiscoverRunner'

# Use an in-memory database for faster tests
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',
    }
}

# Disable migrations during tests
class DisableMigrations:
    def __contains__(self, item):
        return True

    def __getitem__(self, item):
        return None

MIGRATION_MODULES = DisableMigrations()
```

## Unit Testing

Unit tests focus on testing individual components in isolation.

### Testing Models

```python
# plugins/inventory_plugin/tests/test_models.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from ..models import InventoryItem, InventoryCategory
from orgs.models import Organization

User = get_user_model()

class InventoryItemTestCase(TestCase):
    """Tests for the InventoryItem model."""

    def setUp(self):
        """Set up test data."""
        # Create a test organization
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

        # Create a test user
        self.user = User.objects.create_user(
            username="testuser",
            email="test@example.com",
            password="password"
        )
        self.user.organization = self.organization
        self.user.save()

        # Create a test category
        self.category = InventoryCategory.objects.create(
            name="Test Category",
            slug="test-category",
            organization=self.organization
        )

        # Create a test item
        self.item = InventoryItem.objects.create(
            name="Test Item",
            sku="TEST001",
            description="This is a test item",
            quantity=10,
            unit="pieces",
            price=19.99,
            reorder_threshold=5,
            category=self.category,
            organization=self.organization
        )

    def test_item_creation(self):
        """Test that item was created correctly."""
        self.assertEqual(self.item.name, "Test Item")
        self.assertEqual(self.item.sku, "TEST001")
        self.assertEqual(self.item.quantity, 10)
        self.assertEqual(self.item.organization, self.organization)

    def test_string_representation(self):
        """Test the string representation of the item."""
        self.assertEqual(str(self.item), "Test Item")

    def test_is_low_stock(self):
        """Test the is_low_stock method."""
        # Not low stock initially
        self.assertFalse(self.item.is_low_stock())

        # Reduce quantity below threshold
        self.item.quantity = 3
        self.item.save()

        # Should be low stock now
        self.assertTrue(self.item.is_low_stock())

    def test_total_value(self):
        """Test the total_value property."""
        # 10 pieces * $19.99 = $199.90
        self.assertAlmostEqual(self.item.total_value, 199.90)

        # Change quantity
        self.item.quantity = 5
        self.item.save()

        # 5 pieces * $19.99 = $99.95
        self.assertAlmostEqual(self.item.total_value, 99.95)
```

### Testing Forms

```python
# plugins/inventory_plugin/tests/test_forms.py
from django.test import TestCase
from ..forms import InventoryItemForm, StockAdjustmentForm
from orgs.models import Organization
from ..models import InventoryCategory

class InventoryItemFormTestCase(TestCase):
    """Tests for the InventoryItemForm."""

    def setUp(self):
        """Set up test data."""
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

        self.category = InventoryCategory.objects.create(
            name="Test Category",
            slug="test-category",
            organization=self.organization
        )

        self.form_data = {
            'name': 'Test Item',
            'sku': 'TEST001',
            'description': 'This is a test item',
            'quantity': 10,
            'unit': 'pieces',
            'price': 19.99,
            'reorder_threshold': 5,
            'category': self.category.id
        }

    def test_valid_form(self):
        """Test form with valid data."""
        form = InventoryItemForm(
            self.form_data,
            organization=self.organization
        )
        self.assertTrue(form.is_valid())

    def test_blank_name(self):
        """Test form with blank name."""
        self.form_data['name'] = ''
        form = InventoryItemForm(
            self.form_data,
            organization=self.organization
        )
        self.assertFalse(form.is_valid())
        self.assertIn('name', form.errors)

    def test_negative_quantity(self):
        """Test form with negative quantity."""
        self.form_data['quantity'] = -1
        form = InventoryItemForm(
            self.form_data,
            organization=self.organization
        )
        self.assertFalse(form.is_valid())
        self.assertIn('quantity', form.errors)

    def test_duplicate_sku(self):
        """Test form with duplicate SKU."""
        # First create an item
        form1 = InventoryItemForm(
            self.form_data,
            organization=self.organization
        )
        form1.save()

        # Try to create another with the same SKU
        form2 = InventoryItemForm(
            self.form_data,
            organization=self.organization
        )
        self.assertFalse(form2.is_valid())
        self.assertIn('sku', form2.errors)
```

### Testing Views

```python
# plugins/inventory_plugin/tests/test_views.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model
from ..models import InventoryItem, InventoryCategory
from orgs.models import Organization

User = get_user_model()

class InventoryViewsTestCase(TestCase):
    """Tests for inventory views."""

    def setUp(self):
        """Set up test data."""
        # Create test organization
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

        # Create test user
        self.user = User.objects.create_user(
            username="testuser",
            email="test@example.com",
            password="password"
        )
        self.user.organization = self.organization
        self.user.save()

        # Create test category
        self.category = InventoryCategory.objects.create(
            name="Test Category",
            slug="test-category",
            organization=self.organization
        )

        # Create test items
        for i in range(5):
            InventoryItem.objects.create(
                name=f"Test Item {i+1}",
                sku=f"TEST00{i+1}",
                quantity=10 * (i+1),
                price=9.99,
                category=self.category,
                organization=self.organization
            )

        # Set up client
        self.client = Client()
        self.client.login(username="testuser", password="password")

    def test_item_list_view(self):
        """Test the item list view."""
        url = reverse('inventory_plugin:item_list')
        response = self.client.get(url)

        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'inventory_plugin/item_list.html')
        self.assertEqual(len(response.context['items']), 5)

    def test_item_detail_view(self):
        """Test the item detail view."""
        item = InventoryItem.objects.first()
        url = reverse('inventory_plugin:item_detail', args=[item.id])
        response = self.client.get(url)

        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'inventory_plugin/item_detail.html')
        self.assertEqual(response.context['item'], item)

    def test_create_item_view(self):
        """Test the create item view."""
        url = reverse('inventory_plugin:item_create')

        # GET request should show the form
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'inventory_plugin/item_form.html')

        # POST request should create a new item
        data = {
            'name': 'New Test Item',
            'sku': 'NEW001',
            'quantity': 20,
            'unit': 'pieces',
            'price': 29.99,
            'reorder_threshold': 5,
            'category': self.category.id
        }

        response = self.client.post(url, data)

        # Should redirect after successful creation
        self.assertEqual(response.status_code, 302)

        # Check that the item was created
        self.assertTrue(
            InventoryItem.objects.filter(sku='NEW001').exists()
        )

    def test_update_item_view(self):
        """Test the update item view."""
        item = InventoryItem.objects.first()
        url = reverse('inventory_plugin:item_update', args=[item.id])

        # GET request should show the form with item data
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'inventory_plugin/item_form.html')

        # POST request should update the item
        data = {
            'name': 'Updated Item',
            'sku': item.sku,
            'quantity': 50,
            'unit': 'pieces',
            'price': 39.99,
            'reorder_threshold': 10,
            'category': self.category.id
        }

        response = self.client.post(url, data)

        # Should redirect after successful update
        self.assertEqual(response.status_code, 302)

        # Check that the item was updated
        item.refresh_from_db()
        self.assertEqual(item.name, 'Updated Item')
        self.assertEqual(item.quantity, 50)
```

### Testing Permissions

```python
# plugins/inventory_plugin/tests/test_permissions.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from django.urls import reverse
from django.test import Client
from ..permissions import permissions
from orgs.models import Organization

User = get_user_model()

class InventoryPermissionsTestCase(TestCase):
    """Tests for inventory permissions."""

    def setUp(self):
        """Set up test data."""
        # Create test organization
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

        # Create users with different permissions
        self.admin_user = User.objects.create_user(
            username="admin",
            email="admin@example.com",
            password="password"
        )
        self.admin_user.organization = self.organization
        self.admin_user.save()

        self.viewer_user = User.objects.create_user(
            username="viewer",
            email="viewer@example.com",
            password="password"
        )
        self.viewer_user.organization = self.organization
        self.viewer_user.save()

        # Set up permissions
        permissions.add_user_to_group(self.admin_user, "inventory_admin")
        permissions.add_user_to_group(self.viewer_user, "inventory_viewer")

        # Set up clients
        self.admin_client = Client()
        self.admin_client.login(username="admin", password="password")

        self.viewer_client = Client()
        self.viewer_client.login(username="viewer", password="password")

    def test_view_permission(self):
        """Test view permission."""
        # Both users should be able to view the list
        url = reverse('inventory_plugin:item_list')

        response1 = self.admin_client.get(url)
        self.assertEqual(response1.status_code, 200)

        response2 = self.viewer_client.get(url)
        self.assertEqual(response2.status_code, 200)

    def test_add_permission(self):
        """Test add permission."""
        url = reverse('inventory_plugin:item_create')

        # Admin user should be able to access the create page
        response1 = self.admin_client.get(url)
        self.assertEqual(response1.status_code, 200)

        # Viewer user should be redirected or denied
        response2 = self.viewer_client.get(url)
        self.assertNotEqual(response2.status_code, 200)

    def test_has_perm_method(self):
        """Test the has_perm method."""
        self.assertTrue(
            permissions.has_perm(self.admin_user, "view_inventory")
        )
        self.assertTrue(
            permissions.has_perm(self.admin_user, "add_inventory")
        )
        self.assertTrue(
            permissions.has_perm(self.admin_user, "change_inventory")
        )

        self.assertTrue(
            permissions.has_perm(self.viewer_user, "view_inventory")
        )
        self.assertFalse(
            permissions.has_perm(self.viewer_user, "add_inventory")
        )
        self.assertFalse(
            permissions.has_perm(self.viewer_user, "change_inventory")
        )
```

## Integration Testing

Integration tests verify that different components work together correctly.

### Testing Plugin Components Together

```python
# plugins/inventory_plugin/tests/test_integration.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model
from ..models import InventoryItem, StockAdjustment
from orgs.models import Organization
from ..settings import settings

User = get_user_model()

class InventoryIntegrationTestCase(TestCase):
    """Integration tests for inventory plugin."""

    def setUp(self):
        """Set up test data."""
        # Create test organization
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

        # Create test user with admin permissions
        self.user = User.objects.create_user(
            username="admin",
            email="admin@example.com",
            password="password"
        )
        self.user.organization = self.organization
        self.user.save()

        # Set up permissions
        from ..permissions import permissions
        permissions.add_user_to_group(self.user, "inventory_admin")

        # Create test item
        self.item = InventoryItem.objects.create(
            name="Test Item",
            sku="TEST001",
            quantity=10,
            unit="pieces",
            price=19.99,
            reorder_threshold=5,
            organization=self.organization
        )

        # Set up settings
        settings.set_for_organization(
            self.organization,
            "low_stock_threshold",
            5
        )

        # Set up client
        self.client = Client()
        self.client.login(username="admin", password="password")

    def test_stock_adjustment_workflow(self):
        """Test the complete stock adjustment workflow."""
        # 1. View the item detail page
        detail_url = reverse('inventory_plugin:item_detail', args=[self.item.id])
        response = self.client.get(detail_url)
        self.assertEqual(response.status_code, 200)

        # 2. Submit a stock adjustment
        adjust_url = reverse('inventory_plugin:adjust_stock', args=[self.item.id])
        adjustment_data = {
            'quantity': -7,  # Reduce stock by 7
            'reason': 'Used in project'
        }

        response = self.client.post(adjust_url, adjustment_data, follow=True)
        self.assertEqual(response.status_code, 200)

        # 3. Verify item quantity was updated
        self.item.refresh_from_db()
        self.assertEqual(self.item.quantity, 3)  # 10 - 7 = 3

        # 4. Verify stock adjustment record was created
        adjustment = StockAdjustment.objects.latest('created_at')
        self.assertEqual(adjustment.item, self.item)
        self.assertEqual(adjustment.quantity, -7)
        self.assertEqual(adjustment.reason, 'Used in project')

        # 5. Verify low stock status
        self.assertTrue(self.item.is_low_stock())

        # 6. Check if alert was created (if implemented)
        from ..models import StockAlert
        self.assertTrue(
            StockAlert.objects.filter(item=self.item).exists()
        )

    def test_settings_integration(self):
        """Test integration with settings."""
        # 1. Change a setting
        settings_url = reverse('inventory_plugin:settings')
        settings_data = {
            'low_stock_threshold': 8,
            'enable_low_stock_alerts': True,
            'notification_email': 'alerts@example.com'
        }

        response = self.client.post(settings_url, settings_data, follow=True)
        self.assertEqual(response.status_code, 200)

        # 2. Verify setting was updated
        threshold = settings.get_for_organization(
            self.organization,
            "low_stock_threshold"
        )
        self.assertEqual(threshold, 8)

        # 3. Check how the new setting affects behavior
        self.assertTrue(self.item.is_low_stock())  # 10 < 8
```

## Mocking External Dependencies

When testing code that interacts with external services, use mocks to isolate your tests:

```python
# plugins/inventory_plugin/tests/test_external.py
from django.test import TestCase
from unittest.mock import patch, MagicMock
from ..integrations.shopify import sync_with_shopify
from orgs.models import Organization
from ..models import InventoryItem

class ShopifyIntegrationTestCase(TestCase):
    """Tests for Shopify integration."""

    def setUp(self):
        """Set up test data."""
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

        # Create test item with Shopify IDs
        self.item = InventoryItem.objects.create(
            name="Test Item",
            sku="TEST001",
            quantity=10,
            organization=self.organization,
            shopify_inventory_id="12345",
            shopify_location_id="67890"
        )

        # Set up settings mock
        self.settings_patcher = patch('plugins.inventory_plugin.integrations.shopify.settings')
        self.mock_settings = self.settings_patcher.start()

        # Configure mock settings
        self.mock_settings.get_for_organization.side_effect = lambda org, key: {
            "shopify_api_key": "test_api_key",
            "shopify_shop_url": "https://test-shop.myshopify.com"
        }.get(key)

    def tearDown(self):
        """Clean up after tests."""
        self.settings_patcher.stop()

    @patch('plugins.inventory_plugin.integrations.shopify.requests')
    def test_sync_with_shopify(self, mock_requests):
        """Test syncing inventory with Shopify."""
        # Configure the mock response
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"inventory_level": {"available": 10}}
        mock_requests.put.return_value = mock_response

        # Call the function being tested
        sync_with_shopify(self.organization)

        # Verify API call was made correctly
        mock_requests.put.assert_called_once_with(
            "https://test-shop.myshopify.com/admin/api/2022-01/inventory_levels/set.json",
            headers={
                "X-Shopify-Access-Token": "test_api_key",
                "Content-Type": "application/json"
            },
            json={
                "inventory_item_id": "12345",
                "location_id": "67890",
                "available": 10
            }
        )
```

## Testing Settings and Permissions

Testing plugin settings and permissions requires special handling:

```python
# plugins/inventory_plugin/tests/test_settings_permissions.py
from django.test import TestCase
from orgs.models import Organization
from ..settings import settings

class SettingsTestCase(TestCase):
    """Tests for plugin settings."""

    def setUp(self):
        """Set up test data."""
        self.organization = Organization.objects.create(
            name="Test Organization",
            slug="test-org"
        )

    def test_default_settings(self):
        """Test default settings values."""
        # Get default values
        threshold = settings.get_for_organization(
            self.organization,
            "low_stock_threshold"
        )
        alerts_enabled = settings.get_for_organization(
            self.organization,
            "enable_low_stock_alerts"
        )

        # Check against expected defaults
        self.assertEqual(threshold, 5)
        self.assertTrue(alerts_enabled)

    def test_update_settings(self):
        """Test updating settings."""
        # Update a setting
        settings.set_for_organization(
            self.organization,
            "low_stock_threshold",
            10
        )

        # Verify it was updated
        threshold = settings.get_for_organization(
            self.organization,
            "low_stock_threshold"
        )
        self.assertEqual(threshold, 10)

    def test_validation(self):
        """Test settings validation."""
        # Try to set an invalid value
        success, _ = settings.validate_setting(
            "low_stock_threshold",
            -1,
            self.organization
        )

        self.assertFalse(success)

        # Try to set a valid value
        success, _ = settings.validate_setting(
            "low_stock_threshold",
            20,
            self.organization
        )

        self.assertTrue(success)
```

## Running Tests

To run your plugin tests:

```bash
# Run all tests for your plugin
python manage.py test plugins.inventory_plugin

# Run specific test case
python manage.py test plugins.inventory_plugin.tests.test_models

# Run specific test method
python manage.py test plugins.inventory_plugin.tests.test_models.InventoryItemTestCase.test_is_low_stock
```

## Debugging Techniques

Debugging plugins can be challenging. Here are some techniques to help:

### Print Debugging

Simple but effective:

```python
def some_function(arg):
    print(f"Debug: arg={arg}")
    # ... function logic
```

### Python Debugger (pdb)

Insert a breakpoint in your code:

```python
def complex_function(data):
    # ... some code
    import pdb; pdb.set_trace()
    # Execution will pause here
    result = process_data(data)
    # ... more code
```

When execution reaches the breakpoint, you'll get an interactive console where you can:
- Examine variables with `p variable_name`
- Continue execution with `c`
- Step through code with `n` (next) and `s` (step into)
- View the current line with `l`

### Django Debug Toolbar

Add Django Debug Toolbar to your development environment:

```python
# settings.py
INSTALLED_APPS = [
    # ... other apps
    'debug_toolbar',
]

MIDDLEWARE = [
    # ... other middleware
    'debug_toolbar.middleware.DebugToolbarMiddleware',
]

INTERNAL_IPS = [
    '127.0.0.1',
]
```

This provides information about:
- SQL queries
- Templates used
- Request data
- Signals fired
- Cache operations

### Logging

Set up detailed logging for debugging:

```python
# plugins/inventory_plugin/apps.py
import logging

logger = logging.getLogger(__name__)

class InventoryPluginConfig(PluginAppConfig):
    # ... other code

    def ready(self):
        super().ready()
        logger.debug("Inventory plugin is initializing")
        # ... other initialization code
```

Configure logging in settings:

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'DEBUG',
            'class': 'logging.FileHandler',
            'filename': 'debug.log',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'plugins.inventory_plugin': {
            'handlers': ['file'],
            'level': 'DEBUG',
            'propagate': True,
        },
    },
}
```

## Common Plugin Issues and Solutions

### Issue: Plugin Not Being Discovered

**Symptoms**:
- Plugin doesn't appear in the admin interface
- Plugin functionality is not available

**Debugging Steps**:
1. Check the `PLUGIN_METADATA` in `__init__.py`
2. Verify that the plugin is in the correct directory
3. Check for errors in the console output during Django startup
4. Ensure the plugin is included in `INSTALLED_APPS`

**Solution**:
```python
# __init__.py
from plugin_registry import PluginMetadata

PLUGIN_METADATA = PluginMetadata(
    name="Inventory Plugin",
    version="1.0.0",
    description="Manage inventory items",
    author="Your Name",
    email="your.email@example.com",
    django_apps=["plugins.inventory_plugin"],
)

default_app_config = 'plugins.inventory_plugin.apps.InventoryPluginConfig'
```

### Issue: Template Hooks Not Appearing

**Symptoms**:
- Hook content doesn't show up in templates
- No errors in logs

**Debugging Steps**:
1. Check hook registration code
2. Verify the hook name matches exactly with the template
3. Check the context being passed to the hook

**Solution**:
```python
# template_hooks.py
from core.template_hooks import template_hook_registry

def my_hook_function(context, request):
    # Add debugging to see if this is called
    print(f"Hook called with context: {context}")
    return render_to_string('my_template.html', context, request=request)

# Register with exact hook name
template_hook_registry.register_hook(
    "exact_hook_name_in_template",
    my_hook_function,
    plugin_id="my_plugin"
)
```

### Issue: Permissions Not Working

**Symptoms**:
- Users can access views they shouldn't
- Users can't access views they should have permission for

**Debugging Steps**:
1. Check permission registration
2. Verify permission checking in views
3. Look for typos in permission names

**Solution**:
```python
# permissions.py
from core.plugin_permissions import create_permissions

permissions = create_permissions("inventory_plugin")

# Define permissions with correct names
permissions.add_permission(
    "view_inventory",
    "Can view inventory items"
)

# In views.py
@plugin_permission_required("inventory_plugin.view_inventory")  # Correct format
def my_view(request):
    # View code
```

### Issue: Settings Not Being Saved

**Symptoms**:
- Settings values don't persist
- Default values always used

**Debugging Steps**:
1. Check settings registration
2. Verify organization context
3. Look for validation errors

**Solution**:
```python
# Check settings validation
from .settings import settings

# Print validation result
result, error = settings.validate_setting(
    "setting_name",
    value,
    organization
)
print(f"Validation result: {result}, Error: {error}")

# Check actual setting value
actual_value = settings.get_for_organization(
    organization,
    "setting_name"
)
print(f"Actual value: {actual_value}")
```

## Best Practices for Testing and Debugging

1. **Write Tests First**: Consider Test-Driven Development (TDD) for critical functionality
2. **Test Coverage**: Aim for high test coverage, especially for complex logic
3. **Isolation**: Test components in isolation using mocks for dependencies
4. **Real-World Scenarios**: Include tests for typical user workflows
5. **Edge Cases**: Test boundary conditions and edge cases
6. **Error Handling**: Test how your code handles errors and invalid inputs
7. **Reproducibility**: Make bugs reproducible with specific test cases
8. **Documentation**: Document debugging techniques specific to your plugin
9. **Log Responsibly**: Use appropriate log levels and include context
10. **Regular Testing**: Run tests regularly, especially after major changes

By following these practices, you'll create more robust plugins and spend less time debugging issues.

## Next Steps

Congratulations! You've completed the Ritiko plugin development tutorial series. You now have the knowledge to create, test, and debug powerful plugins that extend the Ritiko platform.

To continue your learning:
- Review the existing core plugins for design patterns and best practices
- Join the developer community to share your plugins and get feedback
- Explore advanced topics in Django and Python testing

Happy plugin development!
