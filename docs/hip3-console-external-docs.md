# Hyperliquid HIP‑3 Console & Explorer – External Documentation Guide for a Rust Backend
## Executive overview
This document is a **rules-and-docs companion** for building a Rust-based "HIP‑3 Console & Explorer" that indexes all HIP‑3 markets, exposes analytics and portfolio APIs, and offers deployer tooling and widgets, without the coding agent needing direct web access.
Hyperliquid exposes a unified Info and Exchange API for all markets (validator-operated perps, HIP‑3 builder-deployed perps, spot, outcomes), with HIP‑3-specific behavior encoded mainly in asset IDs, DEX indices, and deployer actions.[^1][^2][^3][^4]

The goal of this file is to centralize:

- Which **external docs and endpoints** matter for this product
- How HIP‑3 assets are **encoded and discovered**
- Which **environment variables and secrets** the backend needs
- How to use **REST vs WebSocket** in a high-RPS Rust API layer
- How to wire **builder codes** and **oracle/HIP‑3-as-a-Service** integrations

Coding agents should treat this as **ground truth for external behavior** and combine it with the internal `rules.md` that defines Rust workspace, architecture, and non-functional constraints.

***
## High-level architecture & external systems
At a high level, the HIP‑3 Console & Explorer interacts with three classes of systems:

1. **Hyperliquid exchange APIs (Info & Exchange)**
   - Read-only Info API for markets, positions, funding, OI, etc.[^5][^4]
   - Signed Exchange API for placing/cancelling orders when offering routed trading or signed-order helpers.[^6][^7]
   - WebSocket endpoints for streaming order books, trades, and account updates.[^8][^9]
2. **HIP‑3-specific protocol layer**
   - HIP‑3 spec: staking requirements, DEX semantics, auction rules, slashing and cross-margin guidelines.[^2]
   - Asset ID scheme for builder-deployed perps: `asset = 100000 + perp_dex_index * 10000 + index_in_meta`.[^3]
   - Info endpoints that expose metadata keyed by DEX and asset IDs (e.g., `meta`, `metaAndAssetCtxs`, `allPerpMetas`).[^10][^11][^5]
3. **Oracle and liquidity infra (primarily for deployer tooling)**
   - Pyth **HIP‑3 as a Service** for managed oracle pushing and infra.[^12]
   - Optional additional price sources (Pyth Core/Pro, SEDA, Hyperliquid’s own prices) for simulations and backtests.[^13][^12]

The Rust backend should treat Hyperliquid’s APIs as **authoritative state** for market data and user positions, and Pyth or other feeds as reference data for risk analytics and oracle configuration simulations.

***
## Hyperliquid network, base URLs, and rate limits
### Networks and base URLs
Hyperliquid exposes a public API surface for **mainnet** and **testnet** with consistent URL patterns:[^1][^8]

- Mainnet REST base (Info + Exchange): `https://api.hyperliquid.xyz`
- Testnet REST base: `https://api.hyperliquid-testnet.xyz`
- Mainnet WebSocket base: `wss://api.hyperliquid.xyz/ws`
- Testnet WebSocket base: `wss://api.hyperliquid-testnet.xyz/ws`

Third-party infra providers expose **private Info endpoints** and RPCs for higher throughput and SLAs, which are relevant for a 100k+ RPS console:

- Example high-throughput Info endpoint: `https://api-hyperliquid-mainnet-info.n.dwellir.com/info` (API-key protected).[^14]
- Data infra like HyperPC’s Data API and private Info streams can provide higher-rate Info API mirrors with routing and analytics.[^9][^15]
### Rate-limiting and design implications
The official docs emphasize that Info and Exchange endpoints use POST requests with rate limits based on **request weights**; third-party docs highlight that public endpoints are shared and subject to latency spikes during traffic bursts.[^9][^14][^15]
A production-grade explorer should:

- Prefer **read-only Info API** for market data, not Exchange endpoints.
- Use **connection pools and backoff** for public endpoints and optionally integrate **private Info endpoints** for core indexing.
- Use **WebSocket subscriptions** for hot paths (books, trades, account events) to reduce REST pressure.[^8][^9]
### Recommended env vars – network & infra
Backend configuration should externalize network and provider choices:

- `HL_NETWORK` – `mainnet` or `testnet`.
- `HL_INFO_BASE_URL` – default `https://api.hyperliquid.xyz/info` or private info endpoint.
- `HL_EXCHANGE_BASE_URL` – default `https://api.hyperliquid.xyz/exchange` (same host, different path; see SDKs/CCXT).
- `HL_WS_URL` – `wss://api.hyperliquid.xyz/ws` or provider-specific stream gateway.
- `HL_INFO_API_KEY` – optional API key for private Info endpoints (e.g., Dwellir, HyperPC).[^14][^15]

