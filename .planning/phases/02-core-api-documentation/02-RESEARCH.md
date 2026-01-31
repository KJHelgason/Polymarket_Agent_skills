# Phase 2: Core API Documentation - Research

**Researched:** 2026-01-31
**Domain:** Polymarket API ecosystem (Gamma, CLOB, Data APIs and WebSocket)
**Confidence:** HIGH

## Summary

Phase 2 documentation covers four distinct API surfaces: the Gamma API for market discovery, the CLOB API for trading operations, the Data API for positions and history, and WebSocket channels for real-time data. Each API serves a specific purpose in the Polymarket ecosystem and requires different documentation approaches.

The Gamma API (`https://gamma-api.polymarket.com`) is a read-only REST service that indexes market metadata, events, categories, and resolution data. The CLOB API (`https://clob.polymarket.com`) handles all trading operations including order placement, cancellation, and orderbook queries. The Data API (`https://data-api.polymarket.com`) provides user-specific data like positions, balances, and trade history. WebSocket services provide real-time updates via two channels: CLOB WebSocket for orderbook and user-specific streams, and RTDS for broader market data.

The py-clob-client library (version 0.34.5, January 2026) provides Python bindings for most operations, but direct REST/WebSocket calls are sometimes needed for full functionality. Documentation must cover both library methods and direct API access patterns.

**Primary recommendation:** Structure documentation by API domain (Gamma for discovery, CLOB for trading, Data for analytics, WebSocket for real-time), with each skill file covering complete workflows including request/response schemas, code examples, and common error handling patterns.

## Standard Stack

The established libraries/tools for Polymarket API integration:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| py-clob-client | 0.34.5+ | Python CLOB client | Official Polymarket library, handles authentication and trading |
| requests | 2.31.0+ | HTTP client | Standard for REST API calls to Gamma and Data APIs |
| websockets | 12.0+ | WebSocket client | Python WebSocket connections for real-time data |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| aiohttp | 3.9.0+ | Async HTTP | High-volume async API calls |
| pydantic | 2.5.0+ | Data validation | Type-safe response parsing |
| python-dotenv | 1.0.0+ | Environment config | Managing API credentials |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| py-clob-client | Direct REST calls | More control but must handle auth manually |
| requests | httpx | Async support but overkill for simple calls |
| websockets | websocket-client | Simpler API but less async support |

**Installation:**
```bash
pip install py-clob-client requests websockets pydantic python-dotenv
```

## Architecture Patterns

### Recommended Skill File Structure
```
skills/polymarket/
├── market-discovery/
│   ├── README.md                    # Index for market discovery
│   ├── gamma-api-overview.md        # Gamma API architecture
│   ├── fetching-markets.md          # Market query patterns
│   ├── events-and-metadata.md       # Event/market data models
│   └── search-and-filtering.md      # Query parameters, pagination
│
├── trading/
│   ├── README.md                    # Index for trading operations
│   ├── clob-api-overview.md         # CLOB API architecture
│   ├── order-types.md               # GTC, GTD, FOK, FAK
│   ├── order-placement.md           # Create, sign, post orders
│   ├── order-management.md          # Cancel, status, batch ops
│   └── positions-and-balances.md    # View positions, check balances
│
├── real-time/
│   ├── README.md                    # Index for real-time data
│   ├── websocket-overview.md        # WebSocket architecture
│   ├── market-channel.md            # Orderbook, price updates
│   ├── user-channel.md              # Order fills, trade notifications
│   └── connection-management.md     # Heartbeat, reconnection
│
└── data-analytics/
    ├── README.md                    # Index for data/analytics
    ├── data-api-overview.md         # Data API architecture
    ├── positions-and-history.md     # Position queries, trade history
    ├── historical-prices.md         # Price timeseries data
    └── portfolio-export.md          # Export patterns for accounting
```

### Pattern 1: Three-API Workflow

**What:** Using Gamma, CLOB, and Data APIs together for complete market interaction

**When to use:** Any application that needs to discover markets, trade, and track results

