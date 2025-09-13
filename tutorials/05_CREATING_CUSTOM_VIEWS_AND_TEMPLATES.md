# Creating Custom Views and Templates

This tutorial explores how to create custom views and templates for your plugins. You'll learn how to build both function-based and class-based views, create templates, handle forms, and integrate your views with the Ritiko system.

## Introduction to Views in Plugins

Views are the components responsible for processing HTTP requests and returning responses. In Django and Ritiko plugins, views can be implemented as:

1. **Function-based views**: Simple Python functions that take a request and return a response
2. **Class-based views**: Python classes that provide a more organized approach with built-in functionality

Both approaches have their advantages, and your plugin can use either or both.

## Function-Based Views

Function-based views are straightforward Python functions that take a request object and return a response. They're simple to implement and understand.

### Basic Function-Based View

```python
# plugins/analytics_plugin/views.py
from django.shortcuts import render
from django.contrib.auth.decorators import login_required
from django.http import HttpResponse

@login_required
def dashboard_view(request):
    """A simple dashboard view."""
    context = {
        'title': 'Analytics Dashboard',
        'metrics': {
            'total_patients': 120,
            'active_patients': 85,
            'appointments_today': 12,
        }
    }
    return render(request, 'analytics_plugin/dashboard.html', context)

def simple_metric_view(request):
    """A simple view that returns plain text."""
    return HttpResponse("Total Patients: 120")
```

### Function-Based View with Parameters

```python
# plugins/analytics_plugin/views.py
from django.shortcuts import render, get_object_or_404
from patients.models.people import Patient

@login_required
def patient_metrics_view(request, patient_id):
    """View displaying metrics for a specific patient."""
    patient = get_object_or_404(Patient, pk=patient_id)

    # Calculate metrics
    metrics = {
        'appointments_count': patient.appointments.count(),
        'last_visit': patient.appointments.order_by('-date').first(),
        'medication_count': patient.medications.count(),
    }

    context = {
        'patient': patient,
        'metrics': metrics,
        'title': f'Metrics for {patient.get_full_name()}',
    }

    return render(request, 'analytics_plugin/patient_metrics.html', context)
```

### Function-Based View with Form Handling

```python
# plugins/analytics_plugin/views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from .forms import ReportForm

@login_required
def generate_report_view(request):
    """View to generate a custom report."""
    if request.method == 'POST':
        form = ReportForm(request.POST)
        if form.is_valid():
            # Process the form data
            report_type = form.cleaned_data['report_type']
            start_date = form.cleaned_data['start_date']
            end_date = form.cleaned_data['end_date']

            # Generate the report
            from .reports import generate_report
            report_data = generate_report(report_type, start_date, end_date)

            # Store report data in session for the results view
            request.session['report_data'] = report_data

            # Redirect to results page
            messages.success(request, "Report generated successfully!")
            return redirect('analytics_plugin:report_results')
    else:
        # Initial form display
        form = ReportForm()

    context = {
        'form': form,
        'title': 'Generate Custom Report',
    }

    return render(request, 'analytics_plugin/generate_report.html', context)

@login_required
def report_results_view(request):
    """View to display report results."""
    report_data = request.session.get('report_data', None)

    if not report_data:
        messages.error(request, "No report data found. Please generate a report first.")
        return redirect('analytics_plugin:generate_report')

    context = {
        'report_data': report_data,
        'title': 'Report Results',
    }

    return render(request, 'analytics_plugin/report_results.html', context)
```

## Class-Based Views

Class-based views provide a more structured approach with built-in functionality for common patterns like displaying lists, handling forms, and providing detail views.

### Basic Class-Based Views

