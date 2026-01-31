# Architecture Research: Polymarket API

**Domain:** Prediction Market Trading Platform
**Researched:** 2026-01-31
**Overall Confidence:** HIGH

## System Overview

Polymarket operates as a **hybrid-decentralized platform** that combines off-chain efficiency with on-chain security. The architecture separates concerns across three distinct layers:

```
Application Layer: Web frontend, mobile apps, third-party integrations
     |
Service Layer: Gamma API | CLOB API | Data API
     |
Protocol Layer: CTF Exchange | NegRisk Exchange | CTF Core | UMA Oracle
     |
Blockchain: Polygon (Chain ID: 137)
```

### Core Design Principles

1. **Hybrid Decentralization**: Centralized order matching for speed + non-custodial on-chain settlement for security
2. **Separation of Concerns**: Three distinct APIs for market data, trading, and user data
3. **Capital Efficiency**: Advanced mechanisms (NegRisk) to minimize collateral requirements
4. **ERC-1155 Tokens**: Conditional outcome tokens backed by USDC collateral

**Confidence:** HIGH - Verified through official documentation and multiple authoritative sources

---

## Three-API Architecture

Polymarket's service layer consists of three specialized APIs, each serving distinct purposes:

### 1. Gamma API (Market Metadata & Discovery)

**Base URL:** `https://gamma-api.polymarket.com`

**Purpose:** Read-only market metadata, discovery, and organization

**Key Responsibilities:**
- Market discovery and filtering
- Event hierarchy (Series → Events → Markets)
- Market metadata (titles, descriptions, end dates)
- Lightweight market state (not real-time prices)

**Authentication:** None required (public read access)

**Rate Limits:**
- General: 4000 requests/10s
- `/markets`: 300 requests/10s
- `/events`: 500 requests/10s
- Search: 350 requests/10s

**Data Hierarchy:**
```
Series (optional grouping)
  └─ Events (organizational container)
      └─ Markets (tradeable instruments)
```

**Market Types:**
- **Single Market Products (SMP)**: Events with 1 market (simple binary YES/NO)
- **Group Market Products (GMP)**: Events with 2+ markets (multi-outcome)

**When to Use:**
- Discovering what markets exist
- Getting market metadata and descriptions
- Building market browsers or explorers
- Non-time-sensitive queries

### 2. CLOB API (Trading Operations)

**Base URL:** `https://clob.polymarket.com`

**Purpose:** Central Limit Order Book - live trading data and order management

**Key Responsibilities:**
- Order placement (limit orders, market orders)
- Order cancellation
- Orderbook depth queries
- Real-time price data
- Trade history
- Order status tracking

**Authentication:** Required for trading operations (API key, secret, passphrase)

**Rate Limits:**
- General: 9000 requests/10s
- Market data (book/price/midprice): 1500 requests/10s
- POST `/order`: 3500 requests/10s burst; 36000/10 minutes sustained
- DELETE `/order`: 3000 requests/10s burst; 30000/10 minutes sustained

**Order Types:**
- **GTC (Good-Till-Cancel)**: Remains active until filled or cancelled
- **FOK (Fill-Or-Kill)**: Execute immediately and completely, or cancel
- **GTD (Good-Till-Date)**: Active until specified expiration

**When to Use:**
- Placing or cancelling orders
- Real-time price/orderbook queries
- Trading automation
- Market-making operations

### 3. Data API (User-Specific Data)

**Base URL:** `https://data-api.polymarket.com`

**Purpose:** User positions, portfolio data, and trade history

**Key Responsibilities:**
- Current positions by user address
- Closed positions and PnL
- User trade history
- Redeemable positions (settled markets)
- Mergeable positions (for capital efficiency)

**Authentication:** None required (queries by public address)

**Rate Limits:**
- General: 1000 requests/10s
- `/trades`: 200 requests/10s
- `/positions` & `/closed-positions`: 150 requests/10s each

**Key Endpoint:** `GET /positions`
```
Required: user (Ethereum address)
Optional: market, eventId, sizeThreshold, redeemable, mergeable, limit, offset, sortBy
Returns: Array of Position objects with size, avgPrice, currentValue, PnL metrics
```

**When to Use:**
- Portfolio tracking
- Position monitoring
- PnL calculations
- Identifying redeemable/mergeable positions

### Rate Limit Behavior

**Critical:** Unlike most APIs, Polymarket does **not reject** requests that exceed rate limits. Instead, requests over the limit are **delayed/queued** rather than dropped. This means:
- No HTTP 429 errors under normal conditions
- Requests may experience increased latency when throttled
- Clients should implement timeout handling for delayed requests

**Confidence:** HIGH - Verified through official documentation

---

## CLOB Structure (Central Limit Order Book)

### Hybrid-Decentralized Model

Polymarket's CLOB operates as a "hybrid-decentralized" system:

**Off-Chain (Centralized):**
- Order matching engine
- Order book maintenance
- Order routing and optimization
- Operator manages matching logic