**Example:**
```python
# Source: Polymarket documentation patterns
import requests
from py_clob_client.client import ClobClient

# 1. GAMMA API: Discover markets
gamma_url = "https://gamma-api.polymarket.com"
response = requests.get(f"{gamma_url}/events", params={
    "active": "true",
    "closed": "false",
    "limit": 10
})
events = response.json()

# 2. CLOB API: Get prices and trade
client = ClobClient(
    host="https://clob.polymarket.com",
    key=PRIVATE_KEY,
    chain_id=137,
    signature_type=0,
    funder=WALLET_ADDRESS
)
client.set_api_creds(client.create_or_derive_api_creds())

# Get market data for a token
token_id = events[0]["markets"][0]["clobTokenIds"][0]
price = client.get_price(token_id, side="BUY")
midpoint = client.get_midpoint(token_id)

# 3. DATA API: Check positions
data_url = "https://data-api.polymarket.com"
positions = requests.get(f"{data_url}/positions", params={
    "user": WALLET_ADDRESS
}).json()
```

### Pattern 2: Order Type Selection

**What:** Choosing the correct order type (GTC, GTD, FOK, FAK) based on trading intent

**When to use:** Every order placement decision

**Example:**
```python
# Source: Polymarket order documentation
from py_clob_client.clob_types import OrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY, SELL

# GTC (Good Till Cancelled) - Default for limit orders
# Use when: You want to wait for your price
order_args = OrderArgs(price=0.45, size=100, side=BUY, token_id=token_id)
signed_order = client.create_order(order_args)
resp = client.post_order(signed_order, OrderType.GTC)

# GTD (Good Till Date) - Limit order with expiration
# Use when: Time-sensitive trades
# Note: Must be at least 1 minute in future
expiration = int(time.time()) + 3600  # 1 hour
order_args = OrderArgs(
    price=0.45, size=100, side=BUY, token_id=token_id,
    expiration=expiration
)
signed_order = client.create_order(order_args)
resp = client.post_order(signed_order, OrderType.GTD)

# FOK (Fill or Kill) - Immediate full execution or nothing
# Use when: All-or-nothing trades, no partial fills acceptable
# WARNING: Stricter decimal precision requirements
order_args = OrderArgs(price=0.45, size=100, side=BUY, token_id=token_id)
signed_order = client.create_order(order_args)
resp = client.post_order(signed_order, OrderType.FOK)

# FAK (Fill and Kill) - Immediate execution, partial fills OK
# Use when: Market orders where you accept partial fills
order_args = OrderArgs(price=0.45, size=100, side=BUY, token_id=token_id)
signed_order = client.create_order(order_args)
resp = client.post_order(signed_order, OrderType.FAK)
```

### Pattern 3: WebSocket Subscription Management

**What:** Managing real-time data subscriptions with proper lifecycle handling

**When to use:** Any real-time market data or order tracking application

**Example:**
```python
# Source: Polymarket WebSocket documentation
import asyncio
import websockets
import json

CLOB_WS_MARKET = "wss://ws-subscriptions-clob.polymarket.com/ws/market"
CLOB_WS_USER = "wss://ws-subscriptions-clob.polymarket.com/ws/user"

async def subscribe_to_market(token_ids: list):
    """Subscribe to market channel for orderbook updates."""
    async with websockets.connect(CLOB_WS_MARKET) as ws:
        # Subscribe message
        subscribe_msg = {
            "type": "market",
            "assets_ids": token_ids
        }
        await ws.send(json.dumps(subscribe_msg))

        # Heartbeat task
        async def heartbeat():
            while True:
                await asyncio.sleep(5)
                await ws.ping()

        heartbeat_task = asyncio.create_task(heartbeat())

        try:
            async for message in ws:
                data = json.loads(message)
                event_type = data.get("event_type")

                if event_type == "book":
                    # Full orderbook snapshot
                    print(f"Book: {data['bids'][:3]} / {data['asks'][:3]}")
                elif event_type == "price_change":
                    # Incremental update
                    print(f"Price change: {data}")
                elif event_type == "last_trade_price":
                    # Trade executed
                    print(f"Trade: {data['price']} x {data['size']}")
        finally:
            heartbeat_task.cancel()

async def subscribe_to_user(api_creds: dict, market_ids: list):
    """Subscribe to user channel for order updates (authenticated)."""
    async with websockets.connect(CLOB_WS_USER) as ws:
        # Authenticated subscription
        subscribe_msg = {
            "type": "user",
            "auth": {
                "apiKey": api_creds["apiKey"],
                "secret": api_creds["secret"],
                "passphrase": api_creds["passphrase"]
            },
            "markets": market_ids
        }
        await ws.send(json.dumps(subscribe_msg))

        async for message in ws:
            data = json.loads(message)
            # Handle order updates, trade notifications
            print(f"User event: {data}")
```

