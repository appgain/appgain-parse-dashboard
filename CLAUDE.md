# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
npm install

# Development (starts Express server + webpack watch)
npm run dev           # uses eval-source-map for faster rebuilds
npm run dashboard     # same but without source map flag

# Production build (outputs to Parse-Dashboard/public/bundles/)
npm run build

# Lint
npm run lint

# Tests (lint runs first via pretest)
npm test

# Run a single test file
NODE_PATH=./node_modules jest src/lib/tests/Authentication.test.js

# Start server only (no webpack)
npm start
```

The dev server runs on port **4040** by default. The dashboard requires a config file or CLI options (`--appId`, `--masterKey`, `--serverURL`).

## Architecture

This is a **fork of Parse Dashboard** customized by Appgain. It is a full-stack app: an Express.js server (`Parse-Dashboard/`) serves a React SPA (`src/`) built with webpack.

### Server layer (`Parse-Dashboard/`)

- `index.js` — CLI entrypoint. Reads config from a JSON file or env vars/flags, constructs the Express app, and starts the HTTP(S) server. Env vars: `PARSE_DASHBOARD_APP_ID`, `PARSE_DASHBOARD_MASTER_KEY`, `PARSE_DASHBOARD_SERVER_URL`, `PARSE_DASHBOARD_CONFIG`, `PORT`, `HOST`, `MOUNT_PATH`.
- `app.js` — Express middleware factory exported as a module (usable as middleware by other apps). Wires up static file serving, authentication, CSRF, the `/parse-dashboard-config.json` endpoint (which delivers app credentials to the SPA), and a catch-all that serves the SPA shell.
- `Authentication.js` — Passport/local-strategy authentication. Supports plaintext and bcrypt-hashed passwords, per-user app restrictions, and read-only access.

The server enforces: HTTPS-only for remote access, CSRF on login, and master key suppression for read-only users.

### Client layer (`src/`)

Webpack entry points (configured in `webpack/build.config.js`) are `src/dashboard/index.js` (main app) and `src/login/index.js`. Bundles are written to `Parse-Dashboard/public/bundles/`.

**Module resolution:** webpack resolves from `src/` directly, so `import Foo from 'lib/Foo'` resolves to `src/lib/Foo.js`. The jest preprocessor (`testing/preprocessor.js`) replicates this with string replacement.

**CSS Modules:** `.scss` files are processed with CSS Modules. Class names in JS appear as `styles.foo` referencing the local scoped name.

#### React component hierarchy

```
Dashboard (Router root, fetches /parse-dashboard-config.json, seeds AppsManager)
└── AppData (React.createClass – provides currentApp + generatePath via context)
    └── DashboardView (base class for all per-app views)
        ├── renders Sidebar (built from server feature flags in DashboardView.react.js)
        └── renderContent() / renderSidebar() — overridden by subclasses
            └── TableView (extends DashboardView, for tabular pages like Jobs, Push, etc.)
```

Feature sections under `src/dashboard/`:
- `Data/Browser/` — Data browser (the most complex feature; uses `SchemaStore`)
- `Data/CloudCode/`, `Data/Jobs/`, `Data/Logs/`, `Data/Config/`, `Data/ApiConsole/`, `Data/Webhooks/`
- `Push/` — Push notification sending, history, audiences
- `Analytics/` — Analytics views (largely commented out pending Parse Server support)
- `Settings/` — App settings pages
- `Apps/` — App selector/index

#### State management (`src/lib/stores/`)

Custom Flux-like store system (not Redux). Key files:
- `StoreManager.js` — Registry. Stores are registered by name and dispatch actions that return either a new state value or a `Parse.Promise` that resolves to one.
- `StateManager.js` — Holds per-app and global state maps.
- `subscribeTo.js` — HOC decorator (`@subscribeTo('Schema', 'schema')`) that subscribes a component to a named store and injects `{ data, dispatch }` as a prop.

Stores: `SchemaStore`, `ConfigStore`, `JobsStore`, `PushAudiencesStore`, `AnalyticsQueryStore`, `WebhookStore`.

#### `src/lib/`

Utility modules and the core `ParseApp` class:
- `ParseApp.js` — Wraps a connected Parse app. Holds credentials and makes API requests via the Parse JS SDK. Components access the current app through React context (`this.context.currentApp`).
- `AppsManager.js` — Singleton in-memory registry of `ParseApp` instances. Seeded from `/parse-dashboard-config.json` at startup.
- `AJAX.js` — XHR wrapper that prepends `basePath`, attaches CSRF tokens, and returns `Parse.Promise`s.
- `Constants.js` — `AsyncStatus`, `DefaultColumns`, `SpecialClasses`, and other shared enums.

#### `src/components/`

Reusable UI primitives (Button, Dropdown, Modal, Sidebar, etc.). Components follow the `ComponentName/ComponentName.react.js` naming convention and co-locate their `.scss` file.

### Webpack configs (`webpack/`)

- `base.config.js` — Shared rules: babel-loader (decorators, object rest spread, regenerator), CSS modules for `.scss`, file-loader for images, custom SVG plugin.
- `build.config.js` — Dev build, outputs to `Parse-Dashboard/public/bundles/`.
- `production.config.js` — Production build with UglifyJS, outputs to `production/bundles/`.
- `publish.config.js` — Pre-publish build for npm package.
- `PIG.config.js` — Parse Interface Guide (component library preview).

### Testing

Tests live in `src/lib/tests/`. Jest is configured to look only in `src/lib` (`jest.roots`). The preprocessor handles SCSS imports and module alias rewriting. Tests use jest's manual mock system (`jest.dontMock(...)` to opt modules out of auto-mocking).

To test server-side logic (e.g. `Authentication.js`), tests import directly from `Parse-Dashboard/`.

### Docker

The multi-stage Dockerfile builds the app and produces a minimal production image. In the production image, `Parse-Dashboard/index.js` is patched to look for config at `conf/config.json` instead of `parse-dashboard-config.json`. The entrypoint always passes `--allowInsecureHTTP`.
