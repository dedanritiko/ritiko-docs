# Plugin System Examples

This document provides practical examples of the plugin system implementation in Ritiko, demonstrating different ways to extend and override functionality.

## Example 1: Patient List Enhancement Plugin

### Overview
The Patient Plugin demonstrates how to create enhanced list views with advanced filtering, multiple display formats, and export capabilities.

### Plugin Structure
```
plugins/patient_plugin/
├── __init__.py
├── apps.py                 # Django app configuration
├── plugin.json            # Plugin metadata
├── settings.py            # Organization settings
├── permissions.py         # Plugin permissions
├── menus.py              # Plugin menus
├── views.py              # Enhanced views
├── urls.py               # URL patterns with overrides
├── tables.py             # Table definitions (if django-tables2 available)
├── filters.py            # Filter definitions
├── templates/
│   └── patient_plugin/
│       ├── patient_list.html
│       ├── compact_patient_list.html
│       ├── patient_stats.html
│       └── actions_column.html
└── static/               # Plugin static files
```

### Key Implementation Details

#### 1. Enhanced Views (`views.py`)
```python
from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic import ListView
from django.http import HttpResponse
from django_filters.views import FilterView

class PatientListView(PermissionRequiredMixin, FilterView, ListView):
    model = Patient
    filterset_class = PatientFilter
    template_name = "patient_plugin/patient_list.html"
    permission_required = "patient_plugin.can_view_patient_list"
    paginate_by = 25

    def get_queryset(self):
        return Patient.objects.filter(
            organization=self.request.user.organization
        ).select_related("organization", "patient_category")

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        queryset = self.get_queryset()
        context.update({
            "total_patients": queryset.count(),
            "active_patients": queryset.filter(is_active=True).count(),
            "inactive_patients": queryset.filter(is_active=False).count(),
        })
        return context
```

#### 2. Advanced Filtering (`filters.py`)
```python
import django_filters
from django import forms
from django.db import models

class PatientFilter(django_filters.FilterSet):
    name = django_filters.CharFilter(
        method="filter_name",
        widget=forms.TextInput(attrs={
            "class": "form-control",
            "placeholder": "Search by first or last name..."
        })
    )

    birth_date = django_filters.DateFromToRangeFilter(
        widget=django_filters.widgets.RangeWidget(attrs={
            "class": "form-control",
            "type": "date"
        })
    )

    def filter_name(self, queryset, name, value):
        if not value:
            return queryset
        return queryset.filter(
            models.Q(first_name__icontains=value) |
            models.Q(last_name__icontains=value)
        )

    class Meta:
        model = Patient
        fields = ["name", "mrn", "gender", "is_active", "patient_category"]
```

#### 3. Template with Bootstrap Styling
```html
<!-- patient_list.html -->
{% extends "base.html" %}
{% load crispy_forms_tags %}

{% block content %}
<div class="container-fluid">
    <div class="row">
        <div class="col-12">
            <div class="card">
                <div class="card-header">
                    <h3 class="card-title">Enhanced Patient List</h3>
                    <div class="card-tools">
                        <a href="?export=csv" class="btn btn-sm btn-outline-primary">
                            <i class="fas fa-file-csv"></i> Export CSV
                        </a>
                    </div>
                </div>
                <div class="card-body">
                    <!-- Statistics -->
                    <div class="row mb-3">
                        <div class="col-md-4">
                            <div class="info-box bg-info">
                                <span class="info-box-icon">
                                    <i class="fas fa-users"></i>
                                </span>
                                <div class="info-box-content">
                                    <span class="info-box-text">Total Patients</span>
                                    <span class="info-box-number">{{ total_patients }}</span>
                                </div>
                            </div>
                        </div>
                        <!-- More statistics... -->
                    </div>

                    <!-- Filter Form -->
                    <form method="get" class="card">
                        <div class="card-body">
                            <div class="row">
                                <div class="col-md-3">
                                    {{ filter.form.name|as_crispy_field }}
                                </div>
                                <div class="col-md-2">
                                    {{ filter.form.mrn|as_crispy_field }}
                                </div>
                                <!-- More filter fields... -->
                            </div>
                        </div>
                    </form>

                    <!-- Patient Table -->
                    <div class="table-responsive">
                        <table class="table table-striped table-hover">
                            <thead class="table-dark">
                                <tr>
                                    <th>Name</th>
                                    <th>MRN</th>
                                    <th>Birth Date</th>
                                    <th>Phone</th>
                                    <th>Active</th>
                                    <th>Actions</th>
                                </tr>
                            </thead>
                            <tbody>
                                {% for patient in object_list %}
                                <tr>
                                    <td>
                                        <a href="{{ patient.get_absolute_url }}">
                                            {{ patient.get_full_name }}
                                        </a>
                                    </td>
                                    <td>{{ patient.mrn|default:"-" }}</td>
                                    <td>{{ patient.birth_date|date:"m/d/Y" }}</td>
                                    <td>{{ patient.phone_number|default:"-" }}</td>
                                    <td>
                                        {% if patient.is_active %}
                                            <span class="badge badge-success">Active</span>
                                        {% else %}
                                            <span class="badge badge-warning">Inactive</span>
                                        {% endif %}
                                    </td>
                                    <td>
                                        <div class="btn-group btn-group-sm">
                                            <a href="{{ patient.get_absolute_url }}"
                                               class="btn btn-outline-primary">
                                                <i class="fas fa-eye"></i>
                                            </a>
                                            <!-- More actions... -->
                                        </div>
                                    </td>
                                </tr>
                                {% endfor %}
                            </tbody>
                        </table>
                    </div>

                    <!-- Pagination -->
                    {% if is_paginated %}
                    <nav>
                        <ul class="pagination">
                            {% if page_obj.has_previous %}
                                <li class="page-item">
                                    <a class="page-link" href="?page={{ page_obj.previous_page_number }}">Previous</a>
                                </li>
                            {% endif %}
                            <!-- Page numbers... -->
                        </ul>
                    </nav>
                    {% endif %}
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

#### 4. URL Configuration with Overrides (`urls.py`)
```python
from django.urls import path
from . import views

