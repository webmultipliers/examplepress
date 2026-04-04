# Distribution

ExamplePress uses two systems for distribution: **Troy Server** handles licensing and plugin delivery, and **Theme Update** handles theme versioning. The **Troy Bridge** plugin connects ExamplePress's scaffolding workflow to a Troy Server instance.

## Troy Server

Troy (`vendor/troy/`) is a WordPress plugin that turns a WordPress site into a software licensing and distribution server. It consists of two parts:

### Troy Server (`troy-server`)

Runs on the distribution server. Provides:

- **Plugin CPT** — a custom post type for managed plugins, with metadata stored across four database tables (`wp_troy_plugins`, `wp_troy_plugin_metas`, `wp_troy_plugin_infos`, `wp_troy_plugin_data_caches`)
- **REST API** — endpoints for plugin registration, slug management, ZIP processing, and editor state
- **Integration system** — connects to GitHub or WordPress.org for tag fetching and release processing
- **Package system** — separate CPT for distributable packages
- **Cron jobs** — scheduled tasks for tag syncing and release processing

Key classes:

| Class | Purpose |
|---|---|
| `Troy\Server\Plugins\REST` | Plugin management REST endpoints |
| `Troy\Server\Plugins\Data` | Read-only plugin data access layer |
| `Troy\Server\API\Plugin` | Lookup helpers (`get_plugin_id_by_slug`, etc.) |
| `Troy\Server\API\Slug` | Slug uniqueness checking |
| `Troy\Server\Integrations\Plugins\Repos\GitHub` | GitHub integration (connect, tag fetching) |

### Troy Client (`troy-client`)

Runs on end-user sites. Intercepts WordPress's plugin update checks and fetches from the Troy Server instead of wordpress.org. Supports private repos and sideloading.

Plugins declare their Troy connection via a `Troy` header in the plugin file:

```php
/**
 * Troy: troy.example.com
 */
```

## Troy Bridge

The bridge plugin (`packages/troy-bridge/`) runs on the Troy Server and adds a one-click provisioning endpoint used by ExamplePress's scaffolding flow.

### Provisioning endpoint

```
POST /wp-json/troy-server/v1/plugins/manage/provision
```

Request:
```json
{
  "name": "My Plugin",
  "slug": "my-plugin",
  "description": "Short description",
  "owner_repo": "webmultipliers/my-plugin",
  "github_pat": "ghp_xxx"
}
```

This creates the plugin entry in Troy, provisions all required database rows, connects GitHub integration, and enables tag fetching — all in one call. Without the bridge, these would be separate manual steps in the Troy admin.

### Health endpoint

```
GET /wp-json/troy-server/v1/plugins/manage/health?slug={slug}
```

Returns plugin status, integration health, and an admin link. Used by the ExamplePress admin UI to verify Troy connectivity.

## Theme Update

The theme update plugin (`packages/theme-update/`) manages ExamplePress theme versioning through GitHub Releases.

### Update flow

1. WordPress fires `pre_set_site_transient_update_themes`
2. The plugin fetches an `updates.json` manifest from GitHub Releases
3. If a newer version is available, it injects the update into WordPress's native `update_themes` transient
4. WordPress shows the update on Dashboard > Updates and Appearance > Themes

### Channels

Updates follow a channel system. Resolution order (highest priority first):

1. **PHP filter**: `examplepress_update_channel`
2. **Constant**: `EP_UPDATE_CHANNEL` in `wp-config.php`
3. **Option**: `ep_update_channel` (set via admin UI or REST)
4. **Auto-detect**: if the installed version contains `dev`, `alpha`, `beta`, or `rc` → `development`; otherwise → `stable`
5. **Default**: `stable`

The **stable** channel pulls from GitHub's latest release. The **development** channel finds the prerelease tagged "development".

### Version pinning

Pin a specific version to prevent updates beyond it:

- `update_option( 'ep_pinned_version', '1.2.3' )`
- If pinned = installed → no update offered
- If pinned > installed → update to pinned version
- If pinned < installed → no downgrade

### Update manifest format

The `updates.json` file in GitHub Releases:

```json
{
  "version": "1.2.3",
  "name": "ExamplePress",
  "requires": "6.9",
  "requires_php": "8.0",
  "download_url": "https://github.com/.../archive.zip",
  "changelog": "<p>...</p>",
  "last_updated": "2026-01-01T00:00:00Z"
}
```

### Caching

| Cache key | TTL | Purpose |
|---|---|---|
| `ep_theme_update_manifest` | 6 hours | Manifest JSON |
| `ep_github_releases` | 30 minutes | Releases list |
| Error sentinel | 5 minutes | Prevents hammering after failures |

Cache is flushed on theme switch, after theme update, or manually via `POST /wp-json/ep-theme-update/v1/check`.

### REST API

All endpoints require the `update_themes` capability. Namespace: `ep-theme-update/v1`.

| Method | Route | Purpose |
|---|---|---|
| GET | `/status` | Current version, latest, channel, pin state |
| GET | `/releases` | GitHub releases list |
| POST | `/check` | Flush cache, return fresh status |
| PUT | `/channel` | Set channel (`stable` or `development`) |
| PUT | `/pin` | Set or clear pinned version |
| POST | `/install` | Trigger theme update |
| POST | `/reinstall` | Reinstall current or specified version |

### Self-protection

While ExamplePress is the active theme, the update plugin protects itself:

- Removes "Deactivate" and "Delete" links from the Plugins list
- Blocks programmatic deactivation with `wp_die()`
- Shows a warning if the plugin is active but the theme is not
