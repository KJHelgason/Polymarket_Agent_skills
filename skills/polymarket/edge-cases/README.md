# Edge Cases & Common Pitfalls

Quick reference for diagnosing and resolving Polymarket integration issues. These edge cases cover the most common failure points that cause hours of debugging.

## Quick Symptom-to-Solution Guide

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Polygon shows USDC, Polymarket shows $0.00 | Wrong USDC token (Native vs USDC.e) | [USDC Token Confusion](./usdc-token-confusion.md) |
| FOK order rejected with "invalid amounts" | Precision requirements not met | [Order Constraints](./order-constraints.md#fok-precision) |
| Order rejected with "INVALID_ORDER_MIN_TICK_SIZE" | Tick size changed at price extremes | [Order Constraints](./order-constraints.md#dynamic-tick-size) |
| Order works as GTC but fails as FOK | FOK has stricter precision rules | [Order Constraints](./order-constraints.md#precision-fallback) |
| Orders failing after successful auth | Missing token allowances | [Token Allowances](../auth/token-allowances.md) |

## Edge Case Documents

### Token & Funding Issues

| Document | Issue | When You Need It |
|----------|-------|------------------|
| [USDC Token Confusion](./usdc-token-confusion.md) | Native USDC vs USDC.e | Balance shows on Polygon but not Polymarket |

### Order Placement Issues

| Document | Issue | When You Need It |
|----------|-------|------------------|
| [Order Constraints](./order-constraints.md) | Minimums, precision, tick sizes | Order rejections, "invalid amounts" errors |

### Coming Soon

| Document | Issue | Status |
|----------|-------|--------|
| Price Interpretation | Midpoint vs executable prices | Phase 3, Plan 2 |
| Resolution Mechanics | UMA disputes, redemption delays | Phase 3, Plan 3 |
| NegRisk Trading | Multi-outcome patterns | Phase 3, Plan 3 |
| Partial Fill Tracking | Order reconciliation | Phase 3, Plan 4 |

## Troubleshooting Flow

```
Issue encountered
|
+-- Is it a funding/balance issue?
|   |
|   +-- Check USDC token type: usdc-token-confusion.md
|   +-- Check allowances: ../auth/token-allowances.md
|
+-- Is it an order rejection?
|   |
|   +-- "invalid amounts" --> order-constraints.md (precision)
|   +-- "INVALID_ORDER_MIN_TICK_SIZE" --> order-constraints.md (tick size)
|   +-- "insufficient balance" --> Check USDC.e balance
|   +-- "401 Unauthorized" --> ../auth/api-credentials.md
|
+-- Is it an authentication issue?
    |
    +-- See ../auth/ directory
```

## Prevention Checklist

Before placing orders, verify:

- [ ] USDC.e balance (not Native USDC) - `check_usdc_balances()`
- [ ] Token allowances set - `check_polymarket_allowances()`
- [ ] Current tick size fetched - `client.get_tick_size(token_id)`
- [ ] FOK orders rounded to 2 decimals
- [ ] GTD expiration in seconds (not milliseconds)

## Related Documentation

- [Authentication Setup](../auth/README.md) - Credentials and allowances
- [Order Types](../trading/order-types.md) - GTC, GTD, FOK, FAK selection
- [Trading Overview](../trading/README.md) - Order placement workflows

## Navigation

[Back to Polymarket Skills](../README.md)
