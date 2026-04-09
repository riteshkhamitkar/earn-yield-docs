# HIP-3 Console API — Runbook

This service indexes Hyperliquid **perp DEXs and markets** via the public **Info API**, stores snapshots in **Postgres**, and exposes a **read-only HTTP API** (`/api/v1/...`).

Authoritative API behavior is defined in the [Hyperliquid GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api). This MVP uses:

- `POST /info` with `{ "type": "perpDexs" }` — list perp DEX rows (see [Perpetuals](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals)).
- `POST /info` with `{ "type": "metaAndAssetCtxs", "dex": "<name>" }` — universe + funding/OI/mark context per DEX (optional `dex` for the first/native DEX).
- [Asset IDs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids) for HIP-3 markets: `100_000 + perp_dex_index * 10_000 + index_in_meta`; native perp indices use the universe index.

Project context: [Product-Plan.md](../Product-Plan.md), [rustrules.md](../rustrules.md), [hip3-console-external-docs.md](hip3-console-external-docs.md).

## Prerequisites

- **Rust** stable (see [rust-toolchain.toml](../rust-toolchain.toml)).
- **PostgreSQL** 14+ reachable from your machine.

## Credentials and secrets

| Item | Required for MVP? | Where to get it |
|------|-------------------|-----------------|
| `DATABASE_URL` | Yes | Your Postgres instance (local Docker, RDS, etc.). |
| `HL_INFO_BASE_URL` | Yes | Public: `https://api.hyperliquid.xyz/info` or testnet `https://api.hyperliquid-testnet.xyz/info`. |
| `HL_INFO_API_KEY` | No | Only if you use a **private Info** provider; obtain from that vendor (header format may differ from the default `Authorization: Bearer …` — adjust code if needed). |
| Hyperliquid signing keys / API wallet | No | Not used for read-only Info indexing. |

No on-chain wallet or HYPE stake is required for this phase.

## Setup

1. Create a database (example name `hip3_console`).

2. Copy environment template and edit:

   ```bash
   cp .env.example .env
   ```

3. Set at minimum:

   - `DATABASE_URL`
   - `HL_INFO_BASE_URL`

4. **Run migrations** — applied automatically on service startup via `sqlx::migrate!` embedded in the binary.

## Run

From the repository root:

```bash
cargo run -p hip3-console-api
```

Or release build:

```bash
cargo build -p hip3-console-api --release
./target/release/hip3-console-api
```

## HTTP endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | Liveness |
| GET | `/ready` | Readiness (Postgres ping) |
| GET | `/api/v1/dexs` | Indexed DEX metadata |
| GET | `/api/v1/markets` | Cursor-paginated markets (`dex`, `search`, `cursor`, `limit`) |
| GET | `/api/v1/markets/{asset_id}` | One market + latest snapshot fields |
| GET | `/api/v1/markets/{asset_id}/history` | Returns **501** until history is implemented |

## Operational notes

- **Indexer:** runs once at startup, then every `INDEXER_POLL_INTERVAL_SECS` (minimum 5 seconds enforced in config).
- **DEX whitelist:** `HL_INFO_DEX_WHITELIST` is comma-separated. To include the native/first DEX (empty `name` in `perpDexs`), use a **leading comma** (see [.env.example](../.env.example)).
- **CORS:** `CORS_PERMISSIVE=true` is for local development only; tighten before production.

## Verification checklist (against GitBook)

Before relying on this in production, re-check:

1. [Perpetuals Info](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals) — request/response shapes for `perpDexs` and `metaAndAssetCtxs`.
2. [Asset IDs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids) — formula for builder-deployed perps.
3. [Rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api) — tune `HL_INFO_MAX_INFLIGHT`, backoffs, and polling interval.

If GitBook changes (field names or types), update parsing in `crates/infrastructure/src/hl_client.rs` and, if needed, migrations in `migrations/`.
