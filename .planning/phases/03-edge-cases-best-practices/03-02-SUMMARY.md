---
phase: 03-edge-cases-best-practices
plan: 02
subsystem: edge-cases
tags: [price, spread, resolution, UMA, oracle, disputes]
requires: [03-RESEARCH.md]
provides: [price-interpretation-guide, resolution-mechanics-guide]
affects: []
tech-stack:
  added: []
  patterns: [price-analysis, resolution-status-detection]
key-files:
  created:
    - skills/polymarket/edge-cases/price-interpretation.md
    - skills/polymarket/edge-cases/resolution-mechanics.md
  modified:
    - skills/polymarket/edge-cases/README.md
decisions:
  - Midpoint as display-only (not executable)
  - Spread analysis thresholds (2%, 5%, 10%)
  - Resolution states (open, closed_pending, proposed, disputed, resolved)
  - UMA timeline (2-hour challenge, 48-hour DVM)
metrics:
  duration: 4 min
  completed: 2026-01-31
---

# Phase 3 Plan 2: Price Interpretation & Resolution Mechanics Summary

Document price interpretation nuances and market resolution mechanics including UMA dispute handling.

## One-liner

Clarified midpoint vs executable prices with spread analysis, documented UMA resolution timeline with 2-hour/48-hour dispute windows.

## What Was Done

### Task 1: Price Interpretation Guide (EDGE-04)

Created `skills/polymarket/edge-cases/price-interpretation.md`:

- **Midpoint vs Executable Prices:** Clear distinction between display price (midpoint) and actual execution prices (bid/ask)
- **Spread Calculation:** Examples showing roundtrip trading costs
- **Order Book Analysis:** Code for estimating execution price for large orders
- **Liquidity Thresholds:** Spread percentages indicating high/medium/low liquidity
- **Common Mistakes:** Using midpoint for orders, PnL with midpoint, ignoring spread

Key code pattern - price analysis:
```python
buy_price = client.get_price(token_id, side="BUY")   # Executable ask
sell_price = client.get_price(token_id, side="SELL") # Executable bid
midpoint = client.get_midpoint(token_id)             # Display only
spread = buy_price - sell_price                       # Trading cost
```

### Task 2: Resolution Mechanics Guide (EDGE-05, EDGE-06)

Created `skills/polymarket/edge-cases/resolution-mechanics.md`:

- **Resolution States Table:** Open, Closed Pending, Proposed, Disputed, Resolved
- **UMA Oracle Process:** Complete timeline from event close to redemption
- **Dispute Timeline:** 2-hour challenge window, DVM 48-hour voting
- **Whitelisted Proposers:** January 2026 update on resolution system
- **Redemption Process:** When and how to redeem winning positions
- **Status Detection Code:** Function to determine current resolution state

Resolution timeline diagram (ASCII):
```
Event Closes -> Proposal (2h) -> [No Dispute] -> Resolved
                    |
                    v
              [Dispute] -> New Proposal (2h) -> [No Dispute] -> Resolved
                                  |
                                  v
                            [Dispute] -> DVM Vote (48h) -> Resolved
```

### README Updates

Updated `skills/polymarket/edge-cases/README.md`:
- Added price-interpretation.md and resolution-mechanics.md to index
- Added new symptoms to Quick Symptom-to-Solution Guide
- Expanded troubleshooting flow with price and resolution sections
- Moved completed items from "Coming Soon"

## Decisions Made

| Decision | Rationale | Impact |
|----------|-----------|--------|
| Midpoint as display-only | Users expect to trade at displayed price but can't | Clear warning prevents bad trading decisions |
| Spread thresholds (2/5/10%) | Quantifiable liquidity assessment | Helps users decide market vs limit orders |
| Resolution states from API | Proposed/Disputed require on-chain queries | Documented limitation, provided API-based detection |
| UMA timeline (2h/48h) | Official dispute process timing | Users understand why payouts take time |

## Requirements Coverage

| Requirement | Status | File | Evidence |
|-------------|--------|------|----------|
| EDGE-04 | Complete | price-interpretation.md | get_midpoint vs get_price distinction |
| EDGE-05 | Complete | resolution-mechanics.md | Resolution states table |
| EDGE-06 | Complete | resolution-mechanics.md | UMA dispute timeline |

## Links Created

| From | To | Pattern |
|------|-----|---------|
| price-interpretation.md | clob-api-overview.md | Reference link |
| resolution-mechanics.md | events-and-metadata.md | Reference link |

## Artifacts

| File | Lines | Purpose |
|------|-------|---------|
| skills/polymarket/edge-cases/price-interpretation.md | 261 | Price vs odds clarification |
| skills/polymarket/edge-cases/resolution-mechanics.md | 368 | Resolution states and disputes |
| skills/polymarket/edge-cases/README.md | +21 | Index updates |

## Deviations from Plan

None - plan executed exactly as written.

## Must-Have Verification

| Truth | Status | Evidence |
|-------|--------|----------|
| User can distinguish midpoint price from executable price | Pass | get_midpoint vs get_price code examples |
| User understands spread represents actual trading cost | Pass | Spread calculation with roundtrip cost |
| User can determine if market is pending resolution, disputed, or finalized | Pass | Resolution states table with API fields |
| User understands UMA oracle resolution timeline | Pass | Timeline diagram with 2h/48h windows |

## Next Phase Readiness

**Completed:** 2/2 tasks
**Blockers:** None
**Ready for:** Phase 3 Plan 3 (NegRisk Trading) or Plan 4 (Partial Fills)

## Commits

| Hash | Message |
|------|---------|
| 94592c4 | docs(03-02): add price interpretation guide |
| f3106ad | docs(03-02): add resolution mechanics guide |
| 360881c | docs(03-02): update edge-cases README with new guides |
