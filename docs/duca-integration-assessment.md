# DUCA GEO integration assessment

## 1. Existing architecture

This repository is the WRI GFW Mapbuilder template, forked as `sura-genesis/gfw-mapbuilder`. It is a web application built around React, TypeScript, Redux, and the ArcGIS JavaScript API, with configuration-driven map layers and UI behaviour.

Key architecture points observed from the codebase:

- App entry point: `src/js/index.tsx`
  - Creates the React root and mounts the app under a Redux Provider.
- Application bootstrap: `src/js/MapBuilderMain.tsx`
  - Passes app settings into the main `App` component.
- Config-driven app settings: `configs/resources.js`
  - Defines defaults for title, language, time range, layer-panel groups, basemaps, and `extraLayers`.
  - Uses a `layerPanel` object with groups like `GROUP_LCD`, `GROUP_LC`, `GROUP_IMAGERY`, and `GROUP_BASEMAP`.
- Layer registry and schema: `configs/layers/layers-content-config.ts`
  - Imports individual layer config modules from `configs/layers/*/index.ts`.
  - Exposes `layersContentConfig` as the registry consumed by the app.
- Layer schema: `configs/layers/types/index.ts`
  - Defines the expected layer metadata and identifiers such as `id`, `order`, `type`, `uuid`, and `groupId`.
- Layer factory: `src/js/helpers/LayerFactory.ts`
  - Switches on `layerConfig.type` and instantiates ArcGIS layer objects for `dynamic`, `image`, `feature`, `loss`, `webtiled`, `tiled`, `wms`, `imagery`, `glad`, and others.
- ArcGIS mapping and controls are built around the MapBuilder app pattern rather than a bespoke DUCA engine.

The repository is not a blank-slate app; it is a reusable template that expects a set of layer definitions and app configuration metadata. For DUCA GEO, the correct integration point is to extend the existing layer configuration and app state rather than replacing the map engine.

## 2. Files to reuse

These are the primary files and directories to build on for the DUCA GEO work without rewriting the app architecture:

- `configs/resources.js`
  - Main app configuration and layer panel definition.
- `configs/layers/layers-content-config.ts`
  - Central registry for layer configs.
- `configs/layers/types/index.ts`
  - Supported layer schema and metadata contract.
- `src/js/helpers/LayerFactory.ts`
  - Layer instantiation / ArcGIS integration.
- `src/js/index.tsx`
  - Entry point for the React app.
- `src/js/components/*`
  - Existing UI and Redux patterns for panels, widgets, and map interaction.
- `src/js/store/*`
  - Redux architecture for app state, settings, and map state.
- `configs/layers/*/index.ts`
  - Existing example layer configs for style, metadata, and layer options.
- `tests/resources.test.js`
  - Existing repo-level config validation tests.
- `tests/layerFactory.test.js`
  - Existing smoke test for layer type coverage.

## 3. Files to add or modify

For the DUCA phase we should keep changes minimally scoped and isolated:

### Frontend configuration
- `configs/resources.js`
  - Add DUCA layer groups: Coffee Plots, Forest Baseline, Tree-Cover Loss, Forest-Change Evidence.
- `configs/layers/duca-*` or a DUCA-specific layer directory
  - Add unique layer definitions that follow the existing schema and use verified URLs.
- `configs/layers/layers-content-config.ts`
  - Register DUCA layer configs.

### Frontend app code
- `src/js/components/*`
  - Add DUCA-specific plot and assessment panels if required by the app shell.
- `src/js/store/*`
  - Extend Redux state for selected plot, assessment results, and loading/error states if the app uses them.
- `src/js/helpers/*`
  - Possibly add a DUCA API client and plot geometry/assessment helpers.

### Backend API
- `duca-geo-api/` (new)
  - Python/FastAPI service with validation, GeoJSON handling, PostGIS schema, and assessment persistence.
- `duca-geo-api/app/`
  - API routes, schemas, geometry validation, adapters, and risk methodology logic.
- `duca-geo-api/migrations/` or `alembic` setup
  - Database migration for plots, datasets, assessment runs, evidence items, and review events.

### Documentation
- `docs/architecture.md`
- `docs/datasets.md`
- `docs/risk-methodology.md`
- `docs/duca-integration-assessment.md` (this file)

## 4. External service dependencies

The repository already depends on a handful of external services and environment variables.

### Existing repo dependencies
- ArcGIS JavaScript API / ArcGIS Online / Esri map services.
- Planet imagery access through the environment variable `REACT_APP_PLANET_API_KEY`.
- Existing layer URLs are served from WRI/ArcGIS/GFW endpoints.
- Mapbuilder configuration includes a `webmap` and remote data layers, meaning it is designed around external map services and metadata endpoints.

### Current `.env` requirement
Observed in `.env.examples`:

- `REACT_APP_PLANET_API_KEY=planet api key`

This is a placeholder requiring an actual key. No other app-specific DUCA envs are present yet.