**On-Chain (Decentralized):**
- Non-custodial settlement
- Exchange smart contracts execute trades
- Users maintain custody of funds
- Operators cannot set prices or execute unauthorized trades

### Order Structure

All orders are represented as **EIP712 signed typed data**:

```typescript
Order = {
  salt: number,           // Unique order identifier
  maker: address,         // Order creator
  signer: address,        // Signing authority
  taker: address,         // Intended counterparty (0x0 for any)
  tokenId: string,        // Outcome token ID
  makerAmount: string,    // Maker's offered amount
  takerAmount: string,    // Taker's required amount
  expiration: number,     // Expiry timestamp
  nonce: number,          // Prevents replay attacks
  feeRateBps: number,     // Fee rate in basis points
  side: number,           // BUY (0) or SELL (1)
  signatureType: number   // 0=EOA, 1=PolyProxy, 2=GnosisSafe
}
```

**Why Complex?** The documentation explicitly states order structuring "is quite involved (structuring, hashing, signing)," which is why Polymarket strongly recommends using official client libraries (TypeScript, Python, Go, Rust) rather than implementing order creation manually.

### Maker/Taker Model

**Matching Relationship:** Always 1:1 or many:1 (maker to taker)

- **Maker:** Places limit orders that add liquidity to the book
- **Taker:** Executes against existing orders, removing liquidity
- **Price Improvement:** Any price improvement is captured by the taker

**Example:**
```
Maker: Limit order to BUY at $0.55
Taker: Market order to SELL
If matched at $0.54, taker captures $0.01 improvement
```

### Security Model

The operator's authority is strictly limited:
- **Can:** Match orders, maintain orderbook, route orders
- **Cannot:** Set prices, execute unauthorized trades, access user funds
- **User Override:** Users can independently cancel orders on-chain if needed

**Confidence:** HIGH - Verified through official documentation

---

## CTF Structure (Conditional Token Framework)

Polymarket uses Gnosis's Conditional Token Framework to tokenize market outcomes.

### Token Hierarchy

```
conditionId (parent identifier)
  ├─ collectionId (YES outcome)
  │   └─ positionId (ERC-1155 token ID)
  └─ collectionId (NO outcome)
      └─ positionId (ERC-1155 token ID)
```

### Key Identifiers

**conditionId** - Parent identifier for the entire market
- Derived from: `hash(oracleAddress, questionId, outcomeSlotCount)`
- Oracle: UMA Adapter V2
- questionId: Hash of UMA ancillary data
- outcomeSlotCount: Always 2 for binary markets

**collectionId** - Outcome-specific identifier
- Derived from: `hash(conditionId, parentCollectionId, indexSet)`
- parentCollectionId: Always 0 for Polymarket
- indexSet: Bitmap indicating outcome (1 = 0b01 for YES, 2 = 0b10 for NO)

**positionId (tokenId)** - The actual ERC-1155 token
- Derived from: `hash(collectionId, collateralAddress)`
- Collateral: USDC (0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174)

### Collateral Splitting & Merging

**Core Mechanism:** Users can atomically exchange collateral for outcome tokens

**Split:** 1 USDC → 1 YES token + 1 NO token
**Merge:** 1 YES token + 1 NO token → 1 USDC

**Why Critical:**
- Enables liquidity creation
- Guarantees arbitrage boundary (YES + NO = $1.00)
- Allows capital-efficient market making

**Process:**
1. User deposits USDC to CTF contract
2. Calls `splitPosition(conditionId, amount)`
3. Receives proportional YES and NO tokens
4. Can trade individual outcomes on CLOB
5. Can merge back to USDC anytime before settlement

### Settlement

After oracle resolution:
- Winning outcome tokens redeemable 1:1 for USDC
- Losing outcome tokens become worthless
- Users call `redeemPositions()` on CTF contract

**Confidence:** HIGH - Verified through official CTF documentation

---

## NegRisk Architecture (Capital Efficiency)

NegRisk (Negative Risk) is an advanced mechanism for **multi-outcome markets** where exactly one outcome will win (mutually exclusive outcomes).

### Problem It Solves

**Traditional Model:**
- Election with 3 candidates (A, B, C)
- To hold "NOT A AND NOT B" requires buying NO shares in both markets
- Cost: ~$2.00 of capital

**NegRisk Model:**
- "NOT A AND NOT B" is equivalent to "YES C"
- Cost: ~$0.XX (price of C winning)
- Capital savings: Significant

### Conversion Mechanism

**Core Principle:** A NO share in any market can be converted into 1 YES share in all other markets

**Example:**
```
Election: Candidate A, B, C (one will win)

Position: 1 NO A + 1 NO B
Equivalent to: 1 YES C + 1 USDC

The NegRiskAdapter enables conversion between these equivalent positions
```

### Architecture Components

**Standard Markets:**
- Use: CTF Exchange contract
- Address: `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E`

**NegRisk Markets:**
- Use: NegRisk_CTFExchange contract
- Address: `0xC5d563A36AE78145C45a50134d48A1215220f80a`
- Plus: NegRisk Adapter at `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296`

### Identifying NegRisk Markets