app_name = "patient_plugin"

# Standard plugin URLs (accessible at /patient_plugin/)
urlpatterns = [
    path("patients/", views.PatientListView.as_view(), name="patient_list"),
    path("patients/compact/", views.CompactPatientListView.as_view(), name="compact_patient_list"),
    path("patients/stats/", views.patient_stats_view, name="patient_stats"),
]

# URL Overrides - these override system URLs while preserving others
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

**How the URL Override Works:**
- `/patients/` → Plugin's enhanced patient list (overridden)
- `/patients/referrals/` → Original system referrals list (preserved)
- `/patients/compact/` → Plugin's compact view (new functionality)
- `/patients/stats/` → Plugin's statistics view (new functionality)
- `/patient_plugin/patients/` → Direct plugin access (still available)

**Template Compatibility:**
```django
<!-- All existing templates continue to work -->
<a href="{% url 'patients:list' %}">Patients</a>                    <!-- Now shows plugin view -->
<a href="{% url 'patients:referrals-list' %}">Referrals</a>         <!-- Original system view -->
<a href="{% url 'patients:compact_patient_list' %}">Compact View</a> <!-- New plugin view -->
```

## Example 2: Patient Email Extension Plugin

### Overview
The Patient Email Plugin demonstrates model extension, page injection, and template hooks without modifying core files.

### Key Implementation Details

#### 1. Model Extension (`model_extensions.py`)
```python
from patients.models.people import Patient

def send_email(self, subject, message=None, html_message=None,
               template_name=None, context=None, from_email=None):
    """Send an email to this patient using their email profile."""
    email_profile = self.get_email_profile()
    return email_profile.send_email(
        subject=subject,
        message=message,
        html_message=html_message,
        template_name=template_name,
        context=context,
        from_email=from_email,
    )

def has_email(self):
    """Check if this patient has an email address configured."""
    try:
        email_profile = self.email_profile
        return bool(email_profile.get_preferred_email())
    except:
        return False

def get_email_profile(self):
    """Get or create the email profile for this patient."""
    from .models import PatientEmail

    email_profile, created = PatientEmail.objects.get_or_create(
        patient=self,
        defaults={"organization": self.organization}
    )
    return email_profile

# Extend Patient model with new methods
Patient.add_to_class("send_email", send_email)
Patient.add_to_class("has_email", has_email)
Patient.add_to_class("get_email_profile", get_email_profile)
```

