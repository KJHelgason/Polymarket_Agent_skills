---
phase: 03-edge-cases-best-practices
plan: 01
subsystem: edge-cases
tags: [usdc, token-confusion, order-precision, tick-size, fok, troubleshooting]
dependency-graph:
  requires: [01-token-allowances, 02-order-types]
  provides: [usdc-diagnostic, precision-helpers, tick-size-patterns]
  affects: [trading-integration, error-handling]
tech-stack:
  added: []
  patterns: [check-usdc-balances, prepare-fok-order, tick-size-validation]
key-files:
  created:
    - skills/polymarket/edge-cases/README.md
    - skills/polymarket/edge-cases/usdc-token-confusion.md
    - skills/polymarket/edge-cases/order-constraints.md
  modified: []
decisions:
  - pattern: "Symptom-to-solution structure"
    rationale: "Troubleshooting-first approach enables fast problem resolution"
  - pattern: "Reference existing docs instead of duplicate"
    rationale: "Avoid code duplication, link to token-allowances.md and order-types.md"
  - pattern: "FOK fallback to GTC"
    rationale: "When precision requirements cannot be met, graceful degradation is better than failure"
metrics:
  duration: 3 min
  completed: 2026-01-31
---

# Phase 03 Plan 01: Edge Cases Index & Token/Order Constraints Summary

**One-liner:** Edge cases directory with USDC token diagnostic (EDGE-01) and order precision/tick size handling (EDGE-02, EDGE-03, EDGE-08) enabling users to diagnose and resolve the two most common integration failures.

## What Was Built

### 1. Edge Cases Index (README.md)

- **Quick symptom-to-solution table** for common issues
- **Document index** organizing all edge case guides
- **Troubleshooting flow** decision tree
- **Prevention checklist** for pre-order validation
- **Navigation links** to related documentation

### 2. USDC Token Confusion Guide (usdc-token-confusion.md)

- **EDGE-01 coverage:** Native USDC vs USDC.e confusion
- **Symptoms section:** Polygon shows balance, Polymarket shows $0.00
- **Root cause explanation:** Exchange defaults, historical context
- **`check_usdc_balances()` diagnostic function:** Detects which token user has
- **Three solutions documented:**
  1. Polymarket UI "Activate Funds" auto-swap
  2. Manual DEX swap (QuickSwap/Uniswap)
  3. Correct deposit next time
- **Contract address reference table:** Both tokens clearly listed
- **Reference to token-allowances.md** for post-swap setup

### 3. Order Constraints Guide (order-constraints.md)

- **EDGE-02 (Minimums):** No enforced minimum, practical considerations
- **EDGE-03 (FOK Precision):**
  - Strict 2-decimal size requirement
  - Size x Price product constraint
  - `prepare_fok_order()` helper function
  - Precision fallback pattern (FOK -> GTC)
- **EDGE-08 (Dynamic Tick Size):**
  - Tick size changes at price extremes (>0.96 or <0.04)
  - ROUNDING_CONFIG table from py-clob-client
  - `get_validated_price()` helper function
  - WebSocket tick_size_change handling
- **Decision tree** for precision handling
- **Pre-order checklist** combining all validations
- **Common error messages** with solutions

## Coverage Analysis

| Requirement | Status | Documentation |
|-------------|--------|---------------|
| EDGE-01: USDC.e vs Native USDC | Complete | usdc-token-confusion.md |
| EDGE-02: Order Minimums | Complete | order-constraints.md |
| EDGE-03: FOK Precision | Complete | order-constraints.md |
| EDGE-08: Dynamic Tick Size | Complete | order-constraints.md |

**Four edge case requirements (EDGE-01, 02, 03, 08) now documented.**

## Key Patterns Established

### USDC Diagnostic Pattern
```python
def check_usdc_balances(wallet_address: str) -> dict:
    # Returns: usdc_e_balance, usdc_native_balance, status, action
    # Enables immediate diagnosis of token confusion issue
```

### FOK Precision Pattern
```python
def prepare_fok_order(price: float, size: float) -> dict:
    # Rounds size to 2 decimals
    # Verifies product precision
    # Returns adjusted parameters or raises ValueError
```

### Tick Size Validation Pattern
```python
def get_validated_price(client, token_id: str, desired_price: float) -> float:
    # Fetches current tick size
    # Rounds price to nearest tick
    # Returns validated price
```

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| Symptom-first structure | Users start with "what went wrong", not "what is X" |
| Reference not duplicate | token-allowances.md already has setup code; link to it |
| FOK -> GTC fallback | Better to place order as GTC than fail entirely |
| Always fetch tick size | Cached values cause rejections at price extremes |

## Deviations from Plan

None - plan executed exactly as written.

## Artifacts Created

| File | Lines | Purpose |
|------|-------|---------|
| skills/polymarket/edge-cases/README.md | 78 | Index and navigation |
| skills/polymarket/edge-cases/usdc-token-confusion.md | 223 | EDGE-01 diagnostic and solutions |
| skills/polymarket/edge-cases/order-constraints.md | 468 | EDGE-02, 03, 08 precision patterns |
| **Total new** | **769** | |

## Must-Have Verification

| Truth | Verified |
|-------|----------|
| User can diagnose USDC.e vs Native USDC | check_usdc_balances() returns token breakdown |
| User can resolve Native USDC with swap | Three solutions documented with steps |
| User can verify minimum order requirements | Practical considerations documented |
| User can apply precision rules per order type | prepare_fok_order() and ROUNDING_CONFIG |

| Artifact | Verified |
|----------|----------|
| README.md min 40 lines | 78 lines |
| usdc-token-confusion.md contains 0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174 | 4 occurrences |
| order-constraints.md contains ROUNDING_CONFIG | Present with full table |

| Link | Verified |
|------|----------|
| README -> usdc-token-confusion.md | 3 links present |
| order-constraints.md -> order-types.md | Link in Related Documentation |

## Next Phase Readiness

**Plan 03-01 Complete:**
- Edge cases directory established
- USDC and precision issues documented
- Ready for Plan 03-02 (additional edge cases)

**Remaining for Phase 3:**
- EDGE-04: Price interpretation (Plan 02)
- EDGE-05, EDGE-06: Resolution mechanics (Plan 03)
- EDGE-07: NegRisk trading (Plan 03)
- EDGE-09: Partial fill tracking (Plan 04)
- LIB-02, LIB-03, LIB-04: Library patterns (Plan 05)

## Commits

| Hash | Type | Description |
|------|------|-------------|
| debbf0a | feat | Edge cases index and USDC token confusion guide |
| f51e6e7 | feat | Order constraints guide (precision, tick size) |