Check the Gamma API event object:
```json
{
  "negRisk": true,           // This is a NegRisk event
  "negRiskAugmented": false  // Standard NegRisk (not augmented)
}
```

### Augmented NegRisk

**Extension:** Handles incomplete outcome universes

**Use Case:** Election with "Other" category
- "Other" is a placeholder that can be clarified later
- Prevents expensive conversions when new candidates added
- Event has `negRisk: true` AND `negRiskAugmented: true`

### Capital Efficiency Impact

For multi-outcome markets in 2026, NegRisk provides significant capital efficiency:
- Reduced collateral requirements
- Simpler hedging strategies
- More efficient market making
- Critical for complex events (elections, tournaments, rankings)

**Confidence:** HIGH - Verified through official NegRisk documentation

---

## Authentication Architecture

Polymarket uses a **two-layer authentication system**: L1 (private key) and L2 (API credentials).

### Layer 1: Private Key Authentication

**Purpose:** Cryptographic identity and order signing

**Components:**
- **Private Key:** User's Ethereum wallet private key
- **Signer:** Wallet that signs orders and transactions
- **Funder:** Wallet that holds USDC and outcome tokens

**Key Separation:** Signer and funder can be different addresses

**Why?** Supports proxy wallets and smart contract wallets:
- Email wallets (Magic)
- Browser extension wallets
- Gnosis Safe multisig
- Smart contract wallets

### Layer 2: API Credentials

**Purpose:** Authenticated API access for order management

**Components:**
- API Key: Public identifier
- API Secret: Signing secret for HMAC
- API Passphrase: Additional authentication factor

**Derivation:**
```
GET {clob-endpoint}/auth/derive-api-key
Headers:
  - Signature (signed with private key)
  - Address
  - Nonce

Response:
  - apiKey
  - secret
  - passphrase
```

**Client Library Approach:**
```python
client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    chain_id=137,
    signature_type=0,  # EOA, Proxy, or GnosisSafe
    funder=None        # Optional: separate funder address
)

# Derive or create API credentials
client.set_api_creds(client.create_or_derive_api_creds())
```

### Signature Types

| Type | ID | Description | Use Case |
|------|-----|-------------|----------|
| EOA | 0 | Standard externally owned account | MetaMask, hardware wallets, direct private key |
| Poly Proxy | 1 | Polymarket proxy wallet | Email/Magic wallets, delegated signing |
| Poly Gnosis Safe | 2 | Gnosis Safe multisig | Smart contract wallets, multisig security |

### Token Allowances (Critical for EOA Users)

**Problem:** Exchange contracts need permission to move tokens on user's behalf

**Solution:** Set ERC-20 and ERC-1155 allowances

**Required Approvals:**

**For USDC Trading:**
- Token: `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` (USDCe)
- Spenders: CTF Exchange, NegRisk Exchange

**For Outcome Token Trading:**
- Token: `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` (CTF)
- Spenders: CTF Exchange, NegRisk Exchange
- Standard: `setApprovalForAll(spender, true)` for ERC-1155

**Note:** Email/Magic wallets handle allowances automatically. Hardware wallet users must execute approval transactions manually before first trade.

### Authentication Flow Diagram

```
1. User has private key
   ↓
2. Create ClobClient with private key + signature type
   ↓
3. Client signs authentication request
   ↓
4. POST to /auth/derive-api-key with signature
   ↓
5. Receive API key, secret, passphrase
   ↓
6. Set API credentials on client
   ↓
7. All subsequent requests use HMAC signing with API credentials
```

### Read-Only Mode

**Option:** Instantiate client without authentication
```python
client = ClobClient(
    host="https://clob.polymarket.com",
    chain_id=137
)
# Can query markets, orderbooks, prices
# Cannot place or cancel orders
```

**Confidence:** HIGH - Verified through official documentation and py-clob-client README

---

## Order Lifecycle

### Phase 1: Order Creation

**Client Side:**
1. Create order parameters (tokenId, price, size, side)
2. Structure as EIP712 typed data
3. Generate order hash
4. Sign with private key (Layer 1)
5. Attach API credentials (Layer 2)

**Order Types:**

**Limit Order:**
```python
order_args = OrderArgs(
    token_id="71321045679252212594626385532706912750332728571942532289631379312455583992563",
    price=0.55,
    size=10.00,
    side=BUY
)

signed_order = client.create_order(order_args)
response = client.post_order(signed_order, OrderType.GTC)
```

**Market Order:**
```python
market_order = MarketOrderArgs(
    token_id="71321045679252212594626385532706912750332728571942532289631379312455583992563",
    amount=10.00,  # dollars to spend/receive
    side=BUY
)

signed_order = client.create_market_order(market_order)
response = client.post_order(signed_order, OrderType.FOK)
```

### Phase 2: Order Submission

**Client → CLOB Operator:**
1. POST signed order to `/order`
2. Operator validates signature
3. Operator checks funder has sufficient balance/allowances
4. Operator adds to orderbook