```python
# plugins/inventory_plugin/views.py
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.urls import reverse_lazy
from .models import InventoryItem

class InventoryListView(LoginRequiredMixin, ListView):
    """Display a list of inventory items."""
    model = InventoryItem
    template_name = 'inventory_plugin/item_list.html'
    context_object_name = 'items'
    paginate_by = 25

    def get_queryset(self):
        """Filter items by the current user's organization."""
        return InventoryItem.objects.filter(
            organization=self.request.user.organization
        ).order_by('name')

    def get_context_data(self, **kwargs):
        """Add extra context data."""
        context = super().get_context_data(**kwargs)
        context['total_items'] = self.get_queryset().count()
        context['low_stock_items'] = self.get_queryset().filter(
            quantity__lt=10
        ).count()
        return context

class InventoryDetailView(LoginRequiredMixin, DetailView):
    """Display details for a specific inventory item."""
    model = InventoryItem
    template_name = 'inventory_plugin/item_detail.html'
    context_object_name = 'item'
```

### Form Handling with Class-Based Views

```python
# plugins/inventory_plugin/views.py
class InventoryCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    """Create a new inventory item."""
    model = InventoryItem
    template_name = 'inventory_plugin/item_form.html'
    form_class = InventoryItemForm
    permission_required = 'inventory_plugin.add_inventoryitem'
    success_url = reverse_lazy('inventory_plugin:item_list')

    def form_valid(self, form):
        """Set the organization before saving."""
        form.instance.organization = self.request.user.organization
        messages.success(self.request, "Inventory item created successfully!")
        return super().form_valid(form)

    def get_context_data(self, **kwargs):
        """Add extra context data."""
        context = super().get_context_data(**kwargs)
        context['title'] = 'Add Inventory Item'
        return context

class InventoryUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    """Update an existing inventory item."""
    model = InventoryItem
    template_name = 'inventory_plugin/item_form.html'
    form_class = InventoryItemForm
    permission_required = 'inventory_plugin.change_inventoryitem'

    def get_success_url(self):
        """Return to the detail view after update."""
        return reverse_lazy('inventory_plugin:item_detail', kwargs={'pk': self.object.pk})

    def form_valid(self, form):
        """Add success message on valid form submission."""
        messages.success(self.request, "Inventory item updated successfully!")
        return super().form_valid(form)

    def get_context_data(self, **kwargs):
        """Add extra context data."""
        context = super().get_context_data(**kwargs)
        context['title'] = f'Edit {self.object.name}'
        return context
```

### Filtering and Searching

```python
# plugins/inventory_plugin/views.py
from django_filters.views import FilterView
from .filters import InventoryItemFilter

class InventoryFilterView(LoginRequiredMixin, FilterView):
    """Display a filtered list of inventory items."""
    model = InventoryItem
    template_name = 'inventory_plugin/item_list.html'
    filterset_class = InventoryItemFilter
    paginate_by = 25

    def get_queryset(self):
        """Filter items by the current user's organization."""
        return InventoryItem.objects.filter(
            organization=self.request.user.organization
        ).order_by('name')

    def get_context_data(self, **kwargs):
        """Add extra context data."""
        context = super().get_context_data(**kwargs)
        context['total_items'] = self.get_queryset().count()
        return context
```

## Creating Templates

Templates define the HTML structure and presentation of your plugin's pages. Ritiko follows Django's template system, which allows for template inheritance, includes, and custom tags.

### Template Directory Structure

```
plugins/your_plugin/
└── templates/
    └── your_plugin/
        ├── base.html             # Plugin-specific base template
        ├── dashboard.html        # Dashboard template
        ├── item_list.html        # List template
        ├── item_detail.html      # Detail template
        ├── item_form.html        # Form template
        └── partials/             # Partial templates for reuse
            ├── _header.html
            ├── _sidebar.html
            └── _item_card.html
```

### Base Template

```html
<!-- plugins/inventory_plugin/templates/inventory_plugin/base.html -->
{% extends "base.html" %}
{% load static %}

{% block extra_css %}
<link rel="stylesheet" href="{% static 'inventory_plugin/css/inventory.css' %}">
{% endblock %}

{% block extra_js %}
<script src="{% static 'inventory_plugin/js/inventory.js' %}"></script>
{% endblock %}

{% block content %}
<div class="container-fluid">
    <div class="row">
        <div class="col-md-3">
            {% include "inventory_plugin/partials/_sidebar.html" %}
        </div>
        <div class="col-md-9">
            <div class="card">
                <div class="card-header">
                    <h3 class="card-title">{% block page_title %}Inventory Management{% endblock %}</h3>
                </div>
                <div class="card-body">
                    {% block plugin_content %}{% endblock %}
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### List Template

```html
<!-- plugins/inventory_plugin/templates/inventory_plugin/item_list.html -->
{% extends "inventory_plugin/base.html" %}
{% load static %}