### Anti-Patterns to Avoid

- **Polling Gamma API for real-time prices:** Use CLOB WebSocket for price updates, Gamma is for discovery only
- **Caching tick sizes indefinitely:** Tick sizes change dynamically when price > 0.96 or < 0.04
- **Using FOK orders without checking precision:** FOK has stricter decimal requirements than GTC
- **Single WebSocket for all symbols:** RTDS supports only ONE symbol per connection; create multiple connections
- **Ignoring WebSocket heartbeats:** Connections drop after ~20 minutes without proper ping/pong
- **Not handling negRisk markets differently:** NegRisk markets have different trading mechanics and capital efficiency
- **Assuming all events have one market:** Events can have multiple markets (multi-outcome questions)

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Order signing | Custom EIP-712 implementation | py-clob-client.create_order() | Complex signature format with specific domain separators |
| Pagination | Manual offset tracking | Generator/iterator patterns | Easy to miss edge cases, off-by-one errors |
| WebSocket reconnection | Simple retry loop | Exponential backoff with subscription state | Need to restore subscriptions, handle partial state |
| Tick size validation | Manual decimal checks | client.get_tick_size() + ROUNDING_CONFIG | Dynamic tick sizes, order-type-specific rules |
| Price precision | Hardcoded decimal places | Use tick size hierarchy table | Different precisions for price vs size vs amount |
| Orderbook maintenance | Full refresh on each update | Incremental updates from price_change events | Bandwidth intensive, misses fast changes |

**Key insight:** The Polymarket API has many subtle rules around precision, tick sizes, and order validation. The py-clob-client handles most of these internally, but direct API users must implement them correctly or face "invalid order" rejections.

## Common Pitfalls

### Pitfall 1: FOK Order Decimal Precision

**What goes wrong:** FOK orders fail with "invalid amounts" error even when GTC orders with same parameters succeed.

**Why it happens:** FOK (Fill-or-Kill) market orders have stricter precision requirements than GTC limit orders. Maker amount limited to 2 decimals, taker amount to 4 decimals, and size x price must not exceed 2 decimals.

**How to avoid:**
- For FOK orders, round size to 2 decimal places
- Ensure price x size product has max 2 decimal places
- Use GTC for orders requiring more precision
- Test with GTC first, then adapt for FOK

**Warning signs:**
- Error: "invalid amounts, the sell orders maker amount supports a max accuracy of 2 decimals"
- Same order works as GTC but fails as FOK

### Pitfall 2: WebSocket Data Stream Stops After ~20 Minutes

**What goes wrong:** WebSocket connection remains open but stops receiving data after approximately 20 minutes.

**Why it happens:** Known issue with Polymarket WebSocket connections. The connection stays "healthy" but data stops flowing without explicit disconnection.

**How to avoid:**
- Implement 5-second heartbeat (ping) interval
- Add data timeout detection (if no data for 5 minutes, reconnect)
- Use automatic reconnection with exponential backoff
- Re-subscribe to all topics after reconnection

**Warning signs:**
- Connection reports as open but no new messages
- Last message timestamp is 15-20+ minutes old
- Ping/pong still succeeds

**Workaround pattern:**
```python
DATA_TIMEOUT_MS = 5 * 60 * 1000  # 5 minutes
last_data_time = time.time()

async def check_data_timeout():
    if time.time() - last_data_time > DATA_TIMEOUT_MS / 1000:
        await reconnect_and_resubscribe()
```

### Pitfall 3: Dynamic Tick Size Changes