#### 2. Related Model (`models.py`)
```python
from django.db import models
from patients.models.people import Patient

class PatientEmail(RitikoModel):
    """Email extension for Patient model via one-to-one relationship."""

    patient = models.OneToOneField(
        Patient,
        on_delete=models.CASCADE,
        related_name="email_profile"
    )

    email = models.EmailField(blank=True, null=True)
    secondary_email = models.EmailField(blank=True, null=True)
    email_verified = models.BooleanField(default=False)
    email_notifications_enabled = models.BooleanField(default=True)
    preferred_email = models.CharField(
        max_length=10,
        choices=[("primary", "Primary"), ("secondary", "Secondary")],
        default="primary"
    )

    def get_preferred_email(self):
        """Get the preferred email address."""
        if self.preferred_email == "secondary" and self.secondary_email:
            return self.secondary_email
        return self.email or self.secondary_email

    def send_email(self, subject, message=None, **kwargs):
        """Send email to this patient."""
        email_address = self.get_preferred_email()
        if not email_address or not self.email_notifications_enabled:
            return False

        # Email sending logic here
        from django.core.mail import send_mail
        try:
            result = send_mail(
                subject=subject,
                message=message or "",
                from_email=kwargs.get('from_email'),
                recipient_list=[email_address],
                fail_silently=False,
            )
            return bool(result)
        except Exception as e:
            print(f"Failed to send email: {e}")
            return False
```

#### 3. Template Hooks (`template_hooks.py`)
```python
from django.template.loader import render_to_string

class PatientTemplateHooks:
    """Template hooks for patient pages."""

    @staticmethod
    def patient_detail_email_section(context, request):
        """Add email section to patient detail page."""
        patient = context.get('patient')
        if not patient:
            return ""

        try:
            email_profile = patient.email_profile
        except:
            email_profile = None

        hook_context = {
            "patient": patient,
            "email_profile": email_profile,
            "has_email_permission": request.user.has_perm(
                "patient_email_plugin.can_manage_patient_emails"
            ),
        }

        return render_to_string(
            "patient_email_plugin/partials/patient_detail_email_section.html",
            hook_context,
            request=request
        )

# Register template hooks
def register_template_hooks():
    try:
        from core.template_hooks import template_hook_registry

        template_hook_registry.register_hook(
            "patient_detail_additional_sections",
            PatientTemplateHooks.patient_detail_email_section,
            plugin_id="patient_email_plugin",
            priority=10
        )

        print("📧 Patient email template hooks registered successfully")
    except ImportError:
        print("⚠️ Template hook registry not available")

# Auto-register hooks
register_template_hooks()
```

#### 4. Page Extension Template
```html
<!-- partials/patient_detail_email_section.html -->
<div class="card mt-3" id="patient-email-section">
    <div class="card-header">
        <h5 class="card-title mb-0">
            <i class="fas fa-envelope"></i>
            Email Information
        </h5>
        {% if has_email_permission %}
            <div class="card-tools">
                <a href="{% url 'patient_email_plugin:edit_patient_email' patient.pk %}"
                   class="btn btn-sm btn-outline-primary">
                    <i class="fas fa-edit"></i> Edit Email Settings
                </a>
            </div>
        {% endif %}
    </div>
    <div class="card-body">
        {% if email_profile %}
            <div class="row">
                <div class="col-md-6">
                    <strong>Primary Email:</strong>
                    {% if email_profile.email %}
                        <span class="text-success">
                            {{ email_profile.email }}
                            {% if email_profile.email_verified %}
                                <i class="fas fa-check-circle text-success" title="Verified"></i>
                            {% endif %}
                        </span>
                    {% else %}
                        <span class="text-muted">Not provided</span>
                    {% endif %}
                </div>
                <div class="col-md-6">
                    <strong>Notifications:</strong>
                    {% if email_profile.email_notifications_enabled %}
                        <span class="badge badge-success">Enabled</span>
                    {% else %}
                        <span class="badge badge-warning">Disabled</span>
                    {% endif %}
                </div>
            </div>

            <!-- Quick action buttons -->
            {% if has_email_permission and email_profile.get_preferred_email %}
                <div class="row mt-3">
                    <div class="col-12">
                        <button type="button" class="btn btn-sm btn-outline-primary"
                                onclick="sendWelcomeEmail()">
                            <i class="fas fa-hand-wave"></i> Send Welcome Email
                        </button>
                        <a href="{% url 'patient_email_plugin:send_email' patient.pk %}"
                           class="btn btn-sm btn-outline-success">
                            <i class="fas fa-paper-plane"></i> Send Custom Email
                        </a>
                    </div>
                </div>
            {% endif %}
        {% else %}
            <div class="text-center text-muted py-4">
                <i class="fas fa-envelope-open-text fa-3x mb-3"></i>
                <p>No email profile configured for this patient.</p>
            </div>
        {% endif %}
    </div>
</div>

<script>
function sendWelcomeEmail() {
    // AJAX call to send welcome email
    fetch("{% url 'patient_email_plugin:ajax_email_quick_actions' patient.pk %}", {
        method: 'POST',
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'X-CSRFToken': document.querySelector('[name=csrfmiddlewaretoken]').value
        },
        body: 'action=send_welcome'
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            alert('Welcome email sent successfully!');
        } else {
            alert('Failed to send email: ' + data.message);
        }
    });
}
</script>
```

