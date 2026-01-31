# Phase 3: Edge Cases & Best Practices - Research

**Researched:** 2026-01-31
**Domain:** Polymarket edge cases, py-clob-client library patterns, production best practices
**Confidence:** HIGH

## Summary

Phase 3 fills critical gaps not covered in the API-focused Phase 2 documentation. This research covers: USDC.e vs Native USDC confusion (EDGE-01), order constraints and precision requirements (EDGE-02, EDGE-03, EDGE-08), price interpretation nuances (EDGE-04), market resolution mechanics (EDGE-05, EDGE-06), NegRisk multi-outcome patterns (EDGE-07), partial fill tracking (EDGE-09), py-clob-client error handling (LIB-02, LIB-03), and production usage best practices (LIB-04).

The existing Phase 2 documentation (20 skill files) provides solid API coverage but intentionally defers edge cases and library internals. Phase 3 documentation should build on Phase 2 by referencing existing files where appropriate and filling gaps with dedicated edge case content.

The py-clob-client library (v0.34.5+) has two primary exception classes: `PolyException` (base) and `PolyApiException` (API errors with status codes). Common error patterns include 401 Unauthorized (invalid API key), 400 Bad Request (insufficient balance/allowance, invalid amounts), and order validation errors. Rate limits are well-documented with burst and sustained tiers.

**Primary recommendation:** Create focused edge case documentation that serves as a troubleshooting reference, with clear symptoms-to-solutions mappings. Link heavily to existing Phase 2 documentation rather than duplicating content.

## Standard Stack

Phase 3 uses the same stack as Phase 2 - no additional libraries needed.

### Core (from Phase 2)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| py-clob-client | 0.34.5+ | Python CLOB client | Official Polymarket library |
| requests | 2.31.0+ | HTTP client | REST API calls |
| websockets | 12.0+ | WebSocket client | Real-time data |
| web3 | 6.0+ | Blockchain interaction | On-chain balance checks |

### Phase 3 Specific Patterns
| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| Try/except with PolyApiException | Error handling | All py-clob-client calls |
| Decimal for precision | Avoid floating point errors | Price/size calculations |
| Rate limiter wrapper | Respect API limits | High-volume trading |

## Architecture Patterns

### Recommended Documentation Structure

Phase 3 documentation should integrate with existing Phase 2 structure:

```
skills/polymarket/
|-- edge-cases/                    # NEW - Phase 3
|   |-- README.md                  # Index of edge cases
|   |-- usdc-token-confusion.md    # EDGE-01
|   |-- order-constraints.md       # EDGE-02, EDGE-03, EDGE-08
|   |-- price-interpretation.md    # EDGE-04
|   |-- resolution-mechanics.md    # EDGE-05, EDGE-06
|   |-- negrisk-trading.md         # EDGE-07
|   `-- partial-fills.md           # EDGE-09
|
|-- library/                       # NEW - Phase 3
|   |-- README.md                  # Library overview
|   |-- error-handling.md          # LIB-03
|   `-- production-patterns.md     # LIB-04
|
|-- trading/                       # EXISTING - Phase 2
|   `-- (update with cross-refs)
```

### Pattern 1: Exception Handling

**What:** Structured error handling for py-clob-client operations

**When to use:** All trading operations, API calls

**Example:**
```python
# Source: py-clob-client/exceptions.py
from py_clob_client.exceptions import PolyApiException

def place_order_safely(client, order_args, order_type):
    """Place order with comprehensive error handling."""
    try:
        signed = client.create_order(order_args)
        response = client.post_order(signed, order_type)
        return {"success": True, "response": response}

    except PolyApiException as e:
        # Extract structured error info
        error_info = {
            "success": False,
            "status_code": e.status_code,
            "error_msg": e.error_msg
        }

        # Categorize error
        if e.status_code == 401:
            error_info["category"] = "authentication"
            error_info["action"] = "Refresh API credentials"
        elif e.status_code == 400:
            if "balance" in str(e.error_msg).lower():
                error_info["category"] = "insufficient_funds"
                error_info["action"] = "Check USDC.e balance"
            elif "amounts" in str(e.error_msg).lower():
                error_info["category"] = "precision"
                error_info["action"] = "Adjust size/price precision"
            else:
                error_info["category"] = "validation"
                error_info["action"] = "Check order parameters"
        else:
            error_info["category"] = "unknown"
            error_info["action"] = "Check full error message"

        return error_info