{% block page_title %}Inventory Items{% endblock %}

{% block plugin_content %}
<div class="row mb-4">
    <div class="col-md-4">
        <div class="info-box bg-info">
            <span class="info-box-icon"><i class="fas fa-boxes"></i></span>
            <div class="info-box-content">
                <span class="info-box-text">Total Items</span>
                <span class="info-box-number">{{ total_items }}</span>
            </div>
        </div>
    </div>
    <div class="col-md-4">
        <div class="info-box bg-warning">
            <span class="info-box-icon"><i class="fas fa-exclamation-triangle"></i></span>
            <div class="info-box-content">
                <span class="info-box-text">Low Stock Items</span>
                <span class="info-box-number">{{ low_stock_items }}</span>
            </div>
        </div>
    </div>
    <div class="col-md-4">
        <div class="info-box bg-success">
            <span class="info-box-icon"><i class="fas fa-plus"></i></span>
            <div class="info-box-content">
                <a href="{% url 'inventory_plugin:item_create' %}" class="text-white">
                    <span class="info-box-text">Add New Item</span>
                </a>
            </div>
        </div>
    </div>
</div>

<!-- Filter Form -->
<form method="get" class="mb-4">
    <div class="row">
        <div class="col-md-3">{{ filter.form.name.label_tag }} {{ filter.form.name }}</div>
        <div class="col-md-3">{{ filter.form.category.label_tag }} {{ filter.form.category }}</div>
        <div class="col-md-3">{{ filter.form.in_stock.label_tag }} {{ filter.form.in_stock }}</div>
        <div class="col-md-3">
            <button type="submit" class="btn btn-primary mt-4">Filter</button>
            <a href="{% url 'inventory_plugin:item_list' %}" class="btn btn-secondary mt-4">Reset</a>
        </div>
    </div>
</form>

<!-- Item List -->
{% if items %}
    <div class="table-responsive">
        <table class="table table-striped table-hover">
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Category</th>
                    <th>Quantity</th>
                    <th>Price</th>
                    <th>Status</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {% for item in items %}
                <tr>
                    <td>{{ item.name }}</td>
                    <td>{{ item.category }}</td>
                    <td>{{ item.quantity }}</td>
                    <td>${{ item.price }}</td>
                    <td>
                        {% if item.quantity < 5 %}
                            <span class="badge badge-danger">Critical</span>
                        {% elif item.quantity < 10 %}
                            <span class="badge badge-warning">Low</span>
                        {% else %}
                            <span class="badge badge-success">In Stock</span>
                        {% endif %}
                    </td>
                    <td>
                        <div class="btn-group btn-group-sm">
                            <a href="{% url 'inventory_plugin:item_detail' item.pk %}" class="btn btn-info">
                                <i class="fas fa-eye"></i>
                            </a>
                            <a href="{% url 'inventory_plugin:item_update' item.pk %}" class="btn btn-primary">
                                <i class="fas fa-edit"></i>
                            </a>
                            <a href="{% url 'inventory_plugin:item_delete' item.pk %}" class="btn btn-danger">
                                <i class="fas fa-trash"></i>
                            </a>
                        </div>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>

    <!-- Pagination -->
    {% if is_paginated %}
    <nav aria-label="Page navigation">
        <ul class="pagination">
            {% if page_obj.has_previous %}
                <li class="page-item">
                    <a class="page-link" href="?{% if request.GET.name %}name={{ request.GET.name }}&{% endif %}page={{ page_obj.previous_page_number }}">Previous</a>
                </li>
            {% endif %}

            {% for num in page_obj.paginator.page_range %}
                {% if num == page_obj.number %}
                    <li class="page-item active">
                        <span class="page-link">{{ num }}</span>
                    </li>
                {% else %}
                    <li class="page-item">
                        <a class="page-link" href="?{% if request.GET.name %}name={{ request.GET.name }}&{% endif %}page={{ num }}">{{ num }}</a>
                    </li>
                {% endif %}
            {% endfor %}

            {% if page_obj.has_next %}
                <li class="page-item">
                    <a class="page-link" href="?{% if request.GET.name %}name={{ request.GET.name }}&{% endif %}page={{ page_obj.next_page_number }}">Next</a>
                </li>
            {% endif %}
        </ul>
    </nav>
    {% endif %}
{% else %}
    <div class="alert alert-info">No inventory items found.</div>
{% endif %}
{% endblock %}
```

### Detail Template

```html
<!-- plugins/inventory_plugin/templates/inventory_plugin/item_detail.html -->
{% extends "inventory_plugin/base.html" %}
{% load static %}