***
## Hyperliquid API surface – mental model
The Hyperliquid API is logically split into two top-level POST endpoints:[^1][^14][^6]

- `POST {BASE_URL}/info` – **Info API** (read-only).
  - Body always includes a `type` field; each value defines a different read operation.
  - Used for markets metadata, price & funding, OI, account summaries, etc.[^10][^5][^4]
- `POST {BASE_URL}/exchange` – **Exchange API** (signed actions).
  - Used for order placement, cancels, withdrawals, etc.[^6][^7]

Most HIP‑3 Console work uses the **Info API**, with optional use of Exchange for:

- Builder-code-routed trading helpers.
- Deployer console flows that trigger `haltTrading` or related actions.

SDKs in Python, Go, TypeScript, and Elixir all follow this split and show how to construct request payloads and signatures.[^16][^17][^18][^19]

***
## Info API – key endpoints for indexing and analytics
### Info endpoint basics
The Info API is accessed via:

- `POST https://api.hyperliquid.xyz/info` (mainnet) or `https://api.hyperliquid-testnet.xyz/info` (testnet).[^10][^5][^4]
- Content type: `application/json`.
- Every request body includes a `type` string that dictates the behavior.

Third-party docs illustrate this pattern with `meta`, `metaAndAssetCtxs`, and user account summaries.[^14][^5][^4][^10]
### Perpetuals metadata and contexts (core for HIP‑3 indexer)
The console’s market indexer relies heavily on the following Info types (names from docs and Chainstack mirrors):

1. `meta` – **perpetuals metadata** (universe, margin tables, leverage limits).
   - Request: `{"type": "meta"}` with optional `"dex"` field to select a specific perp DEX.[^5][^4]
   - Response includes:
     - `universe`: array of perpetual markets with fields like `name`, `szDecimals`, `pxDecimals`, etc.
     - Margin tiers and max leverage data for each asset.[^5]
   - For HIP‑3 builder-deployed DEXs, `dex` corresponds to the DEX name, letting the indexer fetch metadata per DEX.[^11][^5]

2. `metaAndAssetCtxs` – **universe + live market data**.
   - Request: `{"type": "metaAndAssetCtxs"}` (official docs; Chainstack docs show same pattern).[^10]
   - Response includes:
     - `universe` (metadata as above).
     - `assetCtxs` with fields like mark prices, funding rates, OI, 24h volume, price impact, etc.[^10]
   - This endpoint is ideal for **snapshots** used to populate dashboards and aggregates.

3. HIP‑3-specific metadata: `allPerpMetas` / HIP‑3 DEX metadata
   - Elixir SDK docs show an `AllPerpMetas` info call for "Metadata for perpetual assets across all or a specific DEX", which is similar to `meta` but takes a `dex` argument and is specifically aware of builder-deployed DEXs.[^11]
   - A HIP‑3 indexer should periodically call the equivalent functionality via the Info API (either `meta` with different `dex` values or a dedicated `allPerpMetas` type when available) to discover **all HIP‑3 DEXs and their assets**.

4. Perpetuals account summary and positions
   - Docs reference Info types to retrieve a user’s perpetuals account summary: open positions, balances, margin requirements.[^4]
   - This is used by the Console when a user connects their Hyperliquid account to show cross-DEX positions and PnL.
### Other Info types likely needed
While not all Info types are formally listed in the small excerpts, third-party documentation and SDKs show additional patterns:[^14][^20][^21]

- User-centric Info:
  - Open orders by user (optionally with `dex` filter) – used for building open-orders views per HIP‑3 DEX.[^20]
  - Subaccounts and margin breakdowns – used for advanced portfolio and risk views.[^21]
- Spot info (`spotMeta`, etc.) – needed for collateral asset discovery and cross-reference with HIP‑3 perps using those quote assets.[^3]
- Outcome markets meta – relevant when HIP‑4 is added later; same Info pattern with different meta fields.[^16][^3]

The console’s indexer should treat Info types as **extensible**, keeping an internal mapping of `type` names and response schemas in a dedicated module and storing raw JSON in Postgres when necessary to preserve forward compatibility.
### Recommended env vars – Info API
- `HL_INFO_READ_TIMEOUT_MS` – HTTP client timeout for Info calls.
- `HL_INFO_MAX_INFLIGHT` – max concurrent Info requests.
- `HL_INFO_RETRY_POLICY` – JSON/YAML describing retry and backoff (or separate envs for backoff parameters).
- `HL_INFO_DEX_WHITELIST` – comma-separated DEX names the indexer should treat as HIP‑3; used to avoid scanning test or experimental DEXs.

