# Django Tables2 Integration Guide

This guide shows how to use django-tables2 and django-filter in the Ritiko plugin system for creating advanced table views with sorting, filtering, and export capabilities.

## Prerequisites

- `django-tables2==2.7.5` ✅ (already installed)
- `django-filter==25.1` ✅ (already installed)
- Both packages are configured in `INSTALLED_APPS` ✅

## Basic Setup

### 1. INSTALLED_APPS Configuration

The following packages are already configured in `ritiko/settings/base.py`:

```python
INSTALLED_APPS = [
    # ... other apps ...
    "django_tables2",
    "django_filters",
    # ...
]
```

### 2. Plugin Structure for Tables

```
plugins/your_plugin/
├── tables.py          # Table definitions
├── filters.py         # Filter definitions
├── views.py           # Views using tables
├── templates/
│   └── your_plugin/
│       └── list.html  # Template with table rendering
```

## Table Definitions (tables.py)

### Basic Table Example

```python
import django_tables2 as tables
from patients.models.people import Patient


class PatientTable(tables.Table):
    """Enhanced patient table with django-tables2."""

    # Simple column
    name = tables.Column(
        verbose_name="Name",
        accessor="get_full_name"
    )

    # MRN column
    mrn = tables.Column(verbose_name="MRN")

    # Date column with custom format
    birth_date = tables.DateColumn(
        format="m/d/Y",
        verbose_name="Birth Date"
    )

    # Phone column
    phone_number = tables.Column(verbose_name="Phone")

    # Choice field display
    gender = tables.Column(verbose_name="Gender")

    # Boolean column with custom display
    is_active = tables.BooleanColumn(
        verbose_name="Active",
        yesno="✓,✗"
    )

    # Custom column with method rendering
    care_team_count = tables.Column(
        verbose_name="Care Team Size",
        empty_values=(),
        orderable=False
    )

    def render_care_team_count(self, record):
        """Render care team count."""
        if hasattr(record, 'care_team'):
            return record.care_team.count()
        return 0

    # Template column for actions
    actions = tables.TemplateColumn(
        template_name="patient_plugin/actions_column.html",
        verbose_name="Actions",
        orderable=False
    )

    class Meta:
        model = Patient
        template_name = "django_tables2/bootstrap4.html"
        fields = (
            "name",
            "mrn",
            "birth_date",
            "phone_number",
            "gender",
            "is_active",
            "care_team_count",
            "actions"
        )
        attrs = {
            "class": "table table-striped table-hover",
            "thead": {"class": "table-dark"}
        }
        per_page = 25
```

### Link Columns

```python
# Link to detail page (if URL exists)
name = tables.LinkColumn(
    "patients:detail",  # URL name
    args=[tables.A("pk")],
    verbose_name="Name",
    accessor="get_full_name"
)
```

### Custom Column Rendering

```python
def render_phone_number(self, value, record):
    """Render phone with tel: link."""
    if value:
        return format_html('<a href="tel:{}">{}</a>', value, value)
    return "-"
```

## Filter Definitions (filters.py)

### Comprehensive Filter Example

```python
import django_filters
from django import forms
from django.db import models
from patients.models.people import Patient, PatientCategory


class PatientFilter(django_filters.FilterSet):
    """Enhanced patient filter with django-filter."""

    # Text search with custom method
    name = django_filters.CharFilter(
        method="filter_name",
        label="Name",
        widget=forms.TextInput(attrs={
            "class": "form-control",
            "placeholder": "Search by first or last name..."
        })
    )

    # Simple text filter
    mrn = django_filters.CharFilter(
        lookup_expr="icontains",
        label="MRN",
        widget=forms.TextInput(attrs={
            "class": "form-control",
            "placeholder": "Search by MRN..."
        })
    )

    # Date range filter
    birth_date = django_filters.DateFromToRangeFilter(
        label="Birth Date Range",
        widget=django_filters.widgets.RangeWidget(attrs={
            "class": "form-control",
            "type": "date"
        })
    )

    # Choice filter
    gender = django_filters.ChoiceFilter(
        choices=[("", "All")] + Patient._meta.get_field("gender").choices,
        label="Gender",
        widget=forms.Select(attrs={"class": "form-control"})
    )

    # Boolean filter
    is_active = django_filters.BooleanFilter(
        label="Active Only",
        widget=forms.CheckboxInput(attrs={"class": "form-check-input"})
    )

    # Foreign key filter
    patient_category = django_filters.ModelChoiceFilter(
        queryset=PatientCategory.objects.all(),
        label="Category",
        widget=forms.Select(attrs={"class": "form-control"})
    )

    # Custom method filter
    has_care_team = django_filters.BooleanFilter(
        method="filter_has_care_team",
        label="Has Care Team",
        widget=forms.CheckboxInput(attrs={"class": "form-check-input"})
    )

    class Meta:
        model = Patient
        fields = [
            "name",
            "mrn",
            "birth_date",
            "gender",
            "is_active",
            "patient_category",
            "has_care_team"
        ]

    def filter_name(self, queryset, name, value):
        """Filter by first name or last name."""
        if not value:
            return queryset
        return queryset.filter(
            models.Q(first_name__icontains=value) |
            models.Q(last_name__icontains=value)
        )

    def filter_has_care_team(self, queryset, name, value):
        """Filter patients who have care team members."""
        if value is None:
            return queryset
        if value:
            return queryset.filter(care_team__isnull=False).distinct()
        else:
            return queryset.filter(care_team__isnull=True)
```

