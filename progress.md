# Tilt Hydrometer Platform — Progress Log

## 2026-02-14
- **ralph-deep-init Phase 1**: Architecture defined. 6 functional groups identified. tech-stack.md and README.md written.
- **ralph-deep-init Phase 2**: Expanded all 6 groups (Workspace-Scaffolding, Database-Schema, Server-API, Shared-Types, BLE-Client, Infrastructure) into detailed tasks with acceptance criteria.
- **ralph-deep-init Phase 3**: Assembled prd.json with 30 tasks. Cleaned up temp files.

## Working on: Cargo workspace with server, client, and shared crates
**Plan:**
- Create root Cargo.toml with `[workspace]` members = ["server", "client", "shared"]
- Use `cargo init server` (binary), `cargo init client` (binary), `cargo init --lib shared`
- Verify with `cargo build --workspace`
**Files:** Cargo.toml, server/Cargo.toml, server/src/main.rs, client/Cargo.toml, client/src/main.rs, shared/Cargo.toml, shared/src/lib.rs
**Result:** Success

## Working on: Shared crate dependencies and stub types
**Plan:**
- `cargo add` serde (derive), chrono (serde), uuid (v4+serde) to shared crate
- Write shared/src/lib.rs with stub TiltColor (8 variants), TiltReading struct, BrewStatus enum
- Verify with `cargo check -p shared`
**Files:** shared/Cargo.toml, shared/src/lib.rs
**Result:** Success

## Working on: Server crate dependencies
**Plan:**
- `cargo add` rocket (json+secrets), sea-orm (sqlx-postgres, runtime-tokio-rustls, macros), sea-orm-migration, rocket_cors, serde (derive), serde_json, chrono (serde), uuid (v4+serde), dotenvy, tracing, tracing-subscriber, anyhow to server crate
- `cargo add --path ../shared` to server crate
- Update server/src/main.rs to minimally reference shared so the dep is used
- Verify with `cargo check -p server`
**Files:** server/Cargo.toml, server/src/main.rs
**Result:** Success

## Working on: Client crate dependencies
**Plan:**
- `cargo add` btleplug, reqwest (json+rustls-tls), tokio (full), clap (derive), tracing, tracing-subscriber, serde (derive), serde_json, chrono to client crate
- `cargo add --path ../shared` to client crate
- Verify with `cargo check -p client`
**Files:** client/Cargo.toml
**Note:** Required `sudo apt-get install libdbus-1-dev` for btleplug's dbus dependency on Linux.
**Result:** Success

## Working on: Environment and config files
**Plan:**
- Create .env.example with DATABASE_URL, ROCKET_SECRET_KEY, ROCKET_PORT, RUST_LOG, DB_PASSWORD placeholders with comments
- .gitignore already exists from cycle 1 — verify it has target/, .env entries
- Verify with `cargo build --workspace`
**Files:** .env.example, .gitignore
**Result:** Success

## Working on: TiltColor enum with iBeacon UUID constants
**Plan:**
- The stub TiltColor already exists from cycle 2 with uuid() and from_uuid() methods
- Upgrade to use const UUIDs instead of parsing strings at runtime
- Add comprehensive unit tests: round-trip all 8 colors, unknown UUID returns None, serde serialization
- Verify with `cargo test -p shared`
**Files:** shared/src/lib.rs
**Result:** Success — 9 tests pass (round-trip, UUID correctness, uniqueness, serde)

## Working on: TiltReading and CreateReadingsBatch DTOs
**Plan:**
- Add TiltReading::new() constructor
- Add CreateReadingsBatch newtype wrapping Vec<TiltReading> with serde camelCase
- Add unit tests: TiltReading::new() constructs valid instance, serde round-trip, CreateReadingsBatch wraps vec
- Verify with `cargo test -p shared`
**Files:** shared/src/lib.rs
**Result:** Success — 15 tests pass (9 TiltColor + 6 TiltReading/CreateReadingsBatch)