***
## Exchange API – trading, deployer actions, and builder codes
### Exchange endpoint basics
The Exchange endpoint is used for **state-changing actions** on Hyperliquid:[^6][^7]

- `POST https://api.hyperliquid.xyz/exchange`.
- Requests contain a JSON payload describing the action (e.g., order placement) and must be **signed** with the API wallet’s private key or a vault key.
- SDKs show helpers for generating the correct signatures and encodings (e.g., Python and Go clients).[^16][^17]

For the Console & Explorer, Exchange is needed only for:

- Placing/cancelling orders on behalf of users (routed through API wallets + builder codes).
- Deployer actions like `haltTrading` when a deployer uses the console as their control plane.[^2]
### HIP‑3 deployer actions
Hyperliquid’s docs have a dedicated page for **HIP‑3 deployer actions**, which are L1 actions used to deploy and operate builder-deployed perpetual DEXs.[^22][^2]
These are not typical user trading actions, but they matter for the **deployer tooling** side of the product:

- Staking HYPE to deploy a DEX.
- Listing/auctions for new markets.
- `haltTrading` to stop trading and settle positions at mark price.[^2]

The console should **not** directly perform staking or listing on behalf of deployers (those remain in the deployer’s L1 wallet workflows), but it should:

- Surface the parameters required for these actions.
- Generate payloads or transactions for deployers to sign in their wallets.
- Provide slashing and cross-margin risk analyses before a deployer executes such actions.
### Builder codes – revenue share integration
Builder codes are a central revenue mechanism for a B2C/B2B console, enabling per-order revenue share when routing volume:[^23][^24][^25][^26]

- Builder codes are on-chain identifiers that can be attached to orders.
- Users approve a max builder fee for each builder; the fee (up to 0.1% on perps) is taken from the base trading fee.[^24][^26]
- Builder codes apply across **all markets**, including HIP‑3, and revenue is tracked on-chain and claimed via the standard referral reward flow.[^25][^23]

The console backend must:

- Include the builder code field in Exchange request payloads when placing orders.
- Track **per-user approval status** and max builder fee (via on-chain data or user settings from the UI).[^25]
- Aggregate and expose **builder-code revenue metrics** for internal monitoring and B2B reporting.
### Recommended env vars – Exchange & builder codes
- `HL_API_WALLET_PRIVATE_KEY` – private key for the API wallet (used for signed actions; store securely).
- `HL_API_WALLET_ADDRESS` – public address of the API wallet.
- `HL_BUILDER_CODE` – builder code identifier used in order payloads.[^23][^24]
- `HL_BUILDER_MAX_FEE_BP` – default max builder fee (basis points) to propose to users.
- `HL_EXCHANGE_READ_TIMEOUT_MS` – timeout for Exchange responses.

***
## HIP‑3 protocol details relevant for product behavior
### Staking, DEX limits, fee mechanics
The HIP‑3 spec defines the core economic and operational rules for builder-deployed perps:[^2][^16][^27]

- **Staking requirement** – A deployer must stake **500k HYPE** on mainnet to deploy one perp DEX; this requirement is expected to decrease as infrastructure matures.[^27][^2]
- **DEX scope** – Each deployer gets one perp DEX, with independent margining, order books, and settings.[^2]
- **Initial markets** – The first **3 assets deployed** in any perp DEX do **not** require auction participation; additional assets must be acquired via a shared Dutch auction (same hyperparameters as HIP‑1 auction).[^2]
- **Fee split** – From the deployer’s perspective, fee share is a fixed **50% of trading fees** on their DEX; from the user perspective, HIP‑3 fees are **2x validator-operated perps**, leaving protocol revenue unchanged.[^2]

The console should expose these details in deployer dashboards and risk tools, and incorporate them into revenue simulations and educational content for deployers.
### Asset ID scheme for HIP‑3 markets
Asset IDs are integral to treating HIP‑3 markets as part of a unified universe:[^2][^3]

- For validator-operated perps, `asset` is the index in the `universe` array from the `meta` response (e.g., `BTC = 0` on mainnet).[^3]
- For builder-deployed perps (HIP‑3), the asset ID is:
  
  \( asset = 100000 + perp\_dex\_index \times 10000 + index\_in\_meta \)[^28]
  
  Example from testnet: `test:ABC` has `perp_dex_index = 1`, `index_in_meta = 0`, hence `asset = 110000`.[^3]

