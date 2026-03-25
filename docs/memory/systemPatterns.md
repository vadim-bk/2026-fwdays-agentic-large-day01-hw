# System patterns

## Details
For detailed architecture → see [docs/technical/architecture.md](../technical/architecture.md)  
For domain glossary → see [docs/product/domain-glossary.md](../product/domain-glossary.md)

## Repo / module architecture
- **Yarn workspaces monorepo** (root `package.json`):
  - **App**: `excalidraw-app/`
  - **Packages**: `packages/*`
  - **Examples**: `examples/*`
- **TypeScript path-alias “workspace imports”** (root `tsconfig.json`):
  - `@excalidraw/common` → `packages/common/src/index.ts`
  - `@excalidraw/element` → `packages/element/src/index.ts`
  - `@excalidraw/math` → `packages/math/src/index.ts`
  - `@excalidraw/utils` → `packages/utils/src/index.ts`
  - `@excalidraw/excalidraw` → `packages/excalidraw/index.tsx`

## “Develop against source” pattern
- **App uses package source directly** while developing:
  - `excalidraw-app/vite.config.mts` defines `resolve.alias` rules mapping `@excalidraw/*` imports to `../packages/*/src` (and `../packages/excalidraw/index.tsx`).
  - This avoids publishing/packing to iterate locally and keeps a single source of truth.

## Packages as ESM with explicit exports maps
- `packages/excalidraw/package.json`:
  - `"type": "module"` and `exports` map for `.` + typed subpaths + CSS entry (`./index.css`).
  - **Deep subpaths are primarily typed entrypoints** (exports include `"types": ...`), while runtime entry stays at `.`.
- Pattern encourages **stable public API surface** and better bundler compatibility.

## App build: static assets + caching-aware chunking
- **Static SPA build**:
  - Vite outputs to `excalidraw-app/build` (`build.outDir`).
  - Docker image serves the built files via nginx (`Dockerfile` final stage).
- **Manual chunking for long-lived assets** (`excalidraw-app/vite.config.mts`):
  - Locales are chunked separately (except `en`/`percentages`) to support caching/offline behavior.
  - Dedicated chunks for heavy deps (e.g. CodeMirror / Lezer, Mermaid integration).
- **PWA pattern**:
  - `VitePWA` is configured with explicit Workbox runtime caching for fonts, locales, and chunk JS.

## UI composition: “library core + app shell”
- **Library**: `@excalidraw/excalidraw` provides the embeddable `<Excalidraw />` component and related API/provider utilities.
  - Example pattern: `ExcalidrawAPIProvider` enables hooks like `useExcalidrawAPI()` outside the component tree (see `packages/excalidraw/index.tsx`).
- **App shell**: `excalidraw-app/App.tsx` composes the library’s primitives + app-specific integrations:
  - Collaboration, persistence, dialogs/menus, analytics, and external services.
  - This keeps the reusable canvas/editor logic in the package and the “product” concerns in the app.

## Cross-cutting concerns via shared `common` package
- `packages/common/src/index.ts` re-exports shared utilities/constants (e.g. event bus, editor interface helpers).
- This supports a pattern of **single shared foundation** reused by both:
  - The published `@excalidraw/excalidraw` package
  - The `excalidraw-app` product shell

## Environment/config injection pattern
- Package build scripts (e.g. `scripts/buildPackage.js`) inject environment variables into `import.meta.env` for dev/prod builds.
- App config loads env from repo root via `loadEnv(mode, "../")` and uses `envDir: "../"` (`excalidraw-app/vite.config.mts`).

