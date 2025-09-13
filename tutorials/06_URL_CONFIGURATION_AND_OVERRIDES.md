# URL Configuration and Overrides

This tutorial explores how to configure URLs for your plugins and how to override existing system URLs without modifying core files. You'll learn about URL patterns, namespaces, and the powerful URL override system.

## Basic URL Configuration

Every plugin can define its own URL patterns that exist alongside the core system URLs. These are defined in a `urls.py` file within your plugin package.

### Standard URL Configuration

```python
# plugins/inventory_plugin/urls.py
from django.urls import path
from . import views

app_name = 'inventory_plugin'

urlpatterns = [
    path('', views.dashboard, name='dashboard'),
    path('items/', views.item_list, name='item_list'),
    path('items/<int:pk>/', views.item_detail, name='item_detail'),
    path('items/create/', views.item_create, name='item_create'),
    path('items/<int:pk>/update/', views.item_update, name='item_update'),
    path('items/<int:pk>/delete/', views.item_delete, name='item_delete'),
]
```

With this configuration, your plugin's views will be accessible at:
- `/inventory_plugin/` - Dashboard
- `/inventory_plugin/items/` - Item list
- `/inventory_plugin/items/1/` - Item detail for ID 1
- `/inventory_plugin/items/create/` - Create new item
- `/inventory_plugin/items/1/update/` - Update item with ID 1
- `/inventory_plugin/items/1/delete/` - Delete item with ID 1

### URL Namespaces

The `app_name` variable sets the namespace for your URLs, allowing you to reference them in templates and views using the `namespace:name` syntax:

```html
<!-- In templates -->
<a href="{% url 'inventory_plugin:dashboard' %}">Dashboard</a>
<a href="{% url 'inventory_plugin:item_list' %}">View Items</a>
<a href="{% url 'inventory_plugin:item_detail' item.pk %}">View Item</a>
```

```python
# In views or Python code
from django.urls import reverse

dashboard_url = reverse('inventory_plugin:dashboard')
item_list_url = reverse('inventory_plugin:item_list')
item_detail_url = reverse('inventory_plugin:item_detail', args=[item.pk])
```

## URL Overrides

The Ritiko plugin system provides a powerful mechanism to override existing system URLs without modifying core files. This allows your plugin to replace specific views while preserving others.

### How URL Overrides Work

1. Plugin defines URL overrides in `urls.py` using the `URL_OVERRIDES` dictionary
2. System prepends plugin URL patterns to original URL patterns
3. Django checks plugin patterns first due to ordering
4. Original URLs remain accessible if not overridden
5. Namespace compatibility is maintained for templates

### Basic URL Override Example

```python
# plugins/enhanced_patients/urls.py
from django.urls import path
from . import views

app_name = 'enhanced_patients'

# Regular plugin URLs (accessible at /enhanced_patients/)
urlpatterns = [
    path('dashboard/', views.dashboard, name='dashboard'),
    path('analytics/', views.analytics, name='analytics'),
]

# URL Overrides (replace system URLs)
URL_OVERRIDES = {
    "patients/": [
        # Override the main patients list view
        path("", views.enhanced_patient_list, name="list"),

        # Add new functionality
        path("export/", views.patient_export, name="export"),
    ]
}
```

With this configuration:
- The system's `/patients/` URL will show your enhanced list view
- A new `/patients/export/` URL will be available
- All other patient URLs (`/patients/create/`, `/patients/<id>/`, etc.) remain unchanged
- Your plugin's URLs are still accessible at `/enhanced_patients/dashboard/` and `/enhanced_patients/analytics/`

### Multiple URL Overrides

You can override multiple URL patterns:

```python
URL_OVERRIDES = {
    "patients/": [
        path("", views.enhanced_patient_list, name="list"),
        path("export/", views.patient_export, name="export"),
    ],
    "reports/": [
        path("custom/", views.custom_reports, name="custom_reports"),
    ]
}
```

### Template Compatibility

URL overrides maintain full template compatibility:

```html
<!-- Templates continue to work as before -->
<a href="{% url 'patients:list' %}">Patient List</a>          <!-- Enhanced view -->
<a href="{% url 'patients:detail' patient.id %}">Detail</a>   <!-- Original view -->
<a href="{% url 'patients:export' %}">Export</a>              <!-- New view -->
```

## Best Practices for URL Configuration

1. **Use Descriptive Names**: Choose clear, descriptive URL names that reflect their purpose
2. **Follow URL Hierarchy**: Organize URLs in a logical hierarchy
3. **Override Selectively**: Only override the specific URLs you need to change
4. **Preserve Namespaces**: Maintain namespace compatibility with existing templates
5. **Document Changes**: Clearly document which URLs you're overriding
6. **Test Thoroughly**: Ensure all functionality works correctly after overriding URLs

## Practical Example: Enhanced Patient List

Let's create a plugin that enhances the patient list with advanced filtering and export capabilities:

```python
# plugins/enhanced_patients/urls.py
from django.urls import path
from . import views

app_name = 'enhanced_patients'

urlpatterns = [
    path('dashboard/', views.dashboard, name='dashboard'),
]

URL_OVERRIDES = {
    "patients/": [
        # Override main list with enhanced view
        path("", views.EnhancedPatientListView.as_view(), name="list"),

        # Add new functionality
        path("export/csv/", views.export_csv, name="export_csv"),
        path("export/excel/", views.export_excel, name="export_excel"),
        path("stats/", views.patient_stats, name="stats"),
    ]
}
```

With this setup, users benefit from enhanced features while maintaining a consistent user experience and URL structure. Existing links and bookmarks continue to work, but they now provide the improved functionality from your plugin.

## Next Steps

In the next tutorial, we'll explore template hooks and widget zones, which allow you to inject content into existing templates without overriding them completely.
