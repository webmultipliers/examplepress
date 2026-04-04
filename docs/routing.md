# Routing

ExamplePress replaces WordPress's template hierarchy with a single-entry-point router. Every frontend request passes through the same `templates/index.html`, which contains the router block. The router resolves which template block to render based on registered route origins.

## Request flow

```
1. WordPress loads templates/index.html (the only template)
2. The router block executes blockstudio/router/index.php
3. examplepress_resolve_route() walks the route origin registry
4. First matching origin returns { namespace, slug }
5. If no origin matches → defaults to examplepress-theme/template-get-started
6. examplepress_get_template_prefix() resolves the prefix (default: 'template')
7. Block name assembled: {namespace}/{prefix}-{slug}
8. apply_filters( 'examplepress_route_data', [], $slug, $block_name )
9. do_action( 'examplepress_route_resolved', $slug, $block_name )
10. bs_block() dispatches to the resolved template block
```

If the resolved block doesn't exist, the router renders an error `<div>` instead of a blank page.

## Route origins

A **route origin** is a namespace (typically a plugin slug) that claims a set of route slugs, each guarded by a callable condition.

### Registration

```php
examplepress_register_route_origin( 'my-site-core', [
    'front'   => fn() => is_front_page() || is_home(),
    'single'  => fn() => is_singular(),
    'archive' => fn() => is_archive(),
    'search'  => fn() => is_search(),
    '404'     => fn() => is_404(),
], 10 ); // priority — lower = evaluated first
```

Origins are stored in a global `$examplepress_route_origins` array, keyed by priority then namespace.

### Resolution

`examplepress_resolve_route_origin()` walks origins in ascending priority order. For each origin, it tests every route's callable against the current request. The first match wins.

If two apps register the same slug at different priorities, the lower priority (evaluated first) takes precedence. `examplepress_detect_route_conflicts()` reports when multiple origins claim the same slug — useful for debugging multi-app setups.

### Multi-origin example

```php
// Priority 10 — evaluated first
examplepress_register_route_origin( 'my-site-core', [
    'front'  => fn() => is_front_page() || is_home(),
    'single' => fn() => is_singular(),
], 10 );

// Priority 20 — evaluated after priority 10
examplepress_register_route_origin( 'my-blog-addon', [
    'single' => fn() => is_singular( 'post' ),  // narrows to posts only
    'author' => fn() => is_author(),
], 20 );
```

On a single post page, `my-site-core/template-single` wins because priority 10 is evaluated before 20. The blog addon's `single` route would only fire if `my-site-core` didn't claim it.

## Template block resolution

Once a route resolves to `{ namespace, slug }`, the router assembles a fully-qualified block name:

```
{namespace}/{prefix}-{slug}
```

- **namespace** — the route origin's namespace (e.g. `my-site-core`)
- **prefix** — defaults to `template`, filterable via `examplepress_template_prefix`
- **slug** — the matched route slug (e.g. `front`, `single`, `404`)

Result: `my-site-core/template-front`

This block must exist as a Blockstudio block in the app's `app/templates/{slug}/` directory.

## Filters and actions

| Hook | Type | Arguments | Purpose |
|---|---|---|---|
| `examplepress_resolved_origin` | filter | `$origin` | Modify the resolved `{ namespace, slug }` before dispatch |
| `examplepress_template_prefix` | filter | `$prefix` | Override the template prefix (default: `'template'`) |
| `examplepress_template_block_name` | filter | `$block_name, $slug, $prefix, $namespace` | Override the fully-assembled block name |
| `examplepress_route_data` | filter | `$data, $slug, $block_name` | Enrich data passed to the template block |
| `examplepress_route_resolved` | action | `$slug, $block_name` | Fires after resolution, before rendering — use for side effects (enqueue assets, register sidebars) |

## Introspection helpers

| Function | Returns |
|---|---|
| `examplepress_has_route_origins()` | `bool` — whether any origins are registered |
| `examplepress_get_route_origin_namespaces()` | `string[]` — all registered namespaces |
| `examplepress_get_route_origin_map()` | Full origin map for debugging |
| `examplepress_route_slug_owner( $slug )` | Namespace that owns a given slug |
| `examplepress_detect_route_conflicts()` | Slugs claimed by multiple origins |

## Guard system

The router model only works if WordPress can't create competing templates. Three guards enforce this:

- **Template redirect** — redirects site-editor template URLs to the Styles panel
- **Template REST** — blocks template creation/deletion via the REST API
- **Template resolution** — filters the template list in admin to hide FSE templates

All three are features (`guard-template-redirect`, `guard-template-rest`, `guard-template-resolution`) and can be disabled individually or all at once by setting `EP_DEV_MODE` to `true`.

## Default fallback

When no route origin matches (e.g. on a fresh install with no apps), the router falls back to `examplepress-theme/template-get-started` — a built-in template block that shows the "get started" onboarding screen.
