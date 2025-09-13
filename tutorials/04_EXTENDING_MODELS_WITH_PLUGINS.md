# Extending Models with Plugins

One of the most powerful features of the Ritiko plugin system is the ability to extend existing models without modifying their original code. This tutorial will show you how to add fields, methods, and functionality to core system models using plugins.

## Introduction to Model Extension

There are two primary methods for extending models in the Ritiko plugin system:

1. **Related Models**: Creating separate models with one-to-one relationships to existing models
2. **Method Injection**: Adding methods to existing models through monkey patching

Each approach has its own advantages and use cases. Let's explore both methods in detail.

## Method 1: Related Models (Recommended)

The related models approach is the recommended way to extend existing models. It involves creating a new model with a one-to-one relationship to the model you want to extend.

### Advantages of Related Models

- **Clean Database Separation**: Your extension fields are in separate database tables
- **Migration Control**: You control the migrations for your extension fields
- **Type Safety**: Better IDE support and type checking
- **Maintainability**: Cleaner code organization and separation of concerns

### Example: Extending the Patient Model

Let's create a plugin that extends the Patient model with email notification preferences:

```python
# plugins/patient_email_plugin/models.py
from django.db import models
from patients.models.people import Patient

class PatientEmailPreferences(models.Model):
    """Email preferences extension for Patient model."""

    patient = models.OneToOneField(
        Patient,
        on_delete=models.CASCADE,
        related_name="email_preferences"
    )

    # Extension fields
    email = models.EmailField(blank=True, null=True)
    secondary_email = models.EmailField(blank=True, null=True)
    email_verified = models.BooleanField(default=False)
    notifications_enabled = models.BooleanField(default=True)

    # Notification preferences
    receive_appointment_reminders = models.BooleanField(default=True)
    receive_test_results = models.BooleanField(default=True)
    receive_billing_notices = models.BooleanField(default=True)

    # Extension methods
    def get_primary_email(self):
        """Get the patient's primary email address."""
        return self.email

    def get_notification_email(self):
        """Get the email to use for notifications."""
        return self.email or self.secondary_email

    def has_valid_email(self):
        """Check if the patient has a valid email."""
        return bool(self.email and self.email_verified)

    def can_receive_notification(self, notification_type):
        """Check if patient can receive a specific notification type."""
        if not self.notifications_enabled:
            return False

        if notification_type == "appointment":
            return self.receive_appointment_reminders
        elif notification_type == "test_results":
            return self.receive_test_results
        elif notification_type == "billing":
            return self.receive_billing_notices

        return False

    def __str__(self):
        return f"Email preferences for {self.patient}"
```

### Using the Related Model Extension

You can access the extension from any Patient instance:

```python
# Get a patient
patient = Patient.objects.get(pk=1)

# Access the extension (this will raise DoesNotExist if no extension exists)
try:
    email_prefs = patient.email_preferences

    # Use extension methods
    if email_prefs.has_valid_email() and email_prefs.can_receive_notification("appointment"):
        # Send appointment reminder
        send_email(email_prefs.get_notification_email(), "Appointment Reminder", "...")

except PatientEmailPreferences.DoesNotExist:
    # Extension doesn't exist yet for this patient
    email_prefs = PatientEmailPreferences.objects.create(
        patient=patient,
        email=patient.get_contact_email()  # Assuming this method exists on Patient
    )
```

### Helper Method to Get or Create Extension

To simplify accessing the extension, you can add a helper method:

```python
# plugins/patient_email_plugin/model_extensions.py
from patients.models.people import Patient
from .models import PatientEmailPreferences

def get_or_create_email_preferences(self):
    """Get or create email preferences for this patient."""
    try:
        return self.email_preferences
    except PatientEmailPreferences.DoesNotExist:
        return PatientEmailPreferences.objects.create(
            patient=self,
            email=getattr(self, 'email', None)
        )

# Add the method to the Patient model
Patient.add_to_class("get_or_create_email_preferences", get_or_create_email_preferences)
```

Now you can use this method in your code:

```python
patient = Patient.objects.get(pk=1)
email_prefs = patient.get_or_create_email_preferences()

# Now you can safely use the extension
if email_prefs.can_receive_notification("appointment"):
    # Send appointment reminder
    send_email(email_prefs.get_notification_email(), "Appointment Reminder", "...")
```

## Method 2: Method Injection (Monkey Patching)

Method injection allows you to add methods directly to existing models without creating new database tables. This is useful for adding behavior without needing to store additional data.

### Advantages of Method Injection

- **Simplicity**: No additional models or database tables required
- **Direct Integration**: Methods appear directly on the model
- **Performance**: No additional database queries needed
- **No Migrations**: No need to run migrations

### Example: Adding Email Methods to Patient Model

```python
# plugins/patient_email_plugin/model_extensions.py
from patients.models.people import Patient
from django.core.mail import send_mail

def send_email(self, subject, message, html_message=None, from_email=None):
    """Send an email to this patient."""
    if not self.email:
        return False

    try:
        send_mail(
            subject=subject,
            message=message,
            from_email=from_email or 'noreply@example.com',
            recipient_list=[self.email],
            html_message=html_message,
            fail_silently=False
        )
        return True
    except Exception as e:
        print(f"Failed to send email: {e}")
        return False

def has_email(self):
    """Check if the patient has an email address."""
    return bool(self.email)

def send_appointment_reminder(self, appointment_date, appointment_time,
                             location=None, provider=None):
    """Send an appointment reminder email to this patient."""
    if not self.has_email():
        return False

    subject = "Appointment Reminder"

    message = f"""
    Dear {self.get_full_name()},

    This is a reminder for your upcoming appointment:

    Date: {appointment_date}
    Time: {appointment_time}
    """

    if location:
        message += f"\nLocation: {location}"

    if provider:
        message += f"\nProvider: {provider}"

    message += """

    Please contact us if you need to reschedule.

    Thank you,
    Your Healthcare Team
    """

    return self.send_email(subject, message)

# Add methods to Patient model
Patient.add_to_class("send_email", send_email)
Patient.add_to_class("has_email", has_email)
Patient.add_to_class("send_appointment_reminder", send_appointment_reminder)
```

