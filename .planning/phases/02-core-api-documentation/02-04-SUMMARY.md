---
phase: 02
plan: 04
subsystem: trading
tags: [clob, orders, positions, balances, batch]
dependency-graph:
  requires: [02-03]
  provides: [order-placement-workflow, order-management, position-tracking]
  affects: [03-trading-strategies]
tech-stack:
  added: []
  patterns: [pre-order-validation, tick-size-validation, batch-operations, order-lifecycle]
key-files:
  created:
    - skills/polymarket/trading/order-placement.md
    - skills/polymarket/trading/order-management.md
    - skills/polymarket/trading/positions-and-balances.md
  modified:
    - skills/polymarket/trading/README.md
decisions:
  - id: pre-placement-checklist
    choice: "Document 4-step verification before orders"
    rationale: "Prevents common failures from missing auth, token ID, tick size, or balance"
  - id: order-lifecycle-diagram
    choice: "Visual lifecycle showing all state transitions"
    rationale: "Clarifies when cancellation is possible and what triggers each state"
  - id: batch-limit-prominent
    choice: "15-order limit highlighted in batch operations"
    rationale: "Prevents API errors from exceeding undocumented limit"
  - id: on-chain-balance-pattern
    choice: "Direct web3 queries for USDC.e balance"
    rationale: "CLOB doesn't expose balance, on-chain is definitive source"
metrics:
  duration: 4 min
  completed: 2026-01-31
---

# Phase 2 Plan 4: Trading Order Lifecycle Summary

Complete order placement, management, and position tracking documentation for CLOB API trading operations.

## One-liner

Order lifecycle with pre-placement validation, batch operations (max 15), and on-chain balance verification via web3.

## What Was Built

### 1. Order Placement Documentation (order-placement.md)

**647 lines** covering CLOB-01 detailed placement:

- Pre-placement checklist (auth, token ID, tick size, balance)
- Market context retrieval (tick size, midpoint, spread, depth)
- Complete placement workflow with step-by-step process
- OrderArgs request schema with all field constraints
- Response schema with status values (live, matched, delayed)
- Tick size validation with rounding patterns
- Common errors: INVALID_ORDER_MIN_TICK_SIZE, invalid amounts, insufficient balance
- Four placement patterns: simple limit, market-style (FAK), GTD, FOK
- Robust order placement with retries and error handling

### 2. Order Management Documentation (order-management.md)

**862 lines** covering CLOB-02, CLOB-03, CLOB-06:

- Order lifecycle diagram with state transitions
- Single order cancellation with response handling
- Cancel all orders and cancel by market patterns
- Order status checking with fill percentage
- Open orders retrieval with filtering
- Order response schema with all fields
- Batch order placement (max 15 orders per batch)
- Batch cancellation with independent failure handling
- Ladder order creation pattern
- Management patterns: cancel-and-replace, stale cleanup, position-aware
- OrderManager class for convenient operations

### 3. Positions and Balances Documentation (positions-and-balances.md)

**627 lines** covering CLOB-04:

- Overview of balance sources (CLOB, on-chain, Data API)
- Position concepts (YES/NO, entry price, resolution)
- CLOB position methods with fallback inference
- On-chain USDC.e balance verification via web3
- Allowance checking for EOA wallets
- Pre-order validation for BUY and SELL orders
- Cross-reference to Data API for detailed PnL
- TradingValidator class for complete validation
- Common issues: USDC vs USDC.e, wrong address, caching

### 4. README Updates

- Added links to all three new documents
- Updated reading order for first-time traders
- Added specific task navigation
- Updated status to Complete

## Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Pre-placement checklist | 4-step verification | Prevents common failures |
| Order lifecycle diagram | Visual state machine | Clarifies when cancellation possible |
| Batch limit prominent | 15-order max highlighted | Prevents API errors |
| On-chain balance pattern | Direct web3 queries | CLOB doesn't expose balance |

## CLOB Requirements Coverage

| Requirement | Document | Status |
|-------------|----------|--------|
| CLOB-01 (Order placement detailed) | order-placement.md | Complete |
| CLOB-02 (Cancellation) | order-management.md | Complete |
| CLOB-03 (Order status) | order-management.md | Complete |
| CLOB-04 (Positions) | positions-and-balances.md | Complete |
| CLOB-05 (FOK/FAK) | order-types.md (Plan 03) | Complete |
| CLOB-06 (Batch operations) | order-management.md | Complete |

## Key Patterns Documented

### Pre-Order Validation Pattern

```python
def verify_ready_to_trade(client, token_id, side, size):
    # 1. Check auth
    # 2. Get tick size
    # 3. Check liquidity
    return True
```

### Tick Size Validation Pattern

```python
tick_size = float(client.get_tick_size(token_id))
validated_price = round(price / tick_size) * tick_size
```

### Batch Order Pattern

```python
# Max 15 orders per batch
orders = [create_order(args) for args in orders_data[:15]]
responses = client.post_orders(orders, OrderType.GTC)
```

### On-Chain Balance Pattern

```python
from web3 import Web3
balance = contract.functions.balanceOf(wallet).call() / 1e6  # 6 decimals
```

## Files Created/Modified

| File | Lines | Purpose |
|------|-------|---------|
| skills/polymarket/trading/order-placement.md | 647 | Complete placement workflow |
| skills/polymarket/trading/order-management.md | 862 | Cancellation, status, batch |
| skills/polymarket/trading/positions-and-balances.md | 627 | Positions and balance queries |
| skills/polymarket/trading/README.md | +11/-7 | Updated links and status |

**Total new documentation:** 2,136 lines

## Deviations from Plan

None - plan executed exactly as written.

## Next Phase Readiness

Trading skill folder is now complete with 6 documents:
- README.md (Plan 03)
- clob-api-overview.md (Plan 03)
- order-types.md (Plan 03)
- order-placement.md (this plan)
- order-management.md (this plan)
- positions-and-balances.md (this plan)

All CLOB requirements (CLOB-01 through CLOB-06) are fully documented. Ready for Phase 3 trading strategies which can reference these patterns.

## Commits

1. `605b82b` - feat(02-04): create order placement documentation
2. `dd8c25a` - feat(02-04): create order management documentation
3. `673f6dd` - feat(02-04): create positions and balances documentation
4. `fb54878` - docs(02-04): update trading README with new file links
