# ExamplePress

ExamplePress is a modular framework that turns WordPress into a scalable application platform. Instead of a monolithic child-theme approach, ExamplePress enforces clean architectural boundaries — separating infrastructure, application logic, and distribution into dedicated repositories.

This repository aggregates the full ExamplePress codebase into a single context for reference, review, and architectural understanding.

---

## How It Works

ExamplePress replaces the standard WordPress template hierarchy with a router-first architecture. The theme itself acts as an operating system — it routes requests, orchestrates the environment, and provides the hooks that allow individual "apps" (standard WordPress plugins) to render content. Apps are scaffolded from a boilerplate template and built with Blockstudio's zero-build block rendering. The platform keeps itself healthy via GitHub Releases; apps shared across multiple sites are distributed through Troy Server.

```
                                             Platform Updates
                                             (every site)
Site                                         ┌───────────────────────┐
┌──────────────────────────────────────┐     │ GitHub Releases       │
│                                      │     │                       │
│  examplepress-mu              ◄──────┼─────│ MU plugin checks      │
│  MU plugin. Self-updating            │     │ twice daily, downloads │
│  platform kernel — loads before      │     │ + validates updates.   │
│  the theme, bootstraps the env.      │     │                       │
│  Also owns the theme update          │     │ ThemeUpdateProvider    │
│  pipeline (ThemeUpdateProvider).     │◄────│ (inside the MU)        │
│       │                              │     │ fetches theme releases │
│  examplepress-theme                  │     │ on demand or via       │
│  The operating system. Routes        │     │ admin check.           │
│  requests, orchestrates hooks,       │     └───────────────────────┘
│  installs dependencies.              │
│       │                              │     Fleet Distribution
│       ├── App (plugin)               │     (shared apps only)
│       ├── App (plugin)  ◄────────────┼──── ┌───────────────────────┐
│       └── App (plugin)               │     │ troy                  │
│                                      │     │ Central hub for apps  │
│  blockstudio                         │     │ shared across sites.  │
│  Rendering engine. Filesystem-based  │     │                       │
│  block registration — drop a         │     │ examplepress-troy-    │
│  block.json + PHP template,          │     │ bridge                │
│  no build step required.             │     │ Server-side bridge.   │
│                                      │     │ Adds provisioning     │
│                                      │     │ endpoints for app     │
│                                      │     │ scaffolding.          │
│                                      │     └──────▲────────────────┘
│                                      │            │
└──────────────────────────────────────┘     ┌──────┴────────────────┐
                                             │ Connected sites       │
                                             │ pull shared app       │
                                             │ updates and verify    │
                                             │ licenses from Troy.   │
                                             └───────────────────────┘
```

---

## Repositories

### Platform

**[`examplepress-mu`](https://github.com/webmultipliers/examplepress-mu)** — A self-updating MU plugin that acts as the platform kernel. Loaded by WordPress before the theme, it consists of a thin loader that fetches and bootstraps the core application from GitHub releases automatically. A built-in WP-Cron job checks for updates twice daily, validates downloads with SHA-256, and overwrites the kernel in place. Also owns the theme update pipeline (channel/pin/install/reinstall against `webmultipliers/examplepress-theme`) via `Infrastructure/ThemeUpdateProvider`.

### Infrastructure

**[`examplepress-theme`](https://github.com/webmultipliers/examplepress-theme)** — The foundational layer. Removes the standard WordPress template hierarchy and replaces it with a router-first architecture. Routes requests, orchestrates the environment, installs dependencies, and provides the hooks that allow apps to render content. Theme updates are delivered by the MU kernel's `ThemeUpdateProvider`.

### Application Layer

**[`examplepress-theme-app`](https://github.com/webmultipliers/examplepress-theme-app)** — Boilerplate template repository for scaffolding new apps. Instead of custom code in `functions.php`, developers fork this template to create isolated, version-controlled apps (standard WordPress plugins) that handle specific routing and content logic through Blockstudio.

**[`examplepress-demo`](https://github.com/webmultipliers/examplepress-demo)** — Kitchen sink reference implementation. A fully configured working example showing how the theme, apps, and ecosystem tools interact. Serves as a blueprint for developers to understand routing, block rendering, and deployment patterns.

### Distribution

**[`examplepress-troy-bridge`](https://github.com/webmultipliers/examplepress-troy-bridge)** — A server-side bridge plugin installed on the Troy Server. Adds REST endpoints for programmatic plugin provisioning and health checks so that ExamplePress's app scaffolding flow can register new apps on Troy in a single API call. Without it, each step (plugin creation, GitHub integration, tag fetching) would require manual setup in the Troy admin.

### External Dependencies

**[`blockstudio`](https://github.com/inline0/blockstudio)** — The rendering engine. Filesystem-based Gutenberg block framework that eliminates Node.js build steps, Webpack, and React compilation. A `block.json` next to a PHP template file is all that's needed for registration, asset enqueuing, and rendering.

**[`troy`](https://github.com/sybrew/troy)** — The fleet distribution hub. Server-side software that stores versioned releases of apps shared across multiple sites and serves update packages to connected sites. Not used for the MU plugin or theme, which update directly from GitHub. The counterpart to the Troy Bridge — this is the infrastructure required to operate a private plugin repository and push fleet-wide updates.

---

## Repository Structure

```
examplepress/
├── docs/
│   ├── architecture.md          # Four-layer model, config pipeline, bootstrap
│   ├── apps.md                  # App anatomy, lifecycle, scaffolding, registry
│   ├── routing.md               # Router-first architecture, route origins, dispatch
│   ├── distribution.md          # GitHub updaters, Troy fleet distribution, channels
│   ├── blockstudio.md           # Zero-build block rendering, block.json conventions
│   └── glossary.md              # Theme vs App vs Companion, Troy vs Bridge, etc.
├── packages/
│   ├── wp/
│   │   ├── themes/
│   │   │   └── examplepress-theme/
│   │   ├── plugins/
│   │   │   ├── examplepress-theme-app/
│   │   │   ├── examplepress-theme-demo/
│   │   │   └── examplepress-troy-bridge/
│   │   └── mu-plugins/
│   │       └── examplepress-mu/
│   └── vendor/
│       ├── blockstudio/
│       └── troy/
└── README.md
```

The `packages/` directory mirrors the WordPress `wp-content/` layout under `wp/`, with submodules grouped by type (`themes/`, `plugins/`, `mu-plugins/`). External Composer dependencies live under `packages/vendor/`. The `docs/` directory is the only original content in this repo besides this README — it covers the concepts needed to orient yourself in the codebase.