### Using the Injected Methods

Once you've added these methods to the Patient model, you can use them directly:

```python
patient = Patient.objects.get(pk=1)

if patient.has_email():
    # Send a simple email
    patient.send_email(
        subject="Test Results Available",
        message="Your test results are now available. Please log in to view them."
    )

    # Send an appointment reminder
    patient.send_appointment_reminder(
        appointment_date="Monday, June 15, 2023",
        appointment_time="10:00 AM",
        location="Main Clinic, Room 203",
        provider="Dr. Smith"
    )
```

## Combining Both Approaches

For comprehensive extensions, you can combine both approaches:

1. Use **Related Models** for storing additional data
2. Use **Method Injection** for adding convenience methods to the main model

### Example: Combined Approach

```python
# plugins/patient_email_plugin/model_extensions.py
from patients.models.people import Patient
from .models import PatientEmailPreferences

# Helper method to get or create the extension
def get_email_preferences(self):
    """Get or create email preferences for this patient."""
    try:
        return self.email_preferences
    except PatientEmailPreferences.DoesNotExist:
        return PatientEmailPreferences.objects.create(
            patient=self,
            email=getattr(self, 'email', None)
        )

# Convenience methods that use the extension
def send_email(self, subject, message, **kwargs):
    """Send an email to this patient using their email preferences."""
    prefs = self.get_email_preferences()

    if not prefs.has_valid_email():
        return False

    # Delegate to the extension method
    from .email_service import send_patient_email
    return send_patient_email(prefs, subject, message, **kwargs)

def can_receive_notification(self, notification_type):
    """Check if the patient can receive a specific notification."""
    prefs = self.get_email_preferences()
    return prefs.can_receive_notification(notification_type)

# Add methods to Patient model
Patient.add_to_class("get_email_preferences", get_email_preferences)
Patient.add_to_class("send_email", send_email)
Patient.add_to_class("can_receive_notification", can_receive_notification)
```

## Setting Up Extensions on Plugin Load

To ensure your extensions are loaded when your plugin is initialized, import your extension module in the `ready()` method of your plugin's AppConfig:

```python
# plugins/patient_email_plugin/apps.py
from core.plugin_app_config import PluginAppConfig

class PatientEmailPluginConfig(PluginAppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'plugins.patient_email_plugin'
    verbose_name = 'Patient Email Plugin'

    def ready(self):
        super().ready()
        # Import model extensions
        from . import model_extensions
        print(f"✅ {self.verbose_name} model extensions loaded")
```

## Best Practices for Model Extensions

### 1. Choose the Right Approach

- Use **Related Models** when you need to store additional data
- Use **Method Injection** when you only need to add behavior
- Consider a **Combined Approach** for complex extensions

### 2. Use Descriptive Naming

- Prefix related model names with the model being extended (e.g., `PatientEmailPreferences`)
- Use clear method names that indicate what they do
- Add docstrings to explain method functionality

### 3. Handle Missing Extensions

- Use try/except blocks or helper methods when accessing related models
- Provide fallback behavior when extensions aren't available
- Consider auto-creating extensions when they don't exist

### 4. Avoid Conflicts

- Use unique related_name attributes for OneToOneFields
- Check for existing methods before monkey patching
- Use plugin-specific prefixes for method names to avoid collisions

### 5. Document Your Extensions

- Document which models you're extending and how
- Explain the purpose and behavior of added methods
- Provide usage examples for other developers

## Common Patterns and Examples

### Extension Registration and Discovery

For plugins with multiple extensions, you might want to create a registration system:

```python
# plugins/extensions_plugin/registry.py
class ExtensionRegistry:
    def __init__(self):
        self._extensions = {}

    def register(self, model_class, extension_class):
        """Register an extension class for a model."""
        if model_class not in self._extensions:
            self._extensions[model_class] = []

        self._extensions[model_class].append(extension_class)
        print(f"Registered extension {extension_class.__name__} for {model_class.__name__}")

    def get_extensions(self, model_class):
        """Get all registered extensions for a model."""
        return self._extensions.get(model_class, [])

# Create a singleton instance
extension_registry = ExtensionRegistry()

# Usage
from patients.models.people import Patient
from .models import PatientExtension

extension_registry.register(Patient, PatientExtension)
```

### Automatic Extension Creation

For plugins that need to ensure extensions exist for all relevant instances:

```python
# plugins/extensions_plugin/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from patients.models.people import Patient
from .models import PatientExtension

@receiver(post_save, sender=Patient)
def create_patient_extension(sender, instance, created, **kwargs):
    """Create a PatientExtension when a new Patient is created."""
    if created:
        PatientExtension.objects.create(patient=instance)
```

### Extension with Proxy Models

For more complex customizations, you can use proxy models:

```python
# plugins/advanced_plugin/models.py
from patients.models.people import Patient

class EnhancedPatient(Patient):
    """Proxy model for Patient with enhanced functionality."""

    class Meta:
        proxy = True

    def get_enhanced_data(self):
        """Get enhanced data for this patient."""
        # Implementation here
        pass

    @classmethod
    def get_high_risk_patients(cls):
        """Get all high-risk patients."""
        return cls.objects.filter(risk_level__gte=3)
```

## Next Steps

Now that you know how to extend existing models, in the next tutorial, we'll explore creating custom views and templates for your plugins.
