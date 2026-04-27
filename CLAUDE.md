# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev       # Dev server on port 8000 (auto-opens browser)
npm run build     # Production build to dist/
npm run preview   # Preview production build locally
```

No test or lint scripts are configured.

## Architecture

Interactive 3D anatomical viewer — a Vue 3 + Vite PWA that renders a `.glb` model and lets users highlight, inspect, and search anatomical structures.

**Tech stack:** Vue 3 (`<script setup>`), Three.js, TailwindCSS, vite-plugin-pwa (Workbox). No backend; `/static/` assets are proxied to `https://infirmier.jcbuisson.dev/`.

**Two components:**

- `src/components/AnatomyDetail.vue` — the entire 3D viewer. Loads `/static/uploads/arthrologie.glb` via `GLTFLoader`, drives `OrbitControls`, and wires up raycasting for mouse interaction. Mesh state (highlight, transparency, original material) is stored on `mesh.userData`. A normalization function strips accents, underscores, suffixes, and numbers to make search fuzzy. Mouse intent (single-click vs double-click vs pan) is resolved with a 200 ms / 500 ms debounce pattern.

- `src/components/VersionUpdater.vue` — polls the service worker every 10 s and reloads the page when a new version is available.

**Key data file:** `src/assets/meshKey2Name.json` maps internal mesh names (e.g. `"thriblgeneratedmeshfromxd"`) to French anatomical labels (e.g. `"Côte gauche"`). This file is the source of truth for all displayed names.

**Service worker** (`src/sw.ts`) is a custom Workbox config with a 4 MB precache cap, 1-year font cache, and navigation fallback to `index.html`. `SW_VERSION` is manually incremented (currently `'0.0.9'`); it is not tied to `package.json`.

## Key patterns

- Highlighting swaps a mesh's material for a neon-green emissive material; the original is saved on `mesh.userData.originalMaterial`.
- Double-clicking a mesh sets its opacity to 0.05 (`isTransparent`) to expose occluded structures.
- Search normalizes both the query and `meshKey2Name` values before matching — always go through the normalization function when comparing mesh names.
- The 3D model URL is hardcoded; there is commented-out async API loading code that hints at a future model-switching feature.
