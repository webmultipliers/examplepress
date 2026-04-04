# Apps

In ExamplePress, an **app** is a WordPress plugin that declares itself as a companion to the theme. Apps own routes, provide template blocks, and can connect to GitHub and Troy for distribution. The theme discovers apps at runtime by scanning the plugin directory for `examplepress.json` manifests.

## Anatomy of an app

```
wp-content/plugins/my-site-core/
├── app/
│   └── templates/
│       ├── front/
│       │   ├── block.json
│       │   ├── index.php
│       │   └── style.inline.scss
│       ├── single/
│       │   ├── block.json
│       │   └── index.php
│       └── 404/
│           ├── block.json
│           └── index.php
├── examplepress.json       # App manifest
├── my-site-core.php        # Main plugin file
└── composer.json            # Optional
```

### The plugin file

The main PHP file must include a `Theme` header pointing to the ExamplePress theme:

```php
/**
 * Plugin Name: My Site Core
 * Description: Companion app for my site
 * Version:     1.0.0
 * Theme:       examplepress-theme
 * Troy:        troy.example.com
 */
```

The `Theme: examplepress-theme` header is required — it's how the theme identifies the plugin as an app. The `Troy` header is optional and declares the Troy Server URL for distribution.

The plugin file registers routes and initializes Blockstudio:

```php
// Register route origins (routes → conditions)
examplepress_register_route_origin( 'my-site-core', [
    'front'   => fn() => is_front_page() || is_home(),
    'single'  => fn() => is_singular(),
    'archive' => fn() => is_archive(),
    '404'     => fn() => is_404(),
], 10 ); // priority

// Initialize Blockstudio for this app's blocks
add_action( 'init', function () {
    Blockstudio\Build::init( [
        'dir' => plugin_dir_path( __FILE__ ) . 'app',
    ] );
} );
```

### The app manifest (`examplepress.json`)

```json
{
  "$schema": "https://www.examplepress.com/schema/app",
  "name": "My Site Core",
  "slug": "my-site-core",
  "description": "Main companion app",
  "version": "1.0.0",
  "routing": {
    "priority": 10,
    "routes": {
      "front": {
        "condition": "is_front_page() || is_home()",
        "urls": ["/"],
        "desc": "Homepage"
      },
      "single": {
        "condition": "is_singular()",
        "urls": ["/hello-world/"],
        "desc": "Single post"
      }
    }
  },
  "troy": {
    "server_url": "troy.example.com",
    "repo": "webmultipliers/my-site-core",
    "repo_id": "12345"
  }
}
```

The `routing` block is declarative metadata — it documents what the plugin registers in PHP and is used by the admin UI. The `troy` block stores connection details for the distribution server.

> **Note**: The template repo (`packages/theme-app`) ships a minimal stub with a single placeholder route. The example above shows a filled-in manifest like the demo app's. The `condition` field is optional metadata — routing logic lives in the PHP callables registered by `examplepress_register_route_origin()`.

## Discovery and registry

Apps are tracked in two places that merge at read time:

### Filesystem discovery (`inc/apps.php`)

`examplepress_get_apps()` scans `WP_PLUGIN_DIR` for subdirectories containing `examplepress.json`. For each match, `examplepress_parse_app()`:

1. Reads the JSON manifest
2. Finds the main plugin file (prefers `{slug}.php`, falls back to header scanning)
3. Validates the `Theme: examplepress-theme` header
4. Extracts routing config and Troy connection data
5. Returns an app record with `status: 'connected'` or `'disconnected'`

### Persistent registry (`inc/app-registry.php`)

A shadow Custom Post Type (`ep_app`) stores persistent metadata — GitHub repo URLs, Troy registration state, scaffolding source. This survives plugin deactivation/deletion.

`examplepress_registry_list_merged()` combines both sources:

- **Filesystem** supplies: presence, activation state, local version, live Troy config
- **Registry** supplies: GitHub metadata, creation date, scaffolding source

An app can be:

| Status | Meaning |
|---|---|
| `connected` | Local plugin present, Troy configured |
| `disconnected` | Local plugin present, no Troy connection |
| `orphan` | Registry entry exists but plugin not found on disk |

## Lifecycle

### Scaffolding

`POST /wp-json/examplepress/v1/apps/scaffold` creates a new app. Two modes:

**Template mode** (GitHub token available):
1. Create local plugin files from template
2. Create GitHub repo from `webmultipliers/examplepress-theme-app`
3. Replace `__SLUG__`, `__NAME__`, `__DESC__` placeholders
4. Create initial release (v0.0.0)
5. Register in persistent registry

**Local mode** (no GitHub token):
1. Download template archive
2. Replace placeholders
3. Write to plugin directory

The plugin is **not** activated automatically after scaffolding.

### Activation and deactivation

Standard WordPress plugin activation/deactivation. When active, the app's route origins are registered and its template blocks are available. When deactivated, routes disappear and the router falls through to lower-priority origins or the default `get-started` template.

- Activate: WordPress Plugins screen (no REST endpoint — standard WordPress activation)
- Deactivate: `POST /examplepress/v1/apps/{slug}/deactivate`

### Connection

`POST /examplepress/v1/apps/{slug}/connect` links a disconnected local app to GitHub and Troy without re-scaffolding. This pushes existing code to a new GitHub repo and registers with Troy.

### Troy binding

`POST /examplepress/v1/apps/{slug}/troy-bind` initiates the Troy provisioning flow. The client is redirected to the Troy Server URL for setup.

### Destruction

`DELETE /examplepress/v1/apps/{slug}/destroy` triggers `examplepress_destroy_app()`, which deletes an app everywhere:

1. Deactivate and delete the local plugin directory
2. Delete the GitHub repo via API
3. Unregister from Troy Server
4. Remove the persistent registry entry

Each step is attempted independently — partial failures are reported but don't block the rest.

### Health check

`GET /examplepress/v1/apps/{slug}/health` checks local status, Troy reachability, and GitHub repo state. Used by the admin UI to surface connection issues.

## Slug generation

`examplepress_slugify_app_name()` converts a display name to a slug: lowercase, alphanumeric + hyphens only. Example: `"My Site Core"` → `"my-site-core"`.

## Data enrichment

Apps can enrich the data passed to their template blocks by filtering `examplepress_route_data`. The filter receives three arguments: `$data`, `$target_slug`, and `$full_block_name`:

```php
add_filter( 'examplepress_route_data', function ( $data, $slug, $block_name ) {
    if ( $slug === 'single' ) {
        $data['post'] = get_queried_object();
    }
    return $data;
}, 10, 3 );
```

This data is passed as attributes when Blockstudio renders the template block. You can accept fewer parameters if you don't need the block name (the demo plugin uses `10, 2`).