{% block page_title %}{{ item.name }}{% endblock %}

{% block plugin_content %}
<div class="row">
    <div class="col-md-8">
        <div class="card mb-4">
            <div class="card-header">
                <h4 class="card-title">Item Details</h4>
            </div>
            <div class="card-body">
                <div class="row">
                    <div class="col-md-6">
                        <p><strong>Name:</strong> {{ item.name }}</p>
                        <p><strong>Category:</strong> {{ item.category }}</p>
                        <p><strong>SKU:</strong> {{ item.sku }}</p>
                        <p><strong>Price:</strong> ${{ item.price }}</p>
                    </div>
                    <div class="col-md-6">
                        <p><strong>Quantity:</strong> {{ item.quantity }}</p>
                        <p><strong>Status:</strong>
                            {% if item.quantity < 5 %}
                                <span class="badge badge-danger">Critical</span>
                            {% elif item.quantity < 10 %}
                                <span class="badge badge-warning">Low</span>
                            {% else %}
                                <span class="badge badge-success">In Stock</span>
                            {% endif %}
                        </p>
                        <p><strong>Last Updated:</strong> {{ item.updated_at }}</p>
                    </div>
                </div>

                <h5 class="mt-4">Description</h5>
                <p>{{ item.description|default:"No description available." }}</p>
            </div>
        </div>
    </div>

    <div class="col-md-4">
        <div class="card mb-4">
            <div class="card-header">
                <h4 class="card-title">Actions</h4>
            </div>
            <div class="card-body">
                <div class="d-grid gap-2">
                    <a href="{% url 'inventory_plugin:item_update' item.pk %}" class="btn btn-primary">
                        <i class="fas fa-edit"></i> Edit Item
                    </a>
                    <a href="{% url 'inventory_plugin:stock_adjustment' item.pk %}" class="btn btn-info">
                        <i class="fas fa-boxes"></i> Adjust Stock
                    </a>
                    <a href="{% url 'inventory_plugin:item_delete' item.pk %}" class="btn btn-danger">
                        <i class="fas fa-trash"></i> Delete Item
                    </a>
                    <a href="{% url 'inventory_plugin:item_list' %}" class="btn btn-secondary">
                        <i class="fas fa-arrow-left"></i> Back to List
                    </a>
                </div>
            </div>
        </div>

        <div class="card">
            <div class="card-header">
                <h4 class="card-title">Stock History</h4>
            </div>
            <div class="card-body">
                {% if item.stock_adjustments.exists %}
                    <ul class="list-group">
                        {% for adjustment in item.stock_adjustments.all|slice:":5" %}
                            <li class="list-group-item">
                                <span class="badge {% if adjustment.quantity > 0 %}badge-success{% else %}badge-danger{% endif %}">
                                    {{ adjustment.quantity|abs }}
                                </span>
                                {{ adjustment.created_at|date:"M d, Y" }}
                                <small class="text-muted">by {{ adjustment.user.username }}</small>
                            </li>
                        {% endfor %}
                    </ul>
                    {% if item.stock_adjustments.count > 5 %}
                        <a href="{% url 'inventory_plugin:stock_history' item.pk %}" class="btn btn-sm btn-link mt-2">
                            View Full History
                        </a>
                    {% endif %}
                {% else %}
                    <p class="text-muted">No stock adjustments recorded.</p>
                {% endif %}
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### Form Template

