# How to Be a Quant for Prediction Markets — Complete Roadmap (Rust + AWS)

Prediction markets like Polymarket and Kalshi are not gambling platforms. They are **Central Limit Order Book (CLOB) exchanges** where binary contracts (YES/NO) trade between $0 and $1, settling based on real-world event outcomes. A YES contract priced at $0.60 means the market collectively believes there's a 60% probability of that event happening.

This creates an enormous playground for quantitative traders. If your probability model says the true probability is 70% but the market is pricing it at 60%, you have a **10-cent edge** on every contract. Multiply that by thousands of trades, and you're running a systematic, data-driven operation.

This article is a **complete, no-BS technical roadmap** for building a quant system for prediction markets using **Rust** and **AWS**. Every layer — from math, to code, to infrastructure — is covered with the exact tools and services you need.

---

## 1. Understand the Prediction Market Infrastructure

Before writing a single line of code, you need to understand **how the exchange actually works**.

**Polymarket's Architecture:**

| Component | How It Works |
| :-- | :-- |
| **Order Book** | Central Limit Order Book (CLOB) — same structure as NASDAQ or Binance |
| **Order Matching** | Off-chain for speed (Web2 performance), on-chain settlement on Polygon for security (Web3 trust) |
| **Contract Structure** | Binary YES/NO pairs. Always settle to $1.00 total. If YES = $0.42, NO = $0.58 |
| **Order Types** | GTC (Good-Til-Cancelled), GTD (Good-Til-Date), FOK (Fill-Or-Kill), FAK (Fill-And-Kill) |
| **Fees** | Maker = 0 fees (your limit order sits on book). Taker = fees apply (you hit someone's order) |
| **Signing** | Orders are signed using EIP-712 typed structured data on Polygon (chain ID 137) |

**Why this matters:** You're not betting against a house. You're trading peer-to-peer on an order book. This means **spread capture, arbitrage, and market making** are all viable strategies — exactly like on a traditional exchange.

**Data Feeds Available:**

| Feed | Latency | Use Case |
| :-- | :-- | :-- |
| WebSocket (`wss://ws-subscriptions-clob.polymarket.com`) | ~100ms | Real-time order book, price changes |
| Gamma API | ~1s | Market metadata, resolution criteria |
| REST CLOB API (`https://clob.polymarket.com`) | Variable | Place/cancel orders, fetch order book |
| On-chain | Block time | Settlement verification, resolution |

---

## 2. The Math You Actually Need

You don't need a PhD. But you need these four pillars — and you need to understand **why** each one matters.

### Pillar 1: Probability Estimation (Finding Your Edge)

The entire game is: **Can you estimate the true probability of an event better than the market?**

The market price IS the crowd's probability estimate. Prediction market prices are a wealth-weighted average of beliefs among traders and closely approximate the mean belief under most conditions.

Your edge comes from estimating probability better. Methods:

- **Bayesian Updating** — Start with a prior belief, update as new evidence arrives. Every news article, every data point shifts your posterior probability. This is the fundamental framework.
- **News Sentiment (NLP)** — Quantitative analysts increasingly use NLP and sentiment analysis on news feeds to gain edges. FinBERT and similar models can score news sentiment in real-time, giving you signals before the market fully prices in information.
- **Base Rate Analysis** — Historical frequency of similar events. If a certain type of political outcome happened 73% of the time historically, that's your starting prior.
- **Ensemble Models** — Combine multiple probability estimates (polls, models, expert forecasts) weighted by historical accuracy.

### Pillar 2: Kelly Criterion (How Much to Bet)

Once you have an edge, the Kelly Criterion tells you the **optimal fraction of your bankroll to risk** on each trade:

$$
f^* = \frac{P_{true} - P_{market}}{1 - P_{market}}
$$

**Example:** Your model says 55% probability. Market prices at $0.48.

$$
f^* = \frac{0.55 - 0.48}{1 - 0.48} = \frac{0.07}{0.52} = 13.4\% \text{ of capital}
$$

**Critical rule: Never use full Kelly.** Full Kelly produces 50%+ drawdowns. Use **fractional Kelly (25-50%)** — you sacrifice only 25% of growth rate but eliminate 75% of variance. Practitioners typically use 25% Kelly for model uncertainty.

### Pillar 3: Market Making (Avellaneda-Stoikov Model)

If you want to provide liquidity and earn the spread, you need the **Avellaneda-Stoikov framework** adapted for binary outcomes:

- Calculate a **reservation price** (your fair value adjusted for inventory risk)
- Place a **bid** at reservation price minus half the optimal spread
- Place an **ask** at reservation price plus half the optimal spread
- Dynamically adjust based on your inventory position — if you're holding too many YES contracts, skew your quotes to encourage selling

The key adaptation for prediction markets: contracts settle at exactly $0 or $1, which creates **hard boundaries** and different risk dynamics than equity market making.

### Pillar 4: Arbitrage (Risk-Free Money)

The cleanest edge in prediction markets:

| Type | How It Works | Example |
| :-- | :-- | :-- |
| **Intra-Market** | Buy YES + NO on same platform when total < $1.00 | YES=$0.48 + NO=$0.49 = $0.97 → guaranteed $1.00 payout |
| **Cross-Platform** | Buy YES on Polymarket + NO on Kalshi when combined < $1.00 | Poly YES=$0.42 + Kalshi NO=$0.56 = $0.98 → $1.00 payout (2.04% return) |
| **Resolution Bonding** | Buy resolved markets still "in review" at discount | UMA oracle resolves → Polymarket lags → buy YES at $0.95 for $1.00 payout |

Cross-platform arbitrage between Polymarket and Kalshi has been documented to produce consistent small returns, with opportunities actually **peaking** during high-volatility periods like elections.

---

## 3. The Rust Technical Stack (Why Rust, and Exactly What Crates)

**Why Rust and not Python?** Three words: **zero-cost abstractions.** Rust gives you C/C++ level performance with memory safety guarantees. No garbage collector pauses. No runtime overhead. For a trading system where every microsecond counts, this is non-negotiable.

Rust lets you build **deterministic event loops, lock-free concurrency, and memory-safe buffer management** — all critical for a trading engine. The leading HFT firms are increasingly adopting Rust alongside C++ for their trading engines.

Here is the **exact Cargo.toml** you need:

```toml
[dependencies]
# Async Runtime
tokio = { version = "1", features = ["full"] }

# WebSocket Client (for real-time order book feeds)
tokio-tungstenite = "0.21"
futures-util = "0.3"

# HTTP Client (for REST API calls)
reqwest = { version = "0.12", features = ["json"] }

# JSON Serialization/Deserialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Blockchain (Polygon/EVM interaction + EIP-712 signing)
alloy = { version = "0.9", features = ["full"] }

# Lock-Free Data Structures (for concurrent order book)
crossbeam = "0.8"
dashmap = "6"

# Math / Statistics
statrs = "0.17"           # Statistical distributions
nalgebra = "0.33"         # Linear algebra

# Logging & Tracing
tracing = "0.1"
tracing-subscriber = "0.3"

# Time
chrono = "0.4"
```

### What Each Crate Does and Why:

| Crate | Purpose | Why This One? |
| :-- | :-- | :-- |
| `tokio` | Async runtime | Industry standard. Handles thousands of concurrent connections on a single thread |
| `tokio-tungstenite` | WebSocket client | Connects to Polymarket's `wss://ws-subscriptions-clob.polymarket.com` for real-time order book updates |
| `reqwest` | HTTP client | For REST API calls (place orders, cancel orders, fetch market data) |
| `serde` + `serde_json` | JSON parsing | Polymarket's API returns JSON. Using typed structs with `#[derive(Deserialize)]` is significantly faster than parsing into generic `Value` types |
| `alloy` | Ethereum/Polygon interaction | The successor to `ethers-rs` (now deprecated). Handles EIP-712 signing, transaction building, and Polygon RPC calls |
| `crossbeam` | Lock-free concurrency | Provides lock-free queues, epoch-based garbage collection. Essential for sharing data between the WebSocket reader thread and the trading logic thread without locks |
| `dashmap` | Concurrent hashmap | Thread-safe hashmap for storing order book state. Internally manages locking, far faster than `Arc<Mutex<HashMap>>` |

### Core Architecture (5 Components):

```
┌─────────────────────────────────────────────────────┐
│                  TRADING ENGINE                       │
│                                                       │
│  ┌──────────────┐    ┌──────────────┐                │
│  │  WS Data     │───▶│  Pricing     │                │
│  │  Handler     │    │  Engine      │                │
│  │  (tokio +    │    │  (Bayesian   │                │
│  │  tungstenite)│    │  + Stoikov)  │                │
│  └──────────────┘    └──────┬───────┘                │
│                              │                        │
│  ┌──────────────┐    ┌──────▼───────┐                │
│  │  Risk        │◀──▶│  Order       │                │
│  │  Controller  │    │  Manager     │                │
│  │  (Kelly +    │    │  (reqwest +  │                │
│  │  inventory)  │    │  alloy)      │                │
│  └──────────────┘    └──────────────┘                │
│                                                       │
└─────────────────────────────────────────────────────┘
```

**Component 1: WebSocket Data Handler**

This is the "ears" of your bot. It maintains a persistent WebSocket connection to Polymarket and keeps a **local copy of the order book** updated in real-time:

```rust
// Connect to Polymarket WebSocket
let url = "wss://ws-subscriptions-clob.polymarket.com/ws/market";
let (ws_stream, _) = tokio_tungstenite::connect_async(url).await?;
let (write, read) = ws_stream.split();

// Subscribe to order book updates
let subscribe_msg = serde_json::json!({
    "type": "market",
    "assets_ids": [token_id]
});
write.send(Message::Text(subscribe_msg.to_string())).await?;

// Process updates on dedicated thread
tokio::spawn(async move {
    read.for_each(|msg| async {
        if let Ok(Message::Text(data)) = msg {
            let update: OrderBookUpdate = serde_json::from_str(&data)?;
            order_book.apply_update(update); // Lock-free update via crossbeam
        }
    }).await;
});
```

Best practices: implement reconnection with exponential backoff, respond to ping messages, track sequence numbers to detect missed messages.

**Component 2: Pricing Engine**

Takes the local order book + your probability model and outputs: fair value, optimal bid, optimal ask. This is where Bayesian updating and the Avellaneda-Stoikov model live.

**Component 3: Order Manager**

Places and cancels orders via the CLOB REST API. All orders are signed with EIP-712 before submission. You interact with `https://clob.polymarket.com` — POST to `/orders` to place, DELETE to `/orders/{id}` to cancel.

**Component 4: Risk Controller**

Monitors inventory exposure, P&L, position limits. Enforces Kelly sizing. Kills the bot if drawdown exceeds threshold.

**Component 5: Event Processor**

Ingests external signals — news feeds, sentiment scores, resolution events — and feeds them to the Pricing Engine to update probability estimates.

---

## 4. AWS Infrastructure (Exactly What Services, Why, and How)

Your Rust trading engine needs a home. Here's the **exact AWS setup** optimized for low-latency prediction market trading.

### The Critical Decision: Region Selection

**Use `us-east-1` (N. Virginia).** Polymarket's infrastructure runs on AWS. You want your trading engine as physically close to their servers as possible. The difference between same-region and cross-region can be 50-200ms vs. <1ms.

### Compute: EC2 with Cluster Placement Groups

This is the most important infrastructure decision. A **Cluster Placement Group** packs your EC2 instances onto the same physical rack, giving you the absolute lowest network latency between instances.

```bash
# Create the placement group
aws ec2 create-placement-group \
  --group-name prediction-quant-cluster \
  --strategy cluster \
  --tag-specifications 'ResourceType=placement-group,Tags=[{Key=Name,Value=pred-quant}]'
```

**Instance Selection:**

| Instance | vCPUs | RAM | Network | Use Case | Why? |
| :-- | :-- | :-- | :-- | :-- | :-- |
| `c5n.xlarge` | 4 | 10.5 GB | Up to 25 Gbps | Trading Engine | Networking-optimized. The `n` variant has enhanced networking (ENA), critical for lowest latency |
| `c5n.large` | 2 | 5.25 GB | Up to 25 Gbps | Data Processor | Handles news feeds, sentiment analysis, secondary computations |

**Why `c5n` specifically?** The `c5n`, `m5n`, and `r5n` families are AWS's networking-optimized instances. They work best in cluster placement groups and have higher bandwidth than their non-`n` counterparts. Coinbase used similar EC2 configurations (z1d instances in cluster placement groups) to achieve submillisecond round-trip latency on their exchange.

**Launch all instances together** in a single `RunInstances` call. Adding instances later might fail if there's not enough co-located capacity.

### Network Optimization: AWS Global Accelerator

AWS Global Accelerator routes your traffic through AWS's private backbone instead of the public internet. Result: **up to 60% latency reduction** and consistent sub-50ms global routes vs. 150-300ms on public internet.

| Metric | Public Internet | AWS Global Accelerator |
| :-- | :-- | :-- |
| Global Latency | 150-300ms | <50ms |
| Packet Loss | 2-5% | <0.5% |
| Route Optimization | Static | Dynamic Intelligent Routing |

Deriv, a high-frequency trading platform, reduced trading latency by 50% after adopting Global Accelerator.

### State Management: DynamoDB

Your bot needs to persist state — open positions, trade history, P&L tracking, market metadata. DynamoDB is the right choice:

- Single-digit millisecond response times
- Handles time-series data patterns well with date-based partition keys
- DynamoDB Streams can trigger Lambda functions for real-time state change processing

**Table Design:**

```
Positions Table:
  PK: market_id
  SK: timestamp
  Attributes: side, size, entry_price, current_pnl

Trades Table:
  PK: trade_date (YYYY-MM-DD)
  SK: trade_id
  Attributes: market_id, side, price, size, fees, execution_time_us
```

### Monitoring & Alerting: CloudWatch + EventBridge

Use EventBridge for event-driven automation:

- Market resolution events → trigger position closing Lambda
- Drawdown threshold crossed → trigger kill switch Lambda
- New market created → trigger analysis pipeline

CloudWatch for real-time metrics:

- Latency percentiles (p50, p99, p999)
- Order fill rates
- WebSocket connection health
- Inventory exposure per market

### Logging: S3

Store all raw market data, order book snapshots, and trade logs in S3 for backtesting. This data is gold — every improvement to your model starts with historical analysis.

### Complete AWS Service Map:

| Service | Purpose | Why Not Alternatives? |
| :-- | :-- | :-- |
| **EC2 (c5n) + Cluster Placement Group** | Run trading engine | Lowest latency. VPS providers can't match in-region CPG performance |
| **Global Accelerator** | Network optimization | Routes through AWS backbone, not public internet |
| **DynamoDB** | State persistence | Single-digit ms reads, serverless scaling, streams for event triggers |
| **EventBridge** | Event routing | Decouple services. New market alerts, resolution triggers |
| **CloudWatch** | Monitoring | Native integration, custom metrics, alarms |
| **S3** | Data lake / logs | Cheap storage for backtesting data, unlimited capacity |
| **Lambda** | Auxiliary processing | Event-triggered functions (not on the hot path) |
| **Secrets Manager** | API keys, private keys | Never hardcode credentials. Rotate automatically |

---

## 5. Latency Optimization — Where Every Microsecond is Explained

Latency isn't just a nice-to-have. In market making, the slower you are, the more **adverse selection** you face — faster traders will pick off your stale quotes while you can't cancel in time. Even on prediction markets with ~100ms WebSocket feeds, being faster than other bots means better queue priority and less slippage.

### Layer-by-Layer Optimization:

**Layer 1: Network (~50-200μs)**

- Cluster Placement Groups put instances on the same rack
- VPC Peering (if trading across AWS accounts) provides lowest cross-account latency
- Enable Enhanced Networking (ENA) on all instances

**Layer 2: OS Kernel (~1-10μs)**

- **CPU Pinning:** Assign dedicated CPU cores to your trading thread. Prevents the OS scheduler from moving your process between cores (which flushes CPU cache)
- **`io_uring`:** Linux's async I/O interface. Uses shared memory ring buffers between application and kernel, eliminating extra system calls
- **Thread Segregation:** Keep network I/O threads separate from trading logic threads

**Layer 3: Application (<1μs)**

- **Lock-free data structures** via `crossbeam` — no mutex contention
- **Memory-resident order books** — entire order book lives in RAM, no disk I/O
- **Avoid async for the hot path** — async can introduce context switches. For the critical trading loop, use a dedicated synchronous thread with busy-spin
- **Typed deserialization** — `serde` with typed structs is dramatically faster than parsing to `serde_json::Value`
- **Pre-allocated buffers** — avoid heap allocations on the hot path

```rust
// BAD: Allocates on every message
let data: serde_json::Value = serde_json::from_str(&msg)?;

// GOOD: Zero-allocation typed deserialization
#[derive(Deserialize)]
struct OrderBookUpdate {
    bids: Vec<PriceLevel>,
    asks: Vec<PriceLevel>,
    sequence: u64,
}
let data: OrderBookUpdate = serde_json::from_str(&msg)?;
```

### Latency Budget (Realistic for Prediction Markets):

| Step | Target Latency | Tool |
| :-- | :-- | :-- |
| WebSocket message received | ~100ms (Polymarket's feed) | `tokio-tungstenite` |
| JSON deserialization | <50μs | `serde_json` with typed structs |
| Pricing engine computation | <100μs | Pure Rust math, pre-computed tables |
| EIP-712 order signing | <1ms | `alloy` |
| HTTP order submission | <5ms | `reqwest` to CLOB API |
| **Total tick-to-trade** | **~110ms** | Dominated by WebSocket feed latency |

The honest truth: with Polymarket's ~100ms WebSocket feed, your application-layer optimizations matter less than on a traditional exchange. Your edge comes more from **better probability models and smarter risk management** than from shaving microseconds. But building the system right from Day 1 means you're ready when lower-latency feeds become available.

---

## 6. Putting It All Together — The Roadmap

**Phase 1: Foundation (Week 1-2)**

- Learn Rust basics (ownership, borrowing, async/await)
- Understand Polymarket's CLOB API docs
- Build a read-only WebSocket client that prints order book updates

**Phase 2: Pricing Engine (Week 3-4)**

- Implement Bayesian probability updater
- Build Kelly Criterion position sizer
- Create a simple fair value model (start with base rates + manual news assessment)

**Phase 3: Order Management (Week 5-6)**

- Implement EIP-712 order signing with `alloy`
- Build order placement/cancellation via REST API
- Implement paper trading mode (simulated fills against real order book data)

**Phase 4: Risk & Monitoring (Week 7-8)**

- Build risk controller (max position, max drawdown, inventory limits)
- Set up CloudWatch metrics and EventBridge alerts
- Implement kill switch (auto-cancels all orders if something goes wrong)

**Phase 5: Deploy on AWS (Week 9-10)**

- Provision EC2 instances in Cluster Placement Group in `us-east-1`
- Set up DynamoDB tables for state
- Configure Global Accelerator
- Deploy with proper secrets management

**Phase 6: Go Live & Iterate (Ongoing)**

- Start with tiny positions (1-5% of Kelly)
- Log everything. Analyze fill rates, slippage, P&L daily.
- Improve probability models based on historical performance
- Add news sentiment NLP pipeline for automated edge detection
- Expand to cross-platform arbitrage (Polymarket + Kalshi)

---

## Key Takeaways

1. **Prediction markets are real exchanges**, not gambling sites. They use CLOB infrastructure identical to NASDAQ/Binance.
2. **Your edge is probability estimation.** If you can model event probabilities better than the crowd, the Kelly Criterion tells you exactly how much to bet.
3. **Rust is the right language.** Zero-cost abstractions, no GC pauses, lock-free concurrency out of the box. The `tokio` + `crossbeam` + `alloy` + `serde` stack gives you everything you need.
4. **AWS infrastructure matters.** Cluster Placement Groups + `c5n` instances + Global Accelerator is the production-grade setup. Not a random VPS.
5. **Start with better models, not faster code.** On Polymarket's ~100ms WebSocket, your mathematical edge matters 10x more than your latency edge. Build the smart system first, optimize speed later.
6. **Risk management is survival.** Use fractional Kelly. Set hard drawdown limits. Implement kill switches. The best quant system is the one that's still running next month.

---

*If this thread helped you understand the quant stack for prediction markets, follow me for more deep dives into HFT, DeFi infrastructure, and trading systems engineering. I break down complex systems into actionable roadmaps — no fluff, just technical depth.*