Effective indexer behavior:

- Use `meta` or `allPerpMetas` with `dex` filters to get `universe` entries for each HIP‑3 DEX.[^11][^5]
- Compute asset IDs using the formula above and store them in the DB as primary keys for HIP‑3 markets.
- Recognize builder-deployed markets by their naming convention `{dex}:{coin}`.[^3]
### Settlement, `haltTrading`, and recycling markets
HIP‑3 allows deployers to **settle** and **recycle** assets using the `haltTrading` action:[^2]

- `haltTrading` cancels all orders and settles positions to the current mark price for the asset.
- The same action can be used to resume trading, enabling patterns like dated contracts or event-based markets without full re-auctions.[^2]
- Once all assets are settled, a deployer’s staked HYPE can be unstaked after a retention period.

The console’s deployer section should surface:

- Current `haltTrading` status for each asset.
- Simulated impact of halting on open interest and PnL.
- UX flows that prepare `haltTrading` payloads for deployers to sign.
### Slashing and cross-margin eligibility
Slashing is central to HIP‑3 safety and should be treated as a first-class concept in the risk tooling:[^2]

- Validators can slash up to **100%** of a deployer’s stake for irregular inputs that cause invalid state transitions or prolonged downtime.[^2]
- Up to **50%** can be slashed for irregular inputs causing brief downtime; up to **20%** for invalid inputs causing degradation or performance issues.[^2]
- Slashing is independent of whether behavior was malicious or incompetent; only **effects on protocol correctness and uptime** matter.[^2]

Cross-margin eligibility introduces additional constraints:[^2]

- Only assets with **sufficient liquidity, reliable external oracles, and resilience to price manipulation** can be cross-margined.
- If `externalPerpPx` moves more than **50% relative to start-of-day price**, validators review the asset for potential slashing due to manipulation.[^2]
- Assets that are expected to experience **50% daily moves more than once a month** are **ineligible for cross margin** and may trigger slashing if treated otherwise.[^2]

Implications for the console’s risk engine:

- Implement metrics for **50% daily move detection** on external oracles.
- Tag assets as **cross-margin eligible/ineligible** based on historical volatility and validator guidelines.
- Trigger **alerts** when volatility patterns approach or exceed cross-margin thresholds.
### Recommended env vars – HIP‑3 risk logic
- `HIP3_VOL_LOOKBACK_DAYS` – window for volatility and 50% move analysis.
- `HIP3_SLASHING_ALERT_THRESHOLD_BP` – basis-points threshold before firing severity-graded alerts.
- `HIP3_MAX_CROSS_MARGIN_VOL` – configuration for cross-margin eligibility heuristics.

***
## WebSocket API – real-time data streams
### WebSocket endpoints and connection model
Hyperliquid provides WebSocket endpoints on the same host as the REST API:[^8]

- Mainnet: `wss://api.hyperliquid.xyz/ws`.
- Testnet: `wss://api.hyperliquid-testnet.xyz/ws`.

Docs and SDKs show that:

- The WebSocket connection is used for **subscription-style real-time updates** (order books, trades, account events).
- Clients can also **post Info or Exchange requests** over WebSocket for lower latency vs HTTP.[^17]
- There is a limit of around **1000 WebSocket subscriptions per IP address**, as highlighted in a JS SDK README, which matters for large indexers.[^29]
### Using WebSocket for a HIP‑3 explorer
The console should use WebSockets to:

- Subscribe to **order book updates** and **trades** for top HIP‑3 markets.
- Subscribe to **account events** (positions, notifications) for connected user accounts.
- Optionally offload some Info queries to WebSocket if the SDK supports it.[^17]

Design considerations:

- Implement a **subscription budget** to avoid hitting the 1000-subscription limit per IP.[^29]
- Use per-service WebSocket clients (indexer vs portfolio vs risk engine) behind a shared multiplexing abstraction.
- Reconnect with exponential backoff and resubscribe on connection loss.
### Recommended env vars – WebSocket
- `HL_WS_MAX_SUBSCRIPTIONS` – global subscription cap per node (≤ documented limit).[^29]
- `HL_WS_HEARTBEAT_INTERVAL_MS` – ping interval.
- `HL_WS_RECONNECT_BACKOFF_MS` – base backoff for reconnect attempts.