**What goes wrong:** Orders fail with "INVALID_ORDER_MIN_TICK_SIZE" error after price moves significantly.

**Why it happens:** Tick sizes change dynamically when market price exceeds 0.96 or falls below 0.04. A market that previously had 0.01 tick size may now require 0.001 or 0.0001 precision.

**How to avoid:**
- Always fetch current tick size before order placement
- Don't cache tick sizes for extended periods (refresh every few minutes)
- Handle tick_size_change WebSocket events
- Use client.get_tick_size(token_id) with cache=False for critical orders

**Warning signs:**
- Tick size change events in WebSocket stream
- Orders that previously worked now fail
- Market price near 0.96+ or 0.04-

### Pitfall 4: NegRisk Market Confusion

**What goes wrong:** Attempting to buy YES shares in multiple markets of same event at full price, not understanding capital efficiency.

**Why it happens:** NegRisk (Negative Risk) events have special mechanics where only one market can resolve YES. Holding NO shares can be converted to YES shares in all other markets.

**How to avoid:**
- Check event's `negRisk` boolean field before trading
- Understand that in negRisk events, buying all outcomes costs only 1.0 (max payout)
- Trade only on "named" outcomes (unnamed outcomes are placeholders)
- Use the conversion feature for capital efficiency

**Warning signs:**
- Event has multiple markets but only one can win
- `negRisk: true` in Gamma API response
- Higher than expected capital requirements for multi-leg trades

### Pitfall 5: Events vs Markets Confusion

**What goes wrong:** Treating market IDs and event IDs interchangeably, querying wrong endpoints.

**Why it happens:** Polymarket has a hierarchical structure: Events contain 1+ Markets. An event is a proposition (e.g., "2024 Election Winner"), and markets are specific options (e.g., "Biden", "Trump").

**How to avoid:**
- Use Gamma `/events` endpoint for discovering topics
- Use Gamma `/markets` endpoint for specific trading options
- Events have `id` field, markets have `conditionId` and `clobTokenIds`
- Each market has exactly 2 tokens (YES and NO outcomes)

**Identification:**
```python
# Event: Question/proposition level
event = {
    "id": "16085",
    "slug": "fed-decision-october",
    "markets": [...]  # Contains 1+ markets
}

# Market: Specific tradable outcome
market = {
    "conditionId": "0x123...",  # Used for CLOB queries
    "clobTokenIds": ["71321...", "72541..."],  # YES and NO token IDs
    "question": "Will the Fed raise rates?"
}
```

## Code Examples

Verified patterns from official sources:

### Gamma API: Fetch Active Markets with Pagination
```python
# Source: Polymarket Gamma API documentation
import requests

GAMMA_URL = "https://gamma-api.polymarket.com"

def fetch_all_active_events(limit=50):
    """Fetch all active events with pagination."""
    events = []
    offset = 0

    while True:
        response = requests.get(f"{GAMMA_URL}/events", params={
            "active": "true",
            "closed": "false",
            "order": "id",
            "ascending": "false",
            "limit": limit,
            "offset": offset
        })
        response.raise_for_status()

        batch = response.json()
        if not batch:
            break

        events.extend(batch)

        if len(batch) < limit:
            break

        offset += limit

    return events

def fetch_event_by_slug(slug: str):
    """Fetch single event by URL slug."""
    response = requests.get(f"{GAMMA_URL}/events/slug/{slug}")
    response.raise_for_status()
    return response.json()

def fetch_markets_by_tag(tag_id: int, limit=100):
    """Fetch markets filtered by category tag."""
    response = requests.get(f"{GAMMA_URL}/markets", params={
        "tag_id": tag_id,
        "active": "true",
        "closed": "false",
        "limit": limit
    })
    response.raise_for_status()
    return response.json()
```

