# Pitfalls Research: Polymarket API

**Domain:** Polymarket Prediction Markets API Integration
**Researched:** 2026-01-31
**Overall Confidence:** MEDIUM-HIGH

## Executive Summary

Polymarket API integrations commonly fail due to misunderstandings about order constraints, authentication flows, USDC token variants, and price/odds interpretation. The most critical pitfalls involve: (1) USDC.e vs Native USDC confusion causing $0 balances, (2) proxy wallet vs funder address mismatches in authentication, (3) decimal precision errors causing order rejections, and (4) misinterpreting displayed prices as executable prices.

---

## Critical Pitfalls

Mistakes that cause integration failures or major issues.

### Pitfall 1: USDC.e vs Native USDC Confusion

**What goes wrong:**
Developers deposit Native USDC from exchanges (Coinbase, Kraken) and see a $0.00 balance in Polymarket, assuming funds are lost. The funds exist but in the wrong token variant.

**Why it happens:**
Polymarket was built in early Polygon days and exclusively uses Bridged USDC (USDC.e) as collateral. Major exchanges now default to Native USDC on Polygon. Developers don't realize there are two different USDC token contracts on Polygon PoS.

**Consequences:**
- Balance shows $0.00 despite successful deposit
- Trading fails with "insufficient balance" errors
- Users panic, thinking funds are lost
- Wasted time investigating wallet issues

**Prevention:**
```python
# WRONG: Assuming any USDC works
# deposit_usdc()  # Uses Native USDC from Coinbase

# RIGHT: Verify you're using USDC.e
# Token addresses on Polygon PoS:
USDC_E_ADDRESS = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"  # Bridged USDC (USDC.e) - USE THIS
NATIVE_USDC_ADDRESS = "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359"  # Native USDC - WRONG

# Always bridge to USDC.e or use Polymarket's "Activate Funds" feature
```

**Detection:**
- Check if balance is $0.00 after deposit
- Look for "Activate Funds" banner in Polymarket UI
- Verify token contract address matches USDC.e (0x2791Bca...)

**Workaround:**
Click "Activate Funds" in Polymarket UI to perform gasless/tiny-fee swap from Native USDC to USDC.e via Uniswap pool.