***
## Oracle and Pyth HIP‑3-as-a-Service integration
### Pyth HIP‑3 as a Service – overview
Pyth’s "HIP‑3 as a Service" offering is designed for deployers who want to offload oracle operations while retaining full HIP‑3 market control:[^12]

- Provides **first-party data**, managed infra, and capital/liquidity support.
- Ensures **continuous oracle updates every 3 seconds**, which is a practical requirement for HIP‑3 markets.[^12]
- Offers **fallback configurations** and multi-region redundancy to reduce downtime and irregular inputs risk.[^12]

For the Console & Explorer, Pyth integration is relevant mainly for:

- Deployer console flows that configure **oracle sources**.
- Simulating and monitoring **oracle performance** and **update cadence**.
### Pusher service model
Pyth describes a generic HIP‑3 pusher service pipeline:[^12]

1. **Data sourcing** – Subscribe to price data from REST/WebSocket sources (Pyth Core, Pyth Pro, SEDA, Hyperliquid, or custom feeds).
2. **Robustness & redundancy** – Chain multiple sources with fallback behavior.
3. **Submission** – Construct and submit signed transactions to Hyperliquid validators to update oracle state.

The console does not need to operate the pusher itself unless offering a **self-hosted oracle module**; instead, it should:

- Model the pusher’s **update frequency**, **latency**, and **failures** using telemetry.
- Alert deployers when **update intervals exceed thresholds** or **data sources stall**.
### Recommended env vars – oracle integrations
- `PYTH_HIP3_ENABLED` – whether the console should expose Pyth HIP‑3-as-a-Service configuration.
- `PYTH_API_KEY` – API key or auth token for Pyth’s endpoints (if required).
- `PYTH_PRIMARY_FEED_IDS` – list of Pyth price feed IDs used by connected markets.
- `ORACLE_MAX_UPDATE_INTERVAL_MS` – max tolerated time between oracle updates before alerts.

***
## Wallets, API wallets, and auth model
### API wallets (agent wallets)
Hyperliquid uses **API wallets (agent wallets)** to let off-chain bots and apps act on behalf of a user account without withdrawal rights.[^30][^31][^32][^33]

Key properties:

- API wallets are created in the UI under **More → API** or at `https://app.hyperliquid.xyz/API`.[^31]
- Each API wallet has a **private key** (API secret) and linked main account address; the private key is shown only once.[^33][^31]
- API keys usually have a **maximum lifetime of 180 days** and can expire based on conditions like zero funds or liquidations.[^32][^33]

Third-party guides detail the flow:

- User logs in with a Web3 wallet.
- Navigates to the API page and generates an API wallet.
- Stores **private key**, API wallet address, and main account address.
- Authorizes the API wallet with a signed transaction.[^31][^32]

For the console’s backend:

- `HL_API_WALLET_PRIVATE_KEY` must be stored in a secure secret store, not in code.
- `HL_MASTER_ACCOUNT_ADDRESS` can be used to query Info endpoints for that account’s positions.
### User auth inside the console
The console’s own auth system is independent of Hyperliquid:

- Users can create accounts using email/password; the backend should use **Argon2 or scrypt** for hashing.[^33]
- Alternatively, wallet-based login (e.g., via Phantom) can be used for direct Web3 auth.[^34][^35]
- For API access, use short-lived **JWTs** with roles for **viewer**, **trader**, and **deployer**.

Recommended env vars:

- `AUTH_JWT_SECRET` – signing key for JWTs.
- `AUTH_PASSWORD_HASH_MEMORY_COST`, `AUTH_PASSWORD_HASH_TIME_COST` – parameters for Argon2.
- `AUTH_WEB3_LOGIN_ENABLED` – toggle for wallet-based login flows.

***
## Database and caching implications from external APIs
### Schema drivers from Info API
The Info API’s `meta`, `metaAndAssetCtxs`, and account endpoints drive the DB schema design:

- Market tables keyed by `asset_id`, `dex_name`, and `symbol` (e.g., `dex:coin`).[^10][^3][^5]
- Time-series tables for **funding**, **OI**, **volume**, and **prices**, with columns sourced from asset contexts.[^29][^10]
- User-asset views (positions, PnL) keyed by `(user_address, asset_id)` using Info account summary responses.[^4]

Third-party tooling like Artemis shows that hyperliquid trade data includes fields like `order_coin`, `order_side`, `leverage`, `startPosition`, etc., which can guide a normalized schema for trade history tables.[^36][^29]
### Caching and latency
Given that Info endpoints are read-only but POST-based and rate-limited, caching is essential:[^9][^14][^15]

