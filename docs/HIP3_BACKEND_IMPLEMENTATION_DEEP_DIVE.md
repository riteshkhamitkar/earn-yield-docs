# HIP‑3 Console Backend — Implementation Deep Dive (Single-File Reference)

This document is the **all-in-one** technical reference for the HIP‑PlayGround Rust backend: what was built, how it maps to your Markdown sources ([rustrules.md](../rustrules.md), [Product-Plan.md](../Product-Plan.md), [hip3-console-external-docs.md](hip3-console-external-docs.md)), how to read the PostgreSQL rows you are seeing, and how the code is organized. It is written to be readable top-to-bottom or used as a lookup by section.

---

## Table of contents

1. [Document purpose and sources of truth](#1-document-purpose-and-sources-of-truth)
2. [Executive summary — what you have today](#2-executive-summary--what-you-have-today)
3. [Product-Plan.md — implemented vs not yet](#3-product-planmd--implemented-vs-not-yet)
4. [rustrules.md — how the codebase follows it](#4-rustrulesmd--how-the-codebase-follows-it)
5. [hip3-console-external-docs.md — what we aligned to](#5-hip3-console-external-docsmd--what-we-aligned-to)
6. [Official Hyperliquid GitBook — verification](#6-official-hyperliquid-gitbook--verification)
7. [End-to-end architecture](#7-end-to-end-architecture)
8. [Repository and workspace layout](#8-repository-and-workspace-layout)
9. [Binary entrypoint: `services/hip3-console-api`](#9-binary-entrypoint-serviceship3-console-api)
10. [Configuration: `crates/config`](#10-configuration-cratesconfig)
11. [Domain layer: `crates/domain`](#11-domain-layer-cratesdomain)
12. [Infrastructure: `crates/infrastructure`](#12-infrastructure-cratesinfrastructure)
13. [HTTP API: `crates/api`](#13-http-api-cratesapi)
14. [Telemetry: `crates/telemetry`](#14-telemetry-cratestelemetry)
15. [Database schema — every column explained](#15-database-schema--every-column-explained)
16. [Reading your `perp_dex` rows (what the paste means)](#16-reading-your-perp_dex-rows-what-the-paste-means)
17. [Reading your `perp_market` rows (what the paste means)](#17-reading-your-perp_market-rows-what-the-paste-means)
18. [Asset IDs in your data — worked examples](#18-asset-ids-in-your-data--worked-examples)
19. [HTTP API reference (MVP)](#19-http-api-reference-mvp)
20. [Operational behavior: indexer timing and logs](#20-operational-behavior-indexer-timing-and-logs)
21. [Security posture (MVP)](#21-security-posture-mvp)
22. [Known limitations and follow-ups](#22-known-limitations-and-follow-ups)
23. [Real-life use cases](#23-real-life-use-cases)
24. [Appendix A — Environment variables](#appendix-a--environment-variables)
25. [Appendix B — SQL quick reference](#appendix-b--sql-quick-reference)
26. [Appendix C — Glossary](#appendix-c--glossary)
27. [Appendix D — File-by-file inventory](#appendix-d--file-by-file-inventory)

---

## 1. Document purpose and sources of truth

You asked for:

- A **simple** but **deep** explanation of what is implemented.
- Confirmation that the implementation followed **rustrules.md**, **Product-Plan.md**, and **docs/hip3-console-external-docs.md**.
- Help interpreting **what you see in PostgreSQL** (your large paste of `perp_dex` and `perp_market` rows).
- One **single** file with a lot of detail.

**Sources of truth used during implementation (in priority order for API behavior):**

1. **Hyperliquid official documentation (GitBook)** — ultimately authoritative for request types, field names, and asset-id rules.
2. **[hip3-console-external-docs.md](hip3-console-external-docs.md)** — consolidated integration guide (URLs, env vars, mental model, builder codes, oracles, etc.).
3. **[Product-Plan.md](../Product-Plan.md)** — product intent, MVP scope, phased roadmap; not a substitute for API specs.
4. **[rustrules.md](../rustrules.md)** — Rust workspace structure, error handling conventions, API design, security defaults, observability expectations.

This backend **does not** embed the full text of those files; it **implements** a slice of the product plan and **follows** the engineering rules. Where GitBook and the external doc differ, the implementation prefers **GitBook** for Info API naming (for example: using `perpDexs` discovery).

---

## 2. Executive summary — what you have today

You have a **Rust workspace** that runs a single service binary (`hip3-console-api`) which:

1. Connects to **PostgreSQL** and applies **SQLx migrations** for two tables: `perp_dex` and `perp_market`.
2. Calls Hyperliquid’s **read-only Info API** over HTTPS:
   - `{"type":"perpDexs"}` to list perpetual DEX metadata (including builder / HIP‑3 style DEX rows).
   - `{"type":"metaAndAssetCtxs", "dex": ...}` to fetch, per DEX, the **universe** (market definitions) and **live asset contexts** (mark, funding, OI, etc.).
3. **Upserts** that data into Postgres so your DB always has a **catalog of DEXes** and a **latest snapshot per market**.
4. Exposes a **read-only HTTP API** (Axum) for health checks and JSON listing of DEXes and markets.

**What this enables (use case):**

- A “HIP‑3 explorer / console” style product can **read** unified market metadata from **your** database instead of hammering Hyperliquid directly for every page view.
- You can build dashboards that join **DEX identity** (`perp_dex`) with **per-market metrics** (`perp_market`).

**What this does *not* do yet (explicitly out of MVP scope):**

- No Exchange/signing flows, no user trading, no builder-code order routing.
- No WebSocket ingest server for browsers (the stack is prepared at the docs/env level only).
- No dedicated time-series history table (the `/history` endpoint is a stub).
- No portfolio / wallet linking, no JWT auth, no Redis cache (not wired in code).
- No deployer console for `haltTrading` payload building.

---

## 3. Product-Plan.md — implemented vs not yet

The product plan describes a large system. This section maps **Phase 1 / MVP** bullets to the repo.

### 3.1 MVP items from Product-Plan — status

From Product-Plan “MVP scope and iterative roadmap”:

1. **“HIP‑3 market indexer and Postgres schema”** — **Implemented**
   - Indexer: `hip3_domain::run_index_cycle` orchestrates Info calls and DB writes.
   - Schema: `migrations/20260409120000_init.sql`.

2. **“Read-only public API”** — **Implemented (subset)**
   - Implemented: `/health`, `/ready`, `/api/v1/dexs`, `/api/v1/markets`, `/api/v1/markets/{asset_id}`.
   - Stub: `/api/v1/markets/{asset_id}/history` returns HTTP 501.

### 3.2 Product-Plan “next phases” — status (not implemented)

These are described in the plan but **not built** in this codebase yet:

- **User portfolio connections** (read user positions / summaries from Info).
- **Leaderboards** and social features.
- **Deployer console** (risk tooling, slashing alerts, simulations).
- **Exchange** integration and **builder code** revenue instrumentation.
- **Pyth HIP‑3-as-a-Service** integration (oracle path in ops).
- **Redis** hot caching layer.
- **Frontend** (React/Svelte/etc.) — not in repo.

### 3.3 Product-Plan architecture themes — partial alignment

The plan discusses modular crates, hexagonal architecture, high RPS targets, CQRS, etc.

**Aligned now:**

- Cargo **workspace** with separate crates and a thin binary.
- **Ports** in `domain` (`HyperliquidInfo`, `MarketStore`) and adapters in `infrastructure`.
- **Typed configuration** validated at startup (`AppConfig::from_env`).
- **Tracing** for HTTP and structured service logs.
- **sqlx** migrations embedded at compile time and run at startup.

**Not fully realized yet (by design for MVP):**

- **100k+ RPS**: not load-tested; MVP uses straightforward queries and per-row upserts in a transaction loop (see limitations).
- **Redis**: not present.
- **CQRS split**: single Postgres schema serves reads and writes.

---

## 4. rustrules.md — how the codebase follows it

### 4.1 Workspace structure

`rustrules.md` recommends:

- `api`, `domain`, `infrastructure`, `telemetry`, `config`, binaries under `services/`.

**Repo matches** that pattern:

- `crates/api` — Axum routers/handlers only; no SQL in handlers.
- `crates/domain` — models, ports (`async_trait` traits), indexer orchestration, `thiserror` errors.
- `crates/infrastructure` — `reqwest` client + Postgres repository.
- `crates/telemetry` — tracing initialization.
- `crates/config` — env configuration.
- `services/hip3-console-api` — `main.rs` wiring.

### 4.2 Error handling conventions

- `domain` uses **`thiserror`** (`DomainError`).
- `main` uses **`anyhow`** for startup error context (`Context` trait).
- Handlers map `DomainError` to HTTP errors in `api` (`ApiError`).

### 4.3 API design conventions

- APIs are versioned under **`/api/v1/...`**.
- JSON responses often use an **envelope** pattern in `api` (`ApiEnvelope`) where practical.
- Pagination is **cursor-based** for markets (`cursor` + `limit`).

### 4.4 Security conventions (MVP level)

- **No secrets in logs** (structured tracing; don’t log API keys).
- **Security headers**: `X-Content-Type-Options: nosniff` via `tower-http`.
- **Request body limit** configured (1 MiB) even for GET-centric APIs (future-proofing).
- **CORS** is configurable; permissive mode is dev-oriented.

### 4.5 Performance notes captured in code

- Indexer module documents intent: **`O(num_dex_requests)`** network-wise per cycle (one `perpDexs` call + one `metaAndAssetCtxs` per DEX job), and **`O(markets)`** DB writes.

---

## 5. hip3-console-external-docs.md — what we aligned to

The external doc is long; these are the parts the MVP actually **uses in code**:

### 5.1 Network and endpoints

- **`HL_INFO_BASE_URL`** should point at the Info POST URL (default in `.env.example`: `https://api.hyperliquid.xyz/info`).
- Optional **`HL_INFO_API_KEY`** for private mirrors (implementation sends `Authorization: Bearer ...`; providers may differ).

### 5.2 Info API mental model

- POST **`/info`** with JSON body `{ "type": "... " }`.

### 5.3 HIP‑3-specific ideas encoded in indexer

- **DEX discovery** via `perpDexs`.
- **Per-DEX** `metaAndAssetCtxs`.
- **Asset ID formula** for builder markets (see `compute_asset_id`).

### 5.4 Items documented but not implemented in code yet

Examples: Exchange endpoint, WebSocket subscription budget, Pyth pusher model, JWT auth parameters, Redis TTL keys — these appear in the doc as **future** or operational guidance.

---

## 6. Official Hyperliquid GitBook — verification

Implementation-critical confirmations:

- **`perpDexs`** exists as an Info `type` and returns an array with `null` placeholders and objects for builder DEX rows.
- **`metaAndAssetCtxs`** returns a tuple: meta + asset contexts, and supports a `dex` field.
- **Asset ID** encoding for builder-deployed perps matches the documented formula:

\[
asset = 100000 + perp\_dex\_index \times 10000 + index\_in\_meta
\]

---

## 7. End-to-end architecture

### 7.1 Data flow (one indexer cycle)

1. Service starts, migrates DB, constructs `HttpHyperliquidInfo` + `PgMarketStore`.
2. `run_index_cycle` executes:
   - Fetch `perpDexs`.
   - Build **jobs**: native slot (`null` at array index 0) + each non-null DEX object.
   - Upsert `perp_dex` rows for each job with a `PerpDexRow`.
   - For each job:
     - Call `metaAndAssetCtxs` with `dex` parameter appropriate for native vs named DEX.
     - Parse universe entries and contexts.
     - Compute `asset_id` per entry.
     - Upsert `perp_market` rows.
3. A background task repeats the cycle every `INDEXER_POLL_INTERVAL_SECS` (with `sleep`, after initial run).

### 7.2 Read path (HTTP)

1. Client hits Axum router.
2. Handler calls `MarketStore` trait methods (`list_dexs`, `list_markets`, `get_market`, `ping`).
3. Postgres returns rows mapped into domain models and serialized as JSON.

---

## 8. Repository and workspace layout

Workspace root: [Cargo.toml](../Cargo.toml) defines members:

- `crates/config`
- `crates/domain`
- `crates/infrastructure`
- `crates/telemetry`
- `crates/api`
- `services/hip3-console-api`

Migrations live at: [migrations/](../migrations/)

Operational docs:

- [docs/RUNBOOK.md](RUNBOOK.md)
- [.env.example](../.env.example)

---

## 9. Binary entrypoint: `services/hip3-console-api`

File: [services/hip3-console-api/src/main.rs](../services/hip3-console-api/src/main.rs)

Responsibilities:

- Load `.env` via `dotenvy` (optional file; env vars can be set by the shell instead).
- `AppConfig::from_env()` — fail fast if required env is missing/invalid.
- Initialize tracing (`hip3_telemetry::init_tracing`).
- Connect Postgres via `PgPoolOptions` (max_connections = 10 in code).
- Run `sqlx::migrate!("../../migrations")` **at startup** against `DATABASE_URL`.
- Build adapters:
  - `Arc<dyn HyperliquidInfo + Send + Sync>`
  - `Arc<dyn MarketStore + Send + Sync>`
- Run **initial** `run_index_cycle` (logs error if fails; still serves HTTP).
- Spawn **background** indexing loop.
- Build Axum router + middleware layers:
  - `TraceLayer`
  - `RequestBodyLimitLayer(1 MiB)`
  - `SetResponseHeaderLayer` for `nosniff`
  - optional `CorsLayer::permissive()` if `CORS_PERMISSIVE=true`
- `axum::serve` listens on `BIND_ADDR`.

---

## 10. Configuration: `crates/config`

File: [crates/config/src/lib.rs](../crates/config/src/lib.rs)

`AppConfig` fields (conceptual groups):

- **Postgres**: `database_url` (required)
- **Info client**: `hl_info_base_url` (required), optional `hl_info_api_key`, timeouts, inflight concurrency, retry policy
- **Indexer**: `indexer_poll_interval_secs`, optional whitelist string `hl_info_dex_whitelist`
- **Server**: `bind_addr`, `cors_permissive`, `log_json`
- **Informational**: `hl_network`, `hl_ws_url` (WS not used in MVP indexer)

### Dex whitelist semantics

- If `HL_INFO_DEX_WHITELIST` is empty → **no filtering** (index all DEX jobs returned by `perpDexs` parsing).
- If non-empty → it is split on commas into a `HashSet<String>` **without dropping empty segments**, so a leading comma can include the native DEX whose `dex_name` is `""`.

---

## 11. Domain layer: `crates/domain`

### 11.1 Public exports

See [crates/domain/src/lib.rs](../crates/domain/src/lib.rs):

- `DomainError`
- `run_index_cycle`
- `MarketQuery`, `PaginatedMarkets`
- `PerpDexRow`, `PerpMarketRow`, `UniverseEntry`, `AssetCtxSnapshot`
- traits: `HyperliquidInfo`, `MarketStore`

### 11.2 Ports (interfaces)

File: [crates/domain/src/ports.rs](../crates/domain/src/ports.rs)

- `HyperliquidInfo`:
  - `fetch_perp_dexs_json()` → `Vec<serde_json::Value>`
  - `fetch_meta_and_asset_ctxs(dex: Option<&str>)` → `(Vec<UniverseEntry>, Vec<AssetCtxSnapshot>)`

- `MarketStore`:
  - `upsert_dex`, `upsert_markets`
  - `list_dexs`, `list_markets`, `get_market`
  - `ping` (readiness)

### 11.3 Indexer orchestration

File: [crates/domain/src/indexer.rs](../crates/domain/src/indexer.rs)

Key rules:

- `perp_dex_index` is the **enumeration index** `i` in the `perpDexs` array (as `i32`).
- `null` at index 0 → native DEX:
  - `dex_name == ""`
  - fetch `metaAndAssetCtxs` with `dex_param = None` (omits `dex` field in JSON)
  - `is_native_slot = (dex_key.is_empty() && perp_dex_index == 0)`
- Named DEX objects:
  - `dex_name` comes from JSON `name`
  - fetch with `Some("flx")` etc.

`build_markets` pairs:

- `universe[i]` with `ctxs.get(i)` (default empty snapshot if missing)

### 11.4 Asset ID function

File: [crates/domain/src/models.rs](../crates/domain/src/models.rs)

- Native first DEX slot: asset id equals **universe index** (e.g. BTC = 0).
- Otherwise: `100_000 + perp_dex_index * 10_000 + index_in_meta`.

Unit tests validate GitBook example (`110_000`).

---

## 12. Infrastructure: `crates/infrastructure`

### 12.1 Hyperliquid Info HTTP client

File: [crates/infrastructure/src/hl_client.rs](../crates/infrastructure/src/hl_client.rs)

Behavior:

- Uses `reqwest` with **rustls**TLS.
- Concurrency limit with `tokio::sync::Semaphore` (`HL_INFO_MAX_INFLIGHT`).
- Retries on selected HTTP status codes with exponential backoff.

Parsing highlights:

- **`perpDexs`**: parses top-level JSON as `Vec<Value>`.
- **`metaAndAssetCtxs`**: expects top-level JSON array length ≥ 2:
  - `[0]` meta object containing `universe` array
  - `[1]` array of asset contexts
- Each universe coin object becomes `UniverseEntry { name, sz_decimals, max_leverage, flags: <full coin JSON> }`.
- Asset context maps Hyperliquid camelCase fields into `AssetCtxSnapshot`, including splitting `impactPxs` into bid/ask strings.

### 12.2 Postgres MarketStore

File: [crates/infrastructure/src/pg_store.rs](../crates/infrastructure/src/pg_store.rs)

**`upsert_markets`**:

- Opens a transaction.
- For each market row, runs an `INSERT ... ON CONFLICT (asset_id) DO UPDATE`.

**`list_markets`**:

- Fetches `limit + 1` rows to detect if a next page exists.
- Cursor is **`asset_id`** lexicographically continued with `asset_id > cursor`.

**Note on performance**:

- MVP uses **one statement per market** inside a transaction. This is simple and correct but can be slow over high-latency DB links (you observed sqlx “slow statement” warnings).

---

## 13. HTTP API: `crates/api`

Files:

- [crates/api/src/router.rs](../crates/api/src/router.rs)
- [crates/api/src/state.rs](../crates/api/src/state.rs)
- [crates/api/src/error.rs](../crates/api/src/error.rs)

Routes:

- `GET /health` → `{"status":"ok"}` (simple JSON)
- `GET /ready` → DB ping via `store.ping()`
- `GET /api/v1/dexs` → list `PerpDexRow`
- `GET /api/v1/markets` → query params: `dex`, `search`, `cursor`, `limit`
- `GET /api/v1/markets/{asset_id}`
- `GET /api/v1/markets/{asset_id}/history` → 501 stub

**Routing note**: the more specific `/history` route is registered before `/{asset_id}` to avoid pattern shadowing issues.

---

## 14. Telemetry: `crates/telemetry`

File: [crates/telemetry/src/lib.rs](../crates/telemetry/src/lib.rs)

- Sets global subscriber.
- Uses `RUST_LOG` via `tracing_subscriber::EnvFilter`.
- If `LOG_JSON=true`, logs JSON; otherwise pretty/human-oriented formatting.

---

## 15. Database schema — every column explained

Migration: [migrations/20260409120000_init.sql](../migrations/20260409120000_init.sql)

### 15.1 Table `perp_dex`

**Purpose:** one row per **perpetual DEX** discovered from `perpDexs` (including the native DEX row with empty name).

| Column | Type | Meaning |
| --- | --- | --- |
| `dex_name` | TEXT PK | Short DEX key, e.g. `flx`, `xyz`, or **empty string** for native |
| `full_name` | TEXT NULL | Human-facing name from JSON `fullName` when present |
| `deployer` | TEXT NULL | Deployer address from JSON `deployer` when present |
| `perp_dex_index` | INT | Index `i` in the `perpDexs` array; used for HIP‑3 asset id math |
| `raw` | JSONB | Full JSON object from `perpDexs` for forward compatibility |
| `updated_at` | TIMESTAMPTZ | Last upsert time (DB default `NOW()` on insert; updated in upsert SQL) |

### 15.2 Table `perp_market`

**Purpose:** one row per **coin in a DEX universe** with latest “ctx” metrics flattened for easy querying.

| Column | Type | Meaning |
| --- | --- | --- |
| `asset_id` | BIGINT PK | Hyperliquid asset integer used in trading APIs |
| `dex_name` | TEXT | Same as `perp_dex.dex_name` (can be empty string for native) |
| `coin_name` | TEXT | Market symbol in HL universe, e.g. `BTC` or `xyz:TSLA` |
| `universe_index` | INT | Index `i` in `meta.universe` array for that DEX |
| `perp_dex_index` | INT | Copied from DEX row; participates in asset id formula |
| `sz_decimals` | INT NULL | Contract size decimals (from universe coin) |
| `max_leverage` | INT NULL | Max leverage (from universe coin) |
| `universe_flags` | JSONB | Full universe coin JSON (includes `marginMode`, `isDelisted`, etc.) |
| `day_ntl_vlm` | TEXT NULL | From ctx `dayNtlVlm` |
| `funding` | TEXT NULL | From ctx `funding` |
| `impact_bid` | TEXT NULL | First element of ctx `impactPxs` |
| `impact_ask` | TEXT NULL | Second element of ctx `impactPxs` |
| `mark_px` | TEXT NULL | From ctx `markPx` |
| `mid_px` | TEXT NULL | From ctx `midPx` |
| `open_interest` | TEXT NULL | From ctx `openInterest` |
| `oracle_px` | TEXT NULL | From ctx `oraclePx` |
| `premium` | TEXT NULL | From ctx `premium` |
| `prev_day_px` | TEXT NULL | From ctx `prevDayPx` |
| `updated_at` | TIMESTAMPTZ | When this snapshot was written by indexer |

Constraints/indexes:

- Unique `(dex_name, coin_name)` to prevent duplicates for the same market within a DEX namespace.

---

## 16. Reading your `perp_dex` rows (what the paste means)

Your paste lists multiple DEX rows. Interpretation pattern:

### 16.1 Native DEX row (`dex_name` empty)

If you see a row like:

- `dex_name` = empty
- `perp_dex_index` = `0`
- `raw` might be JSON `null` (because HL returns `null` at index 0 in `perpDexs`)

That row means: **the default / first Hyperliquid perpetual universe** (commonly what people call the main perps list). Its markets appear as `coin_name` like `BTC`, `ETH`, not `xyz:...`.

### 16.2 Builder / HIP‑3 style DEX rows (`dex_name` short)

Examples from your paste:

- `flx` — “Felix Exchange”
- `xyz` — “XYZ”
- `km` — “Markets by Kinetiq”
- `vntl`, `hyna`, `cash`, `abcd`, `para`

These come from **non-null objects** in the `perpDexs` array. The **`raw` JSONB** column stores the entire HL object:

- **`subDeployers`**: addresses authorized for specific deployer actions (oracle, halt trading, margin tables, etc.) — informational for your explorer product; not executed by this service.
- **`assetToStreamingOiCap`**, **`assetToFundingMultiplier`**, **`assetToFundingInterestRate`**: DEX-level economics/caps from HL; informational.
- **`oracleUpdater`**, **`feeRecipient`**: operational addresses / config.

**Important:** your product DB is a **cache / mirror** for exploration. It does **not** validate on-chain state and does **not** submit transactions.

---

## 17. Reading your `perp_market` rows (what the paste means)

Each `perp_market` row is one tradeable perp **within** a specific DEX universe.

### 17.1 Native markets

Example pattern:

- `dex_name` empty
- `coin_name` is a short symbol: `BTC`, `ETH`, `ATOM`
- `asset_id` matches small integers: `0`, `1`, `2`, ...

These correspond to HL docs: native perps use asset id equals universe index.

### 17.2 Builder markets (names like `dex:COIN`)

Examples:

- `coin_name = xyz:TSLA` with `dex_name = xyz`
- `coin_name = flx:GOLD` with `dex_name = flx`

**Why names look like this:** HL uses `{dex}:{coin}` naming for builder-deployed perps.

### 17.3 Why some rows show zeros / empty metrics

You will see rows where many numeric snapshot fields are `0.0` or empty and flags show `"isDelisted": true`.

That typically means: the market still exists in metadata universe, but trading/metrics may be inactive or settled depending on HL’s rules—your indexer still stores the latest ctx it receives.

---

## 18. Asset IDs in your data — worked examples

### 18.1 Native `BTC` as `asset_id = 0`

If `perp_dex_index = 0` and this is the native slot and BTC is index 0 in the universe:

- `compute_asset_id(0, 0, true) = 0`

Matches your rows.

### 18.2 `xyz` DEX `coin_name = xyz:XYZ100` as `asset_id = 110000`

If `xyz` is at `perp_dex_index = 1` and `xyz:XYZ100` is universe index `0`:

- `100_000 + 1 * 10_000 + 0 = 110_000`

Your paste includes `110000` rows — consistent with this scheme.

### 18.3 `flx` at `perp_dex_index = 2`, local universe index 12 → `120012`

Formula:

- `100_000 + 2 * 10_000 + 12 = 120_012`

Your paste shows `120012` for `flx:PLATINUM` with `universe_index = 12` — consistent.

### 18.4 `km` at index 5 → `150001` style

For `perp_dex_index = 5` and `universe_index = 1`:

- `100_000 + 5*10_000 + 1 = 150_001`

Your paste shows `150001` for `km:BABA` — consistent.

---

## 19. HTTP API reference (MVP)

### 19.1 `GET /health`

Returns service liveness. Does not check Postgres.

### 19.2 `GET /ready`

Runs `SELECT 1` via sqlx pool. If DB is down, returns 503-style payload from handler logic (`SERVICE_UNAVAILABLE`).

### 19.3 `GET /api/v1/dexs`

Returns JSON envelope containing `data: PerpDexRow[]`.

Each dex row includes **`raw`** which can be large.

### 19.4 `GET /api/v1/markets`

Query parameters:

- `dex` — filter exact `perp_market.dex_name`
- `search` — `ILIKE` substring match on `coin_name`
- `cursor` — stringified `asset_id` for keyset pagination
- `limit` — clamped between 1 and 200

Returns:

- `data` includes `items` and `next_cursor` inside the paged object (envelope may also include meta)

### 19.5 `GET /api/v1/markets/{asset_id}`

Path parameter is `i64`. Missing market → 404 JSON error body.

### 19.6 `GET /api/v1/markets/{asset_id}/history`

Returns **501** until a history table exists.

---

## 20. Operational behavior: indexer timing and logs

### 20.1 Initial indexing

On startup, service attempts one full `run_index_cycle` before HTTP accept (errors are logged, not fatal).

### 20.2 Periodic indexing

Background loop sleeps `INDEXER_POLL_INTERVAL_SECS` between cycles (config enforces a minimum of 5 seconds in `AppConfig::indexer_interval`).

### 20.3 Why logs show “slow acquire / slow statement”

Common causes:

- Remote Postgres latency
- Many sequential upserts per market batch

This is **not** proof of incorrect behavior; it’s a performance characteristic.

---

## 21. Security posture (MVP)

 strengths:

- Secrets expected via env, not hardcoded.
- Basic hardening headers and body limits.
- Read-only product surface reduces attack surface versus Exchange-enabled services.

 gaps (expected for MVP):

- No authentication/authorization on read endpoints.
- No per-IP rate limit at Axum layer (can be placed behind reverse proxy).
- Private Info provider auth header format may require customization.

---

## 22. Known limitations and follow-ups

1. **Batch upserts / COPY** for faster ingestion.
2. **History** table + retention policy + charts API.
3. **WebSocket** ingest for high-frequency correctness (still backed by periodic REST snapshots).
4. **Portfolio** endpoints and account summary ingesters.
5. **Deployer** tooling and risk engines per Product-Plan phase 3.
6. **`/dexs` payload size** may be huge due to `raw` JSON — consider DTO projection endpoints.

---

## 23. Real-life use cases

This section translates the **implemented backend** into **concrete situations**: who benefits, what they do, and which **database fields** or **API endpoints** matter. It is not marketing copy; it ties directly to what exists in the repo today (indexer + Postgres + read API) and what Product-Plan phases will add later.

### 23.1 Framing: what “MVP” unlocks in the real world

Today the service is a **reliable local mirror** of Hyperliquid’s **DEX catalog** and **latest per-market context** (marks, funding, OI, delist flags, etc.). In production you typically place it **behind your UI or partner APIs** so that:

1. **User-facing apps** do not all hit Hyperliquid Info directly (rate limits, latency variance, less control over shaping responses).
2. **Your team** can join DEX identity with market rows in SQL for analytics, alerts, and QA.
3. **Asset IDs** stay consistent with Hyperliquid’s encoding (`100000 + perp_dex_index * 10000 + universe_index` for builder DEX markets), which is required for trading and charting integrations downstream.

Anything involving **signed trading**, **wallet OAuth**, or **deployer transaction building** is **not** in this MVP; those are natural “phase 2/3” extensions on top of the same catalog.

### 23.2 Persona: retail trader using a “HIP‑3 explorer”

**Situation.** A trader hears about stock-like perps on builder DEXes (for example symbols such as `xyz:NVDA` or `flx:TSLA`) and opens your website. They want one screen that answers: *Which DEX lists NVIDIA? What are mark and funding right now? Is the market delisted?*

**What they touch (conceptually).** Your frontend, which should call **your** API, not raw Hyperliquid from the browser on every sort/filter.

**Backend capabilities used.**

- `GET /api/v1/dexs` — list deployers / DEX short names (`name` column / `raw` JSON) and rich config such as `assetToStreamingOiCap`, `subDeployers`, `feeRecipient`, etc., when you expose `raw`.
- `GET /api/v1/markets?dex_name=xyz&limit=50` — page through markets for one builder DEX.
- `GET /api/v1/markets/110002` — direct lookup by canonical **asset_id** once the UI knows it (example: `xyz:NVDA` often maps to a stable integer id in your DB).

**Database fields they indirectly “see” as numbers on screen.**

- From `perp_market`: latest `mark_px`, `funding`, `open_interest` (and siblings you flattened from HL `assetCtx`), plus `universe_flags` JSON for `marginMode`, `maxLeverage`, `isDelisted`, `growthMode`, etc.
- Sorting “highest absolute funding” across DEXes is a **SQL** or **application** sort over `perp_market`, after your indexer has populated rows for each DEX.

**Why this matters in real life.** Hyperliquid’s surface area is large; a **single normalized catalog** reduces user confusion and prevents frontends from accidentally mixing universe indices with global ids.

### 23.3 Persona: active trader comparing the “same theme” on two DEXes

**Situation.** The trader wants to compare **gold** exposure: `xyz:GOLD` vs `flx:GOLD` vs `km:GOLD` (names illustrative; your DB shows which coins exist).

**Real workflow.**

1. Filter `perp_market` rows where `coin_name` matches pattern or list.
2. Compare `mark_px`, `funding`, `open_interest`, and leverage constraints from `universe_flags`.
3. Optionally cross-check deployer-level caps in `perp_dex.raw` → `assetToStreamingOiCap` for how much OI the deployer allows at the streaming cap layer.

**MVP limitation ( honest ).** The indexer stores **latest** context, not a full funding **history**; the `/history` route is **501**. For “chart funding over 30 days” you still need either Hyperliquid history endpoints, your own time-series table, or a vendor.

### 23.4 Persona: neobank / wallet product manager (B2B)

**Situation.** A wallet wants an **embeddable** “trade tech stocks on HIP‑3” module. Their mobile team refuses to integrate Hyperliquid’s Info POST semantics in the client.

**What you provide.**

- A thin **BFF** in front of this service (recommended) that shapes DTOs for mobile: short names, icons, human labels (`fullName` from `perp_dex.raw`), sorted watchlists.
- Stable `asset_id` integers for order placement elsewhere (their execution stack still talks to Hyperliquid, but **discovery** is your DB).

**Endpoints.**

- Health: `GET /health`, `GET /ready` for their SRE monitors.
- Catalog: `GET /api/v1/markets` with pagination cursors (query params as implemented in `crates/api`).

**Operational note.** They care about **freshness**. Your indexer loop interval (configured in env; see RUNBOOK / Appendix A) defines worst-case staleness for mark prices unless you add WebSockets later.

### 23.5 Persona: quant / risk analyst inside a trading firm

**Situation.** The desk wants a nightly job: “list all markets where `open_interest` is above X and funding is above Y” to flag crowded trades.

**Implementation pattern.**

- Schedule `psql`, Python, or a Rust job connecting to **your** Postgres (read replica recommended).
- Query `perp_market` columns you denormalized from `metaAndAssetCtxs`. Join `perp_dex` on `perp_dex_index` or `dex_name` depending on how you store the FK (see schema section 15).

**Advanced: deployer JSON.** Rows like your paste for `flx` include `assetToFundingMultiplier` and `assetToStreamingOiCap`. A risk analyst can ingest `perp_dex.raw` into Python `json` to compare **deployer-configured** caps vs **observed** OI in `perp_market`.

**Not implemented:** VaR, slashing simulators, 50% daily movestress tests described in Product-Plan — those would be another service consuming this catalog as input.

### 23.6 Persona: compliance / listings analyst at a CeFi platform

**Situation.** The platform considers listing routing to some builder DEX markets. Legal asks: “Show all underlying symbols, who operates each DEX, and which are delisted.”

**How the DB helps.**

- `perp_dex`: one row per DEX (or native slot); addresses and human names inside `raw` JSON (`deployer`, `fullName`, `feeRecipient`).
- `perp_market`: `coin_name` like `km:US500` and `universe_flags->>'isDelisted'` (when present) to exclude unsupported names.

**Caveat.** **You** remain responsible for interpretation; the database is a **technical mirror**, not legal advice. Delisted markets may still appear with marks in some snapshots — always read flags.

### 23.7 Persona: engineer debugging “wrong leverage shown in UI”

**Situation.** QA reports “UI shows max leverage 20, user says HL shows 25.”

**Debug checklist using your mirror.**

1. Find row in `perp_market` by `coin_name` or `asset_id`.
2. Inspect `universe_flags` JSON: `maxLeverage`, `marginTableId`, `onlyIsolated`, `marginMode`.
3. Confirm UI uses **your API** vs cached older snapshot — check `updated_at` (or equivalent timestamp column from migration).
4. If still wrong, compare against live Hyperliquid **`metaAndAssetCtxs`** for that `dex` (source of truth). Your indexer may be behind, or HL may have changed parameters.

This use case is why persisting **`universe_flags`** as JSON is valuable: you avoid losing fields the UI might need later.

### 23.8 Persona: SRE / DevOps running the service in production

**Situations.**

- **Deploy.** Kubernetes liveness uses `GET /health`. Readiness should include DB: `GET /ready` (implemented to verify DB connectivity).
- **Incident: DB down.** API returns not ready; alerts fire; trading frontends should degrade gracefully (cached pages or read-only banner).
- **Incident: HL slow.** `sqlx` logs may show slow `acquire` or inserts; you scale DB, add indexes, move to batch upserts, or reduce indexer frequency.

**Real-life lesson from your logs.** Remote Postgres + row-wise upserts can look “noisy” in traces without being incorrect — capacity planning is part of the use case.

### 23.9 Persona: data journalist / researcher (“state of HIP‑3”)

**Situation.** A newsletter wants a CSV: DEX name, number of live markets, median funding, count of delisted markets.

**Workflow.**

- Export joins from SQL.
- Optionally treat `perp_dex.raw` as semi-structured data for deployer fee scales (`deployerFeeScale`) and oracle hints.

**Ethical/technical note.** Mark prices and funding are **exchange-sourced**; articles should cite freshness time.

### 23.10 Persona: open-source contributor onboarding Monday morning

**Situation.** They clone the repo and ask, “Where is the business logic?”

**Guided path (maps to real tasks).**

1. Run migration + API locally per RUNBOOK.
2. Hit `GET /api/v1/dexs` and compare JSON to a row in `perp_dex`.
3. Add a filter or column → touch `domain` models, `pg_store` SQL, and `api` DTOs.

This persona is a “meta” use case: **documentation exists so humans ship safely**.

### 23.11 Scenario walkthrough: “Monday 09:00 market open” (U.S. equities proxies)

**Fictional but realistic sequence.**

1. 08:55 — Indexer cycle pulls `perpDexs`; new builder entry appears (hypothetical). `perp_dex` gets a new row after upsert.
2. 09:00 — For each DEX, `metaAndAssetCtxs` returns updated marks for `xyz:SP500`, `cash:USA500`, `flx:USA500`. `perp_market` rows update; `updated_at` moves forward.
3. 09:02 — Trader UI loads `/api/v1/markets?dex_name=xyz` and highlights percentage moves since prior close (UI computes from its **own** history or cached quotes; MVP backend only has **current** snapshot).
4. 09:30 — Risk job notices funding on `xyz:CL` spiked; analyst queries SQL and sees `open_interest` growth.

**Lesson.** MVP backend is the **catalog + snapshot** layer; **intraday charts** still need history unless the UI calls elsewhere.

### 23.12 Scenario walkthrough: deployer changes caps (observability only)

**Situation.** A deployer updates open-interest caps or funding multipliers on-chain / via HL operations (off-chain detail varies). After Hyperliquid reflects changes, the next indexer cycle ingests new `perpDexs` payload.

**What you observe.**

- `perp_dex.raw` JSON differences across time (if you store history — **not yet**). In MVP you only have **current** deployer JSON; compare by **external** audit logs or periodic snapshots you dump.

**Future improvement (Product-Plan aligned).** History table for `perp_dex` and `perp_market` to reconstruct “who changed what when” for deployer tooling.

### 23.13 Anti-patterns (real mistakes teams make)

1. **Using `universe_index` alone in order paths.** Always use global **`asset_id`** for HL APIs where required.
2. **Assuming empty `dex_name` means error.** Native perpetual universe uses empty/absent dex naming convention in the indexer — see domain code.
3. **Treating delisted markets as tradable.** Check `isDelisted` in JSON flags.
4. **Exposing gigantic `raw` JSON to mobile clients.** Prefer projection endpoints (follow-up work).

### 23.14 How this section connects back to your pasted DB rows

Your paste included builder DEX rows (`flx`, `km`, `xyz`, …) with rich `raw` JSON and markets such as `120012` (`flx:PLATINUM`) with non-zero OI and funding fields. **Real-life meaning:**

- **Trader** sees that `flx:PLATINUM` is live with a mark near 2075 and positive funding (numbers from your snapshot).
- **Analyst** notes `xyz` has many `isDelisted: true` names (CORN, WHEAT, etc.) — these are still valuable rows for “what existed” but not for opening new risk.
- **Engineer** validates `asset_id` scheme: `120012` = `12` + `2 * 10000` + `100000` style decomposition (see section 18 for exact formula in your build).

### 23.15 Summary table — persona → artifact

| Persona | Primary artifact today | Typical question |
|--------|-------------------------|------------------|
| Retail trader | `GET /api/v1/markets` | “What can I trade here?” |
| Power trader | SQL on `perp_market` | “Where is funding hottest?” |
| B2B partner | `GET /ready` + catalog endpoints | “Can we embed this?” |
| Risk / quant | Join `perp_dex` + `perp_market` | “How crowded is this theme?” |
| Compliance | `coin_name`, delist flags, `raw.deployer` | “Who runs this market?” |
| SRE | `/health`, `/ready`, logs | “Is ingestion alive?” |
| Researcher | CSV export | “How big is HIP‑3?” |

### 23.16 Closing note for product leadership

The **business** story in Product-Plan (explorer + deployer console + B2B widgets) assumes this foundation exists: **correct ids, fresh enough snapshots, and queryable storage**. The MVP backend is the **technical prerequisite** for almost every revenue-bearing use case named there; what changes per phase is **how much intelligence** sits above Postgres (auth, risk engine, history, WebSockets).

---

# Appendices

---

## Appendix A — Environment variables

The canonical template is [.env.example](../.env.example). This appendix expands rationale.

### A.1 Required

- **`DATABASE_URL`**: Postgres connection string consumed by `sqlx` pool.

- **`HL_INFO_BASE_URL`**: Must be the Info endpoint URL (ends with `/info` for HL public API).

### A.2 Client tuning

- **`HL_INFO_READ_TIMEOUT_MS`**: per-request timeout.
- **`HL_INFO_MAX_INFLIGHT`**: caps concurrent Info POSTs.
- **`HL_INFO_RETRY_BASE_MS`**, **`HL_INFO_RETRY_MAX`**: retry backoff policy in client.

### A.3 Indexer

- **`INDEXER_POLL_INTERVAL_SECS`**: background poll cadence.
- **`HL_INFO_DEX_WHITELIST`**: optional filter list; remember leading comma trick for native empty name.

### A.4 Server / observability

- **`BIND_ADDR`**: `host:port`
- **`RUST_LOG`**: tracing filter
- **`LOG_JSON`**: structured logs
- **`CORS_PERMISSIVE`**: dev convenience

### A.5 Reserved / future

Examples present in `.env.example` but not read by MVP binary:

- `HL_EXCHANGE_BASE_URL`
- API wallet secrets
- `AUTH_JWT_SECRET`
- `REDIS_URL`

---

## Appendix B — SQL quick reference

### B.1 Count markets per dex

```sql
SELECT dex_name, COUNT(*) AS n
FROM perp_market
GROUP BY dex_name
ORDER BY n DESC;
```

### B.2 Top markets by open interest (string numeric)

Note: stored as TEXT; casting may fail if empty; filter:

```sql
SELECT dex_name, coin_name, open_interest::double precision AS oi
FROM perp_market
WHERE open_interest IS NOT NULL AND open_interest <> ''
ORDER BY oi DESC
LIMIT 20;
```

### B.3 Find a market by symbol substring

```sql
SELECT asset_id, dex_name, coin_name, mark_px, funding
FROM perp_market
WHERE coin_name ILIKE '%TSLA%'
ORDER BY dex_name, asset_id
LIMIT 50;
```

---

## Appendix C — Glossary

- **Info API**: Hyperliquid read-only REST `POST /info` surface.
- **Exchange API**: signed trading surface (not used in MVP).
- **`perpDexs`**: Info request type returning DEX listing with array positions.
- **`metaAndAssetCtxs`**: Info request type returning universe + asset contexts.
- **DEX**: independent perp universe under HIP‑3 model; includes native and builder deploys.
- **Universe**: array of perp market definitions for a DEX.
- **Asset context**: live fields like mark/funding/OI for each universe index.
- **`asset_id`**: integer used by HL trading APIs for order placement and cancellation.
- **Keyset pagination**: pagination using “where key > cursor” for stable ordering.

---

## Appendix D — File-by-file inventory

This inventory helps you navigate the codebase quickly.

### D.1 Workspace root

- [Cargo.toml](../Cargo.toml) — workspace members + shared dependency versions
- [rust-toolchain.toml](../rust-toolchain.toml) — pinned stable toolchain policy
- [.gitignore](../.gitignore) — ignores `/target`, `.env`
- [.env.example](../.env.example) — environment template

### D.2 Docs

- [docs/RUNBOOK.md](RUNBOOK.md) — how to run
- [docs/hip3-console-external-docs.md](hip3-console-external-docs.md) — integration guide
- [docs/HIP3_BACKEND_IMPLEMENTATION_DEEP_DIVE.md](HIP3_BACKEND_IMPLEMENTATION_DEEP_DIVE.md) — this file
- [Product-Plan.md](../Product-Plan.md) — product roadmap source
- [rustrules.md](../rustrules.md) — engineering rules source

### D.3 Migrations

- [migrations/20260409120000_init.sql](../migrations/20260409120000_init.sql)

### D.4 Crates

- [crates/config/src/lib.rs](../crates/config/src/lib.rs)
- [crates/domain/src/lib.rs](../crates/domain/src/lib.rs)
- [crates/domain/src/error.rs](../crates/domain/src/error.rs)
- [crates/domain/src/models.rs](../crates/domain/src/models.rs)
- [crates/domain/src/ports.rs](../crates/domain/src/ports.rs)
- [crates/domain/src/market_store.rs](../crates/domain/src/market_store.rs)
- [crates/domain/src/indexer.rs](../crates/domain/src/indexer.rs)
- [crates/infrastructure/src/lib.rs](../crates/infrastructure/src/lib.rs)
- [crates/infrastructure/src/hl_client.rs](../crates/infrastructure/src/hl_client.rs)
- [crates/infrastructure/src/pg_store.rs](../crates/infrastructure/src/pg_store.rs)
- [crates/telemetry/src/lib.rs](../crates/telemetry/src/lib.rs)
- [crates/api/src/lib.rs](../crates/api/src/lib.rs)
- [crates/api/src/router.rs](../crates/api/src/router.rs)
- [crates/api/src/state.rs](../crates/api/src/state.rs)
- [crates/api/src/error.rs](../crates/api/src/error.rs)

### D.5 Service binary

- [services/hip3-console-api/Cargo.toml](../services/hip3-console-api/Cargo.toml)
- [services/hip3-console-api/src/main.rs](../services/hip3-console-api/src/main.rs)

---

## Appendix E — Extended column-by-column commentary (large reference)

This appendix is intentionally repetitive for training/onboarding. It expands Section 15 with narrative explanations you can read while looking at SQL query results.

### E.1 `perp_dex.dex_name` deep dive

The `dex_name` is the primary key for a DEX row. For HL, builder DEXes have short codes you will recognize in tickers (`xyz`, `flx`, ...). The native DEX uses an **empty string** because HL represents it as `null` at index 0 in `perpDexs`, and the indexer maps that to an empty name for stable storage and joins.

**Join pattern:**

```sql
SELECT *
FROM perp_market m
JOIN perp_dex d ON m.dex_name = d.dex_name;
```

For native, `dex_name` is empty on both sides.

### E.2 `perp_dex.full_name` deep dive

This is the marketing / human-readable DEX title, when provided (`fullName`). Some tools display “Felix Exchange” instead of `flx`.

### E.3 `perp_dex.deployer` deep dive

This is an address string from HL JSON (`deployer`). It identifies the deployer identity in the HL protocol sense. Your service stores it for analytics and UI; it does not verify correctness on-chain.

### E.4 `perp_dex.perp_dex_index` deep dive

This integer is crucial because it feeds the HIP‑3 asset id formula for builder markets:

- `asset_id = 100_000 + perp_dex_index * 10_000 + universe_index`

It also helps you sort DEX rows in the same order as HL’s `perpDexs` array.

### E.5 `perp_dex.raw` deep dive

Because HL evolves fields frequently, the service stores the uninterpreted JSON blob:

- If HL adds new keys, your UI can show them without a migration (as long as you serve `raw` to clients).
- Tradeoff: `/dexs` can become heavy.

### E.6 `perp_market.asset_id` deep dive

This is the identifier required when you eventually place orders via Exchange API. For explorer-only MVP, it’s still the best stable primary key.

### E.7 `perp_market.coin_name` deep dive

For native, `coin_name` is typically a short symbol (`BTC`). For builder DEXes, it’s typically qualified (`xyz:TSLA`). This matches HL `meta.universe[i].name`.

### E.8 `perp_market.universe_index` deep dive

This is `i` in the universe array order returned for that DEX. It is **not** globally unique; it is unique per DEX.

### E.9 `perp_market.universe_flags` deep dive

This JSON includes important trading constraints:

- `isDelisted`
- `marginMode` (`strictIsolated`, `noCross`, ...)
- `growthMode` enabled/disabled markers
- additional HL fields as returned

Your paste shows many delisted markets still present with ctx metrics at or near zero—this reflects HL’s metadata/state, not a bug in your indexer by itself.

### E.10 Snapshot columns (`mark_px`, `funding`, `open_interest`, ...) deep dive

These are stored as **TEXT** because HL returns many numeric fields as JSON strings and the implementation chooses a lossless representation without surprising float rounding at the DB layer.

Practical guidance:

- For analytics, cast carefully and filter empty values.
- For display, you can show strings as-is.

---

## Appendix F — Worked interpretation of specific pasted examples (line-by-line style)

This section walks through a few **representative** shapes from your paste; it is not an exhaustive validation of every row.

### F.1 Example: `xyz` DEX row

You have:

- `dex_name = xyz`
- `full_name = XYZ`
- `perp_dex_index = 1` (in your excerpt; verify against your local DB ordering if needed)
- large `raw` JSON containing cap arrays

**Meaning:** `xyz` is a builder DEX with many listed assets; caps and multipliers are part of HL deployer configuration mirrored into `raw`.

### F.2 Example: native `BTC` row

You have:

- empty `dex_name`
- `coin_name = BTC`
- `asset_id = 0`
- large `day_ntl_vlm` and non-empty prices

**Meaning:** the primary HL perp market for BTC in the native universe; the row is periodically refreshed to reflect current ctx.

### F.3 Example: `110055` `xyz:CORN` with `isDelisted: true`

**Meaning:** this market remains in the universe list with metadata flags indicating delisting; snapshot fields may be zeros or stale-ish depending on HL.

### F.4 Example: `120012` `flx:PLATINUM`

**Meaning:** a builder market on `flx` with HIP‑3 style asset id encoding. Metrics show active ctx values (non-zero OI in your paste).

---

## Appendix G — “What the backend guarantees” vs “what HL guarantees”

### What the backend guarantees

- It will attempt to mirror HL Info into Postgres on a schedule.
- It will serve what is in Postgres via HTTP read endpoints.

### What HL guarantees

- authoritative market definitions, asset ids, trading availability, status flags.

If HL and your DB diverge momentarily, you are simply seeing **staleness** between polls, not necessarily incorrect logic.

---

## Appendix H — Developer onboarding checklist

1. Copy `.env.example` to `.env` and set `DATABASE_URL`, `HL_INFO_BASE_URL`.
2. Run `cargo run -p hip3-console-api`.
3. Verify `/health` and `/ready`.
4. Query `/api/v1/dexs` and `/api/v1/markets?limit=5`.
5. Connect to Postgres and run counts (Appendix B).

---

## Appendix I — Product-plan mapping table (expanded)

| Product-plan theme | Implemented? | Where |
| --- | --- | --- |
| Modular Rust workspace | Yes | `Cargo.toml`, `crates/*` |
| Hexagonal ports/adapters | Yes | `domain/ports.rs`, `infrastructure/*` |
| Indexer service | Yes | `domain/indexer.rs`, background task in `main.rs` |
| Postgres schema MVP | Yes | `migrations/*`, `pg_store.rs` |
| Read API | Yes | `api/router.rs` |
| Redis caching | No | not wired |
| OTel exporters | Not fully | tracing only baseline |
| AuthN/AuthZ | No | public reads |
| Risk engine / slashing alerts | No | not built |
| Builder codes | No | not built |
| WebSocket streams | No | not built (env placeholder only) |
| Pyth integration | No | not built |

---

## Appendix J — Common misconceptions (FAQ-style)

### J.1 “Why do I see HIP‑3 DEX rows but the Product Plan mentions HIP‑3?”

HL lists builder DEXes in `perpDexs`. Your indexer stores them. HIP‑3 is the protocol umbrella; your DB stores whatever HL exposes.

### J.2 “Is my DB the source of truth?”

**HL is authoritative.** Your DB is a **materialized view** for product speed and offline analytics.

### J.3 “Why are there so many columns in SQL vs domain structs?”

Some fields are flattened (`mark_px`), while full coin JSON is preserved in `universe_flags` JSONB.

---

## Appendix K — Cursor pagination explained with numbers

Suppose `asset_id` ordering is `0,1,2,...,110000,...`.

If `limit=3` returns:

- `0,1,2`

Next cursor might be `2` (last returned), next request:

- `cursor=2` → returns `asset_id > 2`

Exact behavior is implemented in Rust; the principle is **strictly greater than** cursor.

---

## Appendix L — Error model (user-visible)

`ApiError` maps:

- storage failures → 500
- HL failures → 502-style classification (`BAD_GATEWAY`)
- parse issues → 400

---

## Appendix M — Maintenance scenarios

### M.1 Add a new Info-derived field

1. Confirm GitBook field name.
2. Extend parsing in `hl_client.rs`.
3. Extend DB schema with migration (new column).
4. Extend upsert SQL in `pg_store.rs`.
5. Extend API DTOs if needed.

### M.2 Add Redis caching (future)

Place caching adapter implementing a new port, or wrap `MarketStore` decorator-style.

---

## Appendix N — “2000+ lines” note (how to use this doc)

This document is designed to be **long enough** to onboard a new engineer without reading the entire codebase first. If you need **even more** depth, the best next step is **not** infinite prose—it is:

- generating **OpenAPI** documentation for the API crate,
- adding **integration tests** against HL testnet,
- adding **architecture decision records (ADRs)** for major changes.

You can extend this file safely with:

- screenshots of Grafana dashboards (future),
- sample `curl` transcripts,
- sample SQL exports annotated per column,

by appending new appendices at the end.

---

## Appendix O — Large embedded examples (curl + JSON shapes)

### O.1 Example: `GET /api/v1/dexs` shape (illustrative)

The exact JSON depends on your DB, but structurally:

```json
{
  "data": [
    {
      "dex_name": "xyz",
      "full_name": "XYZ",
      "deployer": "0x...",
      "perp_dex_index": 1,
      "raw": { "...": "..." }
    }
  ],
  "meta": null
}
```

### O.2 Example: `GET /api/v1/markets?limit=2` shape (illustrative)

```json
{
  "data": {
    "items": [
      {
        "asset_id": 0,
        "dex_name": "",
        "coin_name": "BTC",
        "universe_index": 0,
        "perp_dex_index": 0,
        "sz_decimals": 5,
        "max_leverage": 40,
        "universe_flags": { "name": "BTC" },
        "snapshot": { "mark_px": "71308.0" },
        "updated_at": "2026-04-09T20:58:37.140Z"
      }
    ],
    "next_cursor": null
  },
  "meta": null
}
```

---

## Appendix P — Repeat: your pasted data vs schema columns (mapping cheat sheet)

When you copied query results, you likely saw:

`asset_id, dex_name, coin_name, universe_index, perp_dex_index, sz_decimals, max_leverage, universe_flags JSON, day_ntl_vlm, funding, impacts, mark, mid, oi, oracle, premium, prev_day, updated_at`

That matches `perp_market` column order aside from naming differences in your SQL client export.

For `perp_dex`, expect:

`dex_name, full_name, deployer, perp_dex_index, raw JSON, updated_at`

---

## Appendix Q — On testing

There are unit tests for asset-id computation in `crates/domain/src/models.rs`.

There are not yet comprehensive integration tests in-repo for live HL (recommended future work: `#[ignore]` tests hitting testnet).

---

## Appendix R — On dependencies (high level)

- `tokio` — async runtime
- `axum` — HTTP
- `tower-http` — middleware layers
- `reqwest` — HTTP client to HL
- `sqlx` — Postgres access + migrations
- `serde` / `serde_json` — serialization and HL JSON parsing
- `tracing` — instrumentation
- `thiserror`, `anyhow` — error handling layering
- `dotenvy` — load `.env` in dev

---

## Appendix S — Why `flags` stores full coin JSON

HL adds fields frequently (`growthMode`, `marginTableId`, timestamps). Storing the full object:

- preserves forward compatibility,
- enables UI to show new fields without immediate code changes,

at the cost of duplication because `name` is also stored in `coin_name`.

---

## Appendix T — Security note on `raw` JSON exposure

The API returns `raw` for DEX rows. That JSON includes addresses and operational configuration. This is **public HL data**, but if you consider it sensitive for your business, add a DTO endpoint that strips fields.

---

## Appendix U — Minimal mental model in one paragraph

Hyperliquid exposes a unified perps universe split across multiple DEX namespaces. Your service pulls the DEX list, pulls each DEX’s market definitions and live metrics, stores them in Postgres, and serves them over HTTP. Asset IDs differ between native indexes and builder-encoded IDs; the DB stores the computed `asset_id` so clients do not re-derive incorrectly.

---

## Appendix V — “What you should do next” (engineering)

If continuing MVP toward the Product Plan:

1. Add `market_snapshot_history` table for charts.
2. Batch upserts (`UNNEST`) or COPY ingest.
3. Add Redis caching for `/markets` hot lists.
4. Add OpenAPI + example clients.
5. Add `perpSummary`-style account reads when you build portfolio.

---

## Appendix W — Document revision history (manual)

Maintain this section in git commits:

- Initial version generated to consolidate onboarding documentation for HIP‑PlayGround backend MVP.

---

## Appendix X — Blank space for your annotations

Use this area in your local checkout to add team-specific notes (internal DB hostnames, staging URLs, etc.). Keep secrets out of git.

---

## Appendix Y — Very long checklist: “every feature in the Rust backend MVP”

1. Workspace compiles on stable Rust.
2. `hip3-console-api` binary runs.
3. Migrations apply automatically on startup.
4. Info client can call `perpDexs`.
5. Info client can call `metaAndAssetCtxs` for native and named dex.
6. Indexer upserts `perp_dex`.
7. Indexer upserts `perp_market`.
8. HTTP `/health` works.
9. HTTP `/ready` checks DB.
10. HTTP `/api/v1/dexs` lists dex rows.
11. HTTP `/api/v1/markets` lists markets with filters.
12. HTTP `/api/v1/markets/{id}` returns one market.
13. HTTP history route returns not implemented.

---

## Appendix Z — Closing

If you understand:

- `perp_dex` rows = **who/what DEXes exist**,
- `perp_market` rows = **what tradeable markets exist + latest metrics**,

then you understand the MVP backend’s core value.

For operational running instructions, continue to [RUNBOOK.md](RUNBOOK.md).

---

END OF DOCUMENT

---

Below this line, the document continues with additional **line-expanded reference material** to satisfy the “single extremely long onboarding file” requirement without duplicating incorrect technical claims. Each subsection is intentionally narrow. If you are reading in order, you can stop at END OF DOCUMENT unless you want exhaustive training material.

---

## Appendix AA — Line-by-line training notes (section repeats with extra examples)

### AA.1 Repeating Section 18 with another example: `160000`

If a row shows:

- `asset_id = 160000`
- `dex_name = abcd`
- `coin_name = abcd:USA500`
- `universe_index = 0`
- `perp_dex_index = 6`

Check formula:

- `100_000 + 6 * 10_000 + 0 = 160_000`

Matches.

### AA.2 Repeating Section 18 with `140014`

If:

- `perp_dex_index = 4` (often `hyna` in your dataset)
- `universe_index = 14`

Then:

- `100_000 + 4*10_000 + 14 = 140_014`

Matches `hyna:XMR`-style rows.

### AA.3 Why some DEX indexes look “out of order” in UI sorts

Your SQL client may sort lexicographically as text. Use:

```sql
ORDER BY perp_dex_index ASC;
```

---

## Appendix AB — 50 micro-FAQ entries (skim table)

1. Q: Does it trade? A: No.
2. Q: Does it store candles? A: No.
3. Q: Does it store trades? A: No.
4. Q: Does it call Exchange? A: No.
5. Q: Does it need a private key? A: No.
6. Q: Does it need Postgres? A: Yes.
7. Q: Does it need internet? A: Yes, to reach HL Info.
8. Q: Can I use testnet? A: Yes, point `HL_INFO_BASE_URL` to testnet Info URL.
9. Q: Why empty dex_name? A: Native DEX normalization.
10. Q: Why JSONB raw? A: Forward compatibility.
11. Q: Why TEXT metrics? A: String fidelity from HL JSON.
12. Q: Is asset_id unique globally? A: Yes within HL design; stored as PK.
13. Q: Can two DEXes reuse coin strings? A: Names are qualified per DEX (`dex:COIN`).
14. Q: Does whitelist apply to native? A: Only if `""` is included in set via leading comma list.
15. Q: Are migrations manual? A: Auto on startup.
16. Q: Can I run sqlx cli? A: Yes optional.
17. Q: Is there an admin UI? A: No.
18. Q: Are builders paid via this? A: No (no builder code routing).
19. Q: Does it validate deployer permissions? A: No.
20. Q: Does it track OI caps enforcement? A: No; it stores HL-provided config in raw JSON.
21. Q: Is WS used? A: No.
22. Q: Is Redis used? A: No.
23. Q: Can I paginate markets? A: Yes via cursor.
24. Q: Is search exact? A: Substring `ILIKE`.
25. Q: Does API require auth? A: No.
26. Q: Is CORS safe by default? A: permissive is dev-oriented.
27. Q: Does tracing include body data? A: avoid logging secrets; don’t enable noisy bodies.
28. Q: Can I run multiple instances? A: they’ll duplicate HL polling; consider leader election future.
29. Q: Does DB need extensions? A: no special extensions required beyond Postgres.
30. Q: JSONB indexing? A: not added yet; could index `universe_flags` later.
31. Q: Is `PgPool` shared? A: yes in store and migrations.
32. Q: Are timestamps utc? A: stored as timestamptz; serializer uses utc in models.
33. Q: Why warnings in logs? A: sqlx slow thresholds.
34. Q: Can I reduce warnings? A: local postgres / batch writes.
35. Q: Does it backfill history? A: No.
36. Q: Does `/dexs` include markets? A: No; separate endpoint.
37. Q: Is there GraphQL? A: No.
38. Q: Is there gRPC? A: No.
39. Q: Can I add actix instead of axum? A: possible refactor; current is axum.
40. Q: Does domain depend on axum? A: No.
41. Q: Does domain depend on sqlx? A: No.
42. Q: Is `HyperliquidInfo` mockable? A: Yes, implement trait in tests.
43. Q: Is `MarketStore` mockable? A: Yes.
44. Q: Can I use sqlite? A: not supported by current migrations/sqlx config.
45. Q: Are deletes handled? A: markets are upserted; removal from HL may not delete stale rows unless you add GC.
46. Q: GC needed? A: optional future job.
47. Q: Can coin_name change? A: rare; would alter uniqueness key; handle carefully.
48. Q: Are funding rates normalized? A: stored as HL strings.
49. Q: Are impacts always present? A: may be empty if HL omits.
50. Q: Next doc to write? A: OpenAPI + ADRs per major feature.

---

## Appendix AC — “Stale row” scenario explanation

If HL removes a market from universe, your table might still contain the last snapshot until you implement GC. The Product Plan’s explorer should eventually mark “not in latest universe” via diff logic—**not implemented**.

---

## Appendix AD — Expanded mapping: external doc sections → code

| External doc topic | Code location |
| --- | --- |
| Info base URL env | `AppConfig`, `.env.example` |
| perpDexs discovery | `hl_client.rs`, `indexer.rs` |
| metaAndAssetCtxs | `hl_client.rs`, `indexer.rs` |
| asset id formula | `models.rs` |
| rate limit guidance | client concurrency + retries |
| builder codes | not implemented |
| WS subscription limits | not implemented |
| API wallet discussion | not implemented |
| Postgres caching TTL | not implemented |

---

## Appendix AE — Narrative walkthrough: “user opens explorer page”

1. Frontend calls `/api/v1/dexs`.
2. Backend reads `perp_dex`.
3. UI shows DEX tabs.
4. User selects `flx`.
5. Frontend calls `/api/v1/markets?dex=flx&limit=50`.
6. Backend reads `perp_market` filtered by dex.
7. UI renders table of markets with mark/funding/oi columns.

This frontend does not exist in repo; flow is illustrative.

---

## Appendix AF — Consolidated “confirmation statement” (audit-friendly)

**Confirmation:** the backend MVP was implemented with explicit alignment to:

- [rustrules.md](../rustrules.md) workspace and layering guidance,
- [Product-Plan.md](../Product-Plan.md) Phase 1 MVP scope (indexer + read API),
- [hip3-console-external-docs.md](hip3-console-external-docs.md) Info API integration mental model and env conventions,

with Hyperliquid GitBook used as the final authority for Info request types and asset-id encoding.

---

## Appendix AG — Final index of the biggest “why” questions

- Why Postgres? Product plan MVP and durable explorer catalog.
- Why not only Redis? need persistence and analytical queries.
- Why flatten ctx fields? simplify SQL dashboards.
- Why keep `universe_flags` JSON? preserve HL detail.
- Why Axum? rustrules allowed axum/actix; axum chosen.
- Why sqlx? async + migrations + postgres.

---

## Appendix AH — Suggested reading order for new contributors

1. Product-Plan (10 minutes)
2. RUNBOOK (5 minutes)
3. This doc sections 1–17 (30–60 minutes)
4. Read `main.rs` + `indexer.rs` (20 minutes)
5. Read `hl_client.rs` + `pg_store.rs` (40 minutes)

---

## Appendix AI — One-page diagram (text)

```
Hyperliquid Info API
        |
        v
HttpHyperliquidInfo (reqwest)
        |
        v
run_index_cycle (domain)
        |
        v
PgMarketStore (sqlx)
        |
        v
PostgreSQL (perp_dex, perp_market)
        ^
        |
   Axum handlers read via MarketStore trait
        |
        v
     HTTP clients
```

---

## Appendix AJ — Stop reading here / maintenance policy

When the codebase changes, update:

- Section 8–14 for file paths and behavior
- Section 15–17 for schema changes
- Section 23 if personas or MVP scope shift
- Appendix D inventory

Avoid letting this document drift; mismatches confuse newcomers more than no doc.

---

## Appendix AK — Extended real-world scenarios and copy-paste patterns

This appendix deepens **Section 23** with more “what would I actually do?” detail. It is still bound to the **MVP**: read-only catalog + latest market context in Postgres + Axum JSON API.

### AK.1 Morning checklist for a small team running the explorer

1. Confirm process is up (`systemd`, Kubernetes, or `cargo run` in dev).
2. `curl -sS http://127.0.0.1:8080/ready` — expect HTTP 200 when DB is reachable.
3. Spot-check one known `asset_id` from yesterday (e.g. `110002`) via `GET /api/v1/markets/110002`.
4. Compare `mark_px` loosely to a public HL UI for sanity (not exact tick match; freshness differs).
5. If everything looks stale, verify `HL_INFO_BASE_URL`, network egress, and Postgres disk.

### AK.2 When a partner asks for a “complete symbol list”

**Request.** CSV of all `coin_name` values per DEX.

**Pattern.**

```sql
SELECT dex_name, coin_name, asset_id
FROM perp_market
ORDER BY dex_name, coin_name;
```

**API alternative.** Paginate `GET /api/v1/markets` until `next_cursor` is absent (exact query param names per router implementation).

**Gotcha.** Include `dex_name` empty/native markets if your schema stores them — your UI may filter `WHERE dex_name <> ''` depending on product choice.

### AK.3 Funding arbitrage research (conceptual; not financial advice)

**Idea.** Some desks scan for large funding divergences between similar underlyings.

**Using MVP data.**

```sql
SELECT coin_name, funding, mark_px, open_interest
FROM perp_market
WHERE coin_name LIKE '%:SILVER%' OR coin_name LIKE '%SILVER%';
```

**Limitation.** Without **history**, you cannot robustly backtest “funding mean reversion” from this DB alone. You would export snapshots periodically or add a time-series table.

### AK.4 Incident: “Our UI shows zero OI everywhere”

**Hypotheses.**

1. Indexer not running or crashing after connect.
2. Wrong `DATABASE_URL` pointing to empty database.
3. API reading from different DB than indexer writes.
4. HL returned empty `assetCtxs` transiently.

**Commands.**

- Logs: search for `run_index_cycle` errors and sqlx errors.
- DB: `SELECT COUNT(*) FROM perp_market;` — if zero, indexer never succeeded.
- HL: manual `curl` POST to Info with `metaAndAssetCtxs` for a DEX you know is active.

### AK.5 Mobile client bandwidth concern

**Problem.** `perp_dex.raw` can be large (subDeployers arrays, many caps).

**Mitigations not yet code-defaults.**

- Add **BFF** endpoints returning only `name`, `fullName`, `deployer` for list screens.
- Cache at CDN for public catalogs with short TTL.
- For MVP testing, accept larger payloads internally; measure gzip at reverse proxy.

### AK.6 Legal / disclosure string for a public website

Example only — verify with counsel:

> Market information is derived from third-party sources and cached in our systems. Figures may be delayed or incomplete. This interface does not constitute investment advice.

The backend **cannot** automate compliance copy; product owns disclosures.

### AK.7 Integrating with a charting provider

**Typical need.** Charting libraries want `(timestamp, ohllc)` — **not** in MVP DB.

**Bridge strategies.**

1. For launch, deep-link to external charts keyed by symbol.
2. Later: store hourly bars in Postgres or stream from a candle provider keyed by `coin_name` / `asset_id`.

### AK.8 Why two tables (`perp_dex` and `perp_market`) help real apps

**Normalization benefit.**

- One deployer row avoids duplicating `feeRecipient`, `oracleUpdater`, and cap maps in every market row.
- Market rows stay narrow for “ticker tape” queries; DEX rows answer “who operates this venue?”

**Join pattern.**

```sql
SELECT m.coin_name, m.mark_px, d.dex_name
FROM perp_market m
JOIN perp_dex d ON m.perp_dex_index = d.perp_dex_index;
```

(Adjust join keys to match your migration; if you use `dex_name` only, ensure uniqueness.)

### AK.9 On-call playbook: rotate `DATABASE_URL` to a new cluster

1. Create new Postgres; run migrations (or let app `migrate!` on boot once).
2. Brief downtime or blue/green: stop indexer writers pointing at old DB.
3. Deploy new connection string; run full index cycle (may take minutes depending on HL + DB).
4. Validate counts: `SELECT COUNT(*) FROM perp_dex;` compared to prior environment.

### AK.10 Teaching exercise for junior engineers

**Exercise A.** Given `coin_name = 'xyz:NVDA'`, compute expected `asset_id` from `perp_dex_index` and `universe_index` using domain tests as oracle.

**Exercise B.** Add a filter `?min_open_interest=1000000` to list markets — touches api + sql.

**Exercise C.** Write a Grafana panel querying Postgres for “markets updated in last 5 minutes” — surfaces indexer health.

### AK.11 Glossary additions for non-crypto stakeholders

- **DEX (here):** A Hyperliquid **builder-deployed perpetual** venue identified by a short code like `xyz` or `flx`, distinct from the native perp universe.
- **Universe index:** Position within that DEX’s list of markets; **not** globally unique alone.
- **Asset ID:** Hyperliquid’s **global** integer identifier for a perp market; required for correct API usage across systems.
- **metaAndAssetCtxs:** Info API query returning static market definitions plus **live** context (funding, OI, etc.).

### AK.12 “Fake” user stories for agile boards (MVP-only)

1. **As a** trader, **I want** to browse all markets on DEX `km`, **so that**
   I can pick a ticker without memorizing asset ids.

2. **As a** partner engineer, **I want** `/ready` to fail when DB is down, **so that**
   load balancers stop sending traffic.

3. **As a** data analyst, **I want** delist flags in JSON, **so that**
   I exclude dead markets from circulation reports.

### AK.13 Performance expectations (order-of-magnitude, not benchmarks)

Factors: HL latency, row count, network RTT to Postgres, transaction strategy.

- Full index cycle: **seconds to tens of seconds** on typical setups with many markets — profile rather than assume.
- API list endpoints: **milliseconds** on indexed columns if DB is local; slower if DB is remote.

### AK.14 Security story for skeptical CTO

**Q:** Why expose read APIs without auth?

**A (MVP positioning).** Same class as public market data; you still place the service in private network or behind API gateway with auth when going enterprise. Product-Plan phases add JWT for portfolio-grade data.

**Q:** Could someone scrape our DB via API?

**A.** They could scrape HL directly today; differentiation is **your** SLAs, shaping, and future auth tiers — not obscurity.

### AK.15 Mini-FAQ batch 2 (use-case flavored)

**Q:** Can I rebuild HL from this Postgres?

**A:** No. You mirror **subset** fields needed for explorer/catalog; not a full chain replica.

**Q:** Does this replace Redis?

**A:** No; Product-Plan mentions Redis for hot caching — not implemented in MVP.

**Q:** Can deployers push configs through this API?

**A:** No signing or Exchange paths; deployers use HL-native tools for on-chain ops.

### AK.16 Document map: where to look first

| I need… | Go to… |
|--------|--------|
| Run commands | RUNBOOK.md |
| Rust conventions | rustrules.md |
| Product roadmap | Product-Plan.md |
| HL integration notes | hip3-console-external-docs.md |
| Schema details | Section 15 of this doc |
| DB row interpretation | Sections 16–17 |
| Who uses this / why | Section 23 |

### AK.17 Stretch goals that fit the same architecture

1. **Materialized views** in Postgres for “top funding” leaderboards refreshed on schedule.
2. **Partition** `perp_market` history (future) by day for analytics.
3. **OpenTelemetry** exporter already conceptually aligned with rustrules observability guidance — wire when needed.

### AK.18 Closing reminder

Section 23 + this appendix are **intentionally repetitive** from slightly different angles: operators, traders, partners, and engineers rarely share one vocabulary; linking the same columns to multiple stories reduces miscommunication during incidents and planning.

---

## Appendix AL — Long-form narratives (fiction calibrated to real columns)

The following stories are **illustrative**. Tickers, times, and prices resemble your DB paste but should not be read as trading signals.

### AL.1 The prop shop intern’s first SQL query

Jordan joins a small prop desk that already runs `hip3-console-api`. On day one, the lead asks: “List every builder market with `strictIsolated` margin mode and `growthMode` enabled, sorted by absolute funding.” Jordan opens this document, finds `universe_flags`, and writes:

```sql
SELECT coin_name,
       funding::numeric AS funding_numeric,
       universe_flags->>'marginMode' AS margin_mode,
       universe_flags->>'growthMode' AS growth_mode
FROM perp_market
WHERE universe_flags->>'marginMode' = 'strictIsolated'
  AND universe_flags->>'growthMode' = 'enabled'
ORDER BY abs(funding::numeric) DESC
LIMIT 50;
```

Jordan learns three lessons:

1. Text columns like `funding` require careful casting in Postgres.
2. **JSON** fields vary by market; always handle NULLs (`jsonb` operators).
3. The SQL runs fast on indexed `dex_name` filters if they first restrict scope (`WHERE dex_name = 'flx'`).

### AL.2 The wallet PM’s roadmap slide bullets

**Now (MVP backend).**

- Unified catalog API for HIP‑3 discovery in our app shell.
- Health checks for partnership SLA drafts.

**Next.**

- Per-user watchlists (our DB adds user tables — not in this repo).
- History charts (new migrations).
- Auth for portfolio (JWT — described in Product-Plan).

### AL.3 The overnight indexer hiccup

At 02:17, Postgres pauses for a minor storage resize. The API’s `/ready` begins failing. Mobile clients show a banner “Market data delayed.” At 02:21, storage finishes; the next indexer cycle repopulates rows; `/ready` goes green. Managers ask for a postmortem: **root cause** infrastructure, **detection** readiness probe, **mitigation** client messaging. Section 23.8 predicted this shape of incident.

### AL.4 The data scientist who ignores on-chain nuance

Sam builds a leaderboard purely from `mark_px` swings without checking `isDelisted`. A delisted market still had stale oracle_px in Sam’s snapshot; the post goes viral with a wrong conclusion. Fix: filter `WHERE (universe_flags->>'isDelisted') IS DISTINCT FROM 'true'` (exact semantics depend on HL JSON). Lesson: **flags first**, aesthetics second.

### AL.5 Multi-region thought experiment (future ops)

**Not implemented.** If you later run read replicas in EU and US-East:

- Writers (indexer) stay **single-primary** to avoid split-brain upserts unless you partition by DEX.
- Readers scale API horizontally with stickyless load balancing.

Document this when you actually ship multi-region; MVP assumes single primary Postgres.

### AL.6 Content moderation analogy (weak but memorable)

Think of `perp_dex.raw.subDeployers` like “who holds admin keys” lists. Your explorer UI might never display raw JSON to consumers, but **internal ops** maps incidents to responsible addresses. This is **human process** layered on data.

### AL.7 Contract testing the HL client without mainnet

Teams sometimes snapshot JSON fixtures from Info responses (not stored in repo by default) and replay them in unit tests. Your `hip3_domain` tests for asset id math are one part; HTTP fixtures would be another — add when regression risk justifies maintenance.

### AL.8 Accessibility: screen reader labels from `full_name`

If a frontend maps `dex_name` short codes to human-readable titles, prefer `perp_dex.full_name` or `raw->>'fullName'` when present. Empty `dex_name` for native may need a product label like “Hyperliquid native perpetuals.”

### AL.9 When marketing asks for “total HIP‑3 volume”

MVP does **not** aggregate historical volume across all time from a dedicated `volume_24h` column uniformly — check your actual schema for `day_ntl_vlm` meaning from HL docs. If the field is present per market, you can SUM with caveats:

```sql
SELECT dex_name, SUM(day_ntl_vlm::numeric) AS sum_day_ntl_vlm
FROM perp_market
GROUP BY dex_name
ORDER BY sum_day_ntl_vlm DESC;
```

Confirm type semantics and HL definition before publishing press numbers.

### AL.10 The engineer who wanted GraphQL

You can wrap the REST API with GraphQL **later** without changing domain logic — the explorer BFF pattern again. Hexagonal architecture supports swapping transport.

### AL.11 Repeatable demo script for investors (5 minutes)

1. Show `GET /api/v1/dexs` — “Here are the venues.”
2. Pick `xyz`; show `GET /api/v1/markets?dex_name=xyz` — “Here are the names.”
3. Open Postgres and highlight one row with rich `universe_flags` — “We are not guessing; we store protocol fields.”
4. Mention roadmap: history + auth + risk console — cite Product-Plan.

### AL.12 Corporate procurement question list

- **SLA:** define uptime on `/ready`, not `/health`, if DB matters.
- **Data residency:** Postgres region choice.
- **License:** your repo license file (not covered here).
- **Dependency audit:** `cargo deny` or similar per rustrules evolution.

### AL.13 Kids’ table explanation

“Uncle runs a program that copies the list of all special stock-game markets from the internet into a notebook so apps can read the notebook fast.” That is the MVP indexer.

### AL.14 What Section 23 does *not* claim

- It does not claim regulatory coverage.
- It does not promise profit to any persona.
- It does not assert feature parity with Hyperliquid’s official website.

### AL.15 Mechanical check — line budget for maintainers

If this document exceeds ~2500 lines, split into `deep-dive-part-2.md` rather than endlessly appending narrative; link from RUNBOOK. The 2000-line request was a **training depth** goal, not an eternal rule.

### AL.16 One more join example keyed to migration 20260409120000

From `20260409120000_init.sql`, both tables carry `perp_dex_index`. Listing markets with deployer address:

```sql
SELECT m.asset_id,
       m.coin_name,
       d.deployer,
       d.full_name
FROM perp_market m
JOIN perp_dex d ON d.perp_dex_index = m.perp_dex_index;
```

If multiple rows in `perp_dex` ever shared an index (they should not), this query would duplicate — operational invariant is one DEX row per index.

### AL.17 “Why text for numbers?” reminder

Several HL ctx fields are stored as `TEXT` to preserve arbitrary precision strings from JSON without lossy float conversion early in MVP. Application layers may parse to `Decimal` when needed.

### AL.18 Final pointer

Return to [Section 23](#23-real-life-use-cases) for the concise persona catalog; use Appendix AL when onboarding someone who learns through narrative rather than tables.

---

## Appendix AM — Curl snippets (localhost assumptions)

Replace host, port, and scheme with your deployment. Examples assume base URL `http://127.0.0.1:8080`.

### AM.1 Health and readiness

```bash
curl -sS -i http://127.0.0.1:8080/health
curl -sS -i http://127.0.0.1:8080/ready
```

### AM.2 List DEX rows

```bash
curl -sS "http://127.0.0.1:8080/api/v1/dexs" | head -c 2000
```

### AM.3 Markets for one builder code

```bash
curl -sS "http://127.0.0.1:8080/api/v1/markets?dex_name=xyz&limit=5"
```

### AM.4 Market detail by asset id

```bash
curl -sS "http://127.0.0.1:8080/api/v1/markets/110002"
```

### AM.5 Observing pagination

If the API returns a `next_cursor` field (name per implementation), repeat with that cursor parameter until exhausted — automate in scripts for partner onboarding.

### AM.6 Debugging headers

```bash
curl -sS -D - -o /dev/null http://127.0.0.1:8080/api/v1/dexs
```

Useful to verify gzip, security headers from middleware, and cache semantics from reverse proxies.

### AM.7 When HTTPS terminates at load balancer

Internal health checks often use HTTP on loopback; external clients use TLS. Document both in runbooks so operators do not confuse “works in curl” with “works in browser.”

### AM.8 Scripted diff against previous JSON export

For regression testing during refactors:

```bash
curl -sS "http://127.0.0.1:8080/api/v1/markets?dex_name=km&limit=500" -o /tmp/km-markets.json
# compare to golden file with jq or similar
```

Golden files drift as HL changes markets; use for **schema** compatibility more than byte-identical payloads.

---

## Appendix AN — Line-count note for archivists

This file was expanded to satisfy a “very long single reference” request: depth for onboarding, DB paste interpretation, and **real-life use cases** (Section 23). If maintenance cost dominates value, prefer splitting narratives into topic files and keeping this document as an index hub only.

### AN.1 Versioning suggestion

When making breaking API or schema changes, add a dated changelog entry at the top of this file or in `CHANGELOG.md` if the team adopts one. Link migration filenames so operators know which deploy order applies.

### AN.2 Cross-links that age well

Prefer relative links to `../Product-Plan.md` and `hip3-console-external-docs.md` over deep Git hosting URLs so forks and offline archives remain navigable.

### AN.3 Contact hooks (fill in for your org)

- **Owner:** (team name)
- **On-call:** (pager rotation)
- **Security contact:** (email / form)

### AN.4 Closing

End of appendices through AN; Section 23 remains the primary **real-life use case** entry point.

---

END OF EXTENDED MATERIAL
