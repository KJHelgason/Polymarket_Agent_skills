---
phase: "03"
plan: "05"
title: "Production Patterns Guide"
subsystem: "library"
tags:
  - rate-limiting
  - websocket
  - balance-tracking
  - production
  - py-clob-client
dependencies:
  requires:
    - "03-04"  # library error handling
    - "02-08"  # connection management reference
  provides:
    - "LIB-02 (Common method signatures)"
    - "LIB-04 (Production best practices)"
    - "Rate limiting patterns"
    - "Balance tracking patterns"
  affects:
    - "04-*"  # Phase 4 advanced topics may reference
tech-stack:
  added: []
  patterns:
    - "RateLimiter class with sliding window"
    - "BalanceTracker for reconciliation"
    - "Burst vs sustained rate limit handling"
key-files:
  created:
    - skills/polymarket/library/production-patterns.md
  modified:
    - skills/polymarket/library/README.md
decisions:
  - desc: "RateLimiter with sliding window for burst limits"
    rationale: "Matches Polymarket's 10-second burst windows"
  - desc: "Conservative rate limits (80% of max)"
    rationale: "Safety margin to avoid queue delays"
  - desc: "BalanceTracker with on-chain USDC.e queries"
    rationale: "Direct web3 balance checks for reconciliation"
  - desc: "Reference connection-management.md for WebSocket"
    rationale: "Avoid duplication, existing doc is comprehensive"
metrics:
  duration: "4 min"
  completed: "2026-01-31"
---

# Phase 3 Plan 5: Production Patterns Guide Summary

**One-liner:** Rate limiting with burst/sustained limits, BalanceTracker for reconciliation, and method signatures for production deployments.

## What Was Built

### Production Patterns Guide

Created `skills/polymarket/library/production-patterns.md` (595 lines):

1. **Rate Limiting Section**
   - Complete RATE_LIMITS reference with all endpoints
   - Burst vs sustained limits explanation
   - RateLimiter class with sliding window implementation
   - Recommended limiter configurations (350/s orders, 300/s cancels)
   - TradingRateLimits coordinated limiter class

2. **WebSocket Reconnection Section**
   - Key reliability parameters table
   - Use case-specific timeout recommendations
   - Minimal reconnection pattern with backoff
   - Reference to connection-management.md for full implementation

3. **Balance Tracking Section**
   - On-chain USDC.e balance check function
   - BalanceTracker class for position reconciliation
   - Reconciliation example with tolerance handling

4. **Method Signatures Section**
   - Trading methods (create_order, post_order, cancel, cancel_all)
   - Order query methods (get_order, get_orders, get_trades)
   - Market data methods (get_tick_size, get_midpoint, get_price)
   - Credential methods (create_or_derive_api_creds, set_api_creds)
   - OrderArgs and OrderType reference

5. **Production Checklist**
   - Rate limiting setup
   - WebSocket reliability
   - Error handling
   - Balance tracking
   - Logging and monitoring
   - Graceful shutdown

### Library README Update

Updated `skills/polymarket/library/README.md`:
- Added link to production-patterns.md
- Added "When to use this section" guidance
- Added quick reference (version, GitHub, PyPI)
- Streamlined overview section

## Key Deliverables

### RateLimiter Class

```python
class RateLimiter:
    """Sliding window rate limiter for Polymarket API limits."""
    def __init__(self, max_calls: int, window_seconds: float): ...
    def wait_if_needed(self): ...  # Blocks until call allowed
    def can_proceed(self) -> bool: ...  # Non-blocking check
```

### BalanceTracker Class

```python
class BalanceTracker:
    """Track USDC.e balance changes for reconciliation."""
    def update(self, reason: str = None) -> float: ...
    def reconcile(self, expected: float, tolerance: float) -> dict: ...
```

### Rate Limit Quick Reference

