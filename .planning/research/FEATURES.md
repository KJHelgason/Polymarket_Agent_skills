# Features Research: Polymarket API

**Domain:** Prediction Market Trading API
**Researched:** 2026-01-31
**Overall Confidence:** HIGH

## Executive Summary

Polymarket provides a comprehensive API ecosystem with three main services: the CLOB (Central Limit Order Book) API for trading operations, the Gamma API for market discovery and metadata, and the Data API for user-specific information. The API supports both REST endpoints and WebSocket channels, with a clear separation between read-only operations (public, no auth required) and trading operations (requiring L1/L2 authentication).

The API is built on a hybrid-decentralized architecture where off-chain order matching provides speed while on-chain settlement ensures non-custodial security. All outcome shares are represented as ERC1155 conditional tokens on Polygon.

---

## REST API Capabilities

### CLOB API (`https://clob.polymarket.com`)

**Purpose:** Order management and trading operations

**Core Trading Endpoints:**
- **POST /order** - Place single order (limit/market)
- **POST /orders** - Place batch orders
- - **DELETE /order** - Cancel single order
- **DELETE /orders** - Cancel all orders for a market
- **GET /orders** - Retrieve active orders
- **GET /order** - Get single order by ID

**Market Data Endpoints:**
- **GET /book** - Get order book for single token
- **GET /books** - Get order books for multiple tokens
- **GET /price** - Get current price for token
- **GET /prices** - Get prices for multiple tokens
- **GET /midpoint** - Get midpoint price
- **GET /midpoints** - Get midpoints for multiple tokens
- **GET /spread** - Get bid-ask spread
- **GET /tick-size** - Get minimum price increment
- **GET /prices-history** - Get historical price timeseries

**Trade Data Endpoints:**
- **GET /trades** - Get trades for authenticated user
- **GET /last-trade-price** - Get price of last executed trade

**Infrastructure Endpoints:**
- **GET /ok** - Health check (public, no auth)
- **GET /time** - Get server time (public, no auth)

**Authentication Endpoints:**
- **POST /auth/api-key** - Create L2 API credentials (requires L1 auth)
- **POST /auth/derive-api-key** - Derive deterministic API credentials (requires L1 auth)

**Builder Program Endpoints:**
- **GET /builder/trades** - Get builder-attributed trades (requires builder auth)

### Gamma API (`https://gamma-api.polymarket.com`)

**Purpose:** Market discovery, metadata, and categorization (read-only)

**Market Discovery:**
- **GET /markets** - List markets with filtering/pagination
- **GET /markets/{id}** - Get market by condition ID
- **GET /markets/slug/{slug}** - Get market by slug

**Event Discovery:**
- **GET /events** - List events with filters
- **GET /events/{id}** - Get event by ID
- **GET /events/slug/{slug}** - Get event by slug

**Categorization & Metadata:**
- **GET /tags** - Get tag taxonomy
- **GET /tags/{tag}** - Get related tags
- **GET /series** - Get market series data
- **GET /series/{id}** - Get series by ID

**Sports Metadata:**
- **GET /sports/metadata** - Get sports configuration including images, resolution sources, ordering preferences, tags, and series information
- **GET /sports/teams** - Get team data
- **GET /sports/market-types** - Get available market types for sports

**Search:**
- Cross-search markets, events, and user profiles

**Performance Characteristics:**
- ~1 second latency
- Globally cacheable (read-only)
- No authentication required

### Data API (`https://data-api.polymarket.com`)

**Purpose:** User-specific data and analytics

**Position Management:**
- **GET /positions** - Get current positions for user address
  - Filter by market, event, status
  - Includes valuation data
- **GET /positions/closed** - Get closed/historical positions

**User Activity:**
- **GET /activity** - Get on-chain activity for user
  - Ordered by timestamp (descending)
  - Filter by activity type: TRADE, SPLIT, MERGE, REDEEM, REWARD, CONVERSION
  - Filter by time range, side (BUY/SELL), market
  - Sort by TIMESTAMP, TOKENS, or CASH