## Views Integration (views.py)

### Complete View with Tables and Filters

```python
from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic import ListView
from django_filters.views import FilterView
from django_tables2 import SingleTableMixin
from django_tables2.export import ExportMixin

from patients.models.people import Patient
from .filters import PatientFilter
from .tables import PatientTable


class PatientListView(PermissionRequiredMixin, ExportMixin, SingleTableMixin, FilterView):
    """Enhanced patient list view with tables and filters."""

    model = Patient
    table_class = PatientTable
    filterset_class = PatientFilter
    template_name = "patient_plugin/patient_list.html"
    context_object_name = "patients"
    permission_required = "patient_plugin.can_view_patient_list"
    export_formats = ["csv", "xlsx", "json"]

    def get_queryset(self):
        """Get patients for current organization with optimizations."""
        return Patient.objects.filter(
            organization=self.request.user.organization
        ).select_related(
            "organization",
            "patient_category"
        ).prefetch_related(
            "care_team"
        )

    def get_context_data(self, **kwargs):
        """Add extra context for statistics."""
        context = super().get_context_data(**kwargs)
        queryset = self.get_queryset()
        context.update({
            "total_patients": queryset.count(),
            "active_patients": queryset.filter(is_active=True).count(),
            "inactive_patients": queryset.filter(is_active=False).count(),
        })
        return context
```

### View Mixins Breakdown

- `PermissionRequiredMixin`: Handles permission checking
- `ExportMixin`: Adds export functionality (CSV, Excel, JSON)
- `SingleTableMixin`: Integrates the table with the view
- `FilterView`: Integrates filtering capabilities

## Template Integration

### Basic Template Structure

```html
{% extends "base.html" %}
{% load django_tables2 %}
{% load crispy_forms_tags %}

{% block title %}Enhanced Patient List{% endblock %}

{% block content %}
<div class="container-fluid">
    <div class="row">
        <div class="col-12">
            <div class="card">
                <div class="card-header">
                    <h3 class="card-title">Enhanced Patient List</h3>
                    <div class="card-tools">
                        <!-- Export buttons -->
                        <div class="btn-group">
                            <a href="?export=csv" class="btn btn-sm btn-outline-primary">
                                <i class="fas fa-file-csv"></i> CSV
                            </a>
                            <a href="?export=xlsx" class="btn btn-sm btn-outline-success">
                                <i class="fas fa-file-excel"></i> Excel
                            </a>
                            <a href="?export=json" class="btn btn-sm btn-outline-info">
                                <i class="fas fa-code"></i> JSON
                            </a>
                        </div>
                    </div>
                </div>
                <div class="card-body">
                    <!-- Filter Form -->
                    <form method="get" class="card mb-3">
                        <div class="card-body">
                            <div class="row">
                                <div class="col-md-3">
                                    {{ filter.form.name|as_crispy_field }}
                                </div>
                                <div class="col-md-2">
                                    {{ filter.form.mrn|as_crispy_field }}
                                </div>
                                <div class="col-md-2">
                                    {{ filter.form.gender|as_crispy_field }}
                                </div>
                                <div class="col-md-2">
                                    {{ filter.form.is_active|as_crispy_field }}
                                </div>
                                <div class="col-md-3">
                                    {{ filter.form.patient_category|as_crispy_field }}
                                </div>
                            </div>
                            <div class="row mt-3">
                                <div class="col-12">
                                    <button type="submit" class="btn btn-primary">
                                        <i class="fas fa-search"></i> Filter
                                    </button>
                                    <a href="?" class="btn btn-secondary">
                                        <i class="fas fa-undo"></i> Clear
                                    </a>
                                </div>
                            </div>
                        </div>
                    </form>

                    <!-- The Table (django-tables2 magic happens here) -->
                    {% render_table table %}
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### Action Column Template

Create `templates/patient_plugin/actions_column.html`:

```html
<div class="btn-group btn-group-sm" role="group">
    <a href="{{ record.get_absolute_url }}"
       class="btn btn-outline-primary btn-sm"
       title="View Details">
        <i class="fas fa-eye"></i>
    </a>
    {% if perms.patients.change_patient %}
        <a href="{% url 'patients:edit' record.pk %}"
           class="btn btn-outline-secondary btn-sm"
           title="Edit Patient">
            <i class="fas fa-edit"></i>
        </a>
    {% endif %}
    {% if record.phone_number %}
        <a href="tel:{{ record.phone_number }}"
           class="btn btn-outline-success btn-sm"
           title="Call Patient">
            <i class="fas fa-phone"></i>
        </a>
    {% endif %}
