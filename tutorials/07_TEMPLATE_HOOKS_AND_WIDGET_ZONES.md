# Template Hooks and Widget Zones

This tutorial explores two powerful mechanisms for extending the Ritiko user interface without modifying core templates: Template Hooks and Widget Zones. These features allow plugins to inject content at specific locations throughout the application.

## Template Hooks

Template hooks provide a way to inject content into existing templates at designated points. Unlike URL overrides (which replace entire views), template hooks allow you to add content alongside existing content.

### How Template Hooks Work

1. Core templates include "hook points" using the `{% hook %}` template tag
2. Plugins register callback functions that return HTML content
3. When a template is rendered, the system calls all registered callbacks for each hook point
4. The rendered content from all callbacks is injected into the template

### Registering a Template Hook

To register a template hook, create a `template_hooks.py` file in your plugin:

```python
# plugins/patient_notes_plugin/template_hooks.py
from django.template.loader import render_to_string
from core.template_hooks import template_hook_registry

def patient_detail_notes(context, request):
    """Add notes section to patient detail page."""
    patient = context.get('patient')

    if not patient:
        return ""  # Don't render if no patient in context

    # Get notes for this patient
    from .models import PatientNote
    notes = PatientNote.objects.filter(patient=patient).order_by('-created_at')

    # Prepare context for template
    template_context = {
        'patient': patient,
        'notes': notes,
        'can_add_note': request.user.has_perm('patient_notes_plugin.add_patientnote'),
        'request': request,
    }

    # Render template to string
    return render_to_string(
        'patient_notes_plugin/patient_detail_notes.html',
        template_context,
        request=request
    )

# Register the hook
template_hook_registry.register_hook(
    "patient_detail_additional_sections",  # Hook name
    patient_detail_notes,                  # Callback function
    plugin_id="patient_notes_plugin",      # Plugin identifier
    priority=10                            # Rendering priority (lower numbers render first)
)
```

### Creating the Hook Template

Create a template file for your hook content:

```html
<!-- plugins/patient_notes_plugin/templates/patient_notes_plugin/patient_detail_notes.html -->
<div class="card mb-4">
    <div class="card-header">
        <h4 class="card-title">
            <i class="fas fa-sticky-note"></i> Patient Notes
            {% if can_add_note %}
            <a href="{% url 'patient_notes_plugin:add_note' patient.id %}" class="btn btn-sm btn-primary float-right">
                <i class="fas fa-plus"></i> Add Note
            </a>
            {% endif %}
        </h4>
    </div>
    <div class="card-body">
        {% if notes %}
            <div class="timeline">
                {% for note in notes %}
                <div class="timeline-item">
                    <span class="time">
                        <i class="fas fa-clock"></i> {{ note.created_at|date:"M d, Y H:i" }}
                    </span>
                    <h3 class="timeline-header">
                        <a href="{% url 'staff:detail' note.author.id %}">{{ note.author.get_full_name }}</a>
                        added a note
                    </h3>
                    <div class="timeline-body">
                        {{ note.content|linebreaks }}
                    </div>
                    {% if request.user == note.author or perms.patient_notes_plugin.change_patientnote %}
                    <div class="timeline-footer">
                        <a href="{% url 'patient_notes_plugin:edit_note' note.id %}" class="btn btn-sm btn-info">
                            <i class="fas fa-edit"></i> Edit
                        </a>
                        {% if perms.patient_notes_plugin.delete_patientnote %}
                        <a href="{% url 'patient_notes_plugin:delete_note' note.id %}" class="btn btn-sm btn-danger">
                            <i class="fas fa-trash"></i> Delete
                        </a>
                        {% endif %}
                    </div>
                    {% endif %}
                </div>
                {% endfor %}
            </div>
        {% else %}
            <p class="text-muted">No notes available for this patient.</p>
        {% endif %}
    </div>
</div>
```

### Hook Points in Core Templates

Core templates include hook points using the `{% hook %}` template tag:

```html
<!-- In patients/templates/patients/detail.html -->
<div class="patient-details">
    <!-- Basic patient information -->
</div>

{% load template_hooks %}
{% hook "patient_detail_additional_sections" %}

<div class="patient-actions">
    <!-- Action buttons -->
</div>
```

### Common Hook Points

Ritiko provides several hook points in core templates:

