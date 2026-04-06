# Glossary

ExamplePress redefines several WordPress terms and introduces its own. This glossary maps the ExamplePress vocabulary to what it actually means.

## Core concepts

### Theme

In WordPress, a theme controls appearance. In ExamplePress, the **theme** is the immutable infrastructure layer — a router, guard system, feature registry, and Blockstudio integration. It never contains site-specific templates or styles. Think of it as the OS kernel.

**Package**: `packages/wp/themes/examplepress-theme`

### App

A WordPress plugin that declares itself as an ExamplePress companion via the `Theme: examplepress-theme` header and an `examplepress.json` manifest. Apps own routes and provide the template blocks that actually render pages. What WordPress calls a "plugin," ExamplePress calls an "app."

### Theme App / Companion Plugin

The primary app for a site. Scaffolded from a template repo, it typically owns all major routes (front, single, archive, 404) at priority 10. Most sites have exactly one theme app plus optional additional apps at higher priority numbers.

**Template**: `packages/wp/plugins/examplepress-theme-app`

### Route origin

A namespace (app slug) that registers a set of route slug → condition pairs at a given priority. The router evaluates origins in priority order and dispatches to the first match. See [routing.md](routing.md).

### Route slug

A short identifier for a route within an origin: `front`, `single`, `archive`, `search`, `404`, etc. Combined with the namespace and prefix to form a template block name (e.g. `my-site-core/template-front`).

### Template block

A Blockstudio block that serves as a full-page template. Named `{namespace}/template-{slug}` by convention. Lives in `app/templates/{slug}/` within a companion plugin. Not to be confused with WordPress's FSE templates, which ExamplePress guards against.

### Feature

A registered capability with an ID, enabled state, and optional key/value options. Features are declared in `inc/features/`, configured in `examplepress.json`, and overridable via PHP filters. Examples: `title-tag`, `guard-template-redirect`, `blockstudio-tailwind`.

### Guard

A feature that prevents WordPress's site editor from interfering with the router model. Three guards exist: template redirect, template REST, and template resolution. All disabled by `EP_DEV_MODE`.

## Distribution

### Troy Server

A WordPress plugin (`packages/vendor/troy/`) that turns a WordPress site into a software distribution and licensing server. Used exclusively for distributing apps shared across multiple sites — not for the MU plugin or theme, which update directly from GitHub. Manages plugin entries, GitHub/WordPress.org integrations, release processing, and update delivery.

### Troy Client

The companion to Troy Server, installed on end-user sites. Intercepts WordPress update checks and routes them through a Troy Server instead of wordpress.org.

### Troy Bridge

A plugin (`packages/wp/plugins/examplepress-troy-bridge`) installed on the Troy Server. Adds a provisioning REST endpoint so ExamplePress can create Troy plugin entries in one API call during app scaffolding.

### GitHub Updaters

The collective term for the two mechanisms that keep the ExamplePress platform healthy on every site: the **MU Self-Updater** (built into the MU plugin kernel) and the **Theme Update** plugin. Both pull releases directly from GitHub — neither uses Troy.

### MU Self-Updater

The built-in update mechanism inside the ExamplePress MU plugin kernel (`packages/wp/mu-plugins/examplepress-mu`). A WP-Cron job fires twice daily, fetches the `updates.json` manifest from the latest GitHub release, compares versions, and — if newer — downloads the ZIP, validates its SHA-256 checksum, and overwrites the kernel and loader in place. Requires no admin intervention.

### Theme Update

A companion plugin (`packages/wp/plugins/examplepress-theme-update`) that manages ExamplePress theme versioning through GitHub Releases. Pulls updates directly from GitHub — not through Troy. Supports stable/development channels and version pinning.

**Package**: `packages/wp/plugins/examplepress-theme-update`

### Channel

The update track a site follows: `stable` (GitHub latest release) or `development` (GitHub prerelease tagged "development"). Configurable via filter, constant, option, or auto-detected from the installed version string.

### Version pin

Locks updates to a specific version ceiling. If pinned to 1.2.3, the site won't be offered anything newer (or older).

## Blockstudio

### Blockstudio

A vendor dependency (`packages/vendor/blockstudio/`) that provides zero-build block rendering. Blocks are `block.json` + `index.php` pairs — no webpack, no `@wordpress/scripts`, no JS build pipeline. Blockstudio handles discovery, SCSS compilation, asset scoping, and registration.

### `bs_block()`

The Blockstudio function that renders a block by name and returns the HTML string. The router uses this to dispatch to the resolved template block.

### `block.json`

Standard WordPress block metadata, extended with `"blockstudio": true` to opt into Blockstudio rendering. Template blocks also typically disable the inserter and lock move/remove/rename.

### Inner-block wrapping

Blockstudio wraps inner blocks in a container `<div>` by default. ExamplePress exempts the router block and all template blocks from this wrapping.

## Configuration

### `examplepress.json` (theme)

The master configuration file in the theme root. Contains design tokens, feature flags, Blockstudio settings, dependency declarations, and updater configuration. Normalized into feature options on load.

### `examplepress.json` (app)

The app manifest in a companion plugin's root. Contains the app's name, slug, version, routing configuration, and Troy connection details. Schema: `https://www.examplepress.com/schema/app`.

### `blockstudio.json`

Runtime configuration for Blockstudio in the theme root. Controls asset processing (enqueue, minify, SCSS) and dev tools (grab, canvas).

## Infrastructure

### `EP_DEV_MODE`

A PHP constant (`wp-config.php`) that disables all three template guards. Intended for development environments where you need direct access to the site editor's template management.

### `EP_THEME_VERSION`, `EP_THEME_PATH`, `EP_THEME_URI`

Theme constants defined in `functions.php`. Version comes from the `style.css` header; path and URI from `get_template_directory()` and `get_template_directory_uri()`.

### `ep_app` (CPT)

A shadow Custom Post Type (no UI) used as the persistent app registry. Stores GitHub repo URLs, Troy registration state, and scaffolding metadata. Survives plugin deactivation and deletion.

### Theme Demo

A reference companion plugin (`packages/wp/plugins/examplepress-theme-demo`) that demonstrates route registration, data enrichment, and template block rendering. Useful as a working example alongside the template repo.

**Package**: `packages/wp/plugins/examplepress-theme-demo`