### CLOB API: Order Lifecycle
```python
# Source: Polymarket CLOB documentation
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import OrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY, SELL

def complete_order_lifecycle(client: ClobClient, token_id: str):
    """Demonstrate full order lifecycle."""

    # 1. Check market data
    tick_size = client.get_tick_size(token_id)
    midpoint = client.get_midpoint(token_id)
    book = client.get_order_book(token_id)
    print(f"Tick: {tick_size}, Mid: {midpoint}")
    print(f"Best bid: {book['bids'][0] if book['bids'] else 'None'}")
    print(f"Best ask: {book['asks'][0] if book['asks'] else 'None'}")

    # 2. Create and post limit order (GTC)
    order_args = OrderArgs(
        price=0.40,  # Limit price
        size=50.0,   # Number of shares
        side=BUY,
        token_id=token_id
    )
    signed_order = client.create_order(order_args)
    response = client.post_order(signed_order, OrderType.GTC)

    if response.get("success"):
        order_id = response["orderId"]
        print(f"Order placed: {order_id}")
        print(f"Status: {response['status']}")  # live, matched, delayed

        # 3. Check order status
        order = client.get_order(order_id)
        print(f"Size matched: {order.get('size_matched', 0)}")

        # 4. Cancel if still open
        if order.get("status") == "LIVE":
            cancel_resp = client.cancel(order_id)
            print(f"Cancelled: {cancel_resp}")
    else:
        print(f"Order failed: {response.get('errorMsg')}")

    return response

def batch_order_example(client: ClobClient, token_id: str):
    """Place multiple orders in single request (max 15)."""
    from py_clob_client.clob_types import PostOrdersArgs

    orders = []
    for price in [0.40, 0.41, 0.42]:
        order_args = OrderArgs(
            price=price,
            size=20.0,
            side=BUY,
            token_id=token_id
        )
        signed = client.create_order(order_args)
        orders.append(PostOrdersArgs(
            order=signed,
            orderType=OrderType.GTC
        ))

    # Post batch (max 15 orders)
    responses = client.post_orders(orders)

    for i, resp in enumerate(responses):
        print(f"Order {i+1}: {'OK' if resp['success'] else resp['errorMsg']}")

    return responses
```

### Data API: Positions and Trade History
```python
# Source: Polymarket Data API documentation
import requests

DATA_URL = "https://data-api.polymarket.com"

def get_user_positions(wallet_address: str, limit=100, sort_by="TOKENS"):
    """Get current positions for a user."""
    response = requests.get(f"{DATA_URL}/positions", params={
        "user": wallet_address,
        "limit": limit,
        "sortBy": sort_by,  # CURRENT, INITIAL, TOKENS, CASHPNL, PERCENTPNL
        "sortDirection": "DESC",
        "sizeThreshold": 1  # Minimum position size
    })
    response.raise_for_status()
    return response.json()

def get_trade_history(wallet_address: str, activity_types=None, limit=100):
    """Get trade history for a user."""
    params = {
        "user": wallet_address,
        "limit": limit,
        "sortBy": "TIMESTAMP",
        "sortDirection": "DESC"
    }

    if activity_types:
        # TRADE, SPLIT, MERGE, REDEEM, REWARD, CONVERSION, MAKER_REBATE
        params["type"] = ",".join(activity_types)

    response = requests.get(f"{DATA_URL}/activity", params=params)
    response.raise_for_status()
    return response.json()

def get_historical_prices(token_id: str, interval="1d"):
    """Get historical price data for a token."""
    # Via CLOB API
    response = requests.get("https://clob.polymarket.com/prices-history", params={
        "market": token_id,
        "interval": interval  # 1m, 1h, 6h, 1d, 1w, max
    })
    response.raise_for_status()
    return response.json()

# Response structure for positions:
# {
#     "proxyWallet": "0x...",
#     "asset": "71321...",
#     "conditionId": "0x...",
#     "size": 100.0,
#     "avgPrice": 0.45,
#     "initialValue": 45.0,
#     "currentValue": 50.0,
#     "cashPnl": 5.0,
#     "percentPnl": 11.11,
#     "curPrice": 0.50,
#     "title": "Will X happen?",
#     "outcome": "Yes",
#     "endDate": "2026-12-31T00:00:00Z"
# }
```