**Validation Checks:**
- Signature valid for signer address
- Order not expired
- Nonce not already used
- Funder has USDC (for BUY) or outcome tokens (for SELL)
- Funder has set token allowances
- Price within valid range ($0.00 - $1.00)

### Phase 3: Order Matching

**CLOB Operator Off-Chain:**
1. Maintain sorted orderbook by price-time priority
2. Match incoming orders against resting orders
3. Determine maker/taker relationship
4. Calculate settlement amounts
5. Send matched orders to Exchange contract

**Matching Logic:**
- Price-time priority: Best price first, then earliest timestamp
- Maker/taker: Resting order = maker, incoming = taker
- Partial fills: Large orders can match multiple counterparties
- Price improvement: Captured by taker

### Phase 4: On-Chain Settlement

**Exchange Contract Execution:**

**For Standard Markets:**
```
CTF Exchange (0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E)
  ↓
1. Verify both order signatures
2. Check funder allowances
3. Transfer USDC from buyer to seller
4. Transfer outcome tokens from seller to buyer
5. Emit OrderFilled event
```

**For NegRisk Markets:**
```
NegRisk_CTFExchange (0xC5d563A36AE78145C45a50134d48A1215220f80a)
  ↓
1. Verify both order signatures
2. Check funder allowances
3. Use NegRiskAdapter for efficient settlement
4. Transfer net positions (capital efficient)
5. Emit OrderFilled and OrderMatched events
```

### Phase 5: Order States

| State | Description | Next States |
|-------|-------------|-------------|
| LIVE | Active in orderbook | MATCHED, CANCELLED |
| MATCHED | Filled (fully or partially) | LIVE (if partial), SETTLED |
| CANCELLED | User cancelled | None (terminal) |
| EXPIRED | Past expiration time | None (terminal) |
| RETRYING | Failed transaction being retried | MATCHED, FAILED |
| FAILED | Settlement failed, not retrying | None (terminal) |

### Phase 6: Monitoring

**Client Options:**

**REST Polling:**
```python
# Get open orders
open_orders = client.get_orders()

# Get order by ID
order = client.get_order(order_id)

# Get trades
trades = client.get_trades(market=token_id)
```

**WebSocket Real-Time:**
```python
# Subscribe to user channel
ws.subscribe(type="user", markets=[token_id])

# Receive updates
{
  "event_type": "ORDER_MATCHED",
  "order_id": "...",
  "fill_amount": "5.00",
  "remaining": "5.00"
}
```

### Phase 7: Cancellation

**User-Initiated:**
```python
# Cancel specific order
client.cancel(order_id)

# Cancel all orders
client.cancel_all()

# Cancel all for specific market
client.cancel_orders(OrderCancelationArgs(asset_id=token_id))
```

**Operator Cannot Cancel:** Only user (order signer) can cancel. This is enforced on-chain - operator has no cancellation authority.

**On-Chain Override:** Users can cancel directly on Exchange contract if operator is unresponsive (trustless escape hatch).

### Phase 8: Settlement & Redemption

**After Market Resolves:**
1. UMA Oracle determines outcome
2. Condition resolves on CTF contract
3. Winning tokens become redeemable
4. Users call `redeemPositions()` on CTF
5. Receive USDC 1:1 for winning tokens

**Data API Query:**
```
GET /positions?user=0x...&redeemable=true
Returns: All positions in resolved markets ready to redeem
```

**Confidence:** HIGH - Verified through official documentation, client library code, and architecture documentation

---

## Data Flow: REST vs WebSocket

### When to Use REST APIs

**Use Cases:**
- Market discovery ("What markets exist?")
- Historical data retrieval
- One-time queries
- Low-frequency updates (every few seconds or more)
- Building dashboards and analytics

**Typical Flow:**
```
1. Gamma API: Discover markets
   GET /markets?active=true

2. CLOB API: Get current prices
   GET /price?token_id=...

3. CLOB API: Place order
   POST /order

4. Data API: Check position
   GET /positions?user=0x...
```

**Latency:** ~1 second for Gamma API, faster for CLOB/Data

### When to Use WebSocket

**Use Cases:**
- Real-time price streaming
- Order status monitoring
- Trade notifications
- Low-latency applications
- Market making bots
- Arbitrage detection

**WebSocket Services:**

**1. CLOB WebSocket** (`wss://ws-subscriptions-clob.polymarket.com`)
- Order book updates
- Trade execution notifications
- User order status changes

**2. RTDS WebSocket** (`wss://ws-live-data.polymarket.com`)
- Real-time market prices
- Volume and activity feeds
- Broader market data

**Typical Flow:**
```
1. Connect to WebSocket
   wss://ws-subscriptions-clob.polymarket.com

2. Authenticate (for user channel)
   Send: { "auth": { "apiKey": "...", "secret": "...", "passphrase": "..." }}

3. Subscribe to channels
   Send: { "type": "subscribe", "channel": "user", "markets": ["0x..."] }
   Send: { "type": "subscribe", "channel": "market", "assets": ["tokenId"] }

4. Receive pushed updates
   Receive: { "event_type": "PRICE_CHANGE", "token_id": "...", "price": 0.56 }
   Receive: { "event_type": "ORDER_MATCHED", "order_id": "...", "fill_amount": 10 }

5. Dynamic subscription (without reconnect)
   Send: { "type": "subscribe", "assets": ["newTokenId"] }
```

