# Blockstudio

ExamplePress renders everything through [Blockstudio](https://blockstudio.dev/) — a zero-build block rendering system for WordPress. Instead of the traditional block build pipeline (`@wordpress/scripts`, webpack, `block.json` + `edit.js` + `save.js`), blocks are defined as a `block.json` + `index.php` pair and rendered server-side.

## How Blockstudio works in ExamplePress

Blockstudio is a vendor dependency (`vendor/blockstudio/`), currently v7.1.2. The theme and each app initialize it independently:

**Theme** (in `functions.php`):
- Blockstudio discovers blocks in `blockstudio/` (the router block, site-editor overrides, login screen, patterns)
- Configuration lives in `blockstudio.json` at the theme root

**Apps** (in their main plugin file):
```php
add_action( 'init', function () {
    Blockstudio\Build::init( [
        'dir' => plugin_dir_path( __FILE__ ) . 'app',
    ] );
} );
```

This tells Blockstudio to scan the app's `app/` directory for blocks.

## Block structure

A Blockstudio block is a directory containing at minimum `block.json` and `index.php`:

```
app/templates/front/
├── block.json           # Block metadata (extends WordPress block.json)
├── index.php            # Server-side render template
└── style.inline.scss    # Optional — scoped styles, compiled automatically
```

### block.json

Standard WordPress block metadata with Blockstudio additions:

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "my-site-core/template-front",
  "title": "Front Page",
  "category": "theme",
  "blockstudio": true,
  "supports": {
    "inserter": false,
    "lock": {
      "move": true,
      "remove": true,
      "rename": true
    }
  }
}
```

The `"blockstudio": true` flag tells Blockstudio to handle rendering. Template blocks typically disable the inserter and lock themselves to prevent accidental modification.

The block name follows the convention: `{namespace}/template-{route-slug}`. The namespace is the app's slug, and the route slug maps to the router (see [routing.md](routing.md)).

### index.php

A PHP template that receives block attributes (including enriched route data) and renders HTML:

```php
<?php
// $attributes contains data from examplepress_route_data filter
$post = $attributes['post'] ?? null;
?>
<article>
    <h1><?php echo esc_html( $post->post_title ?? 'Welcome' ); ?></h1>
    <?php echo apply_filters( 'the_content', $post->post_content ?? '' ); ?>
</article>
```

### style.inline.scss

Optional SCSS that Blockstudio compiles and scopes to the block. No build step needed — Blockstudio handles SCSS compilation at runtime when the `scss` asset processing option is enabled.

## Build phases

Blockstudio processes blocks in four phases:

| Phase | Class | What happens |
|---|---|---|
| 1 | `Block_Discovery` | Recursively scans directories for `block.json` files |
| 2 | `Asset_Discovery` | Processes CSS, SCSS, and JS files; compiles SCSS; scopes styles |
| 3 | `Block_Registrar` | Registers blocks with WordPress as `WP_Block_Type` instances |
| 4 | `Extensions` | Applies block overrides and extension configurations |

All phases run on `init` at `PHP_INT_MAX - 1` priority — after everything else has registered.

## Rendering API

Blockstudio provides two functions for programmatic block rendering:

| Function | Returns | Use |
|---|---|---|
| `bs_block( $value )` | String | Get block content (used by the router) |
| `bs_render_block( $value )` | void | Echo block content directly |

The router uses `bs_block()` to dispatch to the resolved template block, passing enriched route data as attributes.

## Configuration

### Theme-level (`blockstudio.json`)

```json
{
  "assets": {
    "enqueue": true,
    "minify": { "css": true, "js": true },
    "process": { "scss": true }
  },
  "dev": {
    "grab": { "enabled": true },
    "canvas": { "enabled": true, "adminBar": true }
  }
}
```

### Feature-controlled settings

ExamplePress bridges `examplepress.json` configuration to Blockstudio via features registered in `inc/features/blockstudio.php`. These use Blockstudio's filter system:

| Feature | Controls |
|---|---|
| `blockstudio-assets` | Asset enqueue and reset behavior |
| `blockstudio-asset-reset` | CSS reset injection |
| `blockstudio-minify` | CSS and JS minification |
| `blockstudio-scss` | SCSS compilation |
| `blockstudio-tailwind` | Tailwind CSS integration |
| `blockstudio-editor` | Editor format, assets, and markup |
| `blockstudio-block-editor` | Block editor CSS variables |
| `blockstudio-ai-context` | AI context generation for blocks |
| `blockstudio-block-tags` | Block HTML tag rendering |
| `blockstudio-dev` | Dev tools (grab, canvas) |
| `blockstudio-users` | User-specific settings |

Each feature maps `examplepress.json` values to Blockstudio's `blockstudio/settings/*` filter hooks.

## Inner-block wrapping exemption

By default, Blockstudio wraps inner blocks in a container `<div>` for the frontend. ExamplePress exempts two categories from wrapping (`functions.php` lines 76–97):

1. **The router block** (`examplepress-theme/router`) — always unwrapped
2. **Template blocks** — unwrapped for all registered route origin namespaces plus the theme namespace

This is handled via the `blockstudio/blocks/components/inner_blocks/frontend/wrap` filter.

## Patterns

Blockstudio's pattern system is wired to the theme via the `blockstudio/patterns/paths` filter. Patterns in `blockstudio/patterns/` are automatically registered and available in the block editor.

## Key Blockstudio hooks used by ExamplePress

| Hook | Purpose |
|---|---|
| `blockstudio/settings/assets/*` | Asset enqueue, reset, minify |
| `blockstudio/settings/tailwind/*` | Tailwind configuration |
| `blockstudio/settings/editor/*` | Editor format and assets |
| `blockstudio/settings/blockEditor/*` | Block editor CSS/variables |
| `blockstudio/settings/ai/*` | AI context generation |
| `blockstudio/settings/blockTags/*` | Block tag rendering |
| `blockstudio/blocks/components/inner_blocks/frontend/wrap` | Inner-block wrapping control |
| `blockstudio/patterns/paths` | Pattern discovery paths |
