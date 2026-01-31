---
phase: 02-core-api-documentation
plan: 03
subsystem: trading
tags: [clob-api, order-types, gtc, gtd, fok, fak, trading, documentation]
dependency-graph:
  requires: [01-authentication, 02-01]
  provides: [clob-api-docs, order-types-reference, trading-skill-index]
  affects: [02-04, 02-05, order-placement, order-management]
tech-stack:
  added: []
  patterns: [limit-orders, immediate-execution, precision-validation, decision-trees]
key-files:
  created:
    - skills/polymarket/trading/clob-api-overview.md
    - skills/polymarket/trading/order-types.md
    - skills/polymarket/trading/README.md
  modified: []
decisions:
  - id: fok-precision-prominent
    summary: FOK precision requirements (2 decimal max) documented prominently with warning
    rationale: Common failure point - order rejected with "invalid amounts" if violated
  - id: fak-as-market-order
    summary: FAK documented as market-order equivalent with best-effort semantics
    rationale: No native market order type - FAK is closest approximation
  - id: decision-tree-selection
    summary: Order type selection via decision tree flowchart
    rationale: Helps users choose correct type without reading all documentation
  - id: gtd-seconds-not-ms
    summary: GTD expiration explicitly documented as Unix seconds (not milliseconds)
    rationale: Common JavaScript/Python confusion point
metrics:
  duration: 5 min
  completed: 2026-01-31
---

# Phase 02 Plan 03: CLOB API Fundamentals & Order Types Summary

**One-liner:** CLOB API architecture with all four order types (GTC, GTD, FOK, FAK) documented including FOK precision requirements and order type selection guidance.

## Objective

Document the CLOB API fundamentals and order types for trading operations. Enable Claude to understand CLOB API architecture and choose the correct order type for any trading scenario. Covers CLOB-01 (GTC orders) and CLOB-05 (FOK, FAK, GTD orders) requirements.

## What Was Built

### 1. CLOB API Overview (524 lines)

**File:** `skills/polymarket/trading/clob-api-overview.md`

Comprehensive documentation of:
- API basics (base URL: https://clob.polymarket.com)
- Authentication requirements (L2 for write operations)
- Public endpoints (price, midpoint, book, tick-size, prices-history)
- Authenticated endpoints (order, orders, cancel operations)
- py-clob-client usage patterns for all methods
- Token ID context and relationship to Gamma API
- Error response patterns (400, 401, 404, 429)
- Best practices for tick size, rate limiting, verification

### 2. Order Types Documentation (672 lines)

**File:** `skills/polymarket/trading/order-types.md`

Complete order type reference:
- Overview table comparing all four types
- **GTC (Good Till Cancelled):** Standard limit orders, resting on book
- **GTD (Good Till Date):** Time-expiring limits with Unix timestamp
- **FOK (Fill or Kill):** All-or-nothing with STRICT precision requirements
- **FAK (Fill and Kill):** Market-style best-effort execution
- Decision tree flowchart for order type selection
- Code examples for each order type
- Response schema with status values
- Common issues and solutions
- Complete working example with smart type selection

### 3. Trading README Index (336 lines)

**File:** `skills/polymarket/trading/README.md`

Skill index providing:
- Quick start with basic GTC order example
- Prerequisites (auth, token ID, USDC.e, positions)
- Documentation index with reading order
- Order type quick reference table
- Decision flowchart for type selection
- Common issues with inline solutions
- Related skills diagram (auth, market-discovery, real-time)
- API quick reference tables

## Technical Decisions

| Decision | Rationale |
|----------|-----------|
| FOK precision prominent | 2 decimal max is a common failure point |
| FAK as market order | Clearest mental model for immediate execution |
| Decision tree selection | Visual aid for quick type selection |
| GTD seconds not ms | Prevent common timestamp confusion |
| Inline diagnostic code | Users can debug without leaving document |

## Verification Results

| Criterion | Status | Evidence |
|-----------|--------|----------|
| skills/polymarket/trading/ exists with 3 files | PASS | 3 files created |
| clob-api-overview.md 80+ lines | PASS | 524 lines |
| order-types.md 120+ lines | PASS | 672 lines |
| README.md 60+ lines | PASS | 336 lines |
| CLOB base URL documented | PASS | "https://clob.polymarket.com" |
| Authentication requirements explained | PASS | L2 auth for write ops |
| All four order types documented | PASS | GTC, GTD, FOK, FAK with examples |
| FOK precision requirements documented | PASS | 2 decimal max prominently noted |
| Decision tree included | PASS | Flowchart in both files |
| Cross-references to auth skill | PASS | Links to ../auth/ |

## Commits

| Commit | Description | Files |
|--------|-------------|-------|
| `ab2587c` | CLOB API overview documentation | clob-api-overview.md |
| `4c12a89` | Order types documentation | order-types.md |
| `8b63004` | Trading README index | README.md |

## Total Documentation

- **3 files created**
- **1,532 lines total**
- **CLOB-01 requirement covered** (GTC order creation)
- **CLOB-05 requirement covered** (FOK, FAK, GTD orders)

## Deviations from Plan

None - plan executed exactly as written.

## Next Phase Readiness

Plan 02-04 (Order Placement and Management) can proceed. This plan provides:
- CLOB API architecture and authentication patterns
- Order types with code examples
- Client library patterns established
- Error handling patterns documented

## Related Plans

- **02-01 Gamma API** - Provides token IDs for trading
- **02-02 Search and Filtering** - Advanced market discovery
- **02-04 Order Placement** - Detailed order workflows (next)
- **02-05 Data API** - Alternative data endpoints