**Sources:**
- [USDC vs. USDC.e on Polymarket: Why Your Balance Shows $0.00 (2026 Guide)](https://www.homesfound.ca/blog/usdc-vs-usdce-polymarket-why-your-balance-shows-000-2026-guide/)
- [How to Bridge Funds to Polymarket: The Complete 2026 Guide](https://www.homesfound.ca/blog/how-bridge-funds-polymarket-complete-2026-guide/)
- [Native USDC Is on Polygon PoS](https://polygon.technology/blog/phase-one-of-native-usdc-migration-on-polygon-pos-is-underway)

**Confidence:** HIGH

---

### Pitfall 2: Proxy Wallet vs Funder Address Confusion

**What goes wrong:**
Developers use their EOA (signing wallet) address as the funder address in API initialization, causing authentication errors or orders attributed to wrong account.

**Why it happens:**
Polymarket uses a dual-address system:
- **Login Address (EOA):** Controls the account, signs transactions
- **Proxy Address (Funder):** Where funds actually sit, receives deposits

When initializing the CLOB client, developers incorrectly specify their signing address instead of their proxy wallet address as the funder.

**Consequences:**
- 401 Unauthorized errors during order placement
- Orders rejected with "insufficient balance" despite having funds
- Orders attributed to wrong account
- Confusion about which address to use for deposits

**Prevention:**
```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import SignatureType

# WRONG: Using your EOA/signing address as funder
client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    signature_type=SignatureType.POLY_PROXY,
    funder="0xYourEOAAddress..."  # WRONG
)

# RIGHT: Use your proxy wallet address (check polymarket.com/settings)
# The proxy address is displayed in Polymarket settings and is where funds sit
client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    signature_type=SignatureType.POLY_PROXY,
    funder="0xYourProxyWalletAddress..."  # From polymarket.com/settings
)

# SDK auto-derives funder with CREATE2 for POLY_PROXY and POLY_GNOSIS_SAFE
# For EOA signatures, EOA address IS the funder address
```

**Detection:**
- 401 Unauthorized errors with "Invalid api key"
- Orders failing when you have balance
- Check polymarket.com/settings for displayed address (this is proxy, not EOA)
- Verify PolygonScan shows funds at proxy address, not EOA

**Mitigation:**
1. Go to polymarket.com/settings to find your proxy wallet address
2. Verify proxy wallet is deployed (happens on first Polymarket login)
3. Use proxy address as funder parameter
4. For EOA signatures, signing address = funder address
5. For POLY_PROXY/GNOSIS_SAFE, SDK derives funder automatically

**Sources:**
- [Polymarket proxy wallet documentation](https://docs.polymarket.com/developers/proxy-wallet)
- [How to Find Your Polymarket Wallet Address: A 2026 Visual Guide](https://www.homesfound.ca/blog/how-find-your-polymarket-wallet-address-2026-visual-guide/)
- [Polymarket authentication documentation](https://docs.polymarket.com/developers/CLOB/authentication)

**Confidence:** HIGH

---

### Pitfall 3: Decimal Precision Errors by Order Type

**What goes wrong:**
Orders are rejected with "invalid amounts, the sell orders maker amount supports a max accuracy of 2 decimals, taker amount a max of 4 decimals" error, despite working fine when changed from FOK to GTC.

**Why it happens:**
Polymarket enforces different decimal precision constraints based on **order type** (FOK vs GTC) and **tick size**. FOK orders have stricter limits than GTC orders.

**Precision Limits:**

| Order Type | Maker Amount | Taker Amount | Notes |
|------------|--------------|--------------|-------|
| **FOK (Fill-or-Kill)** | 2 decimals max | 4 decimals max | size × price must not exceed 2 decimals |
| **GTC (Good-til-Cancelled)** | Based on tick size | Based on tick size | More flexible, typically 6 decimals for binary options |
| **GTD (Good-til-Date)** | Based on tick size | Based on tick size | Same as GTC |

**Consequences:**
- FOK orders rejected when GTC orders succeed with same amounts
- Mysterious "invalid amounts" errors
- Developers assume bug in API rather than precision constraints

**Prevention:**
```python
from decimal import Decimal, ROUND_DOWN

def round_for_fok(amount: float, is_maker: bool) -> str:
    """Round amounts for FOK orders to meet precision requirements."""
    if is_maker:
        # Maker amount: 2 decimals max for FOK
        return str(Decimal(str(amount)).quantize(Decimal('0.01'), rounding=ROUND_DOWN))
    else:
        # Taker amount: 4 decimals max for FOK
        return str(Decimal(str(amount)).quantize(Decimal('0.0001'), rounding=ROUND_DOWN))

def validate_fok_order(size: float, price: float):
    """Validate FOK order meets precision constraints."""
    product = Decimal(str(size)) * Decimal(str(price))
    product_rounded = product.quantize(Decimal('0.01'), rounding=ROUND_DOWN)

    if product != product_rounded:
        raise ValueError(f"FOK order product (size × price = {product}) exceeds 2 decimal places")

    return True

# WRONG: Using high precision for FOK
place_order(
    size=100.123456,  # Too many decimals for FOK maker
    price=0.5234,     # Product might exceed 2 decimals
    time_in_force="FOK"
)

# RIGHT: Round appropriately for FOK
size_rounded = round_for_fok(100.123456, is_maker=True)  # "100.12"
price_rounded = round_for_fok(0.5234, is_maker=False)    # "0.5234"
validate_fok_order(float(size_rounded), float(price_rounded))
place_order(size=size_rounded, price=price_rounded, time_in_force="FOK")

# OR: Use GTC for higher precision
place_order(size=100.123456, price=0.5234, time_in_force="GTC")  # Works fine
```

**Detection:**
- Error message: "invalid amounts, the sell orders maker amount supports a max accuracy of 2 decimals"
- FOK order fails but same amounts work as GTC
- Check py-clob-client ROUNDING_CONFIG for current precision rules

**Additional Complexity:**
- Tick sizes change dynamically when price > 0.96 or price < 0.04
- Listen for `tick_size_change` WebSocket messages
- Binary options typically support 0.0001 tick size (6 decimal places for amounts)

**Sources:**
- [FOK order decimal places error · Issue #121](https://github.com/Polymarket/py-clob-client/issues/121)
- [Polymarket | NautilusTrader Documentation](https://nautilustrader.io/docs/nightly/integrations/polymarket/)
- [Market Channel - Polymarket Documentation](https://docs.polymarket.com/developers/CLOB/websocket/market-channel)

**Confidence:** HIGH

---

### Pitfall 4: Authentication Nonce Reuse and Lost Credentials

**What goes wrong:**
Developers reuse nonces when creating API credentials, causing authentication errors. Or they lose their API credentials and can't recover them because they didn't save the nonce.

**Why it happens:**
Polymarket uses a two-tier authentication system:
- **L1 (Private Key):** Signs EIP-712 messages to create/derive API credentials
- **L2 (API Key):** Uses HMAC-SHA256 signatures for subsequent requests

Nonces must be unique when creating new credentials. Lost nonces make credential recovery impossible.

**Consequences:**
- Reusing nonces causes "nonce already used" errors
- Lost credentials without nonce = must create entirely new credentials
- No recovery mechanism for lost API keys

**Prevention:**
```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import SignatureType

# SCENARIO 1: Creating NEW credentials
# Generate and SAVE the nonce
nonce = 12345  # Use unique value, save this!

client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    signature_type=SignatureType.EOA,
    funder=your_address
)

# Creates new API credentials with this nonce
# CRITICAL: Save the returned api_key, api_secret, api_passphrase AND nonce
creds = client.create_api_key(nonce=nonce)

# Store these securely:
# - creds['api_key']
# - creds['api_secret']
# - creds['api_passphrase']
# - nonce (for recovery via derive_api_key)

# SCENARIO 2: Recovering EXISTING credentials
# Only works if you saved the nonce!
existing_nonce = 12345  # Must be the original nonce

client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    signature_type=SignatureType.EOA,
    funder=your_address
)

# Derives existing credentials from nonce
creds = client.derive_api_key(nonce=existing_nonce)

# WRONG: Trying to recover without nonce
# There is NO way to recover lost credentials without the nonce
# You MUST create new credentials with a NEW nonce
```

**Detection:**
- "Nonce already used" errors when creating credentials
- 401 Unauthorized errors with "Invalid api key"
- Unable to derive credentials because nonce was lost

**Mitigation:**
1. **Always save nonces** when creating credentials
2. Use `derive_api_key(nonce)` to recover existing credentials
3. Use `create_api_key(new_nonce)` for fresh credentials
4. Store credentials in environment variables or secure vault
5. If lost without nonce: create entirely new credentials

**Sources:**
- [Polymarket Authentication Documentation](https://docs.polymarket.com/developers/CLOB/authentication)
- [401 Unauthorized / Invalid api key · Issue #187](https://github.com/Polymarket/py-clob-client/issues/187)
- [Periodically Unauthorized/Invalid api key · Issue #190](https://github.com/Polymarket/py-clob-client/issues/190)

**Confidence:** HIGH

---

## Moderate Pitfalls

Mistakes that cause delays or technical debt.

### Pitfall 5: Misinterpreting Displayed Prices as Executable

**What goes wrong:**
Developers see a market at "37%" and assume they can buy YES shares at $0.37, but actual execution requires consuming liquidity at bid/ask spread, resulting in worse prices.

**Why it happens:**
**Displayed probability/price is the midpoint of bid-ask spread** (or last trade if spread > $0.10). This is NOT an executable price—it's a summary statistic. To execute, you pay the ask (buying) or hit the bid (selling).

**Example:**
```
Market shows: 37% probability
Actual order book:
  - Best bid: $0.34 (what sellers can get)
  - Best ask: $0.40 (what buyers must pay)
  - Midpoint: $0.37 (displayed price)
```

If you want to buy YES shares, you pay $0.40, not $0.37. Large orders walk up the book and pay progressively worse prices.

**Consequences:**
- Backtests using midpoint prices are unrealistic
- Market orders execute at worse prices than expected
- Slippage surprises when large orders move the market
- Incorrect ROI calculations

**Prevention:**
```python
# WRONG: Using displayed price for calculations
market_data = get_market("condition_id")
displayed_price = market_data['price']  # 0.37 (midpoint)
expected_cost = shares * displayed_price  # WRONG

# RIGHT: Use actual order book
order_book = get_order_book("token_id")
best_ask = order_book['asks'][0]['price']  # 0.40 (actual cost to buy)
best_bid = order_book['bids'][0]['price']  # 0.34 (actual proceeds from sell)

# For buying YES shares
actual_cost = shares * float(best_ask)

# For selling YES shares
actual_proceeds = shares * float(best_bid)

# For market impact estimation
def estimate_fill_price(order_book_side, shares_wanted):
    """Calculate average fill price consuming liquidity."""
    total_cost = 0
    shares_remaining = shares_wanted

    for level in order_book_side:
        level_shares = float(level['size'])
        level_price = float(level['price'])

        if shares_remaining <= level_shares:
            total_cost += shares_remaining * level_price
            shares_remaining = 0
            break
        else:
            total_cost += level_shares * level_price
            shares_remaining -= level_shares

    if shares_remaining > 0:
        raise ValueError("Insufficient liquidity for order")

    return total_cost / shares_wanted

# Use for market orders
avg_fill_price = estimate_fill_price(order_book['asks'], 1000)
```

**Detection:**
- Backtest results don't match live trading
- Market orders fill at worse prices than displayed
- Slippage consistently exceeds expectations

**Key Concepts:**
- **Displayed price = midpoint** (or last trade if spread > $0.10)
- **Buy price = ask price** (pay more than midpoint)
- **Sell price = bid price** (receive less than midpoint)
- **Spread = ask - bid** (cost of immediacy)
- **Large orders walk the book** (progressively worse prices)

**Sources:**
- [How Are Prices Calculated? - Polymarket Documentation](https://docs.polymarket.com/polymarket-learn/trading/how-are-prices-calculated)
- [The Complete Polymarket Playbook (2026)](https://medium.com/thecapital/the-complete-polymarket-playbook-finding-real-edges-in-the-9b-prediction-market-revolution-a2c1d0a47d9d)

**Confidence:** HIGH

---

### Pitfall 6: Balance Constraints and Reserved Funds

**What goes wrong:**
Developers try to place multiple orders totaling more than their balance and get "insufficient balance" errors, not realizing that **open orders reserve funds** even if unfilled.

**Why it happens:**
Polymarket enforces per-market balance constraints:

```
maxOrderSize = underlyingAssetBalance - Σ(orderSize - orderFillAmount)
```

Open orders reserve funds proportional to their size, even if they haven't filled. You can't place new orders exceeding your unreserved balance.

**Example:**
```
Your balance: 500 USDC

Order 1: Buy 1000 YES @ $0.50 = 500 USDC reserved (fills order book)
Order 2: Buy 500 YES @ $0.40 = 200 USDC needed

Result: Order 2 REJECTED (only 0 USDC available after Order 1)
```

**Consequences:**
- Orders rejected despite appearing to have sufficient balance
- Confusion about why balance shows funds but orders fail
- Market making strategies broken by reserved funds

**Prevention:**
```python
def get_available_balance(client, market_id, current_balance):
    """Calculate available balance after open orders."""
    open_orders = client.get_orders(market_id, status="open")

    reserved = 0
    for order in open_orders:
        # Reserved = unfilled amount
        unfilled = float(order['size']) - float(order['filled'])
        reserved += unfilled * float(order['price'])

    available = current_balance - reserved
    return available

# WRONG: Checking only wallet balance
balance = get_balance()
if balance >= order_cost:
    place_order(size, price)  # May fail if funds reserved

# RIGHT: Account for reserved funds
available = get_available_balance(client, market_id, balance)
if available >= order_cost:
    place_order(size, price)
else:
    # Cancel old orders to free up balance, or reduce order size
    cancel_some_orders()
```

**Detection:**
- "Insufficient balance" errors when wallet shows funds
- Orders succeed individually but fail when multiple are open
- Check open orders and calculate reserved funds

**Mitigation:**
1. Track reserved funds from open orders
2. Cancel old orders to free balance for new ones
3. Size new orders within available (not total) balance
4. Use batch order cancellation for efficiency

**Sources:**
- [Orders Overview - Polymarket Documentation](https://docs.polymarket.com/developers/CLOB/orders/orders)

**Confidence:** HIGH

---

### Pitfall 7: WebSocket Data Stream Stops After ~20 Minutes

**What goes wrong:**
WebSocket connection remains open (ping/pong working) but stops receiving market data after approximately 20 minutes, causing stale data and missed updates.

**Why it happens:**
Known issue with Polymarket's real-time data client where the data stream silently stops sending messages despite healthy connection status.

**Consequences:**
- Trading bots use stale order book data
- Price alerts don't fire
- Market opportunities missed
- Silent failure (no error, connection appears healthy)

**Prevention:**
```python
import time
from datetime import datetime, timedelta

class PolymarketWebSocketMonitor:
    def __init__(self, reconnect_threshold_seconds=60):
        self.last_message_time = datetime.now()
        self.reconnect_threshold = timedelta(seconds=reconnect_threshold_seconds)

    def on_message(self, message):
        """Called when WebSocket receives a message."""
        self.last_message_time = datetime.now()
        # Process message...

    def check_health(self):
        """Check if we've received recent messages."""
        time_since_last = datetime.now() - self.last_message_time

        if time_since_last > self.reconnect_threshold:
            print(f"No messages for {time_since_last.seconds}s, reconnecting...")
            return False  # Unhealthy, trigger reconnect
        return True  # Healthy

    def run(self):
        """Main loop with health checks."""
        while True:
            if not self.check_health():
                self.reconnect()
            time.sleep(10)  # Check every 10 seconds

# WRONG: Assuming connection stays healthy
ws = connect_websocket()
ws.run_forever()  # Will stop receiving data after ~20 min

# RIGHT: Monitor message timestamps and reconnect
monitor = PolymarketWebSocketMonitor(reconnect_threshold_seconds=60)
ws = connect_websocket()
ws.on_message = monitor.on_message

# Run health checks in separate thread
import threading
health_thread = threading.Thread(target=monitor.run)
health_thread.start()

ws.run_forever()
```

**Detection:**
- WebSocket connected but no new messages
- Ping/pong heartbeat works but data stream silent
- Last message timestamp > 1-2 minutes old (for active markets)

**Mitigation:**
1. Track timestamp of last received message
2. Implement reconnection logic when no data for 60-120 seconds
3. Use separate thread for health monitoring
4. Log reconnection events for debugging
5. Consider primary/backup WebSocket connections

**Sources:**
- [WebSocket data stream stops after some time · Issue #26](https://github.com/Polymarket/real-time-data-client/issues/26)
- [WSS Overview - Polymarket Documentation](https://docs.polymarket.com/developers/CLOB/websocket/wss-overview)

**Confidence:** MEDIUM

---

### Pitfall 8: Dynamic Tick Size Changes

**What goes wrong:**
Orders are rejected or prices quoted incorrectly because tick size changed dynamically when price reached extreme ranges (>0.96 or <0.04), but the trading system didn't update.

**Why it happens:**
Polymarket automatically adjusts minimum tick size when prices approach boundaries:
- Price > 0.96: Tick size increases (coarser granularity)
- Price < 0.04: Tick size increases (coarser granularity)

Systems that cache tick size or don't listen for updates break when these thresholds are crossed.

**Consequences:**
- Orders rejected for invalid price increments
- Market making spreads violated tick size rules
- Quotes at illegal price points

**Prevention:**
```python
class TickSizeManager:
    def __init__(self, initial_tick_size=0.01):
        self.current_tick_size = initial_tick_size

    def on_tick_size_change(self, message):
        """Handle tick_size_change WebSocket message."""
        old_tick = message['old_tick_size']
        new_tick = message['new_tick_size']
        side = message['side']
        timestamp = message['timestamp']

        print(f"Tick size changed: {old_tick} -> {new_tick} (side: {side})")
        self.current_tick_size = new_tick

        # Cancel and replace existing orders at new tick size
        self.update_orders_for_new_tick_size()

    def round_price_to_tick(self, price: float) -> float:
        """Round price to nearest valid tick."""
        from decimal import Decimal, ROUND_HALF_UP

        price_decimal = Decimal(str(price))
        tick_decimal = Decimal(str(self.current_tick_size))

        # Round to nearest tick
        ticks = (price_decimal / tick_decimal).quantize(Decimal('1'), rounding=ROUND_HALF_UP)
        rounded_price = float(ticks * tick_decimal)

        return rounded_price

    def update_orders_for_new_tick_size(self):
        """Cancel and replace orders that violate new tick size."""
        # Implementation: Cancel old orders, place new ones at valid ticks
        pass

# WRONG: Using cached tick size
tick_size = 0.01  # Hardcoded or fetched once
price = 0.975
quote_price = round(price, 2)  # May be invalid if tick size changed

# RIGHT: Listen for tick_size_change and update
tick_manager = TickSizeManager(initial_tick_size=0.01)

# Subscribe to market WebSocket channel
ws.subscribe('market', token_id)
ws.on_message = lambda msg: (
    tick_manager.on_tick_size_change(msg) if msg['type'] == 'tick_size_change' else None
)

# Use current tick size
valid_price = tick_manager.round_price_to_tick(0.975)
place_order(size, valid_price)
```

**Detection:**
- Orders rejected with "invalid price increment" errors
- Happens when market price near 0.96 or 0.04 thresholds
- Listen for `tick_size_change` WebSocket messages

**Key Thresholds:**
- Price > 0.96: Tick size increases
- Price < 0.04: Tick size increases
- Normal range (0.04 - 0.96): Standard tick size (typically 0.01)

**Sources:**
- [Market Channel - Polymarket Documentation](https://docs.polymarket.com/developers/CLOB/websocket/market-channel)
- [[Polymarket] Error when creating QuoteTick after a tick size change · Issue #2980](https://github.com/nautechsystems/nautilus_trader/issues/2980)

**Confidence:** MEDIUM-HIGH

---

### Pitfall 9: Multi-Outcome Market Confusion

**What goes wrong:**
Developers use standard CTF (Conditional Token Framework) contracts for multi-outcome markets (>2 outcomes) and get transaction failures or wrong contract addresses.

**Why it happens:**
Binary markets use standard CTF contracts, but **multi-outcome markets use the Negative Risk CTF Adapter** with different contract addresses and split/merge logic.

**Consequences:**
- Transactions sent to wrong contract addresses
- Token approvals fail
- Split/merge operations fail
- Integration breaks for multi-outcome markets

**Prevention:**
```python
# Binary market (2 outcomes: YES/NO)
BINARY_CTF_EXCHANGE = "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E"  # Standard

# Multi-outcome market (>2 outcomes)
NEG_RISK_CTF_ADAPTER = "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296"  # Different!
NEG_RISK_EXCHANGE = "0xC5d563A36AE78145C45a50134d48A1215220f80a"

def get_ctf_contract(num_outcomes: int):
    """Get correct CTF contract based on market type."""
    if num_outcomes == 2:
        return BINARY_CTF_EXCHANGE
    elif num_outcomes > 2:
        return NEG_RISK_CTF_ADAPTER
    else:
        raise ValueError("Invalid number of outcomes")

# WRONG: Using binary CTF for all markets
ctf_address = BINARY_CTF_EXCHANGE
approve_tokens(ctf_address, amount)  # Fails for multi-outcome

# RIGHT: Check market type and use appropriate contract
market_info = get_market(condition_id)
num_outcomes = len(market_info['outcomes'])

ctf_address = get_ctf_contract(num_outcomes)
approve_tokens(ctf_address, amount)  # Works for both types
```

**Detection:**
- Transaction failures for multi-outcome markets
- Token approvals not working
- Check market metadata for `outcomes` array length
- Binary markets: 2 outcomes (YES/NO)
- Multi-outcome: >2 outcomes (e.g., multiple candidates)

**Sources:**
- [Inventory Management - Polymarket Documentation](https://docs.polymarket.com/developers/market-makers/inventory)
- [GitHub - Polymarket/neg-risk-ctf-adapter](https://github.com/Polymarket/neg-risk-ctf-adapter)

**Confidence:** MEDIUM

---

## Minor Pitfalls

Mistakes that cause annoyance but are fixable.

### Pitfall 10: Rate Limiting Misunderstandings

**What goes wrong:**
Developers implement aggressive backoff/retry logic thinking Polymarket drops requests over rate limits, but Polymarket actually **queues/delays requests** via Cloudflare throttling.

**Why it happens:**
Misunderstanding Cloudflare's throttling vs traditional rate limiting. Polymarket doesn't immediately reject with 429—it delays/queues requests.

**Rate Limit Structure:**

| Endpoint Type | Limit | Window | Notes |
|---------------|-------|--------|-------|
| **CLOB - Market Data** | 500-1,500 req | 10s | Order book, ticker |
| **CLOB - POST /order** | 3,500 burst, 60 sustained | 10s burst, 10min sustained | Key difference! |
| **CLOB - Ledger** | 125-900 req | 10s | Historical data |
| **Data API** | 150-1,000 req | 10s | Trades, positions |
| **Gamma API** | 200-4,000 req | 10s | Market search, events |

**Consequences:**
- Unnecessary exponential backoff logic
- Confused about why requests succeed but arrive late
- Trading latency increases during busy periods

**Prevention:**
```python
import time
from collections import deque

class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests = deque()

    def wait_if_needed(self):
        """Wait if we're at rate limit."""
        now = time.time()

        # Remove requests outside the window
        while self.requests and self.requests[0] < now - self.window:
            self.requests.popleft()

        # If at limit, wait
        if len(self.requests) >= self.max_requests:
            sleep_time = self.requests[0] + self.window - now
            if sleep_time > 0:
                time.sleep(sleep_time)

        self.requests.append(now)

# Different limiters for different endpoints
order_limiter = RateLimiter(max_requests=60, window_seconds=10)  # Sustained
market_data_limiter = RateLimiter(max_requests=500, window_seconds=10)

def place_order(order_params):
    order_limiter.wait_if_needed()
    # Cloudflare may still queue/delay, but we avoid hitting limits
    return api.post_order(order_params)

def get_order_book(token_id):
    market_data_limiter.wait_if_needed()
    return api.get_order_book(token_id)
```

**Key Insight:**
POST /order has **dual limits**:
- Burst: 3,500 requests / 10 seconds
- Sustained: 60 requests / second over 10 minutes

Design for sustained rate, not burst capacity.

**Detection:**
- Requests succeed but with increased latency
- No 429 errors, but responses slow during high traffic
- Cloudflare throttling is transparent (no error code)

**Sources:**
- [API Rate Limits - Polymarket Documentation](https://docs.polymarket.com/quickstart/introduction/rate-limits)
- [API Rate Limit - Burst vs Throttle for /order endpoint · Issue #147](https://github.com/Polymarket/py-clob-client/issues/147)

**Confidence:** HIGH

---

### Pitfall 11: Partial Fill Tracking Complexity

**What goes wrong:**
Developers lose track of order state when orders partially fill across multiple transactions due to gas limits, seeing duplicate or fragmented trade records.

**Why it happens:**
Large orders may be broken into multiple transactions due to gas limits. Each creates a separate trade entity, but they're related through `bucket_index`, `market_order`, and `match_time` fields.

**Consequences:**
- Double-counting filled amounts
- Confusion about order completion status
- Incorrect position tracking

**Prevention:**
```python
def reconcile_partial_fills(trades: list) -> dict:
    """Group related partial fills into complete fill records."""
    fills_by_order = {}

    for trade in trades:
        # Key: (market_order, match_time)
        key = (trade['market_order'], trade['match_time'])

        if key not in fills_by_order:
            fills_by_order[key] = []

        fills_by_order[key].append({
            'bucket_index': trade['bucket_index'],
            'amount': trade['amount'],
            'price': trade['price']
        })

    # Sort each group by bucket_index to get fill sequence
    complete_fills = {}
    for key, fill_parts in fills_by_order.items():
        sorted_parts = sorted(fill_parts, key=lambda x: x['bucket_index'])

        total_amount = sum(p['amount'] for p in sorted_parts)
        avg_price = sum(p['amount'] * p['price'] for p in sorted_parts) / total_amount

        complete_fills[key] = {
            'total_amount': total_amount,
            'average_price': avg_price,
            'num_parts': len(sorted_parts)
        }

    return complete_fills

# WRONG: Treating each trade as independent
total_filled = sum(t['amount'] for t in trades)  # Double-counts partial fills

# RIGHT: Reconcile partial fills
fills = reconcile_partial_fills(trades)
total_filled = sum(f['total_amount'] for f in fills.values())
```

**Detection:**
- Multiple trade records with same `market_order` and `match_time`
- Incrementing `bucket_index` values
- OrderFilled events with partial amounts

**Fields to Track:**
- `bucket_index`: Sequence number for multi-transaction fills
- `market_order`: Order ID that spawned these fills
- `match_time`: Timestamp when matching occurred

**Sources:**
- [Polymarket Volume Is Being Double-Counted](https://www.paradigm.xyz/2025/12/polymarket-volume-is-being-double-counted)
- [How Polymarket Works | The Tech Behind Prediction Markets](https://rocknblock.io/blog/how-polymarket-works-the-tech-behind-prediction-markets)

**Confidence:** MEDIUM

---

### Pitfall 12: Market Resolution Status Ambiguity

**What goes wrong:**
Developers don't handle all market resolution states, assuming markets are simply "open" or "resolved," missing important states like "closed but unresolved," "pending," or "disputed."

**Why it happens:**
Market lifecycle is more complex than binary open/closed:
1. Market Created
2. Trading Begins
3. Market Closes (event occurred)
4. Oracle Proposes Outcome
5. [Optional] Dispute Period
6. [Optional] Escalation to UMA DVM
7. Market Resolves (final outcome)
8. Users Redeem Winnings

Markets can be **closed** (no new trades) but **unresolved** (outcome pending) for days or weeks.

**Consequences:**
- Trying to trade on closed markets
- Missing redemption opportunities
- Confusion about when positions convert to USDC
- Edge case: Disputed resolutions can flip outcomes

**Prevention:**
```python
from enum import Enum

class MarketStatus(Enum):
    ACTIVE = "active"              # Trading open
    CLOSED = "closed"              # Event occurred, no trading
    PENDING_RESOLUTION = "pending" # Closed, outcome proposed
    DISPUTED = "disputed"          # Outcome challenged
    RESOLVED = "resolved"          # Final outcome set
    ARCHIVED = "archived"          # Historical market

def can_trade(market):
    """Check if market accepts trades."""
    return market['status'] == MarketStatus.ACTIVE

def can_redeem(market):
    """Check if winning positions can be redeemed."""
    return market['status'] == MarketStatus.RESOLVED

def check_market_state(market):
    """Handle all market states appropriately."""
    status = market['status']

    if status == MarketStatus.ACTIVE:
        # Normal trading
        return "can_trade"

    elif status == MarketStatus.CLOSED:
        # Event occurred, waiting for resolution
        # Cancel open orders, wait for outcome
        return "wait_for_resolution"

    elif status == MarketStatus.PENDING_RESOLUTION:
        # Outcome proposed but in dispute period
        # Monitor for disputes
        return "monitor_disputes"

    elif status == MarketStatus.DISPUTED:
        # Outcome challenged, escalated to UMA DVM
        # WARNING: Final outcome may differ from proposed!
        return "wait_for_dvm"

    elif status == MarketStatus.RESOLVED:
        # Final outcome confirmed, redeem winnings
        return "redeem_positions"

    elif status == MarketStatus.ARCHIVED:
        # Historical market, read-only
        return "historical_only"

# WRONG: Binary check
if market['closed']:
    redeem_position()  # May fail if not yet resolved

# RIGHT: Check resolution status
if market['status'] == MarketStatus.RESOLVED:
    redeem_position()
```

**Edge Case: Disputed Resolutions**

Recent example (Dec 2025): "Will Polymarket U.S. go live in 2025?" resolved as "yes" despite being invite-only, using UMA's cryptocurrency voting system.

**Implications:**
- Proposed outcomes can be disputed
- UMA DVM vote can override initial proposal
- "Whale" token holders can influence outcomes
- Resolution timing unpredictable during disputes

**Detection:**
- Check `closed` field (market no longer trading)
- Check `resolved` field (final outcome confirmed)
- Query oracle resolution events on-chain
- Monitor for dispute/escalation events

**Sources:**
- [How Are Prediction Markets Resolved? - Polymarket Documentation](https://docs.polymarket.com/polymarket-learn/markets/how-are-markets-resolved)
- [Inside UMA Oracle | How Prediction Markets Resolution Works](https://rocknblock.io/blog/how-prediction-markets-resolution-works-uma-optimistic-oracle-polymarket)
- [Polymarket Fumbles U.S. Launch (controversial resolution)](https://www.sportico.com/business/sports-betting/2026/polymarket-united-states-launch-invite-waitlist-delay-1234879944/)

**Confidence:** MEDIUM

---

### Pitfall 13: Time-in-Force Misuse

**What goes wrong:**
Developers use FOK (Fill-or-Kill) when they meant GTC (Good-til-Cancelled), or vice versa, causing unexpected order behavior.

**Why it happens:**
Confusion about when to use each time-in-force type:
- **FOK:** Immediate full fill or complete cancellation (taker/market behavior)
- **GTC:** Sits on book until filled or manually cancelled (maker/limit behavior)
- **GTD:** Sits on book with expiration timestamp (time-limited maker)

**Consequences:**
- FOK orders cancel when market has insufficient depth
- GTC orders sit on book indefinitely, tying up capital
- GTD orders expire unexpectedly due to 1-minute buffer requirement

**Prevention:**
```python
from datetime import datetime, timedelta

# Use Case 1: Market Order (want immediate execution)
# Use FOK - fills completely or cancels
place_order(
    size=100,
    price=best_ask,  # Marketable price
    time_in_force="FOK"  # Execute now or cancel
)
# If order cancels: insufficient liquidity for full size

# Use Case 2: Limit Order (providing liquidity)
# Use GTC - sits on book
place_order(
    size=100,
    price=0.45,  # Below best ask (limit price)
    time_in_force="GTC"  # Wait for fill
)
# Order stays open until filled or manually cancelled

# Use Case 3: Time-Limited Limit Order
# Use GTD - expires automatically
expiration = datetime.now() + timedelta(hours=2) + timedelta(minutes=1)  # +1 min buffer!
place_order(
    size=100,
    price=0.45,
    time_in_force="GTD",
    expiration=int(expiration.timestamp())  # UTC seconds
)
# Order expires in 2 hours (auto-cancelled)

# WRONG: Using FOK for limit order
place_order(size=100, price=0.45, time_in_force="FOK")
# Immediately cancels because price isn't marketable

# WRONG: Using GTC for market order
place_order(size=100, price=best_ask, time_in_force="GTC")
# Works, but if best_ask moves, order sits on book unfilled

# WRONG: GTD without 1-minute buffer
expiration = datetime.now() + timedelta(seconds=30)  # Want 30s expiration
place_order(size=100, price=0.45, time_in_force="GTD", expiration=int(expiration.timestamp()))
# Order expires in 1min 30s due to minimum buffer, not 30s!
```

**Time-in-Force Decision Tree:**

```
Want immediate execution?
├─ YES → FOK (market order)
└─ NO → Want order to expire automatically?
    ├─ YES → GTD (time-limited limit)
    └─ NO → GTC (standard limit)
```

**GTD 1-Minute Buffer:**
If you want an order to expire in 30 seconds, set expiration to `now + 1 minute + 30 seconds`. The API enforces a minimum 1-minute buffer.

**Sources:**
- [Trading - Polymarket Documentation](https://docs.polymarket.com/developers/market-makers/trading)
- [Polymarket | NautilusTrader Documentation](https://nautilustrader.io/docs/latest/integrations/polymarket/)

**Confidence:** HIGH

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| **Initial Setup** | USDC.e vs Native USDC confusion | Always verify USDC.e token address, use "Activate Funds" |
| **Authentication** | Proxy wallet vs funder address mismatch | Check polymarket.com/settings for proxy address |
| **Order Placement** | Decimal precision errors (FOK vs GTC) | Round FOK orders to 2/4 decimals, validate before submission |
| **Market Data** | Treating displayed price as executable | Always use order book bid/ask, not midpoint |
| **WebSocket Integration** | Data stream stops after 20 minutes | Implement message timestamp monitoring and auto-reconnect |
| **Balance Management** | Not accounting for reserved funds | Track open orders and calculate available balance |
| **Multi-Outcome Markets** | Using wrong CTF contract | Check num_outcomes and use Neg Risk Adapter for >2 outcomes |
| **Resolution Handling** | Assuming closed = resolved | Check resolution status separately from closed status |
| **Rate Limiting** | Hitting sustained limits on /order endpoint | Design for 60 req/s sustained, not 3,500 burst |
| **Partial Fills** | Double-counting fragmented trades | Reconcile by bucket_index, market_order, match_time |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| **USDC.e Confusion** | HIGH | Confirmed by official docs + multiple 2026 user guides |
| **Proxy Wallet Auth** | HIGH | Official docs + GitHub issues confirm pattern |
| **Decimal Precision** | HIGH | Specific GitHub issue (#121) documents FOK constraints |
| **Price Interpretation** | HIGH | Official docs explicitly explain midpoint calculation |
| **WebSocket Stream Stop** | MEDIUM | GitHub issue #26 confirms, but limited discussion |
| **Balance Constraints** | HIGH | Formula documented in official orders page |
| **Tick Size Changes** | MEDIUM-HIGH | Official docs + NautilusTrader issue confirm |
| **Multi-Outcome CTF** | MEDIUM | Official docs mention Neg Risk adapter but limited detail |
| **Rate Limiting** | HIGH | Official rate limits page + GitHub issue discussion |
| **Partial Fills** | MEDIUM | Paradigm article + docs mention, but limited examples |
| **Resolution Status** | MEDIUM | Official docs + real examples, but edge cases under-documented |
| **Time-in-Force** | HIGH | Official docs clearly define FOK/GTC/GTD behavior |

---

## Research Gaps

Areas where documentation was limited or inconclusive:

1. **Specific minimum order amounts:** Project context mentioned "5 shares, $1 minimum" but official docs don't specify hard minimums—only balance constraints. Needs verification with actual API testing.

2. **Complete list of order rejection reasons:** Found examples (decimal precision, insufficient balance) but no comprehensive error code reference.

3. **Resolution status field names:** Docs explain resolution process but don't clearly map to specific API field values (e.g., is it `status: "closed"` or `closed: true`?). Needs API schema inspection.

4. **Exact tick size values at price thresholds:** Docs confirm tick size changes at 0.96/0.04 but don't specify what the new tick sizes are. Needs empirical observation.

5. **WebSocket reconnection best practices:** Known issue but no official guidance on reconnection strategy. Community implementations vary.

6. **Signature type edge cases:** Docs explain EOA, POLY_PROXY, GNOSIS_SAFE but don't detail what happens if wrong type chosen.

---

## Sources

### Official Documentation
- [Polymarket Documentation](https://docs.polymarket.com/)
- [Orders Overview](https://docs.polymarket.com/developers/CLOB/orders/orders)
- [Authentication](https://docs.polymarket.com/developers/CLOB/authentication)
- [API Rate Limits](https://docs.polymarket.com/quickstart/introduction/rate-limits)
- [How Are Prices Calculated?](https://docs.polymarket.com/polymarket-learn/trading/how-are-prices-calculated)
- [How Are Markets Resolved?](https://docs.polymarket.com/polymarket-learn/markets/how-are-markets-resolved)
- [Market Channel WebSocket](https://docs.polymarket.com/developers/CLOB/websocket/market-channel)
- [Proxy Wallet](https://docs.polymarket.com/developers/proxy-wallet)
- [Inventory Management](https://docs.polymarket.com/developers/market-makers/inventory)
- [Trading (Market Makers)](https://docs.polymarket.com/developers/market-makers/trading)

### GitHub Issues
- [FOK order decimal places error · Issue #121](https://github.com/Polymarket/py-clob-client/issues/121)
- [401 Unauthorized / Invalid api key · Issue #187](https://github.com/Polymarket/py-clob-client/issues/187)
- [Periodically Unauthorized/Invalid api key · Issue #190](https://github.com/Polymarket/py-clob-client/issues/190)
- [API Rate Limit - Burst vs Throttle · Issue #147](https://github.com/Polymarket/py-clob-client/issues/147)
- [WebSocket data stream stops · Issue #26](https://github.com/Polymarket/real-time-data-client/issues/26)
- [[Polymarket] Error when creating QuoteTick after tick size change · Issue #2980](https://github.com/nautechsystems/nautilus_trader/issues/2980)

### Community Resources
- [The Polymarket API: Architecture, Endpoints, and Use Cases (2026)](https://medium.com/@gwrx2005/the-polymarket-api-architecture-endpoints-and-use-cases-f1d88fa6c1bf)
- [Market Making on Prediction Markets: Complete 2026 Guide](https://newyorkcityservers.com/blog/prediction-market-making-guide)
- [USDC vs. USDC.e on Polymarket (2026 Guide)](https://www.homesfound.ca/blog/usdc-vs-usdce-polymarket-why-your-balance-shows-000-2026-guide/)
- [How to Bridge Funds to Polymarket (2026)](https://www.homesfound.ca/blog/how-bridge-funds-polymarket-complete-2026-guide/)
- [How to Find Your Polymarket Wallet Address (2026)](https://www.homesfound.ca/blog/how-find-your-polymarket-wallet-address-2026-visual-guide/)
- [Polymarket Volume Is Being Double-Counted](https://www.paradigm.xyz/2025/12/polymarket-volume-is-being-double-counted)
- [Inside UMA Oracle | How Resolution Works](https://rocknblock.io/blog/how-prediction-markets-resolution-works-uma-optimistic-oracle-polymarket)
- [How Polymarket Works | The Tech Behind Prediction Markets](https://rocknblock.io/blog/how-polymarket-works-the-tech-behind-prediction-markets)
- [The Complete Polymarket Playbook (2026)](https://medium.com/thecapital/the-complete-polymarket-playbook-finding-real-edges-in-the-9b-prediction-market-revolution-a2c1d0a47d9d)
- [Polymarket | NautilusTrader Documentation](https://nautilustrader.io/docs/latest/integrations/polymarket/)
- [How Latency Impacts Polymarket Bot Performance](https://www.quantvps.com/blog/how-latency-impacts-polymarket-trading-performance)

### News/Analysis
- [Polymarket, Kalshi contract limits (Jan 2026)](https://www.coindesk.com/policy/2026/01/30/polymarket-kalshi-contract-limits-demonstrated-in-latest-u-s-government-shutdown-fight)
- [Polymarket Fumbles U.S. Launch](https://www.sportico.com/business/sports-betting/2026/polymarket-united-states-launch-invite-waitlist-delay-1234879944/)
- [Native USDC Is on Polygon PoS](https://polygon.technology/blog/phase-one-of-native-usdc-migration-on-polygon-pos-is-underway)

---

**Research completed:** 2026-01-31