### Performance Comparison

| Metric | REST API | WebSocket |
|--------|----------|-----------|
| Latency | ~1000ms (Gamma)<br>~200-500ms (CLOB) | ~100ms |
| Update Model | Pull (poll) | Push (event-driven) |
| Connection | One per request | Persistent connection |
| Overhead | HTTP headers per request | Minimal after connection |
| Best For | Infrequent queries | Real-time streams |

**Critical for Trading:** When milliseconds matter (arbitrage, market making), WebSocket provides ~10x latency improvement over REST polling.

### Hybrid Approach (Best Practice)

**Recommended Pattern:**
```
1. Use Gamma API (REST) to discover markets
2. Use CLOB WebSocket to stream live prices for selected markets
3. Use CLOB API (REST) to place orders
4. Use CLOB WebSocket (user channel) to monitor order fills
5. Use Data API (REST) periodically to reconcile positions
```

**Why Hybrid?**
- Gamma API is not real-time (intentionally - optimized for heavy read traffic)
- CLOB WebSocket provides lowest latency for prices
- Order placement still via REST (signed HTTP request)
- Position data doesn't need real-time (check every few minutes)

### WebSocket Channels

**User Channel:**
- Authentication: Required
- Data: User-specific order and trade updates
- Subscription: By market condition IDs

**Market Channel:**
- Authentication: Not required
- Data: Market-wide orderbook and trade data
- Subscription: By asset token IDs

**Dynamic Subscriptions:**
- Can add/remove subscriptions without reconnecting
- Send subscribe/unsubscribe messages over existing connection
- Critical for multi-market monitoring

**Confidence:** HIGH - Verified through official WebSocket documentation and community resources

---

## Market Structure

### Binary Markets (Standard)

**Structure:**
- Two outcomes: YES and NO
- One market per question
- YES + NO = $1.00 (enforced by arbitrage)

**Token Representation:**
- Each outcome is a separate ERC-1155 token
- Token IDs: Derived from CTF conditionId
- Prices: $0.00 to $1.00 per share

**Example:**
```
Market: "Will Bitcoin reach $100k by end of 2026?"
YES token: tokenId = 0x123...
NO token: tokenId = 0x456...

If YES token trades at $0.62:
  - Implies 62% probability
  - NO token should trade near $0.38
  - Arbitrageurs enforce this relationship
```

**Exchange Contract:** CTF Exchange (standard)

### Multi-Outcome Markets (NegRisk)

**Structure:**
- Multiple mutually exclusive outcomes
- One outcome will resolve YES, all others NO
- Grouped under single event

**Example:**
```
Event: "2026 Presidential Election"
Markets:
  - Candidate A: tokenId = 0xAAA...
  - Candidate B: tokenId = 0xBBB...
  - Candidate C: tokenId = 0xCCC...
  - Other: tokenId = 0xOTH...

Sum of probabilities ≈ 100%
Exactly one resolves YES
```

**Capital Efficiency:**
- Uses NegRisk adapter
- NO positions convertible across markets
- Significant collateral savings

**Exchange Contract:** NegRisk_CTFExchange

### Market Metadata

**From Gamma API:**
```json
{
  "id": "event-id",
  "slug": "will-bitcoin-reach-100k",
  "title": "Will Bitcoin reach $100k by end of 2026?",
  "description": "...",
  "endDate": "2026-12-31T23:59:59Z",
  "resolutionSource": "CoinMarketCap",
  "markets": [
    {
      "id": "market-id",
      "clobTokenIds": ["YES_tokenId", "NO_tokenId"],
      "question": "Will Bitcoin reach $100k by end of 2026?",
      "conditionId": "0x...",
      "questionId": "0x...",
      "negRisk": false,
      "active": true
    }
  ]
}
```

### Market Lifecycle States

| State | Description | Trading Allowed |
|-------|-------------|----------------|
| Active | Open for trading | Yes |
| Closed | Awaiting resolution | No |
| Resolving | Oracle queried | No |
| Resolved | Outcome determined | No (redeem only) |

**Confidence:** HIGH - Verified through official documentation

---

## Smart Contract Addresses (Polygon Mainnet)

### Exchange Contracts