### WebSocket: Market and User Channels
```python
# Source: Polymarket WebSocket documentation
import asyncio
import websockets
import json
from datetime import datetime

MARKET_WS = "wss://ws-subscriptions-clob.polymarket.com/ws/market"
USER_WS = "wss://ws-subscriptions-clob.polymarket.com/ws/user"

class PolymarketWebSocket:
    def __init__(self):
        self.last_data_time = None
        self.running = False

    async def connect_market_channel(self, token_ids: list, callback):
        """Connect to market channel for orderbook updates."""
        self.running = True

        while self.running:
            try:
                async with websockets.connect(MARKET_WS) as ws:
                    # Subscribe
                    await ws.send(json.dumps({
                        "type": "market",
                        "assets_ids": token_ids
                    }))

                    # Start heartbeat
                    heartbeat = asyncio.create_task(self._heartbeat(ws))
                    timeout_check = asyncio.create_task(self._check_timeout())

                    try:
                        async for message in ws:
                            self.last_data_time = datetime.now()
                            data = json.loads(message)
                            await callback(data)
                    finally:
                        heartbeat.cancel()
                        timeout_check.cancel()

            except websockets.ConnectionClosed:
                print("Connection closed, reconnecting in 5s...")
                await asyncio.sleep(5)
            except Exception as e:
                print(f"Error: {e}, reconnecting in 10s...")
                await asyncio.sleep(10)

    async def connect_user_channel(self, api_creds: dict, market_ids: list, callback):
        """Connect to user channel for order/trade updates (authenticated)."""
        self.running = True

        while self.running:
            try:
                async with websockets.connect(USER_WS) as ws:
                    # Authenticated subscribe
                    await ws.send(json.dumps({
                        "type": "user",
                        "auth": {
                            "apiKey": api_creds["apiKey"],
                            "secret": api_creds["secret"],
                            "passphrase": api_creds["passphrase"]
                        },
                        "markets": market_ids
                    }))

                    heartbeat = asyncio.create_task(self._heartbeat(ws))

                    try:
                        async for message in ws:
                            self.last_data_time = datetime.now()
                            data = json.loads(message)
                            await callback(data)
                    finally:
                        heartbeat.cancel()

            except websockets.ConnectionClosed:
                print("User channel closed, reconnecting...")
                await asyncio.sleep(5)

    async def _heartbeat(self, ws, interval=5):
        """Send ping every 5 seconds."""
        while True:
            await asyncio.sleep(interval)
            try:
                await ws.ping()
            except:
                break

    async def _check_timeout(self, timeout_seconds=300):
        """Check for data timeout (5 minutes)."""
        while self.running:
            await asyncio.sleep(60)
            if self.last_data_time:
                elapsed = (datetime.now() - self.last_data_time).seconds
                if elapsed > timeout_seconds:
                    print(f"Data timeout ({elapsed}s), triggering reconnect")
                    raise websockets.ConnectionClosed(None, None)

    def stop(self):
        self.running = False

# Message types from market channel:
# - "book": Full orderbook snapshot (on subscribe and after trades)
# - "price_change": Incremental price level changes
# - "tick_size_change": When tick size changes (price > 0.96 or < 0.04)
# - "last_trade_price": Trade execution notification

# Message types from user channel:
# - Trade messages: MATCHED, MINED, CONFIRMED, RETRYING, FAILED
# - Order messages: PLACEMENT, UPDATE, CANCELLATION
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Single API for all data | Three-API architecture (Gamma/CLOB/Data) | 2024+ | Better separation of concerns, optimized endpoints |
| Manual WebSocket management | Client libraries with reconnection | 2025+ | More reliable real-time data |
| Fixed tick sizes | Dynamic tick sizes | 2024+ | More precision for extreme prices |
| Simple binary markets | NegRisk multi-outcome markets | 2024+ | Capital-efficient multi-leg trading |
| REST-only price data | WebSocket price_change events | Ongoing | Lower latency, reduced API calls |

**Deprecated/outdated:**
- **RTDS for single symbol crypto prices:** Limited to one symbol per connection, use CLOB WebSocket for market data
- **Caching tick sizes without refresh:** Tick sizes change dynamically, must refresh regularly
- **Ignoring augmented negRisk:** Newer markets use augmented negRisk with placeholder outcomes

## Open Questions

Things that couldn't be fully resolved:

1. **Exact L2 HMAC Construction**
   - What we know: Uses HMAC-SHA256 with POLY-* headers, 30-second expiration
   - What's unclear: Precise message format for manual implementation
   - Recommendation: Use py-clob-client library; don't implement manually

2. **WebSocket ~20 Minute Timeout Root Cause**
   - What we know: Data stops after ~20 minutes, connection remains "open"
   - What's unclear: Whether this is server-side or client-side issue
   - Recommendation: Implement 5-minute data timeout check and auto-reconnect

3. **Rate Limits for Data API**
   - What we know: CLOB API has documented rate limits
   - What's unclear: Specific limits for Data API endpoints
   - Recommendation: Implement conservative rate limiting, add exponential backoff

4. **Historical Data Retention Period**
   - What we know: prices-history endpoint exists with various intervals
   - What's unclear: How far back data is available ("max" interval range)
   - Recommendation: Test with target markets, document actual availability

5. **NegRisk Conversion API**
   - What we know: Conversion happens via smart contract (NegRiskAdapter)
   - What's unclear: Whether there's a convenient py-clob-client method
   - Recommendation: Document as on-chain operation if no API method exists

## Sources

### Primary (HIGH confidence)
- [Polymarket Official Documentation](https://docs.polymarket.com/) - All API documentation
- [py-clob-client GitHub](https://github.com/Polymarket/py-clob-client) - Official Python client (v0.34.5)
- [Polymarket Gamma API Overview](https://docs.polymarket.com/developers/gamma-markets-api/overview) - Market discovery
- [Polymarket CLOB Order Documentation](https://docs.polymarket.com/developers/CLOB/orders/create-order) - Order placement
- [Polymarket WebSocket Overview](https://docs.polymarket.com/developers/CLOB/websocket/wss-overview) - WebSocket channels
- [Polymarket Data API Positions](https://docs.polymarket.com/developers/misc-endpoints/data-api-get-positions) - Position queries
- [Polymarket Data API Activity](https://docs.polymarket.com/developers/misc-endpoints/data-api-activity) - Trade history
- [Polymarket NegRisk Overview](https://docs.polymarket.com/developers/neg-risk/overview) - Multi-outcome markets

### Secondary (MEDIUM confidence)
- [Polymarket Fetch Markets Guide](https://docs.polymarket.com/developers/gamma-markets-api/fetch-markets-guide) - Query patterns
- [Polymarket CLOB Public Methods](https://docs.polymarket.com/developers/CLOB/clients/methods-public) - Read-only methods
- [Polymarket CLOB L2 Methods](https://docs.polymarket.com/developers/CLOB/clients/methods-l2) - Authenticated methods
- [Polymarket Historical Timeseries](https://docs.polymarket.com/developers/CLOB/timeseries) - Price history
- [NautilusTrader Polymarket Integration](https://nautilustrader.io/docs/latest/integrations/polymarket/) - Third-party integration patterns

### Tertiary (LOW confidence - marked for validation)
- [WebSocket Data Stream Issue #26](https://github.com/Polymarket/real-time-data-client/issues/26) - ~20 minute timeout
- [FOK Order Decimal Places Issue #121](https://github.com/Polymarket/py-clob-client/issues/121) - Precision requirements
- Community blog posts on Polymarket API usage - May contain errors or outdated info

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Official libraries verified, versions confirmed from PyPI
- Architecture: HIGH - Based on official documentation structure and API design
- Pitfalls: HIGH - Verified through official docs, GitHub issues, and API error messages
- Code examples: HIGH - Derived from official documentation and library source

**Research date:** 2026-01-31
**Valid until:** 2026-03-02 (30 days - APIs generally stable but WebSocket behaviors may evolve)

**Research limitations:**
- Exact Data API rate limits not documented
- WebSocket ~20 minute issue is observed behavior, not officially documented
- NegRisk conversion may have py-clob-client method not found in current docs
- Historical data retention period needs empirical testing

**Recommended validation:**
- Test WebSocket reconnection with 5-minute timeout in production
- Verify FOK precision requirements with small test orders
- Check Data API rate limits through experimentation
- Confirm historical data availability for target markets