## Working on: BrewStatus enum and Brew DTOs
**Plan:**
- BrewStatus stub exists — keep it, it already has Active/Completed/Archived
- Add CreateBrew, UpdateBrew, BrewResponse DTOs with serde camelCase
- BrewResponse includes optional latest_reading (TiltReading)
- Add unit tests: BrewStatus serde, CreateBrew required/optional fields, UpdateBrew all-optional, BrewResponse round-trip
- Verify with `cargo test -p shared`
**Files:** shared/src/lib.rs
**Result:** Success — 21 tests pass

## Working on: Hydrometer DTOs
**Plan:**
- Add CreateHydrometer (color required, name optional), UpdateHydrometer (all optional), HydrometerResponse (all fields) DTOs
- All with serde camelCase
- Add unit tests: CreateHydrometer required/optional, UpdateHydrometer all-optional, HydrometerResponse round-trip
- Verify with `cargo test -p shared`
**Files:** shared/src/lib.rs
**Result:** Success — 26 tests pass

## Working on: ReadingResponse and query parameter types
**Plan:**
- Add ReadingResponse DTO (id, brew_id?, hydrometer_id, color, temperature_f, gravity, rssi?, recorded_at, created_at)
- Add ReadingsQuery struct with filter fields (brew_id, hydrometer_id, since, until, limit) all Option with default limit 1000
- Add unit tests: ReadingResponse round-trip, ReadingsQuery all-optional, default limit handling
- Verify with `cargo test -p shared`
**Files:** shared/src/lib.rs
**Result:** Success — 31 tests pass. Shared-Types group complete!

## Working on: SeaORM migration crate setup
**Plan:**
- Create migration crate at server/migration/ using `cargo init --lib server/migration`
- Add sea-orm-migration dependency with runtime-tokio-rustls + sqlx-postgres features
- Write lib.rs with Migrator struct implementing MigratorTrait (empty migrations list initially)
- Add migration crate to workspace members and as server dependency
- Verify with `cargo check -p migration`
**Files:** server/migration/Cargo.toml, server/migration/src/lib.rs, Cargo.toml
**Result:** Success

