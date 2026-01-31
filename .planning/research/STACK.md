# Stack Research: Polymarket API Integration

**Domain:** Polymarket API Integration and Trading
**Researched:** 2026-01-31
**Overall Confidence:** HIGH

## Executive Summary

Polymarket provides a comprehensive, multi-language SDK ecosystem for integrating with their Central Limit Order Book (CLOB) platform. The standard 2025-2026 stack centers on **py-clob-client** for Python, with additional APIs for market data (Gamma), portfolio analytics (Data), and real-time updates (WebSocket/RTDS).

**Key recommendation:** Use official Polymarket SDKs exclusively. The signing and authentication complexity makes manual REST implementation error-prone and unsafe.

---

## Recommended Stack

### Core Python Client

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| **py-clob-client** | 0.34.5 (2026-01-13) | Official CLOB client | Handles order signing, EIP-712 auth, HMAC signatures. Official and actively maintained (weekly releases). |
| Python | >=3.9.10 | Runtime requirement | Minimum version for py-clob-client compatibility |

**Installation:**
```bash
pip install py-clob-client
```

**Confidence:** HIGH - Official library, actively maintained, latest release January 2026

### Required Dependencies

py-clob-client automatically installs these core dependencies:

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| **eth-account** | >=0.13.0 | Wallet management, private key signing | Standard Ethereum library |
| **eth-utils** | >=4.1.1 | Ethereum utilities | Type conversions, address validation |
| **httpx[http2]** | >=0.27.0 | HTTP client with HTTP/2 support | Replaces requests, better performance |
| **websockets** | 12.0 | WebSocket connections | Real-time order/market updates |
| **poly_eip712_structs** | >=0.0.1 | Polymarket-specific EIP-712 signing | Official Polymarket library |
| **py-order-utils** | >=0.3.2 | Order structuring and hashing | Official Polymarket library |
| **py-builder-signing-sdk** | >=0.0.2 | Builder authentication support | For advanced order signing |
| **python-dotenv** | (any) | Environment variable management | Safe credential storage |

**Confidence:** HIGH - Verified from official setup.py (2026-01-13)

### Optional: Blockchain Interaction

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **web3.py** | 7.12.1 | Token allowances, on-chain operations | Required for EOA wallets to set USDC/CTF approvals |

**Note:** web3.py is NOT a direct dependency of py-clob-client but is needed for:
- Setting token allowances before trading (EOA wallets)
- Interacting with Conditional Token Framework (CTF)
- Direct blockchain operations

**Known issue:** Dependency conflicts with eth-typing reported. Pin web3==7.12.1 or web3==6.14.0 to avoid version conflicts.

**Confidence:** MEDIUM - Version recommendations from NautilusTrader docs and community reports

---

## Polymarket API Ecosystem

### REST APIs

| API | Base URL | Purpose | Authentication |
|-----|----------|---------|----------------|
| **CLOB API** | `https://clob.polymarket.com` | Order management, prices, orderbooks | L2 (API credentials) for trading; public for market data |
| **Gamma API** | `https://gamma-api.polymarket.com` | Market discovery, metadata, events | Public (no auth required) |
| **Data API** | `https://data-api.polymarket.com` | User positions, activity, trade history | L2 (API credentials) required |

### WebSocket Endpoints

| Endpoint | URL | Purpose | Authentication |
|----------|-----|---------|----------------|
| **CLOB WebSocket** | `wss://ws-subscriptions-clob.polymarket.com/ws/` | Orderbook updates, order status | Optional for public channels |
| **RTDS** | `wss://ws-live-data.polymarket.com` | Real-time crypto prices, comments, trade activity | Optional (wallet address for user streams) |

**Confidence:** HIGH - Verified from official documentation

---

## Authentication Architecture

### Two-Level Authentication System

**L1 Authentication (Private Key)**
- Uses wallet's private key to sign EIP-712 messages
- Generates API credentials
- Non-custodial (key stays with user)
- Required for: Creating API keys, signing orders locally

**L2 Authentication (API Credentials)**
- Uses API key, secret, passphrase generated from L1
- HMAC-SHA256 signatures for request authentication
- Required for: Order posting, cancellation, private endpoints
- Expires after 30 seconds (timestamp-based)

**Signature Types:**
- `0` - EOA/MetaMask (standard Ethereum wallet)
- `1` - POLY_PROXY (email/Magic wallets, browser proxies)
- `2` - GNOSIS_SAFE (multi-sig wallets)

**Confidence:** HIGH - Official documentation and SDK implementation

---

## Official SDKs (Multi-Language)

