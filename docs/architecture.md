# Architecture

ExamplePress is a WordPress theme that behaves like an operating system. The theme itself is immutable infrastructure — it ships a router, a guard system, a feature registry, and block rendering via Blockstudio. All site-specific code lives in companion plugins called **apps**.

## The four-layer model

```
┌─────────────────────────────────────────────┐
│  Layer 4 — Additional Apps (extensions)     │
│  Priority 20+, optional, additive           │
├─────────────────────────────────────────────┤
│  Layer 3 — Theme App (main companion)       │
│  Priority 10, owns primary routes           │
├─────────────────────────────────────────────┤
│  Layer 2 — ExamplePress Theme (immutable)   │
│  Router, guards, features, Blockstudio      │
├─────────────────────────────────────────────┤
│  Layer 1 — ExamplePress MU (platform)       │
│  MU plugin, self-updating kernel loader     │
└─────────────────────────────────────────────┘
```

### Layer 1: The MU Plugin

The ExamplePress MU plugin (`packages/mu`) is a self-updating WordPress MU plugin that loads before the theme. It consists of a thin loader (`examplepress-mu.php`) placed in `wp-content/mu-plugins/` and an application directory (`examplepress-mu/`) containing the platform kernel. On first load, if the kernel is missing, the loader fetches the latest release from GitHub and extracts it automatically.

Structure:

```
mu-plugins/
├── examplepress-mu.php          # Thin loader (auto-loaded by WordPress)
└── examplepress-mu/             # Application directory
    └── bootstrap.php            # Platform kernel entry point
```

### Layer 2: The Theme

The theme never changes per-site. It provides:

- **Router** — a single entry point (`templates/index.html`) that dispatches every frontend request to a Blockstudio template block. See [routing.md](routing.md).
- **Guard system** — three feature-based guards that prevent WordPress's site editor from creating or deleting templates, preserving the router-first model.
- **Feature registry** — a centralized, filterable flag system. Features are declared in `inc/features/`, configured in `examplepress.json`, and resolvable at runtime via PHP filters. Resolution order: PHP filter > JSON config > registration default.
- **App scaffolding** — REST endpoints and admin UI to create, connect, and manage companion plugins.
- **Blockstudio integration** — wires Blockstudio settings from `examplepress.json` into features and exempts router/template blocks from inner-block wrapping.

Key constants defined in `functions.php`:

| Constant | Value |
|---|---|
| `EP_THEME_VERSION` | From `style.css` header (currently 1.0.8) |
| `EP_THEME_PATH` | `get_template_directory()` |
| `EP_THEME_URI` | `get_template_directory_uri()` |

Setting `EP_DEV_MODE` to `true` in `wp-config.php` disables all three template guards for development.

### Layer 3: The Theme App

The main companion plugin — scaffolded from a template repo and installed to `wp-content/plugins/{slug}/`. It declares ownership of routes (front page, single post, archive, etc.) at a given priority and provides the template blocks that render those routes.

Structure:

```
wp-content/plugins/my-site-core/
├── app/templates/
│   ├── front/          # block.json + index.php + style.inline.scss
│   ├── single/
│   ├── archive/
│   └── 404/
├── examplepress.json   # App manifest (routing, Troy connection)
├── my-site-core.php    # Plugin file — must declare Theme: examplepress-theme
└── composer.json
```

The plugin header `Theme: examplepress-theme` is how the theme discovers the app. Without it, the plugin is invisible to the app registry.

### Layer 4: Additional Apps

Any number of companion plugins can coexist. Each registers its own route origin at a different priority (lower number = evaluated first). If two apps both claim `front`, the lower-priority app wins. Route conflicts are detectable via `examplepress_detect_route_conflicts()`.

## Configuration pipeline

`examplepress.json` in the theme root is the master configuration file. On load (`inc/config.php`), it is normalized into feature options through two shorthand systems:

1. **Design shorthand** — `design.colors`, `design.layout`, `design.typography`, etc. are expanded into `features.theme-colors.options.palette`, `features.theme-layout.options.wide_size`, and so on.
2. **Blockstudio shorthand** — `blockstudio.assets`, `blockstudio.tailwind`, `blockstudio.dev`, etc. are expanded into corresponding `blockstudio-*` feature options.

This means the JSON file is the single source of truth for design tokens, Blockstudio behavior, and feature flags — but any value can be overridden at runtime via a PHP filter (`examplepress_feature_{id}` or `examplepress_feature_{id}_{key}`).

## Feature registry

Features are registered in `inc/features/` and booted on `after_setup_theme`. Each feature has:

- An **id** (kebab-case, e.g. `guard-template-redirect`)
- An **enabled** state (resolved: PHP filter > JSON > default)
- **Options** (key/value pairs, same resolution order)
- Either a **hook + callback** (simple) or a **setup callable** (complex)

Simple features are wired automatically when enabled. Complex features (like guards) always run their setup callable and check enablement internally.

## Immutability enforcement

`examplepress_check_theme_immutability()` (`inc/route-registry.php`) verifies the theme directory contains only known files and directories. This is used in health checks and CI to ensure no one has modified the theme directly.

The guard system enforces immutability at the WordPress level:

| Guard | What it does |
|---|---|
| `guard-template-redirect` | Redirects site-editor template URLs to the Styles panel |
| `guard-template-rest` | Blocks template creation/deletion via REST API |
| `guard-template-resolution` | Filters templates shown in admin, hiding FSE templates |

## Bootstrap sequence

`functions.php` loads everything in a specific order:

1. Constants (`EP_THEME_VERSION`, `EP_THEME_PATH`, `EP_THEME_URI`)
2. Composer autoloader
3. Core includes: `helpers.php` → `config.php` → `feature-registry.php` → `features.php` → `route-registry.php` → `router.php` → `dependencies.php` → `notifications.php` → `apps.php` → `app-registry.php` → `app-cpt.php` → `github.php` → `github-app.php` → `scaffolder.php` → `api.php`
4. Admin includes (conditional on `is_admin()`): `admin-assets.php`, `settings-data.php`, `admin-registry.php`, and page controllers (`settings`, `apps`, `theme`, `navigation`, `dependencies`, `library`, `notifications`, `system`, `docs`, `editor`)
5. Late includes: `cli.php`, `demo-bootstrap.php`, `updater-bootstrap.php`
6. Blockstudio pattern path filter + inner-block wrapping filter
7. Feature boot on `after_setup_theme`