```

### Pattern 2: Rate Limiting Wrapper

**What:** Decorator/wrapper to respect API rate limits

**When to use:** High-frequency trading, batch operations

**Example:**
```python
# Source: Polymarket rate limits documentation
import time
from threading import Lock
from collections import deque

class RateLimiter:
    """Rate limiter respecting Polymarket API limits.

    Key limits (per 10 seconds):
    - POST /order: 3,500 burst, 36,000/10min sustained
    - DELETE /order: 3,000 burst, 30,000/10min sustained
    - Batch operations: 1,000 burst, 15,000/10min sustained
    """

    def __init__(self, max_calls: int, window_seconds: float):
        self.max_calls = max_calls
        self.window_seconds = window_seconds
        self.calls = deque()
        self.lock = Lock()

    def wait_if_needed(self):
        """Block until call is allowed."""
        with self.lock:
            now = time.time()

            # Remove old calls outside window
            while self.calls and self.calls[0] < now - self.window_seconds:
                self.calls.popleft()

            # Wait if at limit
            if len(self.calls) >= self.max_calls:
                sleep_time = self.calls[0] + self.window_seconds - now
                if sleep_time > 0:
                    time.sleep(sleep_time)
                self.calls.popleft()

            self.calls.append(time.time())

# Recommended limiters
order_limiter = RateLimiter(max_calls=350, window_seconds=1)  # 350/s for orders
cancel_limiter = RateLimiter(max_calls=300, window_seconds=1)  # 300/s for cancels
```

### Pattern 3: Precision-Safe Price/Size Handling

**What:** Use Decimal for all price/size calculations

**When to use:** Any order construction, balance calculations

**Example:**
```python
# Source: FOK precision requirements from Polymarket docs
from decimal import Decimal, ROUND_DOWN

def prepare_order_amounts(price: float, size: float, order_type: str) -> dict:
    """Prepare price and size with correct precision.

    FOK Requirements:
    - Maker amount: max 2 decimal places
    - Taker amount: max 4 decimal places
    - Size x Price: max 2 decimal places

    GTC/GTD: More lenient, but still must align to tick size.
    """
    price_d = Decimal(str(price))
    size_d = Decimal(str(size))

    if order_type == "FOK":
        # FOK strict precision
        size_d = size_d.quantize(Decimal("0.01"), rounding=ROUND_DOWN)

        # Check product precision
        product = size_d * price_d
        product_rounded = product.quantize(Decimal("0.01"), rounding=ROUND_DOWN)

        if product != product_rounded:
            # Adjust size to make product clean
            size_d = (product_rounded / price_d).quantize(
                Decimal("0.01"), rounding=ROUND_DOWN
            )

    return {
        "price": float(price_d),
        "size": float(size_d),
        "notional": float(size_d * price_d)
    }