| Hook Name | Location | Purpose |
|-----------|----------|---------|
| `patient_detail_additional_sections` | Patient detail page | Add sections below patient info |
| `patient_list_actions` | Patient list page | Add action buttons to patient list |
| `appointment_detail_sidebar` | Appointment detail | Add content to appointment sidebar |
| `dashboard_widgets` | Main dashboard | Add widgets to dashboard |
| `org_settings_sections` | Organization settings | Add setting sections |
| `user_profile_tabs` | User profile | Add tabs to user profile |

## Widget Zones

Widget zones are designated areas in templates where plugins can register widgets to appear. They're similar to template hooks but provide more structure and are specifically designed for dashboard and CRUD pages.

### Widget Zone Placement

#### Dashboard Zones

- `dashboard_top` - Full width zone at the top of the dashboard
- `dashboard_medium` - Split zone (40%/60%) in the middle of the dashboard
- `dashboard_bottom` - Full width zone at the bottom of the dashboard
- `dashboard_footer` - Full width zone at the footer of the dashboard

#### CRUD List Page Zones

- `{app_name}_{model_name}_list_top` - Above the list content
- `{app_name}_{model_name}_list_bottom` - Below the list content
- `{app_name}_{model_name}_list_footer` - At the footer of the list page

#### CRUD Form Page Zones

- `{app_name}_{model_name}_form_top` - Above the form content
- `{app_name}_{model_name}_form_bottom` - Below the form content
- `{app_name}_{model_name}_form_footer` - At the footer of the form page

### Registering Widgets for Zones

To register a widget for a zone, create a `widgets.py` file in your plugin:

```python
# plugins/analytics_plugin/widgets.py
from core.plugin_widgets import Widget, plugin_widget_registry

# Create and register a simple widget
plugin_widget_registry.register_widget(
    widget=Widget(
        name="patient_statistics_widget",
        plugin_id="analytics_plugin",
        template="analytics_plugin/widgets/patient_statistics.html",
        priority=5,
        css_classes="statistics-widget"
    ),
    zones=[
        "dashboard_top",                # Show on main dashboard
        "patients_patient_list_top"     # Show on patient list page
    ]
)

# Create and register another widget
plugin_widget_registry.register_widget(
    widget=Widget(
        name="recent_appointments_widget",
        plugin_id="analytics_plugin",
        template="analytics_plugin/widgets/recent_appointments.html",
        priority=10,
        css_classes="recent-data-widget",
        context_processor=get_recent_appointments_context
    ),
    zones=[
        "dashboard_medium"              # Show in middle dashboard zone
    ]
)

# Context processor function
def get_recent_appointments_context(request, context):
    """Add recent appointments to widget context."""
    from appointments.models import Appointment

    # Get user's organization
    organization = getattr(request.user, 'organization', None)
    if not organization:
        return {}

    # Get recent appointments
    recent_appointments = Appointment.objects.filter(
        organization=organization,
        date__gte=datetime.now().date()
    ).order_by('date', 'time')[:5]

    return {
        'recent_appointments': recent_appointments
    }
```

### Creating Widget Templates

Create templates for your widgets:

```html
<!-- plugins/analytics_plugin/templates/analytics_plugin/widgets/patient_statistics.html -->
<div class="card">
    <div class="card-header">
        <h3 class="card-title">
            <i class="fas fa-chart-pie"></i> Patient Statistics
        </h3>
    </div>
    <div class="card-body">
        <div class="row">
            <div class="col-md-3 col-sm-6 col-12">
                <div class="info-box bg-info">
                    <span class="info-box-icon"><i class="fas fa-users"></i></span>
                    <div class="info-box-content">
                        <span class="info-box-text">Total Patients</span>
                        <span class="info-box-number">{{ total_patients }}</span>
                    </div>
                </div>
            </div>
            <div class="col-md-3 col-sm-6 col-12">
                <div class="info-box bg-success">
                    <span class="info-box-icon"><i class="fas fa-user-plus"></i></span>
                    <div class="info-box-content">
                        <span class="info-box-text">New (30 days)</span>
                        <span class="info-box-number">{{ new_patients }}</span>
                    </div>
                </div>
            </div>
            <div class="col-md-3 col-sm-6 col-12">
                <div class="info-box bg-warning">
                    <span class="info-box-icon"><i class="fas fa-calendar-check"></i></span>
                    <div class="info-box-content">
                        <span class="info-box-text">Appts Today</span>
                        <span class="info-box-number">{{ todays_appointments }}</span>
                    </div>
                </div>
            </div>
            <div class="col-md-3 col-sm-6 col-12">
                <div class="info-box bg-danger">
                    <span class="info-box-icon"><i class="fas fa-exclamation-triangle"></i></span>
                    <div class="info-box-content">
                        <span class="info-box-text">Alerts</span>
                        <span class="info-box-number">{{ alert_count }}</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
```