### External data providers likely relevant to DUCA
- Global Forest Watch Data API (official docs: https://data-api.globalforestwatch.org/docs)
- ArcGIS REST services used by the application.
- Optional future PostgreSQL/PostGIS + FastAPI backend services for DUCA GEO.

## 5. Baseline test and build results

### Repository metadata status
The repository currently has the following scripts in `package.json`:

- `npm start` -> `webpack-dev-server --config webpack.development.js`
- `npm run build` -> `webpack --config webpack.production.js --stats verbose`
- `npm run lib-cms` -> library build path

There is no explicit `test` script in `package.json`.

### Existing tests
The repo contains Jest-style smoke tests:

- `tests/resources.test.js`
- `tests/layerFactory.test.js`

These validate config properties and the layer factory entry points, but they are minimal and not yet the coverage needed for DUCA-specific forest screening flows.

### Build/test execution status
This environment does not expose a shell/Node execution tool, so I could not run `npm install`, `npm run build`, or the existing Jest tests directly from here. The baseline status is therefore based on repository inspection rather than actual runtime execution.

This is important: the repo is configured to support a webpack build, but actual build success remains unverified in the current tool environment.

## 6. Blockers and notable considerations

1. The app is heavily configuration-driven and should be adapted in-place rather than rewritten.
2. The current default env example mentions Planet only; DUCA will require additional variables for forest data, backend API URLs, and any live provider integration.
3. Some remote GFW layer URLs and metadata sources may be live-only and may fail without credentials, network access, or API permits.
4. The repo is a template, not a finalized DUCA implementation; there is no plot/assessment engine or Python backend yet.
5. The business logic for EUDR-style deforestation screening is not present; this will need to be implemented in a separate, versioned risk-service layer.
6. Any live data integration must be explicitly documented as requiring credentials or an external service contract; do not claim operational forest verification without verification.

## 7. Proposed implementation sequence

Recommended implementation order for DUCA GEO:

1. Phase 1: repository assessment and documentation
   - Confirm architecture and config points.
   - Add the assessment document.
   - Establish baseline from repo metadata and known scripts.

2. Phase 2: DUCA map customization
   - Add DUCA-specific layer group configuration.
   - Add explicit labels, visibility defaults, and graceful handling for unavailable layers.
   - Keep the map engine unchanged while adding DUCA overlays.

3. Phase 3: forest data adapters
   - Build provider interface and adapter layer for GFW / forest data sources.
   - Normalize responses into a stable DUCA schema.
   - Add mocked tests that do not assume live credentials.

4. Phase 4: backend API service
   - Scaffold FastAPI + SQLAlchemy + PostGIS service under `duca-geo-api/`.
   - Implement validation, GeoJSON geometry logic, and persistence models.
   - Add endpoints for plots and assessments and a migration system.

5. Phase 5: methodology and screening logic
   - Implement cutoff-date logic (default 2020-12-31).
   - Track `NOT_ASSESSED`, `INSUFFICIENT_DATA`, `REVIEW_REQUIRED`, `NO_ALERT_DETECTED`, and `REVIEWED` states.
   - Keep reviewer decisions separate from machine-generated screening states.

6. Phase 6: frontend integration
   - Add typed API client.
   - Add panel components for plot details, forest evidence, and assessment history.
   - Display loading/error/empty states and avoid claiming compliance from lack of alerts.

7. Phase 7: automated tests and validation
   - Run frontend build/tests where available.
   - Run backend pytest suite.
   - Confirm a synthetic coffee plot workflow end-to-end.

## 8. Recommended integration approach

For the initial vertical slice, the best path is:

- Reuse the existing Mapbuilder config pattern and ArcGIS stack.
- Add a small set of DUCA layer configs instead of replacing the app engine.
- Create a separate backend service for plot assessment logic.
- Use a synthetic coffee plot GeoJSON fixture in tests.
- Keep the dataset and evidence logic modular so provider swaps do not require rewriting the risk engine.

This keeps the project aligned with the repo’s architecture and the prompt’s requirement for minimal, isolated DUCA customizations.

## 9. Summary

This repo is a suitable base for DUCA GEO because it already has:

- A layered geospatial frontend.
- A config-driven layer registry.
- ArcGIS map integration.
- Redux-driven app state.
- Existing tests and webpack build scripts.

The main work required is not a rewrite; it is the addition of DUCA-specific map layers, a forest-data adapter layer, a backend assessment service, and a versioned screening methodology. The repository is ready for the next implementation phase once these integrations are added and validated.

## 10. Baseline conclusion

Status: assessment complete; no DUCA implementation code changes yet beyond this assessment file.

Remaining required work before claiming production readiness:

- Verify build/test execution in a real Node environment.
- Add DUCA layer configuration and UI components.
- Add backend service and database schema.
- Implement risk methodology and evidence provenance.
- Validate all acceptance criteria with actual tests.