```html
<!-- plugins/inventory_plugin/templates/inventory_plugin/item_form.html -->
{% extends "inventory_plugin/base.html" %}
{% load static %}
{% load crispy_forms_tags %}

{% block page_title %}{{ title }}{% endblock %}

{% block plugin_content %}
<div class="row">
    <div class="col-md-8 mx-auto">
        <form method="post" enctype="multipart/form-data">
            {% csrf_token %}

            <div class="card mb-4">
                <div class="card-header">
                    <h4 class="card-title">Basic Information</h4>
                </div>
                <div class="card-body">
                    <div class="row">
                        <div class="col-md-6">
                            {{ form.name|as_crispy_field }}
                        </div>
                        <div class="col-md-6">
                            {{ form.category|as_crispy_field }}
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-md-6">
                            {{ form.sku|as_crispy_field }}
                        </div>
                        <div class="col-md-6">
                            {{ form.price|as_crispy_field }}
                        </div>
                    </div>
                    {{ form.description|as_crispy_field }}
                </div>
            </div>

            <div class="card mb-4">
                <div class="card-header">
                    <h4 class="card-title">Inventory Details</h4>
                </div>
                <div class="card-body">
                    <div class="row">
                        <div class="col-md-6">
                            {{ form.quantity|as_crispy_field }}
                        </div>
                        <div class="col-md-6">
                            {{ form.reorder_level|as_crispy_field }}
                        </div>
                    </div>
                    {{ form.location|as_crispy_field }}
                </div>
            </div>

            <div class="card mb-4">
                <div class="card-header">
                    <h4 class="card-title">Additional Information</h4>
                </div>
                <div class="card-body">
                    {{ form.image|as_crispy_field }}
                    {{ form.notes|as_crispy_field }}
                </div>
            </div>

            <div class="form-group">
                <button type="submit" class="btn btn-primary">
                    <i class="fas fa-save"></i> Save
                </button>
                <a href="{% url 'inventory_plugin:item_list' %}" class="btn btn-secondary">
                    <i class="fas fa-times"></i> Cancel
                </a>
            </div>
        </form>
    </div>
</div>
{% endblock %}
```

## URL Configuration

URLs connect your views to specific URL patterns, making them accessible via web browsers.

### Basic URL Configuration

```python
# plugins/inventory_plugin/urls.py
from django.urls import path
from . import views

app_name = 'inventory_plugin'

urlpatterns = [
    # Function-based views
    path('dashboard/', views.dashboard_view, name='dashboard'),
    path('export/', views.export_inventory, name='export_inventory'),

    # Class-based views
    path('items/', views.InventoryListView.as_view(), name='item_list'),
    path('items/<int:pk>/', views.InventoryDetailView.as_view(), name='item_detail'),
    path('items/create/', views.InventoryCreateView.as_view(), name='item_create'),
    path('items/<int:pk>/update/', views.InventoryUpdateView.as_view(), name='item_update'),
    path('items/<int:pk>/delete/', views.InventoryDeleteView.as_view(), name='item_delete'),

    # Stock management
    path('items/<int:pk>/adjust/', views.StockAdjustmentView.as_view(), name='stock_adjustment'),
    path('items/<int:pk>/history/', views.StockHistoryView.as_view(), name='stock_history'),

    # Reports
    path('reports/', views.report_list_view, name='reports'),
    path('reports/<str:report_type>/', views.generate_report_view, name='generate_report'),
]
```

## Integrating with Ritiko UI

Integrating your plugin's templates with the Ritiko UI ensures a consistent look and feel.

### Using Ritiko Base Templates

```html
<!-- plugins/your_plugin/templates/your_plugin/page.html -->
{% extends "base.html" %}
{% load static %}

{% block title %}Your Plugin Page{% endblock %}

{% block content %}
<div class="container-fluid">
    <div class="row">
        <div class="col-12">
            <div class="card">
                <div class="card-header">
                    <h3 class="card-title">Your Plugin Title</h3>
                </div>
                <div class="card-body">
                    <!-- Your plugin content here -->
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### Using Bootstrap Components

Ritiko uses Bootstrap for its UI components. Your plugin should follow the same patterns:

```html
<!-- Card components -->
<div class="card">
    <div class="card-header">Card Title</div>
    <div class="card-body">
        Card content
    </div>
    <div class="card-footer">Card footer</div>