### Primary SDKs

| Language | Package | Version | Status |
|----------|---------|---------|--------|
| **Python** | py-clob-client | 0.34.5 (2026-01-13) | Official, actively maintained |
| **TypeScript** | @polymarket/clob-client | 5.2.1 (2026-01-13) | Official, actively maintained |
| **Rust** | polymarket-client-sdk | 0.3 | Official, production-ready |

### Supporting SDKs

| Language | Package | Purpose | Status |
|----------|---------|---------|--------|
| **TypeScript** | real-time-data-client | Real-time WebSocket (RTDS) | Official |
| **Go** | go-order-utils | Order signing utilities | Official (utilities only, not full client) |

**Confidence:** HIGH - Verified from official Polymarket GitHub organization

---

## Community Libraries

### Python

| Package | Version | Purpose | Recommendation |
|---------|---------|---------|----------------|
| **polymarket-apis** | 0.4.3 (2025-12-16) | Unified API wrapper with Pydantic validation | AVOID - Use official py-clob-client instead |

**Why avoid polymarket-apis:**
- Not official Polymarket package
- Requires Python >=3.12 (more restrictive than official client)
- Adds abstraction layer over official APIs
- May lag behind official API changes
- Pydantic validation is unnecessary overhead for official SDK

**Confidence:** MEDIUM - Package exists but official SDK is superior choice

### Go (Community SDKs)

All Go SDKs are **community-maintained** (not official):
- lixvyang/polymarket-sdk-go (2025-12-01)
- ivanzzeth/polymarket-go-gamma-client (2025-12-26)
- ybina/polymarket-sdk-go (2025-12-19)

**Recommendation:** Use official TypeScript/Python/Rust SDKs instead. If Go is required, Polymarket's official go-order-utils provides signing utilities but not a full client.

**Confidence:** MEDIUM - Community packages exist but lack official support

---

## What to Avoid

### Unofficial Forks and Variants

| Package | Why Avoid |
|---------|-----------|
| **py-clob-client-extended** | Unofficial fork, no clear maintenance, security risk |
| **py-clob-client-crypto** | Purpose unclear, not official, potential security risk |
| **tn-py-clob-client** | Fork with unknown modifications |

**Rule:** Only use packages from the official Polymarket GitHub organization (github.com/Polymarket).

### Manual REST API Implementation

**Don't:** Manually implement REST API calls with requests/httpx

**Why:**
- EIP-712 signing is complex (order structuring, hashing, signing)
- HMAC-SHA256 authentication requires precise timestamp management
- Easy to introduce security vulnerabilities
- Order schema changes require manual updates
- Official SDK handles all complexity automatically

**Exception:** Read-only public endpoints (market data, prices) are safe for manual implementation

**Confidence:** HIGH - Official documentation strongly recommends SDK usage

### Version Pitfalls

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Using web3.py without version pinning | Dependency conflicts with eth-typing | Pin web3==7.12.1 or web3==6.14.0 |
| Creating new API keys repeatedly | Unnecessary key proliferation | Use `createOrDeriveApiKey()` to reuse existing keys |
| Skipping token allowances | Orders fail for EOA wallets | Set USDC and CTF allowances before trading |

**Confidence:** HIGH - Verified from GitHub issues and best practices documentation

---

## Development Environment Setup

### Minimal Python Stack

```bash
# Core client
pip install py-clob-client

# For EOA wallet allowances
pip install web3==7.12.1

# Environment management
pip install python-dotenv
```

### Recommended Project Structure

```
project/
├── .env                    # Private keys, API credentials (NEVER commit)
├── .gitignore              # Include *.env
├── requirements.txt        # py-clob-client, web3==7.12.1
└── src/
    ├── auth.py            # API key management, credential derivation
    ├── client.py          # ClobClient initialization
    ├── orders.py          # Order creation, management
    └── websocket.py       # Real-time data streaming
```

### Environment Variables

```bash
# .env file
PRIVATE_KEY=0x...                           # Wallet private key (L1 auth)
FUNDER_ADDRESS=0x...                        # Proxy wallet address (if using)
POLY_API_KEY=...                            # L2 API key
POLY_SECRET=...                             # L2 API secret
POLY_PASSPHRASE=...                         # L2 API passphrase
```

**Security:** Add `.env` and `*.env` to `.gitignore` immediately

---

## Advanced Tools

### WebSocket Client (Built-in)

py-clob-client includes WebSocket support via the `websockets` library (v12.0):
- Subscribe to orderbook updates (market channel)
- Subscribe to user orders/trades (user channel)
- CLOB WebSocket: `wss://ws-subscriptions-clob.polymarket.com/ws/`