## Working on: Create hydrometers table migration
**Plan:**
- Create migration file m20260215_000001_create_hydrometers.rs in server/migration/src/
- Table: hydrometers with id (UUID PK default gen_random_uuid()), color (VARCHAR NOT NULL UNIQUE), name (VARCHAR nullable), temp_offset_f (DOUBLE NOT NULL DEFAULT 0), gravity_offset (DOUBLE NOT NULL DEFAULT 0), created_at (TIMESTAMPTZ NOT NULL DEFAULT now())
- Register in Migrator's migrations() vec
- Verify with `cargo check -p migration` (can't run migrate up without a live DB)
**Files:** server/migration/src/m20260215_000001_create_hydrometers.rs, server/migration/src/lib.rs
**Result:** Success

## Working on: Create brews table migration
**Plan:**
- Create m20260215_000002_create_brews.rs migration
- Columns: id (UUID PK), name (VARCHAR NOT NULL), style (VARCHAR nullable), og/fg/target_fg/abv (DOUBLE nullable), status (VARCHAR NOT NULL DEFAULT 'Active'), start_date/end_date (TIMESTAMPTZ nullable), notes (TEXT nullable), hydrometer_id (UUID NOT NULL FK→hydrometers.id), created_at/updated_at (TIMESTAMPTZ NOT NULL DEFAULT now())
- Indexes on hydrometer_id and status
- Register in Migrator
- Verify with `cargo check -p migration`
**Files:** server/migration/src/m20260215_000002_create_brews.rs, server/migration/src/lib.rs
**Result:** Success

## Working on: Create readings table migration
**Plan:**
- Create m20260215_000003_create_readings.rs migration
- Columns: id (UUID PK), brew_id (UUID nullable FK→brews.id), hydrometer_id (UUID NOT NULL FK→hydrometers.id), temperature_f (DOUBLE NOT NULL), gravity (DOUBLE NOT NULL), rssi (SMALLINT nullable), recorded_at (TIMESTAMPTZ NOT NULL), created_at (TIMESTAMPTZ NOT NULL DEFAULT now())
- Indexes on brew_id, hydrometer_id, recorded_at
- Register in Migrator
- Verify with `cargo check -p migration`
**Files:** server/migration/src/m20260215_000003_create_readings.rs, server/migration/src/lib.rs
**Result:** Success

## Working on: Generate SeaORM entities from schema
**Plan:**
- Start Postgres via docker compose (create minimal docker-compose.yml for DB)
- Run `sea-orm-cli migrate up` against live DB
- Run `sea-orm-cli generate entity -o server/src/models/entities --with-serde both`
- Wire up server/src/models/mod.rs to re-export entities
- Verify with `cargo check -p server`
**Files:** docker-compose.yml, server/src/models/entities/*.rs, server/src/models/mod.rs
**Note:** Started Postgres via existing docker container, ran `sea-orm-cli migrate up`, then `sea-orm-cli generate entity --with-serde both`. Also added migration binary (main.rs + tokio + async-std deps).
**Result:** Success — Database-Schema group complete!

## Working on: Rocket application bootstrap with SeaORM database pool
**Plan:**
- Rewrite server/src/main.rs: async Rocket launch, dotenvy, tracing-subscriber init
- SeaORM DatabaseConnection as managed state via DATABASE_URL
- CORS fairing via rocket_cors (permissive for dev)
- Health check GET /api/v1/health returning JSON {"status":"ok"}
- JSON catchers for 404, 422, 500
- Verify with `cargo check -p server` (can't run without live DB in check mode)
**Files:** server/src/main.rs, server/src/routes/mod.rs
**Result:** Success

## Working on: Service layer with SeaORM repository pattern
**Plan:**
- Create server/src/services/mod.rs, hydrometer_service.rs, brew_service.rs, reading_service.rs
- Each service takes &DatabaseConnection, returns Result types
- Implement CRUD: find_all, find_by_id, create, update, delete for hydrometers/brews
- batch_create and find_filtered for readings
- From<Model> conversions to shared DTOs
- Verify with `cargo check -p server`
**Files:** server/src/services/*.rs, server/src/main.rs, shared/src/lib.rs (added TiltColor::from_str)
**Result:** Success

## Working on: CLI argument parsing with clap
**Plan:**
- Rewrite client/src/main.rs with clap-derived Args struct
- Args: --server-url (required), --scan-interval (default 15), --log-level (default "info"), --buffer-size (default 100)
- Init tracing-subscriber with log level, print startup banner
- Add env-filter feature to tracing-subscriber for client
- Verify with `cargo check -p client`
**Files:** client/src/main.rs
**Result:** Success — `--help` shows all 4 args with correct defaults

## Working on: Docker Compose with Postgres and server
**Plan:**
- Update docker-compose.yml: add server service (builds from Dockerfile, depends_on db, env vars)
- Named volume pgdata already exists
- Verify with `sudo docker compose config`
**Files:** docker-compose.yml
**Result:** Success — valid config with db + server services

## Working on: Multi-stage Dockerfile with cargo-chef
**Plan:**
- Create server/Dockerfile with 4 stages: chef, planner, builder, runtime
- Uses cargo-chef for dependency caching
- Builder copies full workspace (shared crate needed)
- Runtime is debian:bookworm-slim with libssl3 + ca-certificates
- Verify with `sudo docker build -f server/Dockerfile .`
**Files:** server/Dockerfile
**Result:** Success — 4-stage build completes, image built

## Working on: Environment configuration and .env.example
**Plan:**
- Verify .env.example has all 5 vars (DATABASE_URL, ROCKET_SECRET_KEY, ROCKET_PORT, RUST_LOG, DB_PASSWORD) with comments
- Verify ROCKET_SECRET_KEY has generation hint
- Verify dotenvy::dotenv() is called in server/src/main.rs
- All already in place from earlier cycles
**Files:** .env.example, server/src/main.rs
**Result:** Success — all criteria already met from cycles 5 and 16

## Working on: Hydrometers CRUD routes
**Plan:**
- Create server/src/routes/hydrometers.rs with GET (list), POST (create), GET/<id>, PUT/<id>, DELETE/<id>
- Routes use hydrometer_service functions, take &State<DatabaseConnection>
- Return proper status codes (200, 201, 204, 404, 422)
- Register routes in main.rs
- Verify with `cargo check -p server`
**Files:** server/src/routes/hydrometers.rs, server/src/routes/mod.rs, server/src/main.rs
**Result:** Success

## Working on: Brews CRUD routes
**Plan:**
- Create server/src/routes/brews.rs with GET (list + ?status filter), POST, GET/<id>, PUT/<id>, DELETE/<id>
- Routes use brew_service functions
- Register in routes/mod.rs and mount in main.rs
- Verify with `cargo check -p server`
**Files:** server/src/routes/brews.rs, server/src/routes/mod.rs, server/src/main.rs
**Result:** Success

## Working on: Readings ingestion and query routes
**Plan:**
- Create server/src/routes/readings.rs with POST (batch insert via CreateReadingsBatch) and GET (filtered query)
- POST auto-resolves hydrometer by color, links to active brew
- GET accepts query params: brew_id, hydrometer_id, since, until, limit
- Register in routes/mod.rs and mount in main.rs
- Verify with `cargo check -p server`
**Files:** server/src/routes/readings.rs, server/src/routes/mod.rs, server/src/main.rs
**Result:** Success — Server-API group complete!

## Working on: BLE scanner with btleplug and Tilt UUID filtering
**Plan:**
- Create client/src/scanner.rs with TiltScanner struct
- new() discovers BLE adapter, scan_once(duration) scans and returns Vec<TiltReading>
- Parse iBeacon from ManufacturerSpecificData (Apple 0x004C): UUID/major/minor/tx_power
- Filter by 8 known Tilt UUIDs via TiltColor::from_uuid()
- Unit tests for iBeacon parsing with known byte sequences
- Verify with `cargo check -p client`
**Files:** client/src/scanner.rs, client/src/main.rs
**Result:** Success — 7 iBeacon parsing tests pass

## Working on: HTTP uploader with batch POST
**Plan:**
- Create client/src/uploader.rs with Uploader struct (reqwest::Client + server_url)
- upload_batch(&[TiltReading]) POSTs JSON to {server_url}/api/v1/readings
- Typed UploadError enum: Network, ServerError(StatusCode), Deserialize
- Log request/response via tracing
- Verify with `cargo check -p client`
**Files:** client/src/uploader.rs, client/src/main.rs
**Result:** Success

## Working on: Retry logic with exponential backoff and local buffer
**Plan:**
- Create client/src/buffer.rs with ReadingBuffer (bounded VecDeque<TiltReading>)
- push_batch(): append readings, drop oldest if at capacity
- drain_all(): return all buffered readings and clear
- Backoff struct: initial 1s, max 60s, factor 2x, reset on success
- Unit tests: new, push_batch, drain_all, capacity overflow, backoff doubling, backoff reset
- Verify with `cargo test -p client`
**Files:** client/src/buffer.rs, client/src/main.rs
**Result:** Success — 13 tests pass (7 scanner + 6 buffer/backoff)

## Working on: Main scan-upload loop
**Plan:**
- Wire together scanner, uploader, buffer, backoff in main.rs
- Loop: scan_once → prepend buffered readings → upload_batch → on success reset backoff, on failure buffer + backoff → sleep
- Handle Ctrl+C via tokio::signal::ctrl_c for graceful shutdown
- Verify with `cargo check -p client`
**Files:** client/src/main.rs
**Result:** Success — BLE-Client group complete!

## Working on: Client systemd service unit file
**Plan:**
- Create client/tilt-client.service with [Unit], [Service], [Install] sections
- After=network-online.target bluetooth.target, Wants=network-online.target
- ExecStart=/usr/local/bin/tilt-client --server-url http://YOUR_SERVER:8000
- Restart=always, RestartSec=10, Environment for RUST_LOG
- Installation instructions in comments at top
- Verify by reading file for correctness
**Files:** client/tilt-client.service
**Result:** Success

## Working on: CI-ready test and build verification
**Plan:**
- Run `cargo test --workspace` — fix any failures
- Run `cargo fmt --all -- --check` — fix any formatting issues
- Run `cargo clippy --workspace -- -D warnings` — fix any clippy warnings
- Create Makefile with targets: test, check, fmt, clippy, migrate, run-server, run-client
- Verify all 3 CI commands pass
**Files:** Makefile, shared/src/lib.rs, server/src/services/hydrometer_service.rs, server/src/models/entities/prelude.rs, cargo fmt fixes
**Result:** Success — ALL 30/30 TASKS COMPLETE! 🎉

---
# Web Frontend Development

## Working on: Initialize Vite React TypeScript project
**Plan:**
- Create web/ via `npm create vite@latest web -- --template react-ts`
- Enable strict mode in tsconfig.json
- Install core deps: react-router-dom, @tanstack/react-query, recharts, date-fns, lucide-react
- Create web/.env with VITE_API_URL default
- Verify with `npm run dev`
**Files:** web/ (new directory)
**Result:** Success — upgraded Node.js to v22, Vite 8 beta, all deps installed, build passes

## Working on: Configure TailwindCSS v4 and shadcn/ui
**Plan:**
- Run `npx shadcn@latest init` (this installs Tailwind v4 automatically)
- Install shadcn components: button, card, input, label, badge, separator, dropdown-menu, dialog, table, sonner
- Verify with `npm run build`
**Files:** web/ (various config and component files)
**Result:** Success — Tailwind v4, shadcn/ui with 10 components, build passes

## Working on: TypeScript API types mirroring shared crate DTOs
**Plan:**
- Create web/src/types/index.ts with TS interfaces matching all Rust shared DTOs
- TiltColor union, TiltReading, CreateReadingsBatch, BrewStatus, CreateBrew, UpdateBrew, BrewResponse, CreateHydrometer, UpdateHydrometer, HydrometerResponse, ReadingResponse, ReadingsQuery
- All camelCase field names matching serde serialization
- Verify with `npm run build`
**Files:** web/src/types/index.ts
**Result:** Success — 12 interfaces/types, build passes

## Working on: API client and TanStack Query provider setup
**Plan:**
- Create web/src/lib/api.ts with apiGet, apiPost, apiPut, apiDelete using VITE_API_URL
- Create web/src/lib/query-client.ts with QueryClient config
- Wrap app in QueryClientProvider in main.tsx
- Create web/src/hooks/use-brews.ts, use-hydrometers.ts, use-readings.ts
- Verify with `npm run build`
**Files:** web/src/lib/api.ts, web/src/lib/query-client.ts, web/src/main.tsx, web/src/hooks/*.ts
**Result:** Success — API client + 3 hook files, build passes

## Working on: React Router configuration with route structure
**Plan:**
- Set up BrowserRouter in main.tsx
- Create placeholder pages: dashboard, brew-list, brew-detail, brew-new, hydrometer-list, not-found
- Define routes in App.tsx: /, /brews, /brews/new, /brews/:id, /hydrometers, * (404)
- Verify with `npm run build`
**Files:** web/src/main.tsx, web/src/App.tsx, web/src/pages/*.tsx
**Result:** Success — 6 page components, BrowserRouter + Routes, build passes

## Working on: App shell with sidebar navigation
**Plan:**
- Create web/src/components/layout/app-shell.tsx with sidebar + main content area
- Sidebar: app title, nav links (Dashboard, Brews, Hydrometers) with Lucide icons
- Active route highlighting via useLocation + NavLink
- Use React Router Outlet for nested route rendering
- Refactor App.tsx to use layout route wrapping all pages in AppShell
- Verify with `npm run build`
**Files:** web/src/components/layout/app-shell.tsx, web/src/App.tsx
**Result:** Success — sidebar with NavLink active highlighting, layout route with Outlet, build passes

## Working on: Responsive mobile navigation
**Plan:**
- Install shadcn Sheet component
- Update app-shell.tsx: add mobile header with hamburger (Menu icon) visible below md breakpoint
- Clicking hamburger opens Sheet with same nav links
- Sheet closes on link click via state management
- Verify with `npm run build`
**Files:** web/src/components/layout/app-shell.tsx
**Result:** Success — Sheet mobile nav, hamburger menu, closes on link click, build passes

## Working on: Page header component with breadcrumbs
**Plan:**
- Create web/src/components/layout/page-header.tsx with title, description, actions props
- Create web/src/components/layout/breadcrumbs.tsx using useLocation to parse path segments
- Update placeholder pages to use PageHeader
- Verify with `npm run build`
**Files:** web/src/components/layout/page-header.tsx, web/src/components/layout/breadcrumbs.tsx, web/src/pages/*.tsx
**Result:** Success — PageHeader + Breadcrumbs components, all pages updated, build passes

## Working on: Toast notification system
**Plan:**
- Add Sonner Toaster component to app-shell.tsx
- Create web/src/lib/toast.ts with success/error/info helpers wrapping Sonner
- Verify with `npm run build`
**Files:** web/src/components/layout/app-shell.tsx, web/src/lib/toast.ts
**Result:** Success — Sonner Toaster in app shell, toast helper with success/error/info, build passes

## Working on: Dashboard summary cards
**Plan:**
- Install shadcn skeleton component for loading states
- Rewrite web/src/pages/dashboard.tsx with 4 summary cards: Active Brews, Total Hydrometers, Latest Reading, Readings Today
- Use useBrews("Active"), useHydrometers(), useReadings() hooks
- Responsive grid: 2x2 desktop, stacked mobile
- Show Skeleton placeholders while loading
- Verify with `npm run build`
**Files:** web/src/pages/dashboard.tsx
**Result:** Success — 4 summary cards with Skeleton loading, responsive grid, build passes

## Working on: Active brews quick-view list
**Plan:**
- Add section below summary cards in dashboard.tsx
- Show compact list of active brews: name, color dot, current gravity, days active, link to detail
- Empty state with "Start a Brew" button linking to /brews/new
- Reuse useBrews("Active") data already fetched
- Verify with `npm run build`
**Files:** web/src/pages/dashboard.tsx
**Result:** Success — active brews list with color dots, gravity, days active, empty state, build passes

## Working on: Recent readings mini-chart
**Plan:**
- Create web/src/components/dashboard/recent-readings-chart.tsx using Recharts LineChart
- Fetch readings from last 24h via useReadings with since param
- Group by brew, render separate colored lines with legend
- X-axis: time (HH:mm via date-fns), Y-axis: specific gravity
- Show Skeleton while loading, empty state when no data
- Add to dashboard.tsx below active brews section
- Verify with `npm run build`
**Files:** web/src/components/dashboard/recent-readings-chart.tsx, web/src/pages/dashboard.tsx
**Result:** Success — Recharts LineChart with 24h readings, grouped by brew, color-coded lines, build passes

## Working on: Auto-refresh with configurable interval
**Plan:**
- Add refetchInterval: 30000 to dashboard query hooks
- Add last-refreshed timestamp display in PageHeader actions area
- Add manual RefreshCw button that invalidates all dashboard queries
- Animate RefreshCw icon during refetch with spin class
- Verify with `npm run build`
**Files:** web/src/pages/dashboard.tsx
**Result:** Success — 30s refetchInterval, RefreshCw spin animation, last-updated timestamp, build passes

## Working on: Brew list page with status filtering
**Plan:**
- Install shadcn tabs component
- Rewrite web/src/pages/brew-list.tsx with shadcn Table, status Tabs (All/Active/Completed/Archived)
- Columns: Name, Style, Hydrometer (color dot), Status (Badge), OG, Current SG, ABV, Start Date
- New Brew button in PageHeader actions
- Loading skeleton rows, empty state
- Clicking row navigates to /brews/:id
- Verify with `npm run build`
**Files:** web/src/pages/brew-list.tsx
**Result:** Success — Table with 8 columns, Tabs status filter, Badge color-coding, empty state, build passes

## Working on: Create brew form page
**Plan:**
- Install shadcn select and textarea components
- Rewrite web/src/pages/brew-new.tsx with form: Name (required), Style, Hydrometer (select from API), OG, Target FG, Notes
- POST to /brews on submit, redirect to /brews/:id on success
- Toast on success/error, inline validation
- Verify with `npm run build`
**Files:** web/src/pages/brew-new.tsx
**Result:** Success — form with 6 fields, Select/Textarea, validation, toast, redirect, build passes

## Working on: Brew detail page with readings summary
**Plan:**
- Rewrite web/src/pages/brew-detail.tsx to fetch brew by ID via useBrew hook
- Show brew name as title, status badge, style, stats grid (OG/FG/Target FG/ABV)
- Hydrometer info, start/end dates (date-fns), notes
- Action buttons: Edit, Complete, Archive, Delete (placeholders for dialog wiring)
- Readings section placeholder
- Verify with `npm run build`
**Files:** web/src/pages/brew-detail.tsx
**Result:** Success — stats grid, status badge, action buttons, dates, notes, readings placeholder, build passes

## Working on: Edit brew dialog with inline updates
**Plan:**
- Create web/src/components/brew/edit-brew-dialog.tsx as shadcn Dialog
- Pre-fill fields: Name, Style, OG, FG, Target FG, ABV, Notes, Status (select)
- PUT /brews/:id on submit, invalidate brew query, toast
- Wire Edit button in brew-detail.tsx to open dialog
- Verify with `npm run build`
**Files:** web/src/components/brew/edit-brew-dialog.tsx, web/src/pages/brew-detail.tsx
**Result:** Success — Dialog with 8 pre-filled fields, PUT on submit, toast, query invalidation, build passes

## Working on: Delete brew with confirmation
**Plan:**
- Install shadcn alert-dialog component
- Create web/src/components/brew/delete-brew-dialog.tsx as AlertDialog
- Show brew name, warning, confirm sends DELETE /brews/:id
- On success: toast + navigate to /brews, invalidate queries
- Wire Delete button in brew-detail.tsx to open dialog
- Verify with `npm run build`
**Files:** web/src/components/brew/delete-brew-dialog.tsx, web/src/pages/brew-detail.tsx
**Result:** Success — AlertDialog with warning, DELETE on confirm, toast, redirect to /brews, build passes

## Working on: Gravity and temperature line charts on brew detail
**Plan:**
- Create web/src/components/readings/readings-chart.tsx with dual-axis Recharts LineChart
- Left Y-axis: gravity (blue), Right Y-axis: temperature °F (orange)
- X-axis: time formatted with date-fns
- Time range buttons: 24h, 7d, 30d, All
- Tooltips with exact values
- Accept brewId prop, fetch via useReadings
- Replace readings placeholder in brew-detail.tsx
- Verify with `npm run build`
**Files:** web/src/components/readings/readings-chart.tsx, web/src/pages/brew-detail.tsx
**Result:** Success — dual-axis LineChart, time range buttons, tooltips, responsive, build passes

## Working on: Readings data table with pagination
**Plan:**
- Create web/src/components/readings/readings-table.tsx with shadcn Table
- Columns: Recorded At, Temperature °F, Gravity SG, RSSI
- Sort by recorded_at DESC, client-side pagination (25/page) with Prev/Next
- Reset pagination on brewId change
- Accept brewId prop, reuse useReadings
- Add below chart in brew-detail.tsx
- Verify with `npm run build`
**Files:** web/src/components/readings/readings-table.tsx, web/src/pages/brew-detail.tsx
**Result:** Success — Table with 4 columns, DESC sort, 25/page pagination, empty state, build passes

## Working on: Fermentation progress indicators
**Plan:**
- Create web/src/components/readings/fermentation-stats.tsx
- Calculate: Current Gravity, Apparent Attenuation %, Estimated ABV, Temperature Trend (↑/↓/→), Time Since Last Reading
- Accept brewId + og props, fetch readings via useReadings
- Display as row of stat cards above the chart on brew detail
- Verify with `npm run build`
**Files:** web/src/components/readings/fermentation-stats.tsx, web/src/pages/brew-detail.tsx
**Result:** Success — 5 stat cards (gravity, attenuation, ABV, temp trend, time since), build passes

## Working on: Tilt color utility with hex color mapping
**Plan:**
- Create web/src/lib/tilt-colors.ts with TILT_COLOR_MAP (hex, displayName, bgLight)
- Create web/src/components/ui/color-dot.tsx component
- Replace inline color maps in dashboard.tsx and brew-list.tsx with ColorDot and TILT_COLOR_MAP
- Verify with `npm run build`
**Files:** web/src/lib/tilt-colors.ts, web/src/components/ui/color-dot.tsx, web/src/pages/dashboard.tsx, web/src/pages/brew-list.tsx
**Result:** Success — TILT_COLOR_MAP with 8 colors, ColorDot component, replaced in dashboard + brew-list, build passes

## Working on: Gravity target line and completion detection
**Plan:**
- Add targetFg prop to ReadingsChart, render Recharts ReferenceLine (green dashed) when set
- Create alert banner in brew-detail.tsx when latest gravity <= targetFg
- Banner has "Complete Brew" button that PUTs status to Completed
- Verify with `npm run build`
**Files:** web/src/components/readings/readings-chart.tsx, web/src/pages/brew-detail.tsx
**Result:** Success — ReferenceLine dashed green, alert banner with Complete button, build passes

## Working on: Hydrometer list page with status indicators
**Plan:**
- Rewrite web/src/pages/hydrometer-list.tsx with responsive card grid
- Each card: large ColorDot, color name, alias, calibration offsets, registration date, active brew link
- Register Hydrometer button in page header (placeholder for dialog wiring)
- Loading skeletons, empty state
- Verify with `npm run build`
**Files:** web/src/pages/hydrometer-list.tsx
**Result:** Success — responsive card grid, ColorDot, calibration offsets, active brew link, empty state, build passes

## Working on: Register hydrometer dialog
**Plan:**
- Create web/src/components/hydrometer/register-hydrometer-dialog.tsx as shadcn Dialog
- Color select with ColorDot previews, filtered to exclude already-registered colors
- Name/Alias optional text input
- POST /hydrometers on submit, toast, invalidate queries
- Wire into hydrometer-list.tsx Register button
- Verify with `npm run build`
**Files:** web/src/components/hydrometer/register-hydrometer-dialog.tsx, web/src/pages/hydrometer-list.tsx
**Result:** Success — Dialog with color select (filtered), ColorDot previews, optional name, toast, build passes

## Working on: Edit hydrometer dialog with calibration offsets
**Plan:**
- Create web/src/components/hydrometer/edit-hydrometer-dialog.tsx as shadcn Dialog
- Pre-filled fields: Name/Alias, Temp Offset °F (step 0.1), Gravity Offset (step 0.001)
- Help text for each offset field
- PUT /hydrometers/:id on submit, toast, invalidate queries
- Add Edit button to hydrometer cards in hydrometer-list.tsx
- Verify with `npm run build`
**Files:** web/src/components/hydrometer/edit-hydrometer-dialog.tsx, web/src/pages/hydrometer-list.tsx
**Result:** Success — Dialog with name, temp offset, gravity offset, help text, Edit button on cards, build passes