```

### Anti-Patterns to Avoid

- **Caching tick sizes indefinitely:** Tick sizes change dynamically when price > 0.96 or < 0.04
- **Using float for price arithmetic:** Floating point errors accumulate, use Decimal
- **Ignoring PolyApiException details:** status_code and error_msg contain actionable info
- **Rate limiting with simple sleep:** Use proper windowed rate limiting
- **Assuming Native USDC works:** Must use USDC.e exclusively
- **Trading unnamed NegRisk outcomes:** Only trade named outcomes in augmented negRisk

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Token detection | Manual address checking | `check_usdc_balances()` pattern | Edge cases with checksums |
| Tick size rounding | Simple round() | Tick size hierarchy lookup | Dynamic changes at extremes |
| Error categorization | String matching | PolyApiException attributes | Structured status_code/error_msg |
| Rate limiting | sleep() between calls | Windowed rate limiter class | Need to respect burst + sustained |
| Precision handling | float math | Decimal with explicit rounding | FOK 2-decimal requirement |

**Key insight:** Many edge cases stem from subtle precision, timing, or token contract differences. The solutions exist in library patterns and established conventions - document and reference them rather than improvising.

## Common Pitfalls

### Pitfall 1: USDC.e vs Native USDC Confusion (EDGE-01)

**What goes wrong:** User deposits USDC from exchange, Polymarket shows $0.00 balance, funds appear "lost"

**Why it happens:**
- USDC.e (bridged): `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` - correct for Polymarket
- Native USDC: `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` - NOT supported
- Most exchanges default to native USDC for Polygon withdrawals
- Both show as "USDC" in wallet interfaces

**How to avoid:**
- Always verify contract address before depositing
- Use `check_usdc_balances()` diagnostic function
- Provide swap instructions for native USDC holders

**Warning signs:**
- Polygon wallet shows USDC balance
- Polymarket balance shows $0.00
- User recently withdrew from exchange to Polygon

**Detection code already exists:** See `skills/polymarket/auth/token-allowances.md` - expand with troubleshooting

### Pitfall 2: FOK Precision Rejection (EDGE-03)

**What goes wrong:** FOK order rejected with "invalid amounts" while same order works as GTC

**Why it happens:** FOK orders have stricter precision requirements:
- Maker amount: max 2 decimal places
- Taker amount: max 4 decimal places
- Size x Price product: max 2 decimal places

**How to avoid:**
- Round size to 2 decimals before FOK orders
- Verify size x price has clean 2-decimal result
- Fall back to GTC when precision can't be satisfied

**Warning signs:**
- Error: "invalid amounts, the sell orders maker amount supports a max accuracy of 2 decimals"
- Same parameters work as GTC but fail as FOK

**Existing coverage:** `skills/polymarket/trading/order-types.md` covers FOK precision - Phase 3 expands with failure recovery

### Pitfall 3: Dynamic Tick Size Changes (EDGE-08)

**What goes wrong:** Orders suddenly rejected with "INVALID_ORDER_MIN_TICK_SIZE" that previously worked

**Why it happens:** Tick sizes change dynamically based on market price:
- Standard: 0.01 tick size (1-2 decimal prices)
- When price > 0.96 or < 0.04: tick size decreases (0.001, 0.0001)
- py-clob-client caches tick sizes locally

**How to avoid:**
- Fetch tick size before every order: `client.get_tick_size(token_id)`
- Handle `tick_size_change` WebSocket events
- Don't cache tick sizes for more than a few minutes

**Warning signs:**
- Market price near 0.96+ or 0.04-
- `tick_size_change` event in WebSocket stream
- Orders that worked yesterday fail today

**Configuration (from py-clob-client):**
```python
ROUNDING_CONFIG = {
    "0.1": {"price": 1, "size": 2, "amount": 3},
    "0.01": {"price": 2, "size": 2, "amount": 4},
    "0.001": {"price": 3, "size": 2, "amount": 5},
    "0.0001": {"price": 4, "size": 2, "amount": 6}
}
```

### Pitfall 4: NegRisk Unnamed Outcome Trading (EDGE-07)

**What goes wrong:** Trading on "Other" or unnamed outcomes, confusion about which outcomes to trade

**Why it happens:**
- Augmented negRisk events have unnamed placeholder outcomes
- Unnamed outcomes can change definitions
- Trading should only happen on named outcomes

**How to avoid:**
- Check both `negRisk` and `negRiskAugmented` flags
- Only trade outcomes with explicit names
- Ignore outcomes until they are named or resolution occurs

**Warning signs:**
- Event has `negRiskAugmented: true`
- Outcome name is empty or "Other"
- Polymarket UI doesn't show the outcome

**Detection:**
```python
def is_tradable_negrisk_outcome(event, market):
    """Check if a negRisk outcome should be traded."""
    if not event.get("negRisk"):
        return True  # Not negRisk, normal rules apply

    if event.get("negRiskAugmented"):
        # Only trade named outcomes
        question = market.get("question", "")
        return bool(question) and question.lower() != "other"

    return True