### Using Widget Zones in Templates

Widget zones are included in templates using the `render_widget_zone` template tag:

```html
{% load plugin_widget_tags %}

<!-- Dashboard top zone -->
{% render_widget_zone 'dashboard_top' %}

<!-- Dashboard medium zone (split) -->
<div class="row">
    <div class="col-md-5">
        {% render_widget_zone 'dashboard_medium' position='left' %}
    </div>
    <div class="col-md-7">
        {% render_widget_zone 'dashboard_medium' position='right' %}
    </div>
</div>

<!-- Model-specific zones -->
{% render_widget_zone 'app_model_list_top' app_name='patients' model_name='patient' %}
```

## Differences Between Template Hooks and Widget Zones

While template hooks and widget zones serve similar purposes, they have some key differences:

| Feature | Template Hooks | Widget Zones |
|---------|---------------|--------------|
| **Implementation** | Python callback functions | Widget objects with templates |
| **Structure** | Flexible, can return any HTML | Structured, follows widget pattern |
| **Context** | Inherits template context | Can have custom context processors |
| **Placement** | Specific predefined locations | Standard locations in CRUD pages |
| **Management** | Code-only registration | Can be user-configurable |
| **Visual Style** | Completely custom | Typically follows card/widget pattern |

## Best Practices

### For Template Hooks:

1. **Check Context**: Always verify required context variables are present
2. **Error Handling**: Handle missing data gracefully
3. **Keep It Focused**: Each hook should do one thing well
4. **Prioritize**: Use the priority parameter to control rendering order
5. **User Permissions**: Always check permissions before displaying sensitive content

### For Widget Zones:

1. **Responsive Design**: Design widgets to work in different zone widths
2. **Prioritize**: Use the priority parameter to control widget ordering
3. **Reuse**: Create reusable widgets that can appear in multiple zones
4. **Context Processors**: Use context processors for dynamic content
5. **Performance**: Keep widgets lightweight to avoid impacting page load times

## Practical Example: Patient Alerts Widget

Let's create a patient alerts widget that appears on both the dashboard and patient detail pages:

```python
# plugins/alerts_plugin/widgets.py
from core.plugin_widgets import Widget, plugin_widget_registry
from django.utils import timezone
from datetime import timedelta

def get_alerts_context(request, context):
    """Get alerts for the current context."""
    from .models import PatientAlert

    organization = getattr(request.user, 'organization', None)
    if not organization:
        return {}

    # Different context depending on whether we're on a patient page or dashboard
    patient = context.get('patient')

    if patient:
        # Patient detail page - show this patient's alerts
        alerts = PatientAlert.objects.filter(
            patient=patient,
            resolved=False,
            organization=organization
        ).order_by('-severity', '-created_at')[:10]

        return {
            'alerts': alerts,
            'patient_specific': True,
            'alert_count': alerts.count()
        }
    else:
        # Dashboard - show recent unresolved alerts
        alerts = PatientAlert.objects.filter(
            resolved=False,
            organization=organization,
            created_at__gte=timezone.now() - timedelta(days=7)
        ).order_by('-severity', '-created_at')[:5]

        total_unresolved = PatientAlert.objects.filter(
            resolved=False,
            organization=organization
        ).count()

        return {
            'alerts': alerts,
            'patient_specific': False,
            'alert_count': total_unresolved,
            'recent_count': alerts.count()
        }

# Register the widget for multiple zones
plugin_widget_registry.register_widget(
    widget=Widget(
        name="patient_alerts_widget",
        plugin_id="alerts_plugin",
        template="alerts_plugin/widgets/patient_alerts.html",
        priority=5,
        css_classes="alerts-widget",
        context_processor=get_alerts_context
    ),
    zones=[
        "dashboard_top",                    # Show on main dashboard
        "patients_patient_detail_sidebar"   # Show on patient detail sidebar
    ]
)
```

With this approach, the same widget can display context-aware content in different locations throughout the application.

## Next Steps

In the next tutorial, we'll explore plugin settings and configuration, which allow users to customize plugin behavior for their organization.