</div>

<!-- Alerts -->
<div class="alert alert-success">Success message</div>
<div class="alert alert-danger">Error message</div>

<!-- Buttons -->
<button class="btn btn-primary">Primary Button</button>
<a href="#" class="btn btn-secondary">Secondary Button</a>

<!-- Tables -->
<table class="table table-striped">
    <thead>
        <tr>
            <th>Header 1</th>
            <th>Header 2</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Data 1</td>
            <td>Data 2</td>
        </tr>
    </tbody>
</table>
```

### Including FontAwesome Icons

Ritiko includes FontAwesome icons that you can use in your templates:

```html
<i class="fas fa-home"></i> Home
<i class="fas fa-user"></i> User
<i class="fas fa-cog"></i> Settings
<i class="fas fa-chart-bar"></i> Analytics
```

## Context Processors and Template Tags

### Creating Custom Context Processors

Context processors add variables to the template context for all templates:

```python
# plugins/your_plugin/context_processors.py
from .models import Notification

def notifications_processor(request):
    """Add unread notifications count to all templates."""
    if request.user.is_authenticated:
        unread_count = Notification.objects.filter(
            user=request.user,
            read=False
        ).count()
        return {'unread_notifications_count': unread_count}
    return {'unread_notifications_count': 0}
```

Register the context processor in your settings:

```python
# In your project's settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'OPTIONS': {
            'context_processors': [
                # ...existing context processors...
                'plugins.your_plugin.context_processors.notifications_processor',
            ],
        },
    },
]
```

### Creating Custom Template Tags

Template tags provide custom functionality in templates:

```python
# plugins/your_plugin/templatetags/your_plugin_tags.py
from django import template
from ..models import InventoryItem

register = template.Library()

@register.simple_tag
def get_low_stock_items():
    """Return items with low stock."""
    return InventoryItem.objects.filter(quantity__lt=10)

@register.filter
def currency(value):
    """Format a number as currency."""
    return f"${value:.2f}"
```

Using custom template tags in templates:

```html
{% load your_plugin_tags %}

<!-- Using a simple tag -->
{% get_low_stock_items as low_stock %}
<p>There are {{ low_stock|length }} items with low stock.</p>

<!-- Using a filter -->
<p>Price: {{ item.price|currency }}</p>
```

## Best Practices for Views and Templates

### 1. Keep Views Focused

- Each view should handle a single responsibility
- Break complex views into smaller, more manageable pieces
- Use mixins for shared functionality in class-based views

### 2. Use Proper Authorization

- Always check permissions before allowing access to views
- Use the permission_required decorator for function-based views
- Use PermissionRequiredMixin for class-based views

### 3. Optimize Database Queries

- Use select_related() and prefetch_related() to reduce database queries
- Avoid N+1 query problems by fetching related objects efficiently
- Use Django's QuerySet methods to filter at the database level

### 4. Follow Template Best Practices

- Use template inheritance to avoid code duplication
- Create reusable template snippets in a partials directory
- Use template tags and filters to keep logic out of templates
- Keep templates clean and focused on presentation

### 5. Provide User Feedback

- Use Django's messages framework to provide feedback after actions
- Display appropriate error messages when things go wrong
- Use confirmation dialogs for destructive actions

### 6. Secure Form Handling

- Always validate form data on the server side
- Use Django's CSRF protection for all forms
- Sanitize user input to prevent XSS attacks

## Real-World Example: Dashboard Plugin

Let's put everything together in a real-world example of a dashboard plugin:

### Models

```python
# plugins/dashboard_plugin/models.py
from django.db import models
from django.conf import settings

class DashboardWidget(models.Model):
    """Configurable dashboard widget."""

    WIDGET_TYPES = [
        ('chart', 'Chart'),
        ('counter', 'Counter'),
        ('list', 'List'),
        ('custom', 'Custom'),
    ]

    name = models.CharField(max_length=100)
    widget_type = models.CharField(max_length=20, choices=WIDGET_TYPES)
