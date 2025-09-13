# Widget Zones Reference

This document provides a comprehensive guide to using widget zones in CRUD (Create, Read, Update, Delete) pages within the Ritiko platform. Widget zones allow plugins to insert content at specific locations in dashboard, list, and form pages.

## Overview

Widget zones are designated areas in templates where plugins can register widgets to appear. This enables extending the UI without modifying core templates, providing a clean separation between core functionality and plugin-specific features.

## Zone Placement

### Dashboard Zones

Dashboard zones allow plugins to add widgets to the main dashboard:

- `dashboard_top` - Full width zone at the top of the dashboard
- `dashboard_medium` - Split zone (40%/60%) in the middle of the dashboard (max 2 widgets)
- `dashboard_bottom` - Full width zone at the bottom of the dashboard
- `dashboard_footer` - Full width zone at the footer of the dashboard

### CRUD List Page Zones

List pages can be extended with these zones:

- `{app_name}_{model_name}_list_top` - Above the list content
- `{app_name}_{model_name}_list_bottom` - Below the list content
- `{app_name}_{model_name}_list_footer` - At the footer of the list page

### CRUD Form Page Zones

Form pages can be extended with these zones:

- `{app_name}_{model_name}_form_top` - Above the form content
- `{app_name}_{model_name}_form_bottom` - Below the form content
- `{app_name}_{model_name}_form_footer` - At the footer of the form page

## Implementation

### Adding Zones to Templates

To implement widget zones in your templates:

1. Load the widget tags:
```django
{% load plugin_widget_tags %}
```

2. Add zones at appropriate locations:

#### For List Pages
```django
{% block content %}
{# List Top Zone #}
{% render_widget_zone 'app_model_list_top' app_name='patients' model_name='patient' %}

<div class="row">
    <div class="col-12">
        <h1>Patient List</h1>
        <!-- Your list content here -->
    </div>
</div>

{# List Bottom Zone #}
{% render_widget_zone 'app_model_list_bottom' app_name='patients' model_name='patient' %}

{% endblock content %}

{# List Footer Zone #}
{% render_widget_zone 'app_model_list_footer' app_name='patients' model_name='patient' %}
```

#### For Form Pages
```django
{% block content %}
{# Form Top Zone #}
{% render_widget_zone 'app_model_form_top' app_name='patients' model_name='patient' %}

<div class="row">
    <div class="col-12">
        <h1>{% if object %}Edit{% else %}Add{% endif %} Patient</h1>
        <!-- Your form content here -->
    </div>
</div>

{# Form Bottom Zone #}
{% render_widget_zone 'app_model_form_bottom' app_name='patients' model_name='patient' %}

{% endblock content %}

{# Form Footer Zone #}
{% render_widget_zone 'app_model_form_footer' app_name='patients' model_name='patient' %}
```

### Registering Widgets for Zones

In your plugin's `widgets.py` file:

```python
plugin_widget_registry.register_widget(
    widget=Widget(
        name="patient_helper_widget",
        plugin_id="your_plugin",
        template="your_plugin/widgets/patient_helper.html",
        priority=5,
        css_classes="helper-widget"
    ),
    zones=[
        "patients_patient_form_top",  # Will show on patient form top
        "patients_patient_list_top"   # Will show on patient list top
    ]
)
```

## Best Practices

1. **Prioritize Widgets**: Use the `priority` parameter to control widget ordering within zones
2. **Responsive Design**: Ensure your widgets adapt to different zone widths
3. **Context Awareness**: Make widgets context-aware to display relevant information based on the current model/object
4. **Performance**: Keep widgets lightweight to avoid impacting page load times
5. **Clear Naming**: Use descriptive names for widgets to maintain clarity in the codebase

## Visual Layout

### Dashboard Layout
```
+---------------------------------------+
|        dashboard_top (Full Width)     |
+---------------------------------------+
|  dashboard_medium  |  dashboard_medium|
|      (40%)         |      (60%)       |
+---------------------------------------+
|     dashboard_bottom (Full Width)     |
+---------------------------------------+
|     dashboard_footer (Full Width)     |
+---------------------------------------+
```

### List Page Layout
```
+---------------------------------------+
|    app_model_list_top                 |
+---------------------------------------+
|                                       |
|           List Content                |
|                                       |
+---------------------------------------+
|    app_model_list_bottom              |
+---------------------------------------+
|    app_model_list_footer              |
+---------------------------------------+
```

### Form Page Layout
```
+---------------------------------------+
|    app_model_form_top                 |
+---------------------------------------+
|                                       |
|           Form Content                |
|                                       |
+---------------------------------------+
|    app_model_form_bottom              |
+---------------------------------------+
|    app_model_form_footer              |
+---------------------------------------+
