# Requirements: Polymarket Agent Skills

**Defined:** 2025-01-31
**Core Value:** Claude knows Polymarket as well as someone who's built with it extensively — no guessing, no common mistakes, no ambiguity.

## v1 Requirements

### Authentication & Setup

- [x] **AUTH-01**: Document L1/L2 authentication flow with code examples
- [x] **AUTH-02**: Document proxy wallet vs EOA wallet patterns and detection
- [x] **AUTH-03**: Document API key derivation, storage, and rotation
- [x] **AUTH-04**: Document USDC.e token allowance setup and approval flow

### Market Discovery (Gamma API)

- [x] **GAMMA-01**: Document querying active markets with endpoint details and response schemas
- [x] **GAMMA-02**: Document market metadata fields (outcomes, end dates, resolution status)
- [x] **GAMMA-03**: Document search and filter patterns (keyword, category, status)
- [x] **GAMMA-04**: Document pagination patterns for large result sets

### Trading Operations (CLOB API)

- [x] **CLOB-01**: Document GTC limit order placement with full request/response schemas
- [x] **CLOB-02**: Document order cancellation patterns
- [x] **CLOB-03**: Document order status checking and open orders retrieval
- [x] **CLOB-04**: Document position viewing and balance checking
- [x] **CLOB-05**: Document FOK, FAK, GTD order types with precision requirements
- [x] **CLOB-06**: Document batch order operations (up to 15 orders)

### Real-Time Data (WebSocket)

- [x] **WS-01**: Document WebSocket connection setup and subscription patterns
- [x] **WS-02**: Document price update stream handling
- [x] **WS-03**: Document connection lifecycle (heartbeat, reconnection after ~20min stops)
- [x] **WS-04**: Document user-specific streams (order fills, trade notifications)
- [x] **WS-05**: Document orderbook depth streaming

### Data & Analytics (Data API)

- [x] **DATA-01**: Document position and balance retrieval
- [x] **DATA-02**: Document trade history queries
- [x] **DATA-03**: Document historical price data access
- [x] **DATA-04**: Document portfolio export patterns for accounting

### Edge Cases & Pitfalls

- [x] **EDGE-01**: Document USDC.e token address and common confusion with Native USDC
- [x] **EDGE-02**: Document minimum order constraints (shares, dollar amounts)
- [x] **EDGE-03**: Document decimal precision requirements by order type
- [x] **EDGE-04**: Document price vs odds interpretation (midpoint vs executable)
- [x] **EDGE-05**: Document market resolution status fields and meanings
- [x] **EDGE-06**: Document disputed outcome handling
- [x] **EDGE-07**: Document NegRisk multi-outcome market patterns
- [x] **EDGE-08**: Document dynamic tick size handling
- [x] **EDGE-09**: Document partial fill tracking and reconciliation

### py-clob-client Library

- [x] **LIB-01**: Document client initialization with all configuration options
- [x] **LIB-02**: Document common method signatures and usage patterns
- [x] **LIB-03**: Document error handling and exception types
- [x] **LIB-04**: Document best practices for production usage

### Skill Packaging

- [ ] **SKILL-01**: Create main Polymarket skill that loads core knowledge
- [ ] **SKILL-02**: Structure documentation for selective loading (auth, trading, websocket, etc.)
- [ ] **SKILL-03**: Package for shareability with other Claude users
- [ ] **SKILL-04**: Include version tracking for API changes

## v2 Requirements

### Extended Features

- **V2-01**: Gnosis Safe multi-sig authentication support
- **V2-02**: Builder program attribution documentation
- **V2-03**: Multi-chain bridge integration patterns
- **V2-04**: Conditional token (CTF) split/merge operations

## Out of Scope

| Feature | Reason |
|---------|--------|
| Trading bot logic | This is knowledge/documentation, not a trading system |
| Backtesting framework | Focus is on API expertise, not strategy testing |
| Account management UI | Just the API knowledge layer |
| Geographic restriction workarounds | Legal/compliance concern |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 1 | Complete |
| AUTH-02 | Phase 1 | Complete |
| AUTH-03 | Phase 1 | Complete |
| AUTH-04 | Phase 1 | Complete |
| LIB-01 | Phase 1 | Complete |
| GAMMA-01 | Phase 2 | Complete |
| GAMMA-02 | Phase 2 | Complete |
| GAMMA-03 | Phase 2 | Complete |
| GAMMA-04 | Phase 2 | Complete |
| CLOB-01 | Phase 2 | Complete |
| CLOB-02 | Phase 2 | Complete |
| CLOB-03 | Phase 2 | Complete |
| CLOB-04 | Phase 2 | Complete |
| CLOB-05 | Phase 2 | Complete |
| CLOB-06 | Phase 2 | Complete |
| WS-01 | Phase 2 | Complete |
| WS-02 | Phase 2 | Complete |
| WS-03 | Phase 2 | Complete |
| WS-04 | Phase 2 | Complete |
| WS-05 | Phase 2 | Complete |
| DATA-01 | Phase 2 | Complete |
| DATA-02 | Phase 2 | Complete |
| DATA-03 | Phase 2 | Complete |
| DATA-04 | Phase 2 | Complete |
| EDGE-01 | Phase 3 | Complete |
| EDGE-02 | Phase 3 | Complete |
| EDGE-03 | Phase 3 | Complete |
| EDGE-04 | Phase 3 | Complete |
| EDGE-05 | Phase 3 | Complete |
| EDGE-06 | Phase 3 | Complete |
| EDGE-07 | Phase 3 | Complete |
| EDGE-08 | Phase 3 | Complete |
| EDGE-09 | Phase 3 | Complete |
| LIB-02 | Phase 3 | Complete |
| LIB-03 | Phase 3 | Complete |
| LIB-04 | Phase 3 | Complete |
| SKILL-01 | Phase 4 | Pending |
| SKILL-02 | Phase 4 | Pending |
| SKILL-03 | Phase 4 | Pending |
| SKILL-04 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 37 total
- Mapped to phases: 37 ✓
- Unmapped: 0

---
*Requirements defined: 2025-01-31*
*Last updated: 2026-01-31 (Phase 3 requirements complete)*