**Portfolio Analytics:**
- **GET /portfolio** - Calculate total position values
- **GET /accounting** - Generate accounting snapshot (ZIP with positions.csv and equity.csv)

**Token Holder Data:**
- **GET /holders** - Get top holders of outcome tokens for a market

**Leaderboards:**
- **GET /leaderboards** - Get trader rankings
- **GET /builder/volume** - Get builder volume statistics

### Bridge API

**Purpose:** Multi-chain deposits and withdrawals

**Address Generation:**
- **POST /deposit/address** - Generate unique deposit address for multi-chain deposits (EVM, Solana, Bitcoin)
  - Automatic conversion to USDC.e on Polygon
- **POST /withdrawal/address** - Create destination-specific withdrawal address

**Quote System:**
- **GET /deposit/quote** - Get estimated quote for deposit (output amounts, checkout time, fee breakdown)
- **GET /withdrawal/quote** - Get estimated quote for withdrawal

**Asset Information:**
- **GET /supported-assets** - List available chains/tokens for deposits/withdrawals with minimum USD thresholds

**Transaction Tracking:**
- **GET /transaction/{id}/status** - Track deposit/withdrawal progress
  - Status states: DEPOSIT_DETECTED, PROCESSING, ORIGIN_TX_CONFIRMED, SUBMITTED, COMPLETED, FAILED

### Compliance Endpoints

- **GET /geoblock** - Check if user's IP is allowed to trade (recommended to call before order placement)

---

## WebSocket Capabilities

### CLOB WebSocket (`wss://ws-subscriptions-clob.polymarket.com/ws/`)

**Purpose:** Real-time order book and trading updates

**Channels:**
1. **MARKET channel** (public)
   - Real-time orderbook updates
   - Price changes
   - Trade executions

2. **USER channel** (authenticated)
   - Personal order status notifications
   - Trade confirmations
   - Position updates

**Subscription Management:**
- Dynamic add/remove of asset IDs (token identifiers)
- Dynamic add/remove of market IDs (condition identifiers)
- Toggle custom features on/off

**Maintenance:**
- Requires PING every 10 seconds to maintain connection