### Real-Time Data Socket (TypeScript Only)

For broader market data (comments, crypto prices):
- Use official `real-time-data-client` (TypeScript)
- RTDS endpoint: `wss://ws-live-data.polymarket.com`
- **No official Python RTDS client** - implement custom WebSocket client or use TypeScript SDK

**Confidence:** HIGH - Verified from official repositories

### Trading Bots and Agents

| Tool | Language | Purpose | Status |
|------|----------|---------|--------|
| **Polymarket/agents** | Python | AI agent framework for autonomous trading | Official, experimental |

**Note:** Uses py-clob-client internally. Demonstrates best practices for integration.

---

## API Endpoints Reference

### CLOB API (Trading)

**Base:** `https://clob.polymarket.com`

Common endpoints:
- GET `/price` - Current token price
- GET `/book` - Full orderbook
- GET `/midpoint` - Market midpoint
- POST `/order` - Create order (requires L2 auth)
- DELETE `/order` - Cancel order (requires L2 auth)

**Documentation:** https://docs.polymarket.com/developers/CLOB/introduction

### Gamma API (Market Data)

**Base:** `https://gamma-api.polymarket.com`

Common endpoints:
- GET `/events` - List events (with pagination, filters)
- GET `/markets` - List markets (filter by slug, token_id, tag, active/closed)
- GET `/events/{id}` - Event details

**Documentation:** https://docs.polymarket.com/developers/gamma-markets-api/overview

**Note:** Gamma indexes on-chain market data and adds metadata (categorization, volume, etc.)

### Data API (Portfolio Analytics)

**Base:** `https://data-api.polymarket.com`

Common endpoints:
- GET `/positions` - User positions with P&L (requires L2 auth)
- GET `/activity` - User activity history (requires L2 auth)
- GET `/trades` - User trade history (requires L2 auth)

**Sorting options:**
- `TOKENS` - Position size
- `CURRENT` - Current value
- `CASHPNL` - Cash profit/loss
- `PERCENTPNL` - Percent profit/loss

**Documentation:** https://docs.polymarket.com/quickstart/reference/endpoints

---

## Common Pitfalls and Solutions

### 1. Dependency Conflicts

**Problem:** web3.py version conflicts with eth-typing

**Solution:**
```bash
pip install py-clob-client web3==7.12.1 python-dotenv
```

**Confidence:** MEDIUM - Community-reported issue with documented workaround

### 2. Token Allowances (EOA Wallets)

**Problem:** Orders fail with "insufficient allowance" error

**Solution:** Set allowances before first trade
```python
# Using web3.py
# 1. Approve USDC spend
# 2. Approve CTF (Conditional Token Framework) spend
```

**Affects:** MetaMask, hardware wallets (signature type 0)
**Doesn't affect:** Proxy wallets (Magic, browser proxies)

**Confidence:** HIGH - Official documentation requirement

### 3. FOK Order Decimal Precision

**Problem:** Fill-or-Kill (FOK) orders fail with "invalid amount" error

**Solution:**
- Sell orders: Max 2 decimal places for maker amount
- Sell orders: Max 4 decimal places for taker amount
- Round amounts appropriately before submission

**Confidence:** MEDIUM - Reported in GitHub issues

### 4. API Key Management

**Problem:** Multiple API keys created unnecessarily

**Solution:** Use `createOrDeriveApiKey()` instead of `createApiKey()`
- Derives existing key if one exists
- Only creates new key if necessary
- Prevents key proliferation

**Confidence:** HIGH - Best practice from official SDK

### 5. Proxy Wallet Configuration

**Problem:** Orders attributed to wrong account or fail

**Solution:** Specify funder address on client initialization
```python
client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    chain_id=137,
    signature_type=1,  # POLY_PROXY
    funder=funder_address  # Address holding funds
)
```

**Confidence:** HIGH - Official documentation

---

## Testing and Risk Management

### Best Practices

1. **Test with small amounts first** - Automated trading carries significant risk
2. **Implement risk controls** - Position limits, stop losses
3. **Never trade more than you can afford to lose**
4. **Monitor rate limits** - CLOB API has rate limiting (1000 calls/hour for non-trading)
5. **Use testnet/staging if available** - Verify integration before production

### Rate Limits

| API | Limit | Notes |
|-----|-------|-------|
| CLOB (read) | 1000 calls/hour | Market data, public endpoints |
| CLOB (trading) | Lower (undocumented) | Order creation/cancellation |
| Gamma | Generous (undocumented) | Public API, no auth required |