```

### Pitfall 5: API Key Invalidation Without Warning (LIB-03)

**What goes wrong:** Orders fail with 401 Unauthorized, `create_or_derive_api_creds()` works but `post_order()` fails

**Why it happens:**
- API keys can expire or be rotated
- Creating new credentials may invalidate old ones
- Nonce tracking issues can cause auth failures

**How to avoid:**
- Use `create_or_derive_api_creds()` to retrieve existing (safer than creating new)
- Handle 401 errors by refreshing credentials
- Log nonce values for credential recovery

**Warning signs:**
- Read operations work, write operations fail
- Error: "Unauthorized/Invalid api key"
- Works on first call, fails on subsequent

### Pitfall 6: Disputed Outcome Resolution Delays (EDGE-06)

**What goes wrong:** Market takes days to resolve, position redemption blocked

**Why it happens:**
- UMA Optimistic Oracle has 2-hour challenge period
- Disputed outcomes escalate to DVM (48-hour vote)
- First dispute triggers re-proposal, second goes to vote

**How to avoid:**
- Factor resolution delays into trading strategy
- Monitor UMA proposal status for held positions
- Understand that resolution is NOT instant after event conclusion

**Resolution timeline:**
1. Market closes: Trading stops
2. Proposal submitted: 2-hour challenge period
3. If challenged once: New proposal (restart 2-hour window)
4. If challenged twice: DVM vote (48 hours)
5. Resolution finalized: Redemption enabled

### Pitfall 7: Price vs Odds Confusion (EDGE-04)

**What goes wrong:** Misinterpreting midpoint prices as executable, confusion about what prices mean

**Why it happens:**
- Midpoint is calculated average, not a tradable price
- Bid/ask spread can be significant in low liquidity
- `get_price()` returns executable price, `get_midpoint()` returns theoretical

**How to avoid:**
- Use `get_order_book()` for actual liquidity
- `get_price(token_id, side="BUY")` for immediate buy price
- Midpoint is for display/analytics only

**Interpretation:**
```python
# Midpoint: (best_bid + best_ask) / 2 - display only
midpoint = client.get_midpoint(token_id)  # e.g., 0.50

# Executable prices
buy_price = client.get_price(token_id, side="BUY")   # e.g., 0.52 (ask)
sell_price = client.get_price(token_id, side="SELL") # e.g., 0.48 (bid)

# Spread = actual trading cost
spread = buy_price - sell_price  # 0.04 = 4 cents per share roundtrip
```

## Code Examples

### py-clob-client Exception Classes

```python
# Source: py-clob-client/exceptions.py
from py_clob_client.exceptions import PolyException, PolyApiException

# PolyException - Base exception
class PolyException(Exception):
    def __init__(self, msg):
        self.msg = msg

# PolyApiException - API-specific with status codes
class PolyApiException(PolyException):
    def __init__(self, resp=None, error_msg=None):
        # Either resp or error_msg must be provided
        self.status_code = resp.status_code if resp else None
        self.error_msg = self._get_message(resp) if resp else error_msg

    # Common status codes:
    # 400 - Bad Request (validation, balance, precision)
    # 401 - Unauthorized (invalid/expired API key)
    # 404 - Not Found (invalid endpoint or resource)
    # 429 - Rate Limited (requests queued, not dropped)
    # 5xx - Server Error (retry with backoff)