- Use Redis for **hot market snapshots**, keyed by `(dex_name, asset_id)` with TTLs of a few seconds.
- Cache results of Info calls like `metaAndAssetCtxs` for use in multiple dashboard views.
- Store precomputed leaderboards and aggregates with TTLs tuned to trading frequency.

Recommended env vars:

- `REDIS_URL` – connection URL for Redis cluster.
- `REDIS_CACHE_TTL_MARKET_S` – TTL for market snapshots.
- `REDIS_CACHE_TTL_LEADERBOARD_S` – TTL for leaderboard caches.

***
## Builder codes & B2B instrumentation
### Revenue characteristics
Analyses of the builder-code ecosystem show their growing economic significance:

- As of early 2026 there are **80+ active builders** with cumulative revenue above **$60M** and 30-day revenue around **$5.6M**, indicating strong B2B potential.[^37]
- Builder fees up to **0.1% on perps** and **1% on spot** can be charged per order, within user-approved caps.[^24][^26]

The console should:

- Instrument all routed orders with **builder code attribution**.
- Provide per-partner and per-market revenue dashboards.
- Generate reports suitable for potential acquirers and investors.
### Recommended env vars
- `BUILDER_REPORTING_DB_SCHEMA` – separate schema or DB for builder revenue analytics.
- `BUILDER_PAYOUT_REPORT_CRON` – cron expression for payout summary jobs.

***
## External analytics and monitoring data sources
While Hyperliquid’s own APIs serve core data, external analytics platforms provide complementary views:

- Artemis tracks Hyperliquid perp volume, OI, fees, and revenue across HIP‑3 integrated protocols, useful for ranking DEXs and validating internal metrics.[^29]
- Research and ecosystem overviews document HIP‑3 projects and Builder Code revenue, serving as reference for project listings and case studies.
[^38][^22][^13][^39][^30][^40][^37]

These data sources are not primary for the backend, but the console may:

- Ingest them periodically for **meta-analytics** (e.g., comparing HIP‑3 DEX volume vs entire Hyperliquid).
- Use them to populate **marketing and research sections** in the UI.

Recommended env vars:

- `EXTERNAL_ANALYTICS_ENABLED` – toggle ingestion from Artemis or similar.
- `EXTERNAL_ANALYTICS_API_KEY` – auth token for those APIs.

***
## Minimal endpoint reference for coding agents
This section lists the minimal set of concrete external endpoints and patterns coding agents should treat as canonical when implementing integrations.
It does not include full schemas, but IDs and method names should be respected exactly.
### REST Info endpoints
- `POST {HL_INFO_BASE_URL}` with JSON body:
  - Perpetuals meta: `{ "type": "meta" }` plus optional `"dex": "<dex_name>"`.[^5][^4]
  - Universe + asset contexts: `{ "type": "metaAndAssetCtxs" }`.[^10]
  - Perptuals account summary: `{ "type": "perpSummary", "user": "<address>" }` (name from docs/SDK patterns; treat as canonical placeholder until refined against SDKs).[^14][^4]
### REST Exchange endpoints
- `POST {HL_EXCHANGE_BASE_URL}` with JSON body describing action, signed using API wallet:
  - Place order: action structure as per official SDKs (fields like `asset`, `isBuy`, `sz`, `limitPx`, `reduceOnly`, `builderCode`, etc.).[^16][^17][^7][^26]
  - Cancel order: payload including `asset` and `cloid` (client order ID).[^20][^7]
  - Deployer actions (HIP‑3): use official `hip-3-deployer-actions` schema, particularly `haltTrading` for settlements.[^22][^2]
### WebSocket
- Connect to `HL_WS_URL` and send subscription messages in JSON as per official WebSocket docs and SDK patterns; common channels include:
  - Order books per asset.
  - Trades per asset.
  - User notifications and account updates (positions, balances).[^8][^17]
### Asset IDs
- Use `meta` `universe` index for validator-operated perps.
- Use `asset = 100000 + perp_dex_index * 10000 + index_in_meta` for HIP‑3 assets; names follow `{dex}:{coin}`.[^3]

