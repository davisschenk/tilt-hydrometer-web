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