**Confidence:** MEDIUM - Documentation mentions limits but doesn't specify all tiers

---

## Recent Changes and Breaking Updates

### 2025-2026 API Updates

| Change | Date | Impact |
|--------|------|--------|
| Batch order limit increased (5 → 15) | 2025 | Allows larger bulk operations |
| Price change message structure update | 2025-09-15 23:00 UTC | Breaking change for WebSocket consumers |
| New metadata fields in `/book` endpoints | 2025 | Reduces need for separate Gamma queries |

**Migration Guide:** https://docs.polymarket.com/changelog/changelog

**Confidence:** HIGH - Official changelog

### SDK Release Cadence

py-clob-client releases approximately **weekly** with bug fixes and feature additions:
- v0.34.5 (2026-01-13) - RFQ improvements
- v0.34.4 (2026-01-06) - Orderbook hash fix
- v0.34.3 (2026-01-06) - HeartBeats V1, post-only orders
- v0.34.0 (2024-12-21) - RFQ price logic fix
- v0.33.0 (2024-12-20) - Deterministic JSON serialization

**Recommendation:** Update py-clob-client regularly (monthly) to get bug fixes

**Confidence:** HIGH - Verified from GitHub releases

---

## Documentation Resources

### Official Documentation

| Resource | URL | Purpose |
|----------|-----|---------|
| Main docs | https://docs.polymarket.com/ | Complete documentation index |
| CLOB Introduction | https://docs.polymarket.com/developers/CLOB/introduction | Trading API overview |
| Authentication | https://docs.polymarket.com/developers/CLOB/authentication | L1/L2 auth patterns |
| WebSocket | https://docs.polymarket.com/developers/CLOB/websocket/wss-overview | Real-time data streaming |
| Gamma API | https://docs.polymarket.com/developers/gamma-markets-api/overview | Market metadata |
| Endpoints | https://docs.polymarket.com/quickstart/reference/endpoints | API reference |
| Changelog | https://docs.polymarket.com/changelog/changelog | Breaking changes, updates |
| LLM-friendly index | https://docs.polymarket.com/llms.txt | Complete doc index |

### GitHub Repositories

| Repository | URL | Purpose |
|------------|-----|---------|
| py-clob-client | https://github.com/Polymarket/py-clob-client | Official Python client |
| clob-client | https://github.com/Polymarket/clob-client | Official TypeScript client |
| rs-clob-client | https://github.com/Polymarket/rs-clob-client | Official Rust client |
| real-time-data-client | https://github.com/Polymarket/real-time-data-client | Official RTDS TypeScript client |
| go-order-utils | https://github.com/Polymarket/go-order-utils | Official Go utilities |
| agents | https://github.com/Polymarket/agents | AI agent examples |

**Confidence:** HIGH - Official Polymarket organization

---

## Confidence Assessment

| Area | Confidence | Reasoning |
|------|-----------|-----------|
| **py-clob-client** | HIGH | Official SDK, verified versions from PyPI and GitHub (2026-01-13) |
| **Dependencies** | HIGH | Verified from setup.py in official repository |
| **API Architecture** | HIGH | Official documentation and SDK implementation |
| **Authentication** | HIGH | Documented in official docs, implemented in SDK |
| **TypeScript SDK** | HIGH | Official repository, verified version |
| **Rust SDK** | HIGH | Official repository, crates.io listing |
| **web3.py version** | MEDIUM | Community recommendations, no single official version |
| **Community Go SDKs** | MEDIUM | Packages exist but are unofficial |
| **Rate limits** | MEDIUM | Mentioned in docs but specific limits not fully documented |
| **polymarket-apis** | MEDIUM | Package exists on PyPI but is not official |
| **Pitfalls** | MEDIUM | Based on GitHub issues and community tutorials |

**Overall Stack Confidence: HIGH**

---

## Recommendations Summary

### For Python Developers (Primary Use Case)

**Core Stack:**
```bash
pip install py-clob-client web3==7.12.1 python-dotenv
```

**Use:**
- py-clob-client for all CLOB operations (trading, market data)
- web3.py only for token allowances (EOA wallets)
- Official documentation exclusively
- Environment variables for credentials

**Avoid:**
- polymarket-apis (use official SDK)
- Manual REST implementation
- Unofficial forks
- Creating multiple API keys

### For TypeScript Developers

**Core Stack:**
```bash
npm install @polymarket/clob-client ethers
npm install @polymarket/real-time-data-client  # For RTDS
```

### For Rust Developers