***
## Summary – what coding agents must remember
1. The **Info API** at `POST /info` is the main source of market and account data; everything is driven by the `type` field, especially `meta`, `metaAndAssetCtxs`, and per-user summary calls.[^10][^5][^4]
2. HIP‑3 markets are distinguished primarily by their **asset ID encoding** and **DEX names**; they share the same trading API as other perps, so all trading/risk tooling should treat them as a superset of Hyperliquid perps.[^2][^3]
3. The console’s main integrations are: **Hyperliquid Info/Exchange**, **WebSocket streams**, **Pyth HIP‑3 as a Service**, **builder codes**, and optional external analytics.
4. All secrets and network configuration (API wallet key, Info base URL, WebSocket URL, Pyth keys, Redis/DB credentials) must come from environment variables or secret stores, never hardcoded.
5. Risk tooling must encode HIP‑3 rules on **slashing**, **50% daily moves**, and **cross-margin eligibility**, making these constraints explicit in deployer-facing simulations and alerts.[^2]

Coding agents should use this document as the external reference when implementing Rust services and treat any discrepancies as requiring manual review of the relevant upstream docs or SDKs.

---

## References

1. [API - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api) - API. Documentation for the Hyperliquid public API. Python SDK: https://github.com/hyperliquid-dex/hy...

2. [HIP-3: Builder-deployed perpetuals - Hyperliquid Docs](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-3-builder-deployed-perpetuals) - The Hyperliquid protocol supports permissionless builder-deployed perps (HIP-3), a key milestone tow...

3. [Asset IDs - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids) - Asset IDs are the integer representation for sending orders and cancels via actions. See here for mo...

4. [Perpetuals - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals) - The section documents the info endpoints that are specific to perpetuals. See Rate limits section fo...

