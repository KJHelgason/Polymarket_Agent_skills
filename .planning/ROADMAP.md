# Roadmap: Polymarket Agent Skills

## Overview

This roadmap transforms Polymarket API research into comprehensive Claude skills documentation. Starting with authentication fundamentals and the py-clob-client library, we document all three APIs (Gamma, CLOB, Data) and WebSocket capabilities, then capture critical edge cases and production patterns, and finally package everything as shareable Claude skills. The result: Claude knows Polymarket as well as someone who's built with it extensively.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Authentication & Setup Foundation** - Auth flows, wallet patterns, library initialization
- [ ] **Phase 2: Core API Documentation** - Complete REST and WebSocket documentation for all three APIs
- [ ] **Phase 3: Edge Cases & Best Practices** - Critical pitfalls, production patterns, library usage
- [ ] **Phase 4: Skill Packaging** - Shareable Claude skills with selective loading

## Phase Details

### Phase 1: Authentication & Setup Foundation
**Goal**: Claude understands how to authenticate with Polymarket and initialize the py-clob-client library correctly
**Depends on**: Nothing (first phase)
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, LIB-01
**Success Criteria** (what must be TRUE):
  1. Documentation explains L1/L2 authentication flow with working code examples
  2. Documentation clearly differentiates proxy wallet vs EOA patterns and shows detection method
  3. Documentation covers USDC.e token setup including address, allowance flow, and common confusion points
  4. Documentation shows py-clob-client initialization with all wallet types (EOA, POLY_PROXY, GNOSIS_SAFE)
**Plans**: 3 plans in 2 waves

Plans:
- [ ] 01-01-PLAN.md — L1/L2 authentication flow and API credential management (AUTH-01, AUTH-03)
- [ ] 01-02-PLAN.md — Wallet types detection and token allowance setup (AUTH-02, AUTH-04)
- [ ] 01-03-PLAN.md — Complete client initialization guide and auth skill index (LIB-01)

### Phase 2: Core API Documentation
**Goal**: Complete documentation of all Polymarket API endpoints with request/response schemas and real-time capabilities
**Depends on**: Phase 1
**Requirements**: GAMMA-01, GAMMA-02, GAMMA-03, GAMMA-04, CLOB-01, CLOB-02, CLOB-03, CLOB-04, CLOB-05, CLOB-06, WS-01, WS-02, WS-03, WS-04, WS-05, DATA-01, DATA-02, DATA-03, DATA-04
**Success Criteria** (what must be TRUE):
  1. Documentation covers market discovery workflow (search, filter, pagination) with Gamma API examples
  2. Documentation explains all CLOB trading operations (GTC/FOK/FAK/GTD orders, cancellation, status, positions, batch)
  3. Documentation covers Data API for positions, balances, trade history, and portfolio export
  4. Documentation explains WebSocket connection patterns for prices, orderbook, and user-specific streams
  5. All endpoint schemas include example requests and responses with field explanations
**Plans**: TBD

Plans:
- [ ] TBD (determined during plan-phase)

### Phase 3: Edge Cases & Best Practices
**Goal**: Document critical pitfalls and production-ready patterns that prevent common integration failures
**Depends on**: Phase 2
**Requirements**: EDGE-01, EDGE-02, EDGE-03, EDGE-04, EDGE-05, EDGE-06, EDGE-07, EDGE-08, EDGE-09, LIB-02, LIB-03, LIB-04
**Success Criteria** (what must be TRUE):
  1. Documentation explains USDC.e vs Native USDC confusion with detection and resolution steps
  2. Documentation covers all order constraints (minimums, decimals, tick sizes) with examples
  3. Documentation clarifies ambiguous fields (price vs odds, resolution status, NegRisk patterns)
  4. Documentation explains py-clob-client error handling, exception types, and recovery patterns
  5. Documentation provides best practices for production usage (rate limits, WebSocket reconnection, balance tracking)
**Plans**: TBD

Plans:
- [ ] TBD (determined during plan-phase)

### Phase 4: Skill Packaging
**Goal**: Package all documentation as shareable Claude skills with selective loading capabilities
**Depends on**: Phase 3
**Requirements**: SKILL-01, SKILL-02, SKILL-03, SKILL-04
**Success Criteria** (what must be TRUE):
  1. Main Polymarket skill exists and loads core knowledge when invoked
  2. Documentation is structured for selective loading (separate auth, trading, websocket, edge cases modules)
  3. Package includes installation instructions and works for other Claude users
  4. Version tracking system exists for API changes and documentation updates
**Plans**: TBD

Plans:
- [ ] TBD (determined during plan-phase)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Authentication & Setup Foundation | 0/3 | Planned | - |
| 2. Core API Documentation | 0/TBD | Not started | - |
| 3. Edge Cases & Best Practices | 0/TBD | Not started | - |
| 4. Skill Packaging | 0/TBD | Not started | - |

---
*Roadmap created: 2026-01-31*
*Last updated: 2026-01-31*