```

### Common Error Messages and Solutions

```python
# Source: GitHub issues and documentation
ERROR_SOLUTIONS = {
    # Authentication errors
    "Invalid api key": {
        "category": "auth",
        "solution": "Call client.set_api_creds(client.create_or_derive_api_creds())"
    },
    "Invalid L1 Request headers": {
        "category": "auth",
        "solution": "Check signature_type matches wallet type (0=EOA, 1=Magic, 2=Browser proxy)"
    },

    # Balance errors
    "not enough balance / allowance": {
        "category": "funds",
        "solution": "Check USDC.e balance and allowances are set"
    },
    "insufficient balance": {
        "category": "funds",
        "solution": "Verify funder address has USDC.e, not EOA if using proxy"
    },

    # Precision errors
    "invalid amounts": {
        "category": "precision",
        "solution": "Round FOK size to 2 decimals, ensure size*price has 2 decimals"
    },
    "INVALID_ORDER_MIN_TICK_SIZE": {
        "category": "precision",
        "solution": "Fetch current tick size with client.get_tick_size(token_id)"
    },

    # Order errors
    "order crosses book": {
        "category": "order",
        "solution": "Post-only orders cannot be marketable; adjust price or use GTC"
    },
    "FOK_ORDER_NOT_FILLED_ERROR": {
        "category": "order",
        "solution": "Insufficient liquidity for full fill; use FAK or reduce size"
    },

    # Market errors
    "MARKET_NOT_READY": {
        "category": "market",
        "solution": "Market not yet accepting orders; wait and retry"
    }
}
```

### Rate Limits Reference

```python
# Source: docs.polymarket.com/quickstart/introduction/rate-limits
# Requests over limit are QUEUED (not dropped) via Cloudflare throttling

RATE_LIMITS = {
    # General endpoints
    "general": {"per_10s": 15000},
    "health_check": {"per_10s": 100},

    # Data API
    "data_general": {"per_10s": 1000},
    "data_trades": {"per_10s": 200},
    "data_positions": {"per_10s": 150},

    # Gamma API
    "gamma_general": {"per_10s": 4000},
    "gamma_markets": {"per_10s": 300},  # to 900 depending on operation
    "gamma_search": {"per_10s": 200},

    # CLOB API (trading)
    "clob_general": {"per_10s": 9000},
    "clob_book": {"per_10s": 1500},
    "clob_price": {"per_10s": 1500},

    # Trading operations - DUAL LIMITS (burst + sustained)
    "order_post": {
        "burst": {"per_10s": 3500, "per_second": 500},
        "sustained": {"per_10min": 36000, "per_second": 60}
    },
    "order_delete": {
        "burst": {"per_10s": 3000, "per_second": 300},
        "sustained": {"per_10min": 30000, "per_second": 50}
    },
    "batch_operations": {
        "burst": {"per_10s": 1000},
        "sustained": {"per_10min": 15000}
    },

    # Other
    "relayer_submit": {"per_1min": 25}
}
```

### NegRisk Detection and Handling

```python
# Source: Polymarket negRisk documentation
def analyze_negrisk_event(event: dict) -> dict:
    """Analyze a negRisk event for trading.

    Key fields:
    - negRisk: Basic negative risk event
    - enableNegRisk AND negRiskAugmented: Augmented (can add outcomes)

    Rules:
    - Only one outcome can resolve YES
    - NO share can convert to YES in all other outcomes
    - Only trade named outcomes in augmented events
    """
    result = {
        "is_negrisk": event.get("negRisk", False),
        "is_augmented": (
            event.get("enableNegRisk", False) and
            event.get("negRiskAugmented", False)
        ),
        "tradable_markets": [],
        "total_yes_price": 0.0
    }

    if not result["is_negrisk"]:
        # Standard binary market
        for market in event.get("markets", []):
            result["tradable_markets"].append({
                "question": market["question"],
                "condition_id": market["conditionId"],
                "yes_token": market["clobTokenIds"][0],
                "yes_price": float(market.get("outcomePrices", [0])[0])
            })
        return result

    # NegRisk event
    for market in event.get("markets", []):
        question = market.get("question", "")
        yes_price = float(market.get("outcomePrices", [0])[0])

        # Skip unnamed outcomes in augmented events
        if result["is_augmented"] and (not question or question.lower() == "other"):
            continue

        result["tradable_markets"].append({
            "question": question,
            "condition_id": market["conditionId"],
            "yes_token": market["clobTokenIds"][0],
            "yes_price": yes_price
        })
        result["total_yes_price"] += yes_price

    # Total should be ~1.0 (market edge = overround)
    result["overround"] = result["total_yes_price"] - 1.0

    return result