| Contract | Address | Purpose |
|----------|---------|---------|
| CTF Exchange | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` | Standard binary market trading |
| NegRisk CTF Exchange | `0xC5d563A36AE78145C45a50134d48A1215220f80a` | Multi-outcome market trading |

### Token Contracts

| Contract | Address | Standard | Purpose |
|----------|---------|----------|---------|
| USDCe | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | ERC-20 | Collateral |
| CTF Core | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | ERC-1155 | Outcome tokens |

### Adapter Contracts

| Contract | Address | Purpose |
|----------|---------|---------|
| NegRisk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | Capital-efficient settlement for multi-outcome markets |

### Key Network Parameters

- **Chain ID:** 137 (Polygon)
- **RPC:** Multiple providers (Infura, Alchemy, QuickNode)
- **Block Explorer:** https://polygonscan.com

**Confidence:** HIGH - Verified through official setup documentation

---

## Component Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER / APPLICATION                        │
└────────┬────────────┬────────────┬─────────────────┬────────────┘
         │            │            │                 │
    [Discover]   [Trade]      [Monitor]        [Portfolio]
         │            │            │                 │
         ▼            ▼            ▼                 ▼
┌─────────────┐ ┌─────────────┐ ┌──────────────┐ ┌─────────────┐
│  Gamma API  │ │  CLOB API   │ │ CLOB WebSocket│ │  Data API   │
│  (Market    │ │  (Trading)  │ │ (Real-Time)  │ │  (Position) │
│  Metadata)  │ │             │ │              │ │             │
└──────┬──────┘ └──────┬──────┘ └──────┬───────┘ └──────┬──────┘
       │               │                │                │
       │               ▼                │                │
       │        ┌─────────────┐         │                │
       │        │ CLOB Engine │◄────────┘                │
       │        │ (Off-Chain  │                          │
       │        │  Matching)  │                          │
       │        └──────┬──────┘                          │
       │               │                                 │
       ▼               ▼                                 ▼
┌──────────────────────────────────────────────────────────────┐
│                    POLYGON BLOCKCHAIN                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ CTF Exchange│  │ NegRisk Exch │  │  NegRisk Adapter  │   │
│  │ (Standard)  │  │ (Multi-out)  │  │  (Conversions)    │   │
│  └──────┬──────┘  └──────┬───────┘  └─────────┬─────────┘   │
│         │                │                     │              │
│         ▼                ▼                     ▼              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │           CTF Core (ERC-1155 Tokens)                │     │
│  │  - Condition Management                              │     │
│  │  - Token Minting (Split)                             │     │
│  │  - Token Burning (Merge)                             │     │
│  │  - Settlement & Redemption                           │     │
│  └─────────────────────┬───────────────────────────────┘     │
│                        │                                      │
│                        ▼                                      │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              USDC Token (ERC-20)                     │     │
│  │              Collateral Asset                        │     │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │          UMA Oracle (via Adapter)                    │     │
│  │          Resolution Source                           │     │
│  └──────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

**Flow:**
1. User queries Gamma API to discover markets
2. User places order via CLOB API (signed with private key)
3. CLOB Engine matches order off-chain
4. Matched orders settle on-chain via Exchange contracts
5. CTF Core manages token transfers
6. User monitors via WebSocket (real-time) or Data API (periodic)
7. After resolution, UMA Oracle triggers settlement
8. Users redeem winning tokens for USDC

**Confidence:** HIGH - Synthesized from official documentation

---

## Architecture Patterns to Follow

### Pattern 1: Separation of Read and Write

**What:** Use different APIs for different concerns
- Gamma API: Market discovery (read-heavy, cacheable)
- CLOB API: Trading operations (write, time-sensitive)
- Data API: Portfolio tracking (read, user-specific)

**Why:** Each API optimized for its use case (rate limits, caching, latency)

**Example:**
```python
# Discovery (Gamma API)
markets = gamma_client.get_markets(active=True, category="crypto")

# Select market and get live price (CLOB API)
price = clob_client.get_midpoint(token_id=markets[0].clobTokenIds[0])

# Place order (CLOB API)
order = clob_client.create_and_post_order(OrderArgs(...))

# Check position later (Data API)
positions = data_client.get_positions(user=my_address)
```

### Pattern 2: Hybrid REST + WebSocket

**What:** Use REST for state mutations, WebSocket for state observation

**Why:**
- Orders must be signed (REST POST)
- Price updates are events (WebSocket push)
- Combines reliability of REST with latency of WebSocket

**Example:**
```python
# REST: Place order
response = clob_client.post_order(signed_order)

# WebSocket: Monitor fills
ws.subscribe(channel="user", markets=[token_id])
ws.on_message(lambda msg: handle_fill(msg))
```

### Pattern 3: Token Allowance Pre-Check

**What:** Verify allowances before trading, set if needed

**Why:** Order placement fails if Exchange lacks token permissions

**Example:**
```python
# Check USDC allowance
allowance = usdc_contract.allowance(user_address, CTF_EXCHANGE)

if allowance < required_amount:
    # Set allowance
    usdc_contract.approve(CTF_EXCHANGE, MAX_UINT256)
    # Wait for confirmation before trading
```

### Pattern 4: Funder Address Separation

**What:** For proxy wallets, specify funder explicitly

**Why:** Signing wallet may differ from funding wallet

**Example:**
```python
# Email wallet scenario
client = ClobClient(
    key=email_wallet_key,      # Signs orders
    funder=safe_wallet_address, # Holds funds
    signature_type=1            # Poly Proxy
)
```

### Pattern 5: Orderbook Spread Awareness

**What:** Check orderbook depth before market orders

**Why:** Large orders may experience significant slippage

**Example:**
```python
# Get orderbook
book = client.get_order_book(token_id)