</div>
```

## Advanced Features

### 1. Custom Table Templates

Create custom table templates for different layouts:

```python
# In table Meta class
class Meta:
    template_name = "patient_plugin/custom_table.html"
```

### 2. Column Visibility

```python
# Add column visibility controls
class PatientTable(tables.Table):
    class Meta:
        sequence = ("name", "mrn", "birth_date", "...")
        exclude = ("internal_field",)
```

### 3. Export Configuration

```python
class PatientListView(..., ExportMixin, ...):
    export_formats = ["csv", "xlsx", "json", "tsv"]

    def create_export(self, export_format):
        """Customize export behavior."""
        exporter = super().create_export(export_format)

        if export_format == 'xlsx':
            # Customize Excel export
            pass

        return exporter
```

### 4. Pagination Customization

```python
# In table Meta class
class Meta:
    per_page = 50

# Or in view
class PatientListView(...):
    paginate_by = 25
```

### 5. Ordering

```python
# Default ordering
class PatientTable(tables.Table):
    class Meta:
        order_by = ("-created_on",)  # Default sort
```

## Bootstrap Integration

Django Tables2 works seamlessly with Bootstrap 4/5:

```python
class Meta:
    template_name = "django_tables2/bootstrap4.html"
    attrs = {
        "class": "table table-striped table-hover",
        "thead": {"class": "table-dark"}
    }
```

## URL Configuration

```python
# urls.py
from django.urls import path
from . import views

app_name = "patient_plugin"

urlpatterns = [
    path("patients/", views.PatientListView.as_view(), name="patient_list"),
]
```

## Usage Examples

### 1. Basic Patient List
```
GET /patient_plugin/patients/
```

### 2. Filtered Results
```
GET /patient_plugin/patients/?name=John&is_active=true
```

### 3. Sorted Results
```
GET /patient_plugin/patients/?sort=birth_date
GET /patient_plugin/patients/?sort=-name  # Descending
```

### 4. Export Data
```
GET /patient_plugin/patients/?export=csv
GET /patient_plugin/patients/?export=xlsx
```

### 5. Combined Operations
```
GET /patient_plugin/patients/?name=John&is_active=true&sort=name&export=csv
```

## Best Practices

1. **Performance Optimization**
   - Use `select_related()` and `prefetch_related()` in querysets
   - Limit the number of columns for large datasets
   - Consider pagination for large tables

2. **User Experience**
   - Provide clear filter labels and placeholders
   - Use appropriate input widgets for different field types
   - Include export options for data analysis

3. **Security**
   - Always check permissions in views
   - Validate filter inputs
   - Limit export access based on user roles

4. **Accessibility**
   - Use semantic HTML in templates
   - Provide proper ARIA labels
   - Ensure keyboard navigation works

This integration provides a powerful and flexible way to create advanced table views in the Ritiko plugin system, with full sorting, filtering, and export capabilities.
