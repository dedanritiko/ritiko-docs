# Ritiko Plugins Directory

This directory is reserved for external plugin repositories. Plugins have been moved to separate repositories for better modularity and maintenance.

## Available Plugin Repositories

- **Blog Plugin**: https://github.com/dedanritiko/ritiko-blog-plugin
- **Dashboard Widgets Plugin**: https://github.com/dedanritiko/ritiko-dashboard-widgets-plugin
- **Patient Email Plugin**: https://github.com/dedanritiko/ritiko-patient-email-plugin
- **Patient Plugin**: https://github.com/dedanritiko/ritiko-patient-plugin
- **Shop Plugin**: https://github.com/dedanritiko/ritiko-shop-plugin

## Adding Plugins

To add a plugin to your development environment:

1. **Clone the plugin repository** into this directory:
   ```bash
   git clone https://github.com/dedanritiko/ritiko-blog-plugin.git plugins/blog_plugin
   ```

2. **Install plugin dependencies** (if any):
   ```bash
   cd plugins/blog_plugin
   # Follow plugin-specific installation instructions
   ```

3. **The plugin will be auto-discovered** by the Ritiko plugin system on next server start.

## Removing Plugins

To remove a plugin from your development environment:

1. **Stop the development server**

2. **Remove the plugin directory**:
   ```bash
   rm -rf plugins/blog_plugin
   ```

3. **Clean up database** (if needed):
   ```bash
   # Run migrations to remove plugin-specific tables/data
   pipenv run python manage.py migrate
   ```

4. **Restart the development server**

## Plugin Development

When developing plugins:

1. **Never commit plugin code** to the main repository
2. **Use the individual plugin repositories** for version control
3. **Follow the plugin development guidelines** in the main project documentation

## Notes

- The `.gitignore` file in this directory ensures no plugin code is accidentally committed
- Plugins are loaded automatically when present in this directory
- Each plugin should have its own repository for independent versioning and maintenance
