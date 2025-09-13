# Plugin Management Command Reference

The `plugin` management command provides a unified interface for managing all aspects of the Ritiko plugin system, including plugin information, enable/disable functionality, and permission management.

## Overview

This command consolidates the functionality of the previous separate commands (`plugin_info`, `plugin_manage`, `plugin_permissions`) into a single, easy-to-use interface with subcommands.

## Basic Syntax

```bash
python manage.py plugin <subcommand> [options]
```

## Available Subcommands

### 1. Plugin Information (`info`)

Display information about registered plugins.

#### List All Plugins
```bash
python manage.py plugin info
```

#### Show Only Enabled Plugins
```bash
python manage.py plugin info --enabled-only
```

#### Show Specific Plugin Details
```bash
python manage.py plugin info --plugin blog_plugin
```

**Example Output:**
```
Plugin: Blog System
ID: blog_plugin
Version: 1.0.0
Description: Blog functionality for Ritiko platform
Author: Ritiko Team
Email: dev@ritiko.com
URL: https://github.com/dedanritiko/ritiko-blog-plugin
Enabled: Yes
Django Apps: blog_plugin
Dependencies: None
```

### 2. Plugin Management

#### Enable a Plugin
```bash
python manage.py plugin enable --plugin blog_plugin --user admin@example.com
```

#### Disable a Plugin
```bash
python manage.py plugin disable --plugin blog_plugin --user admin@example.com
```

#### Show Plugin Status
```bash
python manage.py plugin status
```

**Example Output:**
```
🔌 System-wide Plugin Status:
==================================================
blog_plugin: ✅ Enabled (Blog System)
shop_plugin: ❌ Disabled (E-commerce Shop)
patient_plugin: ✅ Enabled (Patient Management)
```

**Requirements:**
- The `--user` parameter must be a superadmin user
- Server restart may be required for newly enabled plugins

### 3. Permission Management (`permissions`)

Manage plugin-specific permissions.

#### Show All Plugin Permissions
```bash
python manage.py plugin permissions
```

#### Show Permissions for Specific Plugin
```bash
python manage.py plugin permissions --plugin blog_plugin
```

#### Sync Permissions to Database
```bash
python manage.py plugin permissions --sync
```

#### Force Recreation of Permissions
```bash
python manage.py plugin permissions --sync --force
```

#### Create Default Permission Groups
```bash
python manage.py plugin permissions --create-groups
```

#### Assign Permission to User
```bash
python manage.py plugin permissions --assign user@example.com can_create_blog_posts
```

#### Remove Permission from User
```bash
python manage.py plugin permissions --remove user@example.com can_create_blog_posts
```

#### Show User's Plugin Permissions
```bash
python manage.py plugin permissions --user-permissions user@example.com
```

## Common Workflows

### Initial Plugin Setup
1. **Install plugin**: Clone repository to `plugins/` directory
2. **Check plugin info**: `python manage.py plugin info --plugin plugin_name`
3. **Sync permissions**: `python manage.py plugin permissions --sync`
4. **Enable plugin**: `python manage.py plugin enable --plugin plugin_name --user admin`
5. **Restart server**: Required for plugin activation

### Plugin Development
1. **Check plugin status**: `python manage.py plugin status`
2. **View plugin permissions**: `python manage.py plugin permissions --plugin plugin_name`
3. **Test permission assignment**: `python manage.py plugin permissions --assign test@example.com permission_name`

### Troubleshooting
1. **Check dependency issues**: `python manage.py plugin info` (shows missing dependencies)
2. **Verify permissions**: `python manage.py plugin permissions --plugin plugin_name`
3. **Check user permissions**: `python manage.py plugin permissions --user-permissions user@example.com`

## Error Handling

### Common Errors and Solutions

**Plugin not found**
```bash
# Error: Plugin 'unknown_plugin' not found
# Solution: Check available plugins
python manage.py plugin info
```

**User must be superadmin**
```bash
# Error: User 'user@example.com' must be a superadmin
# Solution: Use a superadmin account or grant superadmin rights
```

**Permission not found**
```bash
# Error: Failed to assign permission "invalid_permission"
# Solution: Check available permissions for the plugin
python manage.py plugin permissions --plugin plugin_name
```

## Plugin Repository Structure

Each plugin should be cloned into the `plugins/` directory:

```
plugins/
├── blog_plugin/           # Blog functionality
├── shop_plugin/           # E-commerce features
├── patient_plugin/        # Patient management
├── patient_email_plugin/  # Patient email features
└── dashboard_widgets_plugin/  # Dashboard widgets
```

## Available Plugin Repositories

- **Blog Plugin**: https://github.com/dedanritiko/ritiko-blog-plugin
- **Dashboard Widgets Plugin**: https://github.com/dedanritiko/ritiko-dashboard-widgets-plugin
- **Patient Email Plugin**: https://github.com/dedanritiko/ritiko-patient-email-plugin
- **Patient Plugin**: https://github.com/dedanritiko/ritiko-patient-plugin
- **Shop Plugin**: https://github.com/dedanritiko/ritiko-shop-plugin

## Migration from Old Commands

This unified `plugin` command replaces the following deprecated commands:

### Old Command → New Command

**Plugin Info (DEPRECATED: `plugin_info`)**
```bash
# OLD (no longer available)
python manage.py plugin_info
python manage.py plugin_info --plugin blog_plugin
python manage.py plugin_info --enabled-only

# NEW
python manage.py plugin info
python manage.py plugin info --plugin blog_plugin
python manage.py plugin info --enabled-only
```

**Plugin Management (DEPRECATED: `plugin_manage`)**
```bash
# OLD (no longer available)
python manage.py plugin_manage enable --plugin blog_plugin --user admin
python manage.py plugin_manage disable --plugin blog_plugin --user admin
python manage.py plugin_manage status

# NEW
python manage.py plugin enable --plugin blog_plugin --user admin
python manage.py plugin disable --plugin blog_plugin --user admin
python manage.py plugin status
```

**Plugin Permissions (DEPRECATED: `plugin_permissions`)**
```bash
# OLD (no longer available)
python manage.py plugin_permissions
python manage.py plugin_permissions --sync
python manage.py plugin_permissions --assign user@example.com permission

# NEW
python manage.py plugin permissions
python manage.py plugin permissions --sync
python manage.py plugin permissions --assign user@example.com permission
```

## Notes

- **Breaking Change**: The old separate commands (`plugin_info`, `plugin_manage`, `plugin_permissions`) have been removed
- **Migration Required**: Update any scripts or documentation to use the new unified command syntax
- **Permission Management**: All permission operations require appropriate Django permissions
- **Server Restart**: Enabling new plugins typically requires a server restart
- **Database Migrations**: Some plugins may require running migrations after enabling

## Examples by Use Case

### System Administrator
```bash
# Daily plugin management
python manage.py plugin status
python manage.py plugin info --enabled-only

# Enable new plugin
python manage.py plugin enable --plugin new_plugin --user admin@company.com
python manage.py plugin permissions --sync
```

### Developer
```bash
# Check plugin during development
python manage.py plugin info --plugin my_plugin

# Test permissions
python manage.py plugin permissions --plugin my_plugin
python manage.py plugin permissions --assign dev@company.com test_permission
```

### User Management
```bash
# Assign blog permissions to content manager
python manage.py plugin permissions --assign content@company.com can_create_blog_posts
python manage.py plugin permissions --assign content@company.com can_edit_blog_posts

# Check what permissions a user has
python manage.py plugin permissions --user-permissions content@company.com
```