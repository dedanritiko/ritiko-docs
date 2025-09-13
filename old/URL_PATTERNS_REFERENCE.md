# URL Patterns Reference for Plugin Development

This document provides a reference for common URL patterns used in the Ritiko system, helpful for plugin development.

## Patient URLs (`patients:`)

Based on `patients/urls.py`, the available URL patterns are:

### Core Patient URLs
- `patients:new` - Create new patient
- `patients:detail` - Patient detail view (`<int:pk>/`)
- `patients:update` - Edit/update patient (`<int:pk>/update`) ⚠️ **Note: Use 'update', not 'edit'**
- `patients:schedule` - Patient schedule (`<int:pk>/schedule`)
- `patients:log` - Patient log (`<int:pk>/log`)

### Patient Actions
- `patients:deactivate` - Deactivate patient (`<int:pk>/deactivate`)
- `patients:reactivate` - Reactivate patient (`<int:pk>/reactivate`)
- `patients:care-team` - Patient care team (`<int:pk>/careteam`)

### Reports and Downloads
- `patients:portfolio` - Patient portfolio PDF (`<int:pk>/portfolio`)
- `patients:pdf-detail` - Patient detail PDF (`<int:pk>/pdf`)
- `patients:inactive` - Inactive patient view (`<int:pk>/inactive`)

### Eligibility and Forms
- `patients:eligibility-check` - Patient eligibility check (`<int:pk>/eligibility_check`)
- `patients:organization-eligibility-check` - Organization eligibility check
- `patients:blank-form` - Download blank form (`blank-form/<int:event_type_id>/`)

## Common Patterns for Plugin Development

### Action Buttons in Templates

```html
<!-- Correct way to create patient action buttons -->
<div class="btn-group btn-group-sm" role="group">
    <!-- View patient details -->
    <a href="{{ patient.get_absolute_url }}" class="btn btn-outline-primary btn-sm">
        <i class="fas fa-eye"></i> View
    </a>

    <!-- Edit patient (use 'update', not 'edit') -->
    {% if perms.patients.change_patient %}
        <a href="{% url 'patients:update' patient.pk %}" class="btn btn-outline-secondary btn-sm">
            <i class="fas fa-edit"></i> Edit
        </a>
    {% endif %}

    <!-- Call patient (if phone available) -->
    {% if patient.phone_number %}
        <a href="tel:{{ patient.phone_number }}" class="btn btn-outline-success btn-sm">
            <i class="fas fa-phone"></i> Call
        </a>
    {% endif %}
</div>
```

### Redirects in Views

```python
from django.shortcuts import redirect

# Correct way to redirect to patient detail
def some_view(request, patient_id):
    # ... do something ...
    return redirect("patients:detail", patient_id)

# Redirect to patient update/edit form
def some_other_view(request, patient_id):
    # ... do something ...
    return redirect("patients:update", patient_id)
```

### Link Generation in Python Code

```python
from django.urls import reverse

# Get patient detail URL
patient_detail_url = reverse("patients:detail", args=[patient.pk])

# Get patient update URL
patient_edit_url = reverse("patients:update", args=[patient.pk])
```

## Plugin URL Patterns

### Plugin URLs Naming Convention

Plugins should follow this pattern:
```
/<plugin_name>/<feature>/
```

Examples from current plugins:
- `/patient_plugin/patients/` - Enhanced patient list
- `/patient_email_plugin/emails/` - Patient email management

### Plugin URL Configuration

```python
# plugins/your_plugin/urls.py
from django.urls import path
from . import views

app_name = "your_plugin"

urlpatterns = [
    path("main-feature/", views.MainView.as_view(), name="main"),
    path("patient/<int:patient_id>/action/", views.PatientActionView.as_view(), name="patient_action"),
]
```

### URL Overrides (Advanced)

Plugins can override system URLs while preserving existing functionality:

```python
# plugins/patient_plugin/urls.py
from django.urls import path
from . import views

app_name = "patient_plugin"

# Standard plugin URLs (accessible at /patient_plugin/)
urlpatterns = [
    path("patients/", views.PatientListView.as_view(), name="patient_list"),
    path("patients/compact/", views.CompactPatientListView.as_view(), name="compact_patient_list"),
    path("patients/stats/", views.patient_stats_view, name="patient_stats"),
]

# URL Overrides - these override system URLs
URL_OVERRIDES = {
    "patients/": [
        # Override the main patient list view
        path("", views.PatientListView.as_view(), name="list"),
        # Add new plugin-specific functionality
        path("compact/", views.CompactPatientListView.as_view(), name="compact_patient_list"),
        path("stats/", views.patient_stats_view, name="patient_stats"),
    ]
}
```

**How URL Overrides Work:**
- `/patients/` → Plugin's enhanced patient list (overridden)
- `/patients/referrals/` → Original system referrals list (preserved)
- `/patients/compact/` → Plugin's compact view (new)
- `/patients/stats/` → Plugin's statistics view (new)
- `/patient_plugin/patients/` → Direct plugin access (also available)

**Template Compatibility:**
```django
<!-- All existing templates continue to work -->
<a href="{% url 'patients:list' %}">Patients</a>           <!-- Now shows plugin view -->
<a href="{% url 'patients:referrals-list' %}">Referrals</a> <!-- Original view -->
<a href="{% url 'patients:compact_patient_list' %}">Compact</a> <!-- New plugin view -->
```

## Permission Patterns

### Common Patient Permissions
- `patients.view_patient` - View patient information
- `patients.change_patient` - Edit/modify patient information
- `patients.add_patient` - Create new patients
- `patients.delete_patient` - Delete patients

### Plugin Permission Patterns
- `plugin_name.can_action_name` - Custom plugin permissions
- Example: `patient_plugin.can_view_patient_list`

## Common Mistakes to Avoid

### ❌ Incorrect URL References
```html
<!-- Wrong - 'edit' doesn't exist -->
<a href="{% url 'patients:edit' patient.pk %}">Edit</a>

<!-- Wrong - missing app namespace -->
<a href="{% url 'update' patient.pk %}">Edit</a>
```

### ✅ Correct URL References
```html
<!-- Correct - use 'update' -->
<a href="{% url 'patients:update' patient.pk %}">Edit</a>

<!-- Correct - with namespace -->
<a href="{% url 'patients:detail' patient.pk %}">View</a>
```

### ❌ Incorrect Permission Checks
```python
# Wrong - incorrect permission name
if request.user.has_perm('patients.edit_patient'):
    # ...
```

### ✅ Correct Permission Checks
```python
# Correct - actual permission name
if request.user.has_perm('patients.change_patient'):
    # ...
```

## Testing URL Patterns

You can test URL patterns in Django shell:

```python
# Test URL reversal
from django.urls import reverse
print(reverse('patients:detail', args=[1]))
# Should output: /patients/1/

print(reverse('patients:update', args=[1]))
# Should output: /patients/1/update

# Test URL resolution
from django.urls import resolve
match = resolve('/patients/1/')
print(match.view_name)  # Should output: patients:detail
```

This reference helps ensure plugin developers use the correct URL patterns and avoid common naming mistakes.