**Performance:**
- ~100ms latency (10x faster than REST API's ~1 second)
- Push-based updates (no polling needed)

### RTDS - Real Time Data Socket (`wss://ws-live-data.polymarket.com`)

**Purpose:** Broader market data and activity feeds

**Available Feeds:**
- Low-latency cryptocurrency price feeds
- Community comment streams
- Real-time sports results (structured message format)
- Market activity feeds

**Maintenance:**
- Requires PING every 5 seconds to maintain connection

**Performance:**
- Lower latency than Gamma API
- Suitable for real-time dashboards and monitoring

---

## Table Stakes Features

Features every Polymarket API user needs. Missing these = unusable for basic trading.

| Feature | Endpoints | Complexity | Auth Level | Notes |
|---------|-----------|------------|------------|-------|
| **Market Discovery** | GET /markets, /events | Low | None | Must be able to find what markets exist |
| **Price Checking** | GET /price, /midpoint | Low | None | Essential for decision-making |
| **Order Book Access** | GET /book | Low | None | See current liquidity |
| **Place Limit Order** | POST /order (GTC) | Medium | L2 | Core trading operation |
| **Place Market Order** | POST /order (FOK) | Medium | L2 | Quick execution at current price |
| **Cancel Orders** | DELETE /order | Low | L2 | Must be able to back out |
| **View Open Orders** | GET /orders | Low | L2 | Track active positions |
| **View Positions** | GET /positions | Low | None | Know what you own |
| **Authentication Setup** | POST /auth/api-key | Medium | L1 | Prerequisite for trading |
| **Health Check** | GET /ok, /time | Low | None | Verify API availability |

**Why Table Stakes:**
- **Market Discovery:** Can't trade without knowing what markets exist
- **Price Checking:** Can't make informed decisions without current prices
- **Order Placement:** The fundamental purpose of the API
- **Cancel Orders:** Essential risk management capability
- **Positions:** Must know what you own and its value
- **Authentication:** Required for any write operations

---

## Advanced Features

Sophisticated capabilities that differentiate power users from basic traders.

| Feature | Endpoints/Methods | Complexity | Value Proposition | Notes |
|---------|-------------------|------------|-------------------|-------|
| **Batch Order Placement** | POST /orders | Medium | Execute complex strategies atomically | Place multiple orders in single request |
| **WebSocket Real-Time Feeds** | CLOB WebSocket, RTDS | High | 10x faster updates (~100ms vs ~1s) | Critical for HFT and arbitrage |
| **Historical Price Analysis** | GET /prices-history | Low | Backtest strategies, trend analysis | Timeseries data for each token |
| **Conditional Token Operations** | CTF Contract methods | High | Market making, liquidity provision | Split/merge/redeem outcome tokens |
| **Builder Attribution** | Builder Program endpoints | Medium | Track and monetize order flow | Earn rebates on attributed volume |
| **Multi-Chain Bridge** | Bridge API | High | Deposit from any chain (EVM, Solana, BTC) | Automatic conversion to USDC.e |
| **Post-Only Orders** | POST /order with postOnly | Medium | Market making without taking liquidity | Guarantee maker rebates |
| **Portfolio Accounting** | GET /accounting | Low | Tax reporting, P&L tracking | Automated CSV generation |
| **Activity Filtering** | GET /activity with filters | Medium | Analyze trading patterns | Filter by type, time, side, market |
| **Geographic Compliance** | GET /geoblock | Low | Avoid rejected orders | Check before trading |
| **Tick Size Optimization** | GET /tick-size | Low | Price orders correctly | Ensure orders meet minimum increment |
| **Spread Analysis** | GET /spread | Low | Measure market efficiency | Identify arbitrage opportunities |
| **Batch Price Queries** | GET /prices, /midpoints, /books | Medium | Efficient multi-market monitoring | Single request for multiple markets |
| **Trade History** | GET /trades, /activity | Low | Performance tracking, compliance | Full audit trail |
| **Negative Risk Markets** | negRisk field on events | High | Access asymmetric market structures | Advanced market types |
| **Liquidity Rewards Tracking** | Reward eligibility checks | Medium | Maximize earnings | Track maker rebates and incentives |

**Why Advanced:**
- **Not required for basic trading:** Can trade without these features
- **Provide competitive edge:** Speed, efficiency, or revenue advantages
- **Higher complexity:** Require deeper understanding of market mechanics
- **Enable sophisticated strategies:** Market making, arbitrage, automation

---

## Limitations & Constraints

### Geographic Restrictions

**CRITICAL:** Trading is geo-blocked in 33+ countries including US and UK.

- **Enforcement:** IP-based blocking at CLOB level
- **Scope:** Applies to trading via UI, API, and AI agents
- **Data Access:** Market data viewable globally (only trading restricted)
- **Verification:** Use GET /geoblock before placing orders
- **Impact:** Orders from blocked IPs are rejected

**Workarounds:** None recommended (violates Terms of Service)

### Rate Limits

**Free Tier:**
- Non-trading queries: ~100 requests/minute
- Order endpoint: 3000 requests/10 minutes
- Authenticated trading: ~60 orders/minute per API key

**Premium Tiers:**
- Basic ($99/month): 1,000 calls/hour, WebSocket access, 30+ days history
- Enterprise ($500+/month): Dedicated nodes, no throttling, priority support

**Implementation:**
- Cloudflare throttling (requests throttled, not rejected)
- Sliding time windows
- Burst allowance above sustained rate

**Best Practices:**
- Use WebSocket for real-time data (avoid polling)
- Batch requests when possible (GET /prices vs GET /price)
- Cache Gamma API results (rarely changes)

### Authentication Complexity

**Two-Level System:**

**L1 (Private Key):**
- Required to generate L2 credentials
- Must be kept absolutely secret
- Used for: Creating/deriving API keys

**L2 (API Key):**
- Triple credentials: apiKey, secret, passphrase
- HMAC-SHA256 signed requests
- Used for: All trading operations

**Proxy Wallet Complications:**
- Email wallets (Magic) and browser extensions use different signing vs funding addresses
- Must specify "funder" address separately from "signer" address
- Three signature types supported for different wallet configurations

**Security Requirements:**
- Never expose private keys in code
- Use environment variables
- Rotate API keys periodically
- IP whitelist when possible

### Order Type Limitations

**All orders are limit orders internally:**
- "Market orders" are limit orders with marketable prices
- No true IOC (use FOK or FAK instead)
- No stop-loss or trailing-stop orders
- No conditional orders (if-then logic)

**Available Order Types:**
- **GTC (Good-Til-Cancelled):** Standard limit order
- **GTD (Good-Til-Date):** Expires at timestamp
- **FOK (Fill-Or-Kill):** Market order, all-or-nothing immediate execution
- **FAK (Fill-And-Kill):** Polymarket's IOC equivalent

**Workarounds:**
- Implement stop-loss logic client-side
- Use WebSocket to monitor prices and trigger orders
- Build conditional logic in your application layer

### Data Availability Limitations

**Historical Data:**
- Free tier: Limited history depth
- Premium tier: 30+ days history
- Enterprise: Full historical data access

**Market Metadata:**
- Gamma API has ~1 second latency (not suitable for HFT without WebSocket)
- Some metadata only available through event/market objects (not standalone endpoints)

**Position Tracking:**
- Positions API shows current state only (not historical snapshots)
- Must use /activity endpoint to reconstruct historical positions
- Closed positions available but not timestamped snapshots

### Blockchain Constraints

**Settlement Delays:**
- Off-chain matching is instant (~100ms)
- On-chain settlement takes Polygon block time (~2 seconds)
- Withdrawals subject to bridge time (varies by destination chain)

**Gas Costs:**
- On-chain operations (split, merge, redeem) require gas on Polygon
- User pays gas for conditional token operations
- Orders themselves are gas-free (settled via signatures)

**Conditional Token Complexity:**
- Outcome tokens are ERC1155 (not ERC20)
- Must use CTF contract for split/merge/redeem operations
- Token IDs derived from collateralToken + collectionId
- Negative risk markets use separate NegRisk_CTFExchange contract

### API Operational Limits

**What the API Cannot Do:**

**No Order Modification:**
- Cannot edit existing orders
- Must cancel and re-create

**No Bulk Cancel by Market:**
- Can cancel all orders globally (DELETE /orders)
- Cannot cancel "all orders in market X" (must cancel individually)

**No Market Creation:**
- Cannot create new markets via API
- API is read/trade only, not market administration

**No Direct Blockchain Interaction:**
- API abstracts blockchain
- For advanced CTF operations, must interact with smart contracts directly

**No Historical Orderbook Snapshots:**
- Can get current orderbook
- Cannot query "orderbook state at timestamp X"
- Must build your own snapshot system via WebSocket

**No Guaranteed Fill:**
- Market orders (FOK) can fail if insufficient liquidity
- No "fill at any price" guarantee
- Must check orderbook depth before large orders

**No Cross-Market Atomicity:**
- Cannot place orders across multiple markets atomically
- Batch orders are for efficiency, not atomicity guarantees

### Operator Trust Assumptions

**What You Must Trust the Operator For:**
- Correct order matching (FIFO, price-time priority)
- Not censoring orders
- Not removing cancellations

**What You Don't Trust the Operator For:**
- Custody of funds (non-custodial, on-chain settlement)
- Order execution (cryptographically verified signatures)
- Price manipulation (orderbook is transparent)

**Mitigation:**
- Orders can be cancelled on-chain if operator not trusted
- Settlement is non-custodial (operator cannot steal funds)
- Transparent orderbook (can verify correct matching)

### Signature Type Complexity

**Three Signature Types (signatureType field):**
- Type 0: EOA (Externally Owned Account) wallets
- Type 1: Email/Magic wallets
- Type 2: Proxy wallets (browser extensions)

**Must Match Wallet Type:**
- Wrong signature type = rejected orders
- Must know your wallet architecture
- py-clob-client defaults to type 1 (email wallets)

### WebSocket Maintenance Requirements

**Connection Keep-Alive:**
- CLOB WebSocket: PING every 10 seconds or disconnect
- RTDS: PING every 5 seconds or disconnect
- Must implement reconnection logic
- No automatic reconnect (client responsibility)

**Subscription Limits:**
- Unknown maximum subscriptions per connection
- Recommended to use multiple connections for many markets

---

## REST vs WebSocket Use Cases

### Use REST When:

**Market Discovery & Metadata:**
- Listing available markets (Gamma API)
- Getting event information
- Retrieving market metadata (description, tags, categories)
- One-time or infrequent queries

**Historical Data:**
- Price history analysis
- Trade history retrieval
- Position snapshots
- Accounting exports

**Order Management:**
- Placing orders (POST /order)
- Cancelling orders (DELETE /order)
- Querying open orders

**Low-Frequency Operations:**
- Authentication setup
- Portfolio checks
- Periodic health monitoring

**Batch Operations:**
- Fetching prices for multiple markets
- Placing multiple orders simultaneously
- Bulk orderbook retrieval

### Use WebSocket When:

**Real-Time Price Monitoring:**
- HFT strategies
- Arbitrage detection
- Live price displays
- Market-making

**Order Book Tracking:**
- Depth-of-book visualization
- Liquidity analysis
- Front-running detection
- Order flow analysis

**Personal Order Updates:**
- Real-time fill notifications
- Order status changes
- Trade confirmations
- Position changes

**Event-Driven Trading:**
- News-driven strategies
- Automated execution on price movements
- Risk management triggers

**High-Frequency Needs:**
- Millisecond-level latency requirements
- Continuous data streams
- Avoiding polling overhead

### Hybrid Approach (Recommended):

**Pattern:** Use Gamma REST API to discover interesting markets, then subscribe to those markets via WebSocket for real-time updates.

**Example Workflow:**
1. GET /markets with filters → List of relevant markets
2. GET /events/{id} → Event details and metadata
3. Subscribe to MARKET channel via WebSocket → Real-time price/orderbook
4. Subscribe to USER channel via WebSocket → Personal order updates
5. POST /order via REST → Place orders based on WebSocket data
6. WebSocket USER channel → Receive fill confirmations

**Performance Benefit:**
- Gamma API latency: ~1 second
- WebSocket latency: ~100ms
- 10x speed improvement for price-sensitive operations

---

## Order Types Deep Dive

### Limit Orders (GTC/GTD)

**Good-Til-Cancelled (GTC):**
```
{
  "tokenId": "123456",
  "price": "0.65",
  "size": "100",
  "side": "BUY",
  "orderType": "GTC"
}
```
- Active until filled or cancelled
- Standard limit order
- Can use with postOnly flag to guarantee maker rebate

**Good-Til-Date (GTD):**
```
{
  "tokenId": "123456",
  "price": "0.65",
  "size": "100",
  "side": "BUY",
  "orderType": "GTD",
  "expiration": 1738368000  // UTC seconds timestamp
}
```
- Automatically cancelled at expiration timestamp
- Useful for time-sensitive strategies
- Can also be manually cancelled before expiration

**Post-Only Option:**
- Available for GTC/GTD only
- Prevents immediate matching against resting orders
- Guarantees maker rebate (not taker fee)
- Order rejected if it would take liquidity

### Market Orders (FOK)

**Fill-Or-Kill (FOK):**
```
{
  "tokenId": "123456",
  "amount": "100",  // Dollar amount to spend
  "side": "BUY",
  "orderType": "FOK"
}
```
- Must execute immediately and completely
- If insufficient liquidity, entire order cancelled (nothing filled)
- Implemented as limit order with marketable price
- Gets best available price (with slippage)

**Use Cases:**
- Immediate execution needed
- Willing to accept current market price
- Small orders relative to orderbook depth

**Risks:**
- Slippage (price worse than expected)
- Total rejection if liquidity insufficient
- No price limit (could execute at unfavorable price)

**Best Practice:** Check orderbook depth before large FOK orders

### Fill-And-Kill (FAK) - Polymarket's IOC Equivalent

- Similar to FOK but allows partial fills
- Fills what it can immediately, cancels the rest
- Polymarket's terminology for IOC (Immediate-Or-Cancel) semantics

---

## Authentication Levels

### Public Methods (No Auth)

**Available to anyone without credentials:**
- GET /markets, /events (Gamma API)
- GET /book, /price, /midpoint (CLOB API)
- GET /ok, /time (health checks)
- GET /prices-history (historical data)
- GET /positions (if querying public addresses)

**Use Cases:**
- Market research
- Price monitoring
- Public data analysis
- Building dashboards without trading

### L1 Methods (Private Key Required)

**Operations that create/derive API credentials:**
- POST /auth/api-key - Generate new API credentials
- POST /auth/derive-api-key - Derive deterministic credentials

**Security:**
- Private key must be kept absolutely secret
- Never expose in client-side code
- Use environment variables
- Only needed once to generate L2 credentials

**Implementation:**
```python
from py_clob_client import ClobClient

# L1: Use private key to create API credentials
client = ClobClient(
    host="https://clob.polymarket.com",
    key=os.getenv("PRIVATE_KEY"),  # L1 auth
    chain_id=137
)
creds = client.create_or_derive_api_creds()
# creds = {"apiKey": "...", "secret": "...", "passphrase": "..."}
```

### L2 Methods (API Key Required)

**Trading and account operations:**
- POST /order, POST /orders - Place orders
- DELETE /order, DELETE /orders - Cancel orders
- GET /orders - View personal orders
- GET /trades - View personal trades

**Authentication:**
- Triple credentials: apiKey, secret, passphrase
- HMAC-SHA256 signed requests
- Each request includes timestamp and signature

**Implementation:**
```python
# L2: Use API credentials for trading
client = ClobClient(
    host="https://clob.polymarket.com",
    key=os.getenv("PRIVATE_KEY"),  # Still needed for signing
    chain_id=137,
    creds=creds  # L2 credentials from above
)
client.create_order(...)  # L2 method
```

### Builder Program Methods

**Exclusive to builder program participants:**
- GET /builder/trades - Builder-attributed trades
- Builder volume statistics
- Requires special builder authentication headers

**Purpose:**
- Track order flow attribution
- Earn rebates on attributed volume
- Monetize order flow

---

## Conditional Token Framework (CTF)

### What Are Conditional Tokens?

Polymarket uses Gnosis's Conditional Token Framework (CTF) to represent outcome shares as ERC1155 tokens.

**Key Concepts:**
- **Collateral Token:** USDC.e on Polygon
- **Condition:** Unique identifier for a market event
- **Outcome Tokens:** YES/NO tokens for binary markets
- **Position IDs:** ERC1155 token IDs representing ownership

**Token Math:**
- 1 USDC.e can be split into 1 YES + 1 NO token
- Winning tokens redeem for 1 USDC.e
- Losing tokens redeem for 0 USDC.e

### CTF Operations

**Split:**
```
USDC.e → YES token + NO token
```
- Deposit collateral, receive outcome tokens
- Used to create liquidity
- Requires USDC.e approval and gas

**Merge:**
```
YES token + NO token → USDC.e
```
- Combine outcome tokens back into collateral
- Used to exit positions without trading
- Requires gas

**Redeem:**
```
Winning token → USDC.e
Losing token → 0
```
- After market resolves, exchange winning tokens for collateral
- Only available after oracle reports payout
- Requires gas

**API Support:**
- Split/Merge/Redeem NOT directly in CLOB API
- Must interact with CTF smart contract directly
- Position data available via GET /positions

### Negative Risk Markets

**Special market type for asymmetric outcomes:**
- Uses separate NegRisk_CTFExchange contract
- Identified by negRisk: true field on events
- More complex token structures
- Requires understanding of NegRiskAdapter

**Example:** "Team X wins championship"
- YES: Single outcome (Team X wins)
- NO: Multiple outcomes (any other team wins)
- Asymmetric probability distribution

---

## Rate Limits Summary

| Endpoint Category | Free Tier | Premium ($99/mo) | Enterprise ($500+/mo) |
|-------------------|-----------|------------------|-----------------------|
| Non-trading queries | ~100/min | 1,000/hour | Dedicated nodes |
| Order endpoint | 3000/10min | Higher limits | No throttling |
| Authenticated trading | ~60 orders/min | Higher limits | Priority access |
| Historical data depth | Limited | 30+ days | Full history |
| WebSocket access | Yes | Yes | Yes |

**Throttling Behavior:**
- Cloudflare throttling (slows requests, doesn't reject)
- Sliding time windows
- Burst allowance above sustained rate
- Reset varies by endpoint (10s, 1min, 1hour)

---

## Confidence Assessment

| Area | Confidence | Source | Notes |
|------|------------|--------|-------|
| REST API Endpoints | HIGH | Official docs, WebFetch | Complete endpoint list verified |
| WebSocket Capabilities | HIGH | Official docs | Channel types, latency, maintenance confirmed |
| Order Types | HIGH | Official docs, WebFetch | GTC/GTD/FOK/FAK documented |
| Authentication Flow | HIGH | Official docs, py-clob-client | L1/L2 system well-documented |
| Rate Limits | MEDIUM | Official docs + WebSearch | Specific numbers from docs, tier pricing from community |
| Geographic Restrictions | HIGH | Official docs, WebSearch | 33+ countries, IP-based, /geoblock endpoint |
| CTF Operations | MEDIUM | WebSearch + official docs | Framework understood, some details inferred |
| Limitations | MEDIUM | WebSearch + inference | Some limitations documented, others inferred from capabilities |
| Historical Data | MEDIUM | Official docs + community | Endpoint exists, depth limits from community sources |
| Builder Program | LOW | WebSearch only | Endpoints documented but program details sparse |

---

## Sources

**Official Polymarket Documentation:**
- [Endpoints Overview](https://docs.polymarket.com/quickstart/reference/endpoints)
- [CLOB Introduction](https://docs.polymarket.com/developers/CLOB/introduction)
- [WebSocket Overview](https://docs.polymarket.com/developers/CLOB/websocket/wss-overview)
- [Authentication](https://docs.polymarket.com/developers/CLOB/authentication)
- [Place Single Order](https://docs.polymarket.com/developers/CLOB/orders/create-order)
- [Methods Overview](https://docs.polymarket.com/developers/CLOB/clients/methods-overview)
- [Gamma API Overview](https://docs.polymarket.com/developers/gamma-markets-api/overview)
- [Rate Limits](https://docs.polymarket.com/quickstart/introduction/rate-limits)
- [Historical Timeseries](https://docs.polymarket.com/developers/CLOB/timeseries)
- [Get User Positions](https://docs.polymarket.com/developers/misc-endpoints/data-api-get-positions)
- [Get User Activity](https://docs.polymarket.com/developers/misc-endpoints/data-api-activity)

**Official GitHub Repositories:**
- [py-clob-client](https://github.com/Polymarket/py-clob-client)

**Technical Analysis (2026):**
- [The Polymarket API: Architecture, Endpoints, and Use Cases](https://medium.com/@gwrx2005/the-polymarket-api-architecture-endpoints-and-use-cases-f1d88fa6c1bf)

**Community Resources:**
- [Polymarket WebSocket Tutorial](https://www.polytrackhq.app/blog/polymarket-websocket-tutorial)
- [How to Use the Polymarket API](https://apidog.com/blog/polymarket-api/)
- [Order Types Guide](https://polymarketguide.gitbook.io/polymarketguide/trading/order-types)
- [Automated Trading on Polymarket](https://www.quantvps.com/blog/automated-trading-polymarket)