# Calculate worst execution price for market buy of 100 shares
shares_remaining = 100
cost = 0
for ask in book.asks:
    if shares_remaining == 0:
        break
    fill = min(shares_remaining, ask.size)
    cost += fill * ask.price
    shares_remaining -= fill

avg_price = cost / 100  # Estimate before executing
```

### Pattern 6: NegRisk Event Detection

**What:** Check `negRisk` flag before trading multi-outcome markets

**Why:** Different exchange contract, different mechanics

**Example:**
```python
event = gamma_client.get_event(event_id)

if event.negRisk:
    # Can convert NO positions across markets
    # Use NegRisk_CTFExchange
    exchange_address = NEGRISK_CTF_EXCHANGE
else:
    # Standard binary market
    exchange_address = CTF_EXCHANGE
```

**Confidence:** HIGH - Synthesized from documentation and best practices

---

## Architecture Anti-Patterns to Avoid

### Anti-Pattern 1: Polling Gamma API for Price Updates

**Problem:** Gamma API is not real-time, has lower rate limits

**Why Bad:** Wasted API calls, stale data, rate limit exhaustion

**Instead:** Use CLOB API or WebSocket for real-time prices

**Example (BAD):**
```python
while True:
    market = gamma_client.get_market(market_id)
    price = market.price  # Stale, not real-time
    time.sleep(1)  # Hitting rate limits
```

**Example (GOOD):**
```python
ws = ClobWebSocket()
ws.subscribe(channel="market", assets=[token_id])
ws.on_message(lambda msg: process_price(msg.price))
```

### Anti-Pattern 2: Ignoring Token Allowances

**Problem:** Attempting to trade without setting ERC-20/ERC-1155 approvals

**Why Bad:** Orders fail validation, wasted gas on reverted transactions

**Detection:** Order submission returns allowance error

**Instead:** Check and set allowances before first trade

### Anti-Pattern 3: Using Same Key for Multiple Bots

**Problem:** Single private key for multiple trading strategies

**Why Bad:**
- Nonce conflicts (parallel transactions)
- No per-strategy risk isolation
- Security: Single key compromise affects all bots

**Instead:** Use separate wallets per strategy, or implement nonce management

### Anti-Pattern 4: Hardcoding Token IDs

**Problem:** Token IDs embedded in code

**Why Bad:** Token IDs are market-specific and change for new markets

**Instead:** Query Gamma API for market metadata, extract token IDs dynamically

**Example (BAD):**
```python
TOKEN_ID = "71321045679252212594626385532706912750332728571942532289631379312455583992563"
```

**Example (GOOD):**
```python
market = gamma_client.get_market_by_slug("will-bitcoin-reach-100k")
yes_token_id = market.clobTokenIds[0]
no_token_id = market.clobTokenIds[1]
```

### Anti-Pattern 5: Assuming Binary Markets Only

**Problem:** Code assumes all markets are binary (YES/NO)

**Why Bad:** NegRisk markets have multiple outcomes

**Instead:** Check event structure and `negRisk` flag

**Example (BAD):**
```python
# Assumes 2 tokens per market
yes_token = market.clobTokenIds[0]
no_token = market.clobTokenIds[1]  # Crashes for NegRisk
```

**Example (GOOD):**
```python
if market.negRisk:
    # Multi-outcome market
    tokens = market.clobTokenIds  # Variable length
else:
    # Binary market
    yes_token = market.clobTokenIds[0]
    no_token = market.clobTokenIds[1]
```

### Anti-Pattern 6: No Error Handling for Rate Limits

**Problem:** Assuming requests always succeed

**Why Bad:** Polymarket queues requests over limit, causing timeouts

**Detection:** Requests hang or timeout under load

**Instead:** Implement timeout handling and backoff

**Example (BAD):**
```python
response = client.get_order_book(token_id)  # May hang if rate limited
```

**Example (GOOD):**
```python
try:
    response = client.get_order_book(token_id, timeout=5)
except TimeoutError:
    # Request was queued, retry with exponential backoff
    time.sleep(2 ** retry_count)