5. [meta | Hyperliquid info](https://docs.chainstack.com/reference/hyperliquid-info-meta) - The info endpoint with type: "meta" retrieves perpetuals metadata including universe and margin tabl...

6. [Exchange endpoint - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint) - The exchange endpoint is used to interact with and trade on the Hyperliquid chain. See the Python SD...

7. [hyperliquid](https://docs.ccxt.com/exchanges/hyperliquid) - Instead of trading to increase the address based rate limits, this action allows reserving additiona...

8. [Websocket - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket) - WebSocket endpoints are available for real-time data streaming and as an alternative to HTTP request...

9. [A Guide to Hyperliquid Data API vs. Streams](https://hyperpc.app/blog/hyperliquid-data-api-vs-streams-guide) - Public endpoints are shared infrastructure. When traffic spikes, so does your latency. A private gat...

10. [metaAndAssetCtxs | Hyperliquid info](https://docs.chainstack.com/reference/hyperliquid-info-meta-and-asset-ctxs) - Retrieve comprehensive asset contexts for perpetuals trading, including universe metadata and real-t...

11. [Hyperliquid.Api.Info.AllPerpMetas](https://hexdocs.pm/hyperliquid/0.2.1/Hyperliquid.Api.Info.AllPerpMetas.html) - Metadata for perpetual assets across all or a specific DEX. This is similar to the meta endpoint but...

12. [HIP-3 as a Service - Developer Hub - Pyth Network](https://docs.pyth.network/price-feeds/hip-3-service) - Pyth Network provides HIP-3 as a Service, a complete end-to-end solution for deployers launching per...

13. [Getting Started | Pyth Developer Hub](https://docs.pyth.network/price-feeds/pro/getting-started) - This guide will walk you through setting up and running a basic JavaScript example to subscribe to P...

14. [Complete Hyperliquid Infrastructure Stack for Developers](https://www.dwellir.com/blog/hyperliquid-infrastructure-stack) - Info Endpoint: REST API for Market Data. The Info Endpoint serves as Hyperliquid's primary REST API ...

15. [Hyperliquid Data API for production apps](https://hyperpc.app/products/data/data-api) - Higher Rate Limits. Designed for production workloads. Throughput limits significantly higher than p...

16. [SDK for Hyperliquid API trading with Python.](https://github.com/hyperliquid-dex/hyperliquid-python-sdk) - Generate and authorize a new API private key on https://app.hyperliquid.xyz/API, and set the API wal...

17. [hyperliquid package - github.com/slicken/go_hyperliquid](https://pkg.go.dev/github.com/slicken/go_hyperliquid) - A high-performance Go SDK for the Hyperliquid API, optimized for low-latency trading with automatic ...

18. [nomeida/hyperliquid: Simple and easy way to access the ...](https://github.com/nomeida/hyperliquid) - All info on the Hyperliquid API can be found here: HyperLiquid API Documentation. Features. Complete...

19. [API Reference — hyperliquid v0.2.2](https://hexdocs.pm/hyperliquid/api-reference.html) - Root API module providing unified access to all Hyperliquid endpoints. Hyperliquid.Api.Common. Commo...

20. [hyperliquid/docs/API.md at main](https://github.com/carter2099/hyperliquid/blob/main/docs/API.md) - Client Order IDs (Cloid). Client order IDs must be 16 bytes in hex format ( 0x + 32 hex characters)....

21. [Hyperliquid Info Endpoints](https://docs.coinmarketman.com/hyperliquid-info-endpoints) - The Info Endpoints allow you to query the state of the Hyperliquid exchange, specific user accounts,...

22. [HIP-3 deployer actions - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/hip-3-deployer-actions) - HIP-3 deployer actions. The API for deploying and operating builder-deployed perpetual dexs involves...

23. [Builder Codes](https://docs.privy.io/recipes/hyperliquid/builder-codes) - Builder codes enable order-level revenue sharing - completely independent of which market the order ...

24. [Builder Codes and the Future of Hyperliquid as Infra](https://x.com/0xBroze/status/1938603649917153661) - With Hyperliquid Builder Code revenue nearing $10m, a breakdown is needed on the emerging builder co...

25. [Builder codes - Hyperliquid Docs - GitBook](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/builder-codes) - Builder codes allow builders to receive a fee on fills that they send on behalf of a user. They are ...

26. [Hyperliquid Builder Codes: Rewarding DeFi Innovation](https://okto.tech/blog/hyperliquid-builder-codes-rewarding-defi-innovation) - Builder codes let you earn a small fee every time someone uses your tool to trade. It's like a thank...

27. [Hyperliquid announces HIP-3 feature on testnet, allowing ...](https://cryptorank.io/news/feed/7a03f-hyperliquid-responds-to-the-challenge-prepares-for-builder-deployed-markets) - The platform will allow builders to turn any market into a perpetual futures pair, tapping a wider a...

28. [Hyperliquid.Api.Info](https://hexdocs.pm/hyperliquid/Hyperliquid.Api.Info.html) - Search documentation of hyperliquid. Settings. View Source Hyperliquid.Api.Info (hyperliquid v0.2.2)...

29. [README.md - Hyperliquid API SDK](https://github.com/nomeida/hyperliquid/blob/main/README.md) - The Hyperliquid API imposes a limit of 1000 WebSocket subscriptions per IP address. The SDK automati...

30. [API Wallet](https://app.hyperliquid.xyz/API) - API wallets (also known as agent wallets) can perform actions on behalf of your account without havi...

31. [Hyperliquid](https://hummingbot.org/exchanges/hyperliquid/) - In the top menu, go to More → API, or open: https://app.hyperliquid.xyz/API. Create an API wallet. E...

32. [Hyperliquid](https://docs.tealstreet.io/docs/connect/hyperliquid) - Hyperliquid requires you to deposit funds before you can create an API Key. So make sure you take ca...

33. [Connecting Hyperliquid to MoonTrader](https://docs.moontrader.com/hyperliquid-en) - This section describes the process of connecting the HyperLiquid exchange to the MoonTrader terminal...

34. [Quickstart](https://docs.privy.io/recipes/hyperliquid-guide) - To withdraw funds from Hyperliquid back to your wallet, use the withdraw3 method. Funds will be cred...

35. [How to trade perpetuals on Hyperliquid exchange from the ...](https://www.reddit.com/r/defi/comments/1mwja6w/how_to_trade_perpetuals_on_hyperliquid_exchange/) - To on-board to HL I believe u need a wallet that supports the Arbitrum chain. No kyc should be neede...

36. [Hyperliquid - Artemis API Docs](https://app.artemis.xyz/docs/snowflake-share/tables/hyperliquid) - Client-supplied order ID for tracking the order externally. order_coin, string, The base asset for t...

37. [Exploring Hyperliquid's Million-Dollar Builder Ecosystem](https://www.techflowpost.com/en-US/article/30829) - As of March 2026, the Hyperliquid Builder Code ecosystem comprises over 80 active Builders, with his...

38. [Getting started | Hyperliquid](https://docs.chainstack.com/reference/hyperliquid-getting-started) - The Hyperliquid API allows developers to interact with the Hyperliquid DEX to build trading applicat...

39. [quiknode-labs/hyperliquid-api-examples](https://github.com/quiknode-labs/hyperliquid-api-examples) - Clone-and-run examples for the Hyperliquid API builder API. Trade perps, spot, and HIP-3 markets on ...

40. [Hyperliquid improvement proposals: What is HIP-3?](https://phantom.com/learn/crypto-101/hyperliquid-hip-3) - HIP-3, or “Builder-Deployed Perpetuals,” enables anyone staking 1 million HYPE to permissionlessly c...