```

### Market Resolution Status Detection

```python
# Source: Phase 2 events-and-metadata.md, enhanced for edge cases
def get_market_resolution_status(market: dict) -> dict:
    """Get comprehensive resolution status.

    States:
    - open: Trading active
    - closed_pending: Trading stopped, awaiting resolution
    - proposed: Resolution proposed, in challenge period
    - disputed: Resolution disputed, escalated to UMA DVM
    - resolved: Final, redemption available
    """
    status = {
        "state": "unknown",
        "trading_allowed": False,
        "redemption_allowed": False,
        "details": {}
    }

    # Check resolved first
    if market.get("resolved") or market.get("resolvedAt"):
        status["state"] = "resolved"
        status["redemption_allowed"] = True
        status["details"] = {
            "winner": market.get("winner") or market.get("winningOutcome"),
            "resolved_at": market.get("resolvedAt")
        }
        return status

    # Check if closed but not resolved
    if market.get("closed") or not market.get("active"):
        status["state"] = "closed_pending"
        status["details"]["closed_at"] = market.get("updatedAt")

        # Note: Proposal/dispute status requires on-chain query
        # or monitoring UMA oracle events
        return status

    # Open for trading
    status["state"] = "open"
    status["trading_allowed"] = True
    status["details"]["end_date"] = market.get("endDate")

    return status
```

### Partial Fill Tracking

```python
# Source: CLOB API order response schema
class PartialFillTracker:
    """Track partial fills for order reconciliation.

    Key fields from order response:
    - original_size: What was requested
    - size_matched: What has filled
    - price_average: Weighted average fill price
    """

    def __init__(self):
        self.orders = {}  # order_id -> order state

    def track_order(self, order_id: str, original_size: float):
        """Start tracking a new order."""
        self.orders[order_id] = {
            "original_size": original_size,
            "filled_size": 0.0,
            "remaining_size": original_size,
            "fills": [],  # List of (size, price, timestamp)
            "avg_price": None,
            "status": "pending"
        }

    def update_from_response(self, response: dict):
        """Update tracking from order response."""
        order_id = response.get("orderID") or response.get("orderId")
        if not order_id or order_id not in self.orders:
            return

        order = self.orders[order_id]
        size_matched = float(response.get("size_matched", 0))

        if size_matched > order["filled_size"]:
            # New fill
            new_fill_size = size_matched - order["filled_size"]
            fill_price = response.get("price_average")

            order["fills"].append({
                "size": new_fill_size,
                "price": fill_price,
                "time": response.get("transactTime")
            })

            order["filled_size"] = size_matched
            order["remaining_size"] = order["original_size"] - size_matched
            order["avg_price"] = fill_price

        # Update status
        status = response.get("status", "unknown")
        if size_matched >= order["original_size"]:
            order["status"] = "filled"
        elif status == "LIVE":
            order["status"] = "partial"
        else:
            order["status"] = status.lower()

    def get_fill_summary(self, order_id: str) -> dict:
        """Get summary of fills for an order."""
        if order_id not in self.orders:
            return None

        order = self.orders[order_id]
        return {
            "order_id": order_id,
            "original_size": order["original_size"],
            "filled_size": order["filled_size"],
            "remaining_size": order["remaining_size"],
            "fill_count": len(order["fills"]),
            "avg_price": order["avg_price"],
            "status": order["status"],
            "fill_percentage": (
                order["filled_size"] / order["original_size"] * 100
                if order["original_size"] > 0 else 0
            )
        }
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Fixed tick sizes | Dynamic tick sizes at price extremes | 2024+ | Must check before every order |
| Simple resolution | UMA with whitelist proposals | Jan 2026 | More reliable but longer resolution |
| Manual rate limiting | Cloudflare throttling with burst tiers | 2025+ | Requests queued not dropped |
| Batch limit 5 | Batch limit 15 | Jan 2026 | Higher throughput |