| Endpoint | Burst (10s) | Sustained (10min) |
|----------|-------------|-------------------|
| POST /order | 3,500 | 36,000 |
| DELETE /order | 3,000 | 30,000 |
| Batch operations | 1,000 | 15,000 |
| Batch max orders | 15 per request | - |

## Commits

| Hash | Description |
|------|-------------|
| 12c95d6 | feat(03-05): create production patterns guide |
| 76a05b0 | docs(03-05): finalize library README with production patterns |

## Files Changed

| File | Lines | Change |
|------|-------|--------|
| skills/polymarket/library/production-patterns.md | 595 | Created |
| skills/polymarket/library/README.md | 93 | Updated |

## Deviations from Plan

None - plan executed exactly as written.

## Requirements Satisfied

| Requirement | Status |
|-------------|--------|
| LIB-02 (Common method signatures) | Complete |
| LIB-04 (Production best practices) | Complete |
| Rate limiting patterns with RateLimiter | Complete |
| Burst vs sustained limits documented | Complete |
| WebSocket reconnection reference | Complete |
| Balance tracking patterns | Complete |
| Method signatures quick reference | Complete |
| Production checklist | Complete |

## Verification Results

- [x] library/production-patterns.md exists with all sections (595 lines)
- [x] RateLimiter class included with proper configuration
- [x] RATE_LIMITS reference with all endpoint limits
- [x] Method signatures table complete (4 tables)
- [x] library/README.md updated with full navigation
- [x] Cross-references to connection-management.md and error-handling.md correct

## Key Links Established

| From | To | Via |
|------|-----|-----|
| library/production-patterns.md | real-time/connection-management.md | WebSocket reconnection reference |
| library/production-patterns.md | library/error-handling.md | Error handling integration |
| library/README.md | library/production-patterns.md | Library navigation |
| library/README.md | library/error-handling.md | Library navigation |

## Phase 3 Complete

Phase 3 Plan 5 completes the Edge Cases & Best Practices phase:

| Plan | Title | Status |
|------|-------|--------|
| 03-01 | USDC Confusion & Order Constraints | Complete |
| 03-02 | Price Interpretation & Resolution | Complete |
| 03-03 | NegRisk Trading & Partial Fills | Complete |
| 03-04 | Library Error Handling | Complete |
| 03-05 | Production Patterns Guide | Complete |

### Phase 3 Deliverables Summary

**Edge Cases (6 documents):**
- usdc-token-confusion.md
- order-constraints.md
- price-interpretation.md
- resolution-mechanics.md
- negrisk-trading.md
- partial-fills.md

**Library (3 documents):**
- README.md
- error-handling.md
- production-patterns.md

### Requirements Complete

| ID | Requirement | Document |
|----|-------------|----------|
| EDGE-01 | USDC.e vs Native USDC | usdc-token-confusion.md |
| EDGE-02 | Minimum order size | order-constraints.md |
| EDGE-03 | FOK precision requirements | order-constraints.md |
| EDGE-04 | Midpoint vs executable | price-interpretation.md |
| EDGE-05 | Market resolution timing | resolution-mechanics.md |
| EDGE-06 | UMA dispute process | resolution-mechanics.md |
| EDGE-07 | NegRisk multi-outcome | negrisk-trading.md |
| EDGE-08 | Dynamic tick size | order-constraints.md |
| EDGE-09 | Partial fill tracking | partial-fills.md |
| LIB-02 | Method signatures | production-patterns.md |
| LIB-03 | Exception handling | error-handling.md |
| LIB-04 | Production patterns | production-patterns.md |

## Next Phase Readiness

Phase 3 complete. Ready for Phase 4 (Production Patterns & Advanced Topics).

The library section now provides:
1. Error handling with recovery patterns
2. Production deployment patterns with rate limiting
3. Balance tracking for reconciliation
4. Quick reference for common methods

All Phase 2 and Phase 3 documentation is interconnected with proper cross-references.
