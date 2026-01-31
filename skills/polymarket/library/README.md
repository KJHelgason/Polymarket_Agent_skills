# py-clob-client Library Reference

Reference documentation for the official Polymarket Python client library.

## Overview

| Attribute | Value |
|-----------|-------|
| Library | `py-clob-client` |
| Version | 0.34.5+ |
| Repository | [github.com/Polymarket/py-clob-client](https://github.com/Polymarket/py-clob-client) |
| Purpose | Python client for Polymarket CLOB API |
| License | MIT |

## Installation

```bash
pip install py-clob-client
```

**Recommended companion packages:**

```bash
pip install web3 python-dotenv
```

## Quick Start

```python
from py_clob_client.client import ClobClient

# Initialize client
client = ClobClient(
    host="https://clob.polymarket.com",
    key=os.getenv("POLYMARKET_PRIVATE_KEY"),
    chain_id=137,
    signature_type=0,  # 0=EOA, 1=Magic, 2=Safe
    funder=os.getenv("WALLET_ADDRESS")
)

# Set up credentials
creds = client.create_or_derive_api_creds()
client.set_api_creds(creds)

# Ready to trade
print(client.get_ok())
```

See [Client Initialization Guide](../auth/client-initialization.md) for complete setup instructions.

## Library Documentation

### Error Handling

| Document | Description |
|----------|-------------|
| [Error Handling](./error-handling.md) | Exception types, common errors, and recovery patterns |

### Related Guides

| Topic | Location | Description |
|-------|----------|-------------|
| Client Setup | [auth/client-initialization.md](../auth/client-initialization.md) | Complete ClobClient initialization |
| Authentication | [auth/api-credentials.md](../auth/api-credentials.md) | API credential management |
| Order Placement | [trading/order-placement.md](../trading/order-placement.md) | Order creation and submission |
| Edge Cases | [edge-cases/README.md](../edge-cases/README.md) | Common pitfalls and solutions |

## Key Imports

```python
# Client
from py_clob_client.client import ClobClient

# Order types
from py_clob_client.clob_types import OrderArgs, OrderType

# Side constants
from py_clob_client.order_builder.constants import BUY, SELL

# Exceptions
from py_clob_client.exceptions import PolyException, PolyApiException
```

## Navigation

[Back to Polymarket Skills](../README.md) | [Edge Cases](../edge-cases/README.md) | [Trading](../trading/README.md)