**Deprecated/outdated:**
- Assuming tick size is always 0.01 - changes at price > 0.96 or < 0.04
- Simple retry on 401 - need full credential refresh
- Native USDC on Polygon - must use USDC.e

## Open Questions

Things that couldn't be fully resolved:

1. **Exact Minimum Order Size**
   - What we know: Orderbook has no enforced minimum, but rewards have minimum shares threshold
   - What's unclear: Whether there's a practical minimum for order acceptance
   - Recommendation: Document as "no minimum" with note about liquidity considerations

2. **Price Range 0.99 vs 0.999**
   - What we know: API rejects 0.999 with range 0.01-0.99, but UI allows it
   - What's unclear: Whether this is a bug or intentional discrepancy
   - Recommendation: Document 0.01-0.99 as safe range, note UI discrepancy as known issue

3. **NegRisk Conversion API**
   - What we know: Conversion happens via NegRiskAdapter contract
   - What's unclear: Whether py-clob-client has a convenience method
   - Recommendation: Document as on-chain operation, defer to V2 if API method found

4. **Data API Rate Limits for Activity Endpoint**
   - What we know: General Data API is 1000/10s, trades is 200/10s
   - What's unclear: Specific limit for /activity endpoint
   - Recommendation: Use 200/10s as conservative estimate

## Sources

### Primary (HIGH confidence)
- [py-clob-client GitHub](https://github.com/Polymarket/py-clob-client) - Exception classes, issues
- [Polymarket Rate Limits](https://docs.polymarket.com/quickstart/introduction/rate-limits) - Official limits
- [Polymarket NegRisk Overview](https://docs.polymarket.com/developers/neg-risk/overview) - NegRisk mechanics
- [py-clob-client Issue #121](https://github.com/Polymarket/py-clob-client/issues/121) - FOK precision
- [py-clob-client Issue #122](https://github.com/Polymarket/py-clob-client/issues/122) - Tick size caching
- [py-clob-client Issue #109](https://github.com/Polymarket/py-clob-client/issues/109) - Balance/allowance errors
- [py-clob-client Issue #187](https://github.com/Polymarket/py-clob-client/issues/187) - 401 Unauthorized

### Secondary (MEDIUM confidence)
- [Polymarket CLOB Orders](https://docs.polymarket.com/developers/CLOB/orders/create-order) - Order validation
- [Polymarket Market Resolution](https://docs.polymarket.com/polymarket-learn/markets/how-are-markets-resolved) - UMA process
- [UMA Oracle Update](https://www.theblock.co/post/366507/polymarket-uma-oracle-update) - Whitelist proposers
- [NautilusTrader Polymarket](https://nautilustrader.io/docs/latest/integrations/polymarket/) - Third-party patterns

### Tertiary (LOW confidence - marked for validation)
- GitHub issues for specific error messages - community-reported
- Tick size change behavior at extremes - observed but not officially documented

## Metadata

**Confidence breakdown:**
- Exception types: HIGH - verified from source code
- Rate limits: HIGH - official documentation
- Pitfalls: HIGH - combination of docs and GitHub issues
- NegRisk mechanics: HIGH - official documentation
- Resolution process: MEDIUM - documentation + external sources

**Research date:** 2026-01-31
**Valid until:** 2026-03-02 (30 days - API generally stable)

**Research limitations:**
- Exact minimum order size not found in documentation
- Price range discrepancy (0.99 vs 0.999) unresolved
- NegRisk conversion API method not confirmed in py-clob-client

**Existing documentation to reference:**
- `skills/polymarket/auth/token-allowances.md` - USDC.e detection already documented
- `skills/polymarket/trading/order-types.md` - FOK precision partially covered
- `skills/polymarket/market-discovery/events-and-metadata.md` - NegRisk fields documented
- `skills/polymarket/real-time/connection-management.md` - WebSocket patterns covered

**Phase 3 should expand rather than duplicate these existing files.**
