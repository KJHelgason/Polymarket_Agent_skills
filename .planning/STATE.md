# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2025-01-31)

**Core value:** Claude knows Polymarket as well as someone who's built with it extensively — no guessing, no common mistakes, no ambiguity.
**Current focus:** Phase 3 - Edge Cases & Best Practices

## Current Position

Phase: 3 of 4 (Edge Cases & Best Practices)
Plan: 5 of 5 complete
Status: Phase complete
Last activity: 2026-01-31 — Completed 03-04-PLAN.md (Library Error Handling)

Progress: [█████████░] 90%

## Performance Metrics

**Velocity:**
- Total plans completed: 16
- Average duration: 3 min
- Total execution time: 0.85 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1 - Authentication & Setup Foundation | 3/3 | 10 min | 3 min |
| 2 - Core API Documentation | 8/8 | 26 min | 3 min |
| 3 - Edge Cases & Best Practices | 5/5 | 18 min | 4 min |

**Recent Trend:**
- Last 5 plans: 02-08 (4 min), 03-01 (3 min), 03-02 (4 min), 03-03 (4 min), 03-04 (3 min)
- Trend: Consistent velocity

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Skills-based approach: Allows selective loading of Polymarket knowledge when needed, shareable format
- Deep research before documenting: Official docs may be incomplete or ambiguous; need to understand actual behavior
- Use create_or_derive_api_creds() as recommended method: Safer for initial setup, retrieves existing rather than invalidating (01-01)
- Document nonce tracking as critical: Without nonce, credential recovery impossible (01-01)
- Separate L1/L2 authentication documentation: Clarifies different purposes and when each is used (01-01)
- Unlimited approvals (MaxUint256) as standard pattern: Industry standard for Polymarket token setup (01-02)
- USDC.e as exclusive token: Native USDC requires swap, prominently documented (01-02)
- Proxy wallet allowance exemption: Reduces confusion about which wallets need manual setup (01-02)
- Client initialization as main entry point: Synthesizes all auth concepts into single setup flow (01-03)
- README index with 30-second navigation goal: Quick reference enables fast problem resolution (01-03)
- Common issues inline solutions: Top 6 problems cover 90% of failures, provide diagnosis code (01-03)
- Gamma API no-auth pattern: Public market data enables simpler discovery workflow (02-01)
- Token index convention [0]=YES, [1]=NO: Consistent extraction for trading (02-01)
- Conservative rate limiting (0.5s delay): No published limits, defensive approach (02-01)
- Generator pattern for pagination: Memory efficient for large datasets (02-02)
- State detection function: Single source of truth for market states (02-02)
- NegRisk introduction only: Full trading mechanics deferred to Phase 3 (02-02)
- FOK precision prominent: 2 decimal max is common failure point, documented with warnings (02-03)
- FAK as market order: Clearest mental model for immediate execution (02-03)
- Decision tree for order selection: Visual aid helps users choose correct type (02-03)
- GTD seconds not milliseconds: Prevent common timestamp confusion (02-03)
- Pre-placement checklist: 4-step verification prevents common order failures (02-04)
- Order lifecycle diagram: Visual state machine clarifies cancellation timing (02-04)
- Batch limit prominent: 15-order max per batch highlighted (02-04)
- On-chain balance pattern: Direct web3 queries for USDC.e balance (02-04)
- Data API no-auth pattern: Public wallet queries simplify portfolio analysis (02-05)
- CLOB for price history: Historical prices via CLOB /prices-history endpoint (02-05)
- Pagination generator pattern: Memory-efficient for large trade histories (02-05)
- OrderbookManager class pattern: Encapsulates local state management with Decimal precision (02-07)
- Token IDs vs Condition IDs explicit: Different ID types for different channels prevents subscription errors (02-07)
- FIFO tax lots: Standard accounting method for cost basis calculation (02-06)
- ISO date format for exports: Universal format compatible with accounting software (02-06)
- Bounded collections for history: Prevents memory growth in long-running connections (02-07)
- 5-second heartbeat interval: Balance between keeping connection alive and minimizing overhead (02-08)
- 5-minute data timeout threshold: Long enough for low-activity markets, short enough to detect issues (02-08)
- Exponential backoff with jitter: Prevents thundering herd on reconnection (02-08)
- Symptom-to-solution structure: Troubleshooting-first approach enables fast problem resolution (03-01)
- FOK -> GTC fallback: When precision requirements cannot be met, graceful degradation is better than failure (03-01)
- Always fetch tick size: Cached values cause rejections at price extremes (03-01)
- NegRisk unnamed outcome detection as critical safety: Augmented negRisk events have placeholder outcomes (03-03)
- PartialFillTracker as reusable class pattern: Full implementation for order fill lifecycle tracking (03-03)
- Three reconciliation patterns: Real-time WebSocket, polling-based, and end-of-day reconciliation (03-03)
- Midpoint as display-only: Users expect to trade at displayed price but can't (03-02)
- Spread thresholds (2/5/10%): Quantifiable liquidity assessment for order type selection (03-02)
- UMA timeline (2h/48h): Official dispute process timing documented (03-02)
- ERROR_SOLUTIONS as lookup dictionary: Programmatic error categorization for automated recovery (03-04)
- 5 recovery patterns: Auth, precision, rate limit, server error, comprehensive handler (03-04)

### Pending Todos

None yet.

### Blockers/Concerns

None yet.

## Session Continuity

Last session: 2026-01-31
Stopped at: Completed 03-04-PLAN.md
Resume file: None

**Phase 3 Complete:** All edge cases and library documentation finished. Created:
- edge-cases/: usdc-token-confusion.md, order-constraints.md, price-interpretation.md, resolution-mechanics.md, negrisk-trading.md, partial-fills.md
- library/: README.md, error-handling.md

Requirements satisfied: EDGE-01 through EDGE-09, LIB-03. Ready for Phase 4 (Production Patterns & Advanced Topics).