**Core Stack:**
```toml
[dependencies]
polymarket-client-sdk = { version = "0.3", features = ["clob", "ws", "data"] }
```

### Multi-Language Projects

If building tooling that needs multiple languages:
- **Trading logic:** Python (py-clob-client) or TypeScript (clob-client)
- **Real-time data:** TypeScript (real-time-data-client) - no Python equivalent
- **High-performance:** Rust (polymarket-client-sdk)

**Do NOT use:** Community Go SDKs in production (unofficial, not maintained by Polymarket)

---

## Sources

### Official Polymarket Documentation
- [Polymarket Documentation](https://docs.polymarket.com/)
- [CLOB Introduction](https://docs.polymarket.com/developers/CLOB/introduction)
- [Methods Overview](https://docs.polymarket.com/developers/CLOB/clients/methods-overview)
- [Authentication](https://docs.polymarket.com/developers/CLOB/authentication)
- [Quickstart](https://docs.polymarket.com/developers/CLOB/quickstart)
- [WebSocket Overview](https://docs.polymarket.com/developers/CLOB/websocket/wss-overview)
- [RTDS Overview](https://docs.polymarket.com/developers/RTDS/RTDS-overview)
- [Gamma API Overview](https://docs.polymarket.com/developers/gamma-markets-api/overview)
- [API Endpoints](https://docs.polymarket.com/quickstart/reference/endpoints)
- [Changelog](https://docs.polymarket.com/changelog/changelog)

### Official GitHub Repositories
- [py-clob-client](https://github.com/Polymarket/py-clob-client)
- [py-clob-client README](https://github.com/Polymarket/py-clob-client/blob/main/README.md)
- [py-clob-client setup.py](https://github.com/Polymarket/py-clob-client/blob/main/setup.py)
- [py-clob-client requirements.txt](https://github.com/Polymarket/py-clob-client/blob/main/requirements.txt)
- [py-clob-client releases](https://github.com/Polymarket/py-clob-client/releases)
- [clob-client (TypeScript)](https://github.com/Polymarket/clob-client)
- [rs-clob-client (Rust)](https://github.com/Polymarket/rs-clob-client)
- [real-time-data-client](https://github.com/Polymarket/real-time-data-client)
- [go-order-utils](https://github.com/Polymarket/go-order-utils)
- [Polymarket agents](https://github.com/Polymarket/agents)

### PyPI and Package Registries
- [py-clob-client on PyPI](https://pypi.org/project/py-clob-client/)
- [polymarket-apis on PyPI](https://pypi.org/project/polymarket-apis/)

### Community Resources
- [Polymarket Python Tutorial 2025 | PolyTrack](https://www.polytrackhq.app/blog/polymarket-python-tutorial)
- [Polymarket API Guide 2025 | PolyTrack](https://www.polytrackhq.app/blog/polymarket-api-guide)
- [Polymarket WebSocket Tutorial 2025 | PolyTrack](https://www.polytrackhq.app/blog/polymarket-websocket-tutorial)
- [The Polymarket API: Architecture, Endpoints, and Use Cases | Medium](https://medium.com/@gwrx2005/the-polymarket-api-architecture-endpoints-and-use-cases-f1d88fa6c1bf)
- [Polymarket integration | NautilusTrader](https://nautilustrader.io/docs/latest/integrations/polymarket/)

### Third-Party Tools
- [How to Use the Polymarket API | Apidog](https://apidog.com/blog/polymarket-api/)
- [Polymarket API | Bitquery](https://docs.bitquery.io/docs/examples/polymarket-api/)

---

## Next Steps for Documentation Creation

Based on this stack research, the comprehensive Polymarket knowledge package should include:

1. **py-clob-client Quick Reference**
   - Installation and setup
   - Authentication patterns (L1/L2)
   - Common operations (market data, orders, WebSocket)
   - Error handling

2. **REST API Endpoint Catalog**
   - CLOB API endpoints with examples
   - Gamma API endpoints with examples
   - Data API endpoints with examples
   - Authentication requirements per endpoint

3. **WebSocket Integration Guide**
   - CLOB WebSocket channels (market, user)
   - RTDS channels (activity, comments, prices)
   - Subscription patterns
   - Message formats

4. **Edge Cases and Pitfalls**
   - Token allowance requirements
   - Decimal precision limits
   - API key management
   - Proxy wallet configuration
   - Dependency version conflicts

5. **Best Practices**
   - Security (credential management)
   - Testing strategies
   - Rate limit handling
   - Error recovery patterns

**Priority:** Focus on py-clob-client since it's the official Python SDK and handles most complexity automatically.
