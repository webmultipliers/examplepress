# ExamplePress

ExamplePress is a modular framework that turns WordPress into a scalable application platform. Instead of a monolithic child-theme approach, ExamplePress enforces clean architectural boundaries — separating infrastructure, application logic, and distribution into dedicated repositories.

This repository aggregates the full ExamplePress codebase into a single context for reference, review, and architectural understanding.

---

## How It Works

ExamplePress replaces the standard WordPress template hierarchy with a router-first architecture. The theme itself acts as an operating system — it routes requests, orchestrates the environment, and provides the hooks that allow individual "apps" (standard WordPress plugins) to render content. Apps are scaffolded from a boilerplate template, built with Blockstudio's zero-build block rendering, and distributed across fleets of sites through Troy Server.

```
Site                                         Fleet Infrastructure
┌──────────────────────────────────────┐
│                                      │
│  examplepress-theme                  │
│  The operating system. Routes        │
│  requests, orchestrates hooks,       │
│  installs dependencies.              │
│       │                              │
│       ├── App (plugin)               │     ┌───────────────────────┐
│       ├── App (plugin)  ◄────────────┼─────│ troy                  │
│       └── App (plugin)               │     │ Central distribution  │
│                                      │     │ hub. Stores and       │
│  examplepress-theme-update           │     │ serves versioned      │
│  Standalone updater. Survives        │     │ app releases.         │
│  theme fatal errors to pull          │     └──────▲────────────────┘
│  patches from source.                │            │
│                                      │     ┌──────┴────────────────┐
│  examplepress-troy-bridge            │     │ Connected sites       │
│  Client-side connector to Troy.      │     │ browse, verify        │
│  Handles license verification        │     │ licenses, and pull    │
│  and fleet-wide app updates.         │     │ updates from Troy.    │
│                                      │     └───────────────────────┘
│  blockstudio                         │
│  Rendering engine. Filesystem-based  │
│  block registration — drop a         │
│  block.json + PHP template,          │
│  no build step required.             │
│                                      │
└──────────────────────────────────────┘
```

---

## Repositories

### Infrastructure

**[`examplepress-theme`](https://github.com/webmultipliers/examplepress-theme)** — The foundational layer. Removes the standard WordPress template hierarchy and replaces it with a router-first architecture. Routes requests, orchestrates the environment, installs dependencies, and provides the hooks that allow apps to render content.

**[`examplepress-theme-update`](https://github.com/webmultipliers/examplepress-theme-update)** — A decoupled updater for the theme. Separated from the theme itself for resilience — if the theme hits a fatal error, the standalone updater remains functional and can still pull patches and hotfixes. Bypasses WordPress.org for private distribution.

### Application Layer

**[`examplepress-theme-app`](https://github.com/webmultipliers/examplepress-theme-app)** — Boilerplate template repository for scaffolding new apps. Instead of custom code in `functions.php`, developers fork this template to create isolated, version-controlled apps (standard WordPress plugins) that handle specific routing and content logic through Blockstudio.

**[`examplepress-theme-demo`](https://github.com/webmultipliers/examplepress-theme-demo)** — Kitchen sink reference implementation. A fully configured working example showing how the theme, apps, and ecosystem tools interact. Serves as a blueprint for developers to understand routing, block rendering, and deployment patterns.

### Distribution

**[`examplepress-troy-bridge`](https://github.com/webmultipliers/examplepress-troy-bridge)** — Client-side connector between an ExamplePress site and a central Troy server. Distinct from custom apps deployed via GitHub to a single site — the Troy Bridge handles browsing, license verification, and pulling unified app updates from a central hub. Powers fleet management, distributed product ecosystems, and WordPress-as-a-Service models.

### External Dependencies

**[`blockstudio`](https://github.com/inline0/blockstudio)** — The rendering engine. Filesystem-based Gutenberg block framework that eliminates Node.js build steps, Webpack, and React compilation. A `block.json` next to a PHP template file is all that's needed for registration, asset enqueuing, and rendering.

**[`troy`](https://github.com/sybrew/troy)** — The distribution hub. Server-side software that stores versioned app releases and serves update packages to connected sites. The counterpart to the Troy Bridge — this is the infrastructure required to operate a private plugin repository and push fleet-wide updates.

---

## Repository Structure

```
examplepress/
├── docs/
│   ├── architecture.md     # Three-layer model, config pipeline, bootstrap
│   ├── apps.md             # App anatomy, lifecycle, scaffolding, registry
│   ├── routing.md          # Router-first architecture, route origins, dispatch
│   ├── distribution.md     # Troy Server, Troy Bridge, theme updates, channels
│   ├── blockstudio.md      # Zero-build block rendering, block.json conventions
│   └── glossary.md         # Theme vs App vs Companion, Troy vs Bridge, etc.
├── packages/
│   ├── theme/              ← examplepress-theme
│   ├── theme-update/       ← examplepress-theme-update
│   ├── theme-app/          ← examplepress-theme-app
│   ├── theme-demo/         ← examplepress-theme-demo
│   └── troy-bridge/        ← examplepress-troy-bridge
├── vendor/
│   ├── blockstudio/        ← inline0/blockstudio
│   └── troy/               ← sybrew/troy
└── README.md
```

Each directory under `packages/` and `vendor/` is a Git submodule pointing to its upstream repository. The `docs/` directory is the only original content in this repo besides this README — it covers the concepts needed to orient yourself in the codebase.