## Example 3: Simple Override Plugin

### Overview
A minimal plugin that overrides a single view without extensive functionality.

### Implementation

#### Plugin Structure
```
plugins/simple_override_plugin/
├── __init__.py
├── apps.py
├── plugin.json
├── views.py
├── urls.py
└── templates/
    └── simple_override_plugin/
        └── custom_page.html
```

#### View Override (`views.py`)
```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required
from patients.models.people import Patient

@login_required
def custom_patient_summary(request):
    """Custom patient summary page with different layout."""
    patients = Patient.objects.filter(
        organization=request.user.organization,
        is_active=True
    ).order_by('-created_on')[:10]  # Latest 10 patients

    stats = {
        'total': Patient.objects.filter(organization=request.user.organization).count(),
        'active': patients.count(),
        'recent_additions': patients.count(),
    }

    context = {
        'patients': patients,
        'stats': stats,
        'page_title': 'Custom Patient Summary',
    }

    return render(request, 'simple_override_plugin/custom_page.html', context)
```

#### URL Configuration (`urls.py`)
```python
from django.urls import path
from . import views

app_name = "simple_override_plugin"

urlpatterns = [
    path("patient-summary/", views.custom_patient_summary, name="patient_summary"),
]
```

## Example 4: Function Enhancement Plugin

### Overview
Demonstrates how to enhance existing functionality with decorator patterns.

### Implementation

#### Function Decorators (`decorators.py`)
```python
from functools import wraps
import logging

logger = logging.getLogger(__name__)

def audit_patient_access(func):
    """Decorator to audit patient data access."""
    @wraps(func)
    def wrapper(request, *args, **kwargs):
        # Log patient access
        patient_id = kwargs.get('patient_id') or args[0] if args else None
        logger.info(f"User {request.user.username} accessed patient {patient_id}")

        # Call original function
        response = func(request, *args, **kwargs)

        # Log successful access
        logger.info(f"Patient {patient_id} access completed successfully")

        return response
    return wrapper

def require_patient_permission(permission):
    """Decorator to check patient-specific permissions."""
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            if not request.user.has_perm(permission):
                from django.http import HttpResponseForbidden
                return HttpResponseForbidden("Permission denied")
            return func(request, *args, **kwargs)
        return wrapper
    return decorator
```

#### Enhanced Views (`views.py`)
```python
from .decorators import audit_patient_access, require_patient_permission

@audit_patient_access
@require_patient_permission('patients.view_patient')
def enhanced_patient_detail(request, patient_id):
    """Enhanced patient detail view with auditing."""
    # View logic here
    pass
```

## Usage Examples

### 1. Using Enhanced Patient List
```python
# Access the enhanced patient list
# URL: /patient_plugin/patients/

# With filtering
# URL: /patient_plugin/patients/?name=John&is_active=true

# Export functionality
# URL: /patient_plugin/patients/?export=csv
```

### 2. Using Patient Email Extensions
```python
# In Django shell or views
from patients.models.people import Patient

# Get a patient
patient = Patient.objects.get(pk=1)

# Use extended functionality
if patient.has_email():
    # Send a welcome email
    success = patient.send_welcome_email()

    # Send custom email
    patient.send_email(
        subject="Appointment Reminder",
        message="Your appointment is tomorrow at 10:00 AM"
    )

    # Send templated email
    patient.send_email(
        subject="Care Plan Update",
        template_name="patient_email_plugin/emails/care_plan_update.html",
        context={"care_plan_details": "Updated care plan details"}
    )
```

### 3. Configuration Examples
```python
# Check plugin settings in views
from core.plugin_organization_settings import get_organization_plugin_setting

def my_view(request):
    # Get plugin setting for current organization
    page_size = get_organization_plugin_setting(
        request.user.organization,
        "patient_plugin",
        "patient_list_page_size"
    )

    # Use setting in query
    patients = Patient.objects.all()[:page_size]
```

These examples demonstrate the flexibility and power of the plugin system, showing how to extend functionality without modifying core application files while maintaining clean separation of concerns.