```

### Anti-Pattern 7: Mixing L1 and L2 Authentication

**Problem:** Trying to use API credentials without proper derivation

**Why Bad:** Credentials are wallet-specific and must be derived correctly

**Instead:** Always use `create_or_derive_api_creds()` from client library

**Confidence:** MEDIUM - Synthesized from common issues and best practices

---

## Integration Checklist

For Claude agents integrating with Polymarket:

### Discovery Phase
- [ ] Query Gamma API for market list
- [ ] Filter by category, status, or search terms
- [ ] Extract token IDs from market metadata
- [ ] Check `negRisk` flag for multi-outcome markets

### Authentication Phase
- [ ] Load private key securely
- [ ] Determine signature type (EOA=0, Proxy=1, GnosisSafe=2)
- [ ] Initialize ClobClient with key and chain_id=137
- [ ] Derive or create API credentials
- [ ] Verify authentication with test query

### Allowance Phase (EOA wallets only)
- [ ] Check USDC allowance for CTF_EXCHANGE
- [ ] Check USDC allowance for NEGRISK_CTF_EXCHANGE
- [ ] Check CTF setApprovalForAll for both exchanges
- [ ] Execute approval transactions if needed
- [ ] Wait for confirmations

### Trading Phase
- [ ] Get current orderbook or midpoint price
- [ ] Create OrderArgs (limit) or MarketOrderArgs (market)
- [ ] Sign order with client library
- [ ] Submit order via POST /order
- [ ] Store order ID for tracking

### Monitoring Phase
- [ ] Subscribe to WebSocket user channel (authenticated)
- [ ] Subscribe to WebSocket market channel (price updates)
- [ ] Handle ORDER_MATCHED events
- [ ] Handle ORDER_CANCELLED events
- [ ] Periodically query open orders via REST

### Position Management Phase
- [ ] Query Data API for current positions
- [ ] Calculate PnL from avgPrice and currentPrice
- [ ] Identify redeemable positions (resolved markets)
- [ ] Identify mergeable positions (full sets)
- [ ] Execute redemptions or merges as needed

### Error Handling Phase
- [ ] Implement timeout handling (queued requests)
- [ ] Implement retry with exponential backoff
- [ ] Handle signature errors (wrong signature_type)
- [ ] Handle allowance errors (insufficient approvals)
- [ ] Handle balance errors (insufficient USDC or tokens)

**Confidence:** HIGH - Synthesized from documentation and integration patterns

---

## Sources

### Official Documentation (HIGH Confidence)
- [CLOB Introduction](https://docs.polymarket.com/developers/CLOB/introduction) - Hybrid-decentralized architecture
- [CTF Overview](https://docs.polymarket.com/developers/CTF/overview) - Conditional Token Framework
- [Orders Overview](https://docs.polymarket.com/developers/CLOB/orders/orders) - Order structure and lifecycle
- [WebSocket Overview](https://docs.polymarket.com/developers/CLOB/websocket/wss-overview) - Real-time data channels
- [Methods Overview](https://docs.polymarket.com/developers/CLOB/clients/methods-overview) - Authentication flow
- [NegRisk Overview](https://docs.polymarket.com/developers/neg-risk/overview) - Capital efficiency mechanism
- [Gamma Structure](https://docs.polymarket.com/developers/gamma-markets-api/gamma-structure) - Market hierarchy
- [Data API Positions](https://docs.polymarket.com/developers/misc-endpoints/data-api-get-positions) - User position queries
- [Rate Limits](https://docs.polymarket.com/quickstart/introduction/rate-limits) - API rate limit specifications
- [Setup Documentation](https://docs.polymarket.com/developers/market-makers/setup) - Contract addresses

### Official Repositories (HIGH Confidence)
- [py-clob-client GitHub](https://github.com/Polymarket/py-clob-client) - Python client architecture
- [polymarket-architecture GitHub](https://github.com/ahollic/polymarket-architecture) - Comprehensive technical documentation
- [neg-risk-ctf-adapter GitHub](https://github.com/Polymarket/neg-risk-ctf-adapter) - NegRisk implementation

### Technical Articles (MEDIUM Confidence)
- [The Polymarket API: Architecture, Endpoints, and Use Cases (Jan 2026)](https://medium.com/@gwrx2005/the-polymarket-api-architecture-endpoints-and-use-cases-f1d88fa6c1bf) - Recent architecture overview

### Community Resources (MEDIUM Confidence)
- [Decoding Polymarket's On-Chain Order Data](https://yzc.me/x01Crypto/decoding-polymarket) - On-chain analysis
- [Polymarket WebSocket Tutorial 2025](https://www.polytrackhq.app/blog/polymarket-websocket-tutorial) - WebSocket patterns

**All findings verified against official documentation where possible.**

---

## Summary

Polymarket's architecture is a sophisticated hybrid-decentralized system that:

1. **Separates Concerns**: Three APIs (Gamma, CLOB, Data) each optimized for specific use cases
2. **Balances Tradeoffs**: Off-chain matching for speed + on-chain settlement for security
3. **Enables Efficiency**: NegRisk mechanism dramatically reduces capital requirements for multi-outcome markets
4. **Supports Flexibility**: Multiple wallet types, signature schemes, and authentication flows
5. **Provides Real-Time**: WebSocket channels for low-latency price and order updates
6. **Uses Standards**: EIP712 signing, ERC-1155 tokens, ERC-20 collateral

**Critical for Claude Agents:**
- Use official client libraries (py-clob-client) - order signing is complex
- Check token allowances before first trade (EOA wallets)
- Separate concerns: Gamma for discovery, CLOB for trading, Data for positions
- Use WebSocket for real-time data, REST for mutations
- Detect NegRisk markets (different exchange, different mechanics)
- Implement timeout handling (rate-limited requests are queued, not rejected)

**Architecture Confidence:** HIGH - Comprehensive verification across official documentation, repositories, and technical resources.
