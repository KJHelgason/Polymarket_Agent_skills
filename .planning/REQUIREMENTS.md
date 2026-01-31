# Requirements: Polymarket Agent Skills

**Defined:** 2025-01-31
**Core Value:** Claude knows Polymarket as well as someone who's built with it extensively — no guessing, no common mistakes, no ambiguity.

## v1 Requirements

### Authentication & Setup

- [ ] **AUTH-01**: Document L1/L2 authentication flow with code examples
- [ ] **AUTH-02**: Document proxy wallet vs EOA wallet patterns and detection
- [ ] **AUTH-03**: Document API key derivation, storage, and rotation
- [ ] **AUTH-04**: Document USDC.e token allowance setup and approval flow

### Market Discovery (Gamma API)

- [ ] **GAMMA-01**: Document querying active markets with endpoint details and response schemas
- [ ] **GAMMA-02**: Document market metadata fields (outcomes, end dates, resolution status)
- [ ] **GAMMA-03**: Document search and filter patterns (keyword, category, status)
- [ ] **GAMMA-04**: Document pagination patterns for large result sets

### Trading Operations (CLOB API)

- [ ] **CLOB-01**: Document GTC limit order placement with full request/response schemas
- [ ] **CLOB-02**: Document order cancellation patterns
- [ ] **CLOB-03**: Document order status checking and open orders retrieval
- [ ] **CLOB-04**: Document position viewing and balance checking
- [ ] **CLOB-05**: Document FOK, FAK, GTD order types with precision requirements
- [ ] **CLOB-06**: Document batch order operations (up to 15 orders)

### Real-Time Data (WebSocket)

- [ ] **WS-01**: Document WebSocket connection setup and subscription patterns
- [ ] **WS-02**: Document price update stream handling
- [ ] **WS-03**: Document connection lifecycle (heartbeat, reconnection after ~20min stops)
- [ ] **WS-04**: Document user-specific streams (order fills, trade notifications)
- [ ] **WS-05**: Document orderbook depth streaming

### Data & Analytics (Data API)

- [ ] **DATA-01**: Document position and balance retrieval
- [ ] **DATA-02**: Document trade history queries
- [ ] **DATA-03**: Document historical price data access
- [ ] **DATA-04**: Document portfolio export patterns for accounting

### Edge Cases & Pitfalls

- [ ] **EDGE-01**: Document USDC.e token address and common confusion with Native USDC
- [ ] **EDGE-02**: Document minimum order constraints (shares, dollar amounts)
- [ ] **EDGE-03**: Document decimal precision requirements by order type
- [ ] **EDGE-04**: Document price vs odds interpretation (midpoint vs executable)
- [ ] **EDGE-05**: Document market resolution status fields and meanings
- [ ] **EDGE-06**: Document disputed outcome handling
- [ ] **EDGE-07**: Document NegRisk multi-outcome market patterns
- [ ] **EDGE-08**: Document dynamic tick size handling
- [ ] **EDGE-09**: Document partial fill tracking and reconciliation

### py-clob-client Library

- [ ] **LIB-01**: Document client initialization with all configuration options
- [ ] **LIB-02**: Document common method signatures and usage patterns
- [ ] **LIB-03**: Document error handling and exception types
- [ ] **LIB-04**: Document best practices for production usage

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
| AUTH-01 | Phase 1 | Pending |
| AUTH-02 | Phase 1 | Pending |
| AUTH-03 | Phase 1 | Pending |
| AUTH-04 | Phase 1 | Pending |
| LIB-01 | Phase 1 | Pending |
| GAMMA-01 | Phase 2 | Pending |
| GAMMA-02 | Phase 2 | Pending |
| GAMMA-03 | Phase 2 | Pending |
| GAMMA-04 | Phase 2 | Pending |
| CLOB-01 | Phase 2 | Pending |
| CLOB-02 | Phase 2 | Pending |
| CLOB-03 | Phase 2 | Pending |
| CLOB-04 | Phase 2 | Pending |
| CLOB-05 | Phase 2 | Pending |
| CLOB-06 | Phase 2 | Pending |
| WS-01 | Phase 2 | Pending |
| WS-02 | Phase 2 | Pending |
| WS-03 | Phase 2 | Pending |
| WS-04 | Phase 2 | Pending |
| WS-05 | Phase 2 | Pending |
| DATA-01 | Phase 2 | Pending |
| DATA-02 | Phase 2 | Pending |
| DATA-03 | Phase 2 | Pending |
| DATA-04 | Phase 2 | Pending |
| EDGE-01 | Phase 3 | Pending |
| EDGE-02 | Phase 3 | Pending |
| EDGE-03 | Phase 3 | Pending |
| EDGE-04 | Phase 3 | Pending |
| EDGE-05 | Phase 3 | Pending |
| EDGE-06 | Phase 3 | Pending |
| EDGE-07 | Phase 3 | Pending |
| EDGE-08 | Phase 3 | Pending |
| EDGE-09 | Phase 3 | Pending |
| LIB-02 | Phase 3 | Pending |
| LIB-03 | Phase 3 | Pending |
| LIB-04 | Phase 3 | Pending |
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
*Last updated: 2026-01-31 (traceability mapped to phases)*
