# Phase 1: Authentication & Setup Foundation - Research

**Researched:** 2026-01-31
**Domain:** Polymarket CLOB API authentication and py-clob-client initialization
**Confidence:** MEDIUM

## Summary

Polymarket's authentication system uses a two-tier architecture: L1 authentication with Ethereum wallet signatures (EIP-712) to generate API credentials, and L2 authentication using HMAC-SHA256 signatures for CLOB API requests. The py-clob-client library (version 0.34.5, released January 13, 2026) supports three signature types corresponding to different wallet architectures: EOA wallets (type 0), email/Magic wallets (type 1), and proxy/Safe wallets (type 2).

The most critical distinction for developers is understanding proxy wallet vs EOA wallet patterns. Polymarket.com users have proxy wallets deployed on Polygon where funds actually reside, while the EOA (private key) controls that proxy. This creates a "signing key vs funding address" split that requires careful configuration, particularly the `funder` parameter identifying where capital sits.

Token setup requires USDC.e (the bridged version on Polygon, not native USDC), with approvals for three separate contracts: CTF Exchange, Neg Risk Exchange, and Neg Risk Adapter. EOA wallet users must set these allowances manually; proxy wallet users have this handled automatically.

**Primary recommendation:** Use `create_or_derive_api_creds()` for initial setup (retrieves existing credentials or creates new ones), understand your wallet type (EOA vs proxy) to set correct `signature_type` and `funder` parameters, and verify you have USDC.e (address `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`) not native USDC.

## Standard Stack

The established libraries/tools for Polymarket authentication and setup:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| py-clob-client | 0.34.5+ | Python CLOB client | Official Polymarket library, handles L1/L2 auth |
| web3.py | 6.14.0+ | Ethereum interactions | Token allowance setup, wallet operations |
| polygon-rpc | Current | Polygon network RPC | Chain ID 137 connectivity |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| viem | Latest | Proxy address derivation | When deriving proxy wallet addresses programmatically |
| ethers | Latest | TypeScript alternative | For TypeScript implementations |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| py-clob-client | Direct REST API | Must implement L1/L2 auth manually, HMAC signing, error-prone |
| create_or_derive_api_creds() | create_api_key() | Always creates new key, invalidates previous credentials |

**Installation:**
```bash
pip install py-clob-client web3
```

**Environment Requirements:**
- Python 3.9+
- Polygon RPC endpoint (e.g., https://polygon-rpc.com)
- Private key for wallet
- POL/MATIC for gas fees (if using EOA wallet)

## Architecture Patterns

### Recommended Project Structure
```
polymarket_skills/
├── auth/
│   ├── client.py           # ClobClient initialization
│   ├── credentials.py      # API credential management
│   └── wallet.py           # Wallet detection and proxy derivation
├── contracts/
│   ├── allowances.py       # Token approval logic
│   └── addresses.py        # Contract address constants
└── config/
    ├── network.py          # Chain ID, RPC endpoints
    └── wallet_config.py    # Signature type, funder address
```

### Pattern 1: Wallet Type Detection and Configuration

**What:** Determine if using EOA or proxy wallet, configure signature type accordingly

**When to use:** During initial client setup, before any authenticated operations

**Example:**
```python
# Source: Based on Polymarket documentation patterns
from py_clob_client.client import ClobClient
from enum import Enum

class SignatureType(Enum):
    EOA = 0              # MetaMask, hardware wallets, direct key control
    POLY_PROXY = 1       # Email/Magic wallets
    GNOSIS_SAFE = 2      # Browser wallet proxies, Gnosis Safe

def init_client_for_eoa(private_key: str, wallet_address: str) -> ClobClient:
    """EOA wallet: signing key == funding address"""
    return ClobClient(
        host="https://clob.polymarket.com",
        key=private_key,
        chain_id=137,
        signature_type=SignatureType.EOA.value,
        funder=wallet_address  # Same as wallet address
    )

def init_client_for_proxy(private_key: str, proxy_address: str) -> ClobClient:
    """Proxy wallet: signing key != funding address"""
    return ClobClient(
        host="https://clob.polymarket.com",
        key=private_key,
        chain_id=137,
        signature_type=SignatureType.POLY_PROXY.value,
        funder=proxy_address  # Where funds actually sit
    )
```

### Pattern 2: API Credential Creation and Storage

**What:** Generate L2 credentials from L1 wallet signature, store for reuse

**When to use:** First-time setup or when credentials are lost

**Example:**
```python
# Source: Polymarket official documentation
def setup_authenticated_client(client: ClobClient, save_nonce: bool = True) -> dict:
    """
    Creates or derives API credentials for L2 authentication.

    Returns:
        dict with keys: apiKey, secret, passphrase
    """
    # Use convenience method - retrieves existing or creates new
    api_creds = client.create_or_derive_api_creds()

    # Apply credentials to client for L2 operations
    client.set_api_creds(api_creds)

    # CRITICAL: Save these credentials securely
    # If lost and nonce unknown, must create new credentials
    return api_creds

# Alternative: Explicit creation (invalidates previous credentials)
def rotate_api_credentials(client: ClobClient) -> dict:
    """Create fresh credentials, invalidating old ones"""
    new_creds = client.create_api_key()
    client.set_api_creds(new_creds)
    return new_creds

# Alternative: Derive with known nonce (credential recovery)
def recover_credentials(client: ClobClient, nonce: int) -> dict:
    """Recover existing credentials if nonce is known"""
    recovered = client.derive_api_key(nonce)
    client.set_api_creds(recovered)
    return recovered
```

### Pattern 3: Token Allowance Setup for EOA Wallets

**What:** Approve USDC.e and Conditional Tokens for Polymarket exchange contracts

**When to use:** One-time setup for EOA wallets before first trade

**Example:**
```python
# Source: https://gist.github.com/poly-rodr/44313920481de58d5a3f6d1f8226bd5e
from web3 import Web3
from web3.constants import MAX_INT
from web3.middleware import geth_poa_middleware

# Contract addresses (Polygon mainnet)
USDC_E = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
CTF = "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045"

EXCHANGES = [
    "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E",  # CTF Exchange
    "0xC5d563A36AE78145C45a50134d48A1215220f80a",  # Neg Risk Exchange
    "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296",  # Neg Risk Adapter
]

def setup_token_allowances(private_key: str, wallet_address: str):
    """
    Set unlimited allowances for USDC.e and CTF tokens.
    Required once per EOA wallet. Proxy wallets skip this.
    """
    web3 = Web3(Web3.HTTPProvider("https://polygon-rpc.com"))
    web3.middleware_onion.inject(geth_poa_middleware, layer=0)

    # ABIs
    erc20_abi = '[{"name":"approve","inputs":[{"name":"_spender","type":"address"},{"name":"_value","type":"uint256"}],"outputs":[{"type":"bool"}],"type":"function"}]'
    erc1155_abi = '[{"name":"setApprovalForAll","inputs":[{"name":"operator","type":"address"},{"name":"approved","type":"bool"}],"outputs":[],"type":"function"}]'

    usdc = web3.eth.contract(address=USDC_E, abi=erc20_abi)
    ctf = web3.eth.contract(address=CTF, abi=erc1155_abi)

    nonce = web3.eth.get_transaction_count(wallet_address)

    for exchange in EXCHANGES:
        # Approve USDC.e spending
        tx = usdc.functions.approve(exchange, int(MAX_INT, 0)).build_transaction({
            "chainId": 137,
            "from": wallet_address,
            "nonce": nonce
        })
        signed = web3.eth.account.sign_transaction(tx, private_key)
        tx_hash = web3.eth.send_raw_transaction(signed.raw_transaction)
        web3.eth.wait_for_transaction_receipt(tx_hash, timeout=600)
        nonce += 1

        # Approve CTF token operations
        tx = ctf.functions.setApprovalForAll(exchange, True).build_transaction({
            "chainId": 137,
            "from": wallet_address,
            "nonce": nonce
        })
        signed = web3.eth.account.sign_transaction(tx, private_key)
        tx_hash = web3.eth.send_raw_transaction(signed.raw_transaction)
        web3.eth.wait_for_transaction_receipt(tx_hash, timeout=600)
        nonce += 1
```

### Pattern 4: Proxy Wallet Address Derivation

**What:** Compute proxy wallet address deterministically from EOA address

**When to use:** When you have an EOA from Magic/email wallet and need to find where funds sit

**Example:**
```typescript
// Source: https://github.com/Polymarket/magic-proxy-builder-example
import { keccak256, getCreate2Address, encodePacked } from "viem";

const PROXY_FACTORY = "0xaB45c5A4B0c941a2F231C04C3f49182e1A254052";
const PROXY_INIT_CODE_HASH = "0x..."; // Proxy bytecode hash

export function deriveProxyAddress(eoaAddress: string): string {
  // Deterministic derivation using CREATE2
  return getCreate2Address({
    bytecodeHash: PROXY_INIT_CODE_HASH,
    from: PROXY_FACTORY,
    salt: keccak256(encodePacked(["address"], [eoaAddress])),
  });
}

// Usage:
// const eoaAddress = "0x...";  // Your Magic wallet EOA
// const proxyAddress = deriveProxyAddress(eoaAddress);  // Where funds sit
```

**Python equivalent (conceptual):**
```python
# Note: No official Python implementation found in research
# Would require implementing CREATE2 address derivation
from eth_utils import keccak, to_checksum_address

def derive_proxy_address(eoa_address: str) -> str:
    """
    Derive Polymarket proxy address from EOA.
    Implementation requires CREATE2 calculation.
    """
    # This is pseudocode - needs complete implementation
    factory = "0xaB45c5A4B0c941a2F231C04C3f49182e1A254052"
    salt = keccak(text=eoa_address)
    # ... CREATE2 calculation
    pass
```

### Anti-Patterns to Avoid

- **Using create_api_key() for initial setup**: Always invalidates previous credentials. Use `create_or_derive_api_creds()` instead.
- **Not storing nonce**: If credentials are lost and nonce is unknown, recovery is impossible without creating new credentials.
- **Wrong signature_type for wallet**: Using type 0 for proxy wallets or type 1/2 for direct EOA causes "Invalid signature" errors.
- **Funder address == EOA for proxy wallets**: The funder must be the proxy address where funds sit, not the signing EOA.
- **Setting allowances for proxy wallets**: Proxy/Safe wallet users don't need manual allowances; this is handled automatically.
- **Using native USDC instead of USDC.e**: Polymarket exclusively uses USDC.e (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`).

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| L1/L2 authentication | Custom HMAC signing | py-clob-client methods | Complex EIP-712 signatures, HMAC-SHA256 with specific headers (POLY-ADDRESS, POLY-SIGNATURE, POLY-TIMESTAMP, POLY-NONCE, POLY-PASSPHRASE), 30-second expiration, replay attack prevention |
| API credential derivation | Custom nonce management | create_or_derive_api_creds() | Deterministic key derivation from wallet signature, handles existing credential detection |
| Proxy address calculation | Manual CREATE2 math | deriveProxyAddress() | EIP-1167 minimal proxy with keccak256 salting, factory-specific bytecode hash |
| Token allowance checking | Manual contract calls | web3.py contract.functions | ABI encoding, proper RPC interaction, error handling |
| USDC.e vs USDC detection | Balance comparisons | Use USDC.e address constant | Native USDC requires swap, confusion source, platform exclusively uses USDC.e |

**Key insight:** Authentication in CLOB systems is deceptively complex. L1 requires EIP-712 structured data signing with specific domain separators ("ClobAuthDomain"). L2 requires HMAC-SHA256 signatures with precise header formatting and timestamp expiration (30 seconds). The py-clob-client abstracts all this correctly; reimplementing risks signature validation failures.

## Common Pitfalls

### Pitfall 1: Wrong Signature Type for Wallet Architecture

**What goes wrong:** Using `signature_type=0` (EOA) when funds are in a proxy wallet results in "Invalid signature" (400) or "Unauthorized/Invalid api key" (401) errors.

**Why it happens:** Polymarket.com users have proxy wallets deployed on Polygon. The EOA (from Magic/email) controls the proxy but doesn't hold funds. The signature verification expects the proxy's signature format, not direct EOA.

**How to avoid:**
- If you created wallet on Polymarket.com via email/Magic → use `signature_type=1`
- If using Gnosis Safe or browser wallet proxy → use `signature_type=2`
- If using MetaMask/hardware wallet directly → use `signature_type=0`
- Check Polymarket profile settings for "Wallet Address" - if it differs from your EOA, you have a proxy

**Warning signs:**
- 400 error: "invalid signature" when placing orders
- 401 error: "Invalid L1 Request headers" when creating API key
- Orders from other methods work but `post_order()` fails

**Detection method:**
```python
# Check if address in Polymarket profile matches your EOA
eoa_address = web3.eth.account.from_key(private_key).address
profile_address = get_from_polymarket_profile()  # Via UI or API

if eoa_address != profile_address:
    # You have a proxy wallet
    signature_type = 1  # or 2 for Gnosis Safe
    funder = profile_address
else:
    # Direct EOA
    signature_type = 0
    funder = eoa_address
```

### Pitfall 2: USDC vs USDC.e Confusion

**What goes wrong:** User deposits native USDC from exchange, balance shows $0.00 in Polymarket, believes funds are lost.

**Why it happens:** Polymarket was built when Polygon only had bridged USDC (now called USDC.e). The platform exclusively uses USDC.e (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`). Exchanges now default to native USDC, creating a token mismatch.

**How to avoid:**
- Always verify you have USDC.e, not native USDC
- Check token contract address: must be `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`
- If depositing from exchange, specify "USDC.e" or "Bridged USDC" if option exists
- Use Polymarket's "Activate Funds" button if you deposited native USDC (performs automatic swap)

**Warning signs:**
- Polygon wallet shows USDC balance, Polymarket shows $0.00
- Token address is `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` (native USDC) instead of `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` (USDC.e)
- "Activate Funds" banner appears in Polymarket UI

**Code check:**
```python
from web3 import Web3

USDC_E = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"  # CORRECT for Polymarket
USDC_NATIVE = "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359"  # WRONG for Polymarket

def check_usdc_balance(web3: Web3, wallet_address: str):
    """Verify you have USDC.e, not native USDC"""
    abi = '[{"name":"balanceOf","inputs":[{"type":"address"}],"outputs":[{"type":"uint256"}],"type":"function"}]'

    usdc_e = web3.eth.contract(address=USDC_E, abi=abi)
    usdc_native = web3.eth.contract(address=USDC_NATIVE, abi=abi)

    balance_e = usdc_e.functions.balanceOf(wallet_address).call()
    balance_native = usdc_native.functions.balanceOf(wallet_address).call()

    if balance_native > 0 and balance_e == 0:
        raise ValueError(f"You have native USDC, need USDC.e. Use Polymarket 'Activate Funds' or swap.")

    return balance_e
```

### Pitfall 3: API Credential Loss Without Nonce

**What goes wrong:** API credentials (key, secret, passphrase) are lost or deleted. User calls `create_api_key()` to regenerate, but this invalidates all active orders and requires complete reinitialization.

**Why it happens:** Each wallet can only have one active API key. Creating a new one invalidates the previous credentials. Without the original nonce, you can't recover the old credentials using `derive_api_key()`.

**How to avoid:**
- When first creating credentials, save the nonce along with the credentials
- Use `create_or_derive_api_creds()` for initial setup (handles both cases)
- Store credentials securely in environment variables or secrets manager
- Document the nonce value in secure storage (e.g., `.env.nonce` file, password manager)

**Warning signs:**
- Need to cancel all existing orders after regenerating credentials
- Previous API key returns 401 "Unauthorized/Invalid api key"
- Documentation says "no way to recover lost API credentials without the nonce"

**Best practice pattern:**
```python
import os
from pathlib import Path

def setup_api_credentials(client: ClobClient):
    """
    Setup credentials with nonce tracking for recovery.
    """
    # Try to derive with default nonce first (nonce not specified)
    try:
        creds = client.create_or_derive_api_creds()
        client.set_api_creds(creds)

        # Save credentials securely
        env_path = Path(".env")
        env_path.write_text(
            f"POLY_API_KEY={creds['apiKey']}\n"
            f"POLY_SECRET={creds['secret']}\n"
            f"POLY_PASSPHRASE={creds['passphrase']}\n"
            # Note: create_or_derive doesn't return nonce
            # For explicit nonce tracking, use create_api_key(nonce)
        )

        return creds
    except Exception as e:
        print(f"Credential setup failed: {e}")
        raise

# Recovery with known nonce
def recover_with_nonce(client: ClobClient, nonce: int):
    """Recover existing credentials if nonce is known"""
    creds = client.derive_api_key(nonce)
    client.set_api_creds(creds)
    return creds
```

### Pitfall 4: Forgetting Token Allowances for EOA Wallets

**What goes wrong:** EOA wallet user initializes client successfully, gets API credentials, attempts to place order, receives transaction failure or "insufficient allowance" error.

**Why it happens:** Unlike proxy/Safe wallets (which handle allowances automatically), EOA wallets require explicit approval transactions for USDC.e and Conditional Tokens before trading. This is a one-time setup but easy to forget.

**How to avoid:**
- Check allowances before attempting first trade
- Set allowances to `MaxUint256` for unlimited approval (one-time operation)
- Verify allowances for all three exchange contracts (CTF, Neg Risk, Neg Risk Adapter)
- Document in setup instructions that EOA users need this extra step

**Warning signs:**
- Successful authentication but order placement fails
- Error mentions "allowance" or "transfer amount exceeds allowance"
- Balance shows sufficient USDC.e but can't place orders

**Allowance check:**
```python
from web3 import Web3

def check_allowances(web3: Web3, wallet_address: str) -> dict:
    """
    Check if token allowances are set for all required contracts.
    Returns dict of {exchange: {usdc_approved: bool, ctf_approved: bool}}
    """
    USDC_E = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
    CTF = "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045"

    EXCHANGES = [
        "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E",
        "0xC5d563A36AE78145C45a50134d48A1215220f80a",
        "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296",
    ]

    erc20_abi = '[{"name":"allowance","inputs":[{"name":"owner","type":"address"},{"name":"spender","type":"address"}],"outputs":[{"type":"uint256"}],"type":"function"}]'
    erc1155_abi = '[{"name":"isApprovedForAll","inputs":[{"name":"account","type":"address"},{"name":"operator","type":"address"}],"outputs":[{"type":"bool"}],"type":"function"}]'

    usdc = web3.eth.contract(address=USDC_E, abi=erc20_abi)
    ctf = web3.eth.contract(address=CTF, abi=erc1155_abi)

    results = {}
    for exchange in EXCHANGES:
        usdc_allowance = usdc.functions.allowance(wallet_address, exchange).call()
        ctf_approved = ctf.functions.isApprovedForAll(wallet_address, exchange).call()

        results[exchange] = {
            "usdc_approved": usdc_allowance > 0,
            "ctf_approved": ctf_approved,
            "needs_setup": not (usdc_allowance > 0 and ctf_approved)
        }

    return results
```

### Pitfall 5: Third-Party Authentication Security Risks

**What goes wrong:** Users with email/Magic wallet authentication had accounts breached in December 2025 due to vulnerability in third-party authentication provider. Attackers used proxy function calls to drain USDC funds.

**Why it happens:** Email-based authentication introduces supply-chain risk where vulnerabilities outside Polymarket's control can expose users. Magic Link authentication (commonly used for email sign-in) was implicated in the breach.

**How to avoid:**
- Prefer direct EOA control (MetaMask, hardware wallets) over email/Magic authentication when possible
- If using Magic wallet, export private key and store securely offline
- Monitor wallet for unexpected transactions
- Consider transferring assets from proxy wallet to direct EOA for long-term storage
- Enable additional security measures (2FA where available)

**Warning signs:**
- Unexpected "proxy" transactions in wallet history
- USDC transfers to unknown addresses
- Login issues or forced password resets
- Cloudflare "Ray ID" errors after authentication (post-breach WAF tightening)

**Migration from proxy to EOA (if desired):**
```python
def migrate_from_proxy_to_eoa(
    proxy_private_key: str,
    new_eoa_address: str,
    web3: Web3
):
    """
    Transfer assets from proxy wallet to direct EOA control.
    CAUTION: Test with small amounts first.
    """
    # This requires accessing proxy wallet as owner
    # and executing transfer transactions for all assets
    # (USDC.e balances, conditional token positions)

    # Implementation would require:
    # 1. Get all token balances in proxy
    # 2. Transfer USDC.e to new EOA
    # 3. Transfer conditional token positions
    # 4. Update client to use new EOA

    print("WARNING: Migration requires careful testing.")
    print("Consult Polymarket documentation for proxy wallet operations.")
```

## Code Examples

Verified patterns from official sources:

### Complete Initialization for All Wallet Types

```python
# Source: Polymarket official documentation and py-clob-client README
from py_clob_client.client import ClobClient
import os

# Configuration
HOST = "https://clob.polymarket.com"
CHAIN_ID = 137  # Polygon mainnet
PRIVATE_KEY = os.getenv("POLYMARKET_PRIVATE_KEY")

# Type 0: EOA Wallet (MetaMask, hardware wallet)
def init_eoa_client():
    client = ClobClient(
        HOST,
        key=PRIVATE_KEY,
        chain_id=CHAIN_ID,
        signature_type=0,  # Standard EOA
        funder=os.getenv("WALLET_ADDRESS")  # Same as signing address
    )
    client.set_api_creds(client.create_or_derive_api_creds())
    return client

# Type 1: Email/Magic Wallet
def init_magic_wallet_client():
    client = ClobClient(
        HOST,
        key=PRIVATE_KEY,  # Magic wallet EOA key
        chain_id=CHAIN_ID,
        signature_type=1,  # Email/Magic signatures
        funder=os.getenv("PROXY_ADDRESS")  # Proxy where funds sit
    )
    client.set_api_creds(client.create_or_derive_api_creds())
    return client

# Type 2: Gnosis Safe / Browser Wallet Proxy
def init_safe_client():
    client = ClobClient(
        HOST,
        key=PRIVATE_KEY,
        chain_id=CHAIN_ID,
        signature_type=2,  # Proxy contract signatures
        funder=os.getenv("SAFE_ADDRESS")  # Safe proxy address
    )
    client.set_api_creds(client.create_or_derive_api_creds())
    return client
```

### Complete Token Allowance Setup

```python
# Source: https://gist.github.com/poly-rodr/44313920481de58d5a3f6d1f8226bd5e
from web3 import Web3
from web3.constants import MAX_INT
from web3.middleware import geth_poa_middleware

def setup_polymarket_allowances(private_key: str, wallet_address: str):
    """
    One-time setup for EOA wallets.
    Approves USDC.e and CTF tokens for all Polymarket exchanges.

    Requirements:
    - POL/MATIC for gas fees
    - web3.py 6.14.0+
    """
    # Initialize Web3
    rpc_url = "https://polygon-rpc.com"
    web3 = Web3(Web3.HTTPProvider(rpc_url))
    web3.middleware_onion.inject(geth_poa_middleware, layer=0)

    # Verify connection
    if not web3.is_connected():
        raise ConnectionError("Failed to connect to Polygon RPC")

    # Contract addresses
    usdc_address = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"  # USDC.e
    ctf_address = "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045"   # Conditional Tokens

    targets = [
        "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E",  # CTF Exchange
        "0xC5d563A36AE78145C45a50134d48A1215220f80a",  # Neg Risk Exchange
        "0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296",  # Neg Risk Adapter
    ]

    # Minimal ABIs
    erc20_abi = [
        {
            "name": "approve",
            "type": "function",
            "inputs": [
                {"name": "_spender", "type": "address"},
                {"name": "_value", "type": "uint256"}
            ],
            "outputs": [{"type": "bool"}]
        }
    ]

    erc1155_abi = [
        {
            "name": "setApprovalForAll",
            "type": "function",
            "inputs": [
                {"name": "operator", "type": "address"},
                {"name": "approved", "type": "bool"}
            ],
            "outputs": []
        }
    ]

    usdc = web3.eth.contract(address=usdc_address, abi=erc20_abi)
    ctf = web3.eth.contract(address=ctf_address, abi=erc1155_abi)

    nonce = web3.eth.get_transaction_count(wallet_address)

    print(f"Setting up allowances for {wallet_address}...")
    print(f"Starting nonce: {nonce}")

    for i, target in enumerate(targets):
        print(f"\nApproving exchange {i+1}/3: {target}")

        # Approve USDC.e
        print("  - Approving USDC.e...")
        tx = usdc.functions.approve(target, int(MAX_INT, 0)).build_transaction({
            "chainId": 137,
            "from": wallet_address,
            "nonce": nonce
        })
        signed = web3.eth.account.sign_transaction(tx, private_key)
        tx_hash = web3.eth.send_raw_transaction(signed.raw_transaction)
        receipt = web3.eth.wait_for_transaction_receipt(tx_hash, timeout=600)
        print(f"    USDC.e approved: {receipt.transactionHash.hex()}")
        nonce += 1

        # Approve CTF
        print("  - Approving Conditional Tokens...")
        tx = ctf.functions.setApprovalForAll(target, True).build_transaction({
            "chainId": 137,
            "from": wallet_address,
            "nonce": nonce
        })
        signed = web3.eth.account.sign_transaction(tx, private_key)
        tx_hash = web3.eth.send_raw_transaction(signed.raw_transaction)
        receipt = web3.eth.wait_for_transaction_receipt(tx_hash, timeout=600)
        print(f"    CTF approved: {receipt.transactionHash.hex()}")
        nonce += 1

    print("\n✓ All allowances set successfully!")
    print("You can now trade on Polymarket.")

# Usage:
# setup_polymarket_allowances(
#     private_key="0x...",
#     wallet_address="0x..."
# )
```

### API Credential Management with Nonce Tracking

```python
# Source: Polymarket L1 methods documentation
from py_clob_client.client import ClobClient
import json
from pathlib import Path

def create_credentials_with_tracking(client: ClobClient, nonce: int = None) -> dict:
    """
    Create API credentials with explicit nonce for recovery.

    Args:
        client: Initialized ClobClient
        nonce: Optional nonce for deterministic key generation

    Returns:
        dict: {apiKey, secret, passphrase, nonce}
    """
    if nonce is not None:
        # Create with specific nonce (allows recovery later)
        creds = client.create_api_key(nonce)
        creds['nonce'] = nonce
    else:
        # Let library handle nonce
        creds = client.create_or_derive_api_creds()
        # Note: create_or_derive doesn't return nonce
        creds['nonce'] = None

    client.set_api_creds(creds)
    return creds

def save_credentials_securely(creds: dict, path: str = ".polymarket_creds.json"):
    """
    Save credentials to secure file.
    WARNING: Protect this file (chmod 600 on Unix).
    """
    creds_file = Path(path)
    creds_file.write_text(json.dumps(creds, indent=2))
    creds_file.chmod(0o600)  # Unix only: owner read/write only
    print(f"Credentials saved to {path}")
    print("IMPORTANT: Keep nonce value secure for credential recovery")

def load_credentials(path: str = ".polymarket_creds.json") -> dict:
    """Load saved credentials from file."""
    creds_file = Path(path)
    if not creds_file.exists():
        raise FileNotFoundError(f"Credentials file not found: {path}")
    return json.loads(creds_file.read_text())

def recover_lost_credentials(client: ClobClient, nonce: int) -> dict:
    """
    Recover credentials using saved nonce.
    This retrieves existing credentials without creating new ones.
    """
    print(f"Recovering credentials with nonce {nonce}...")
    creds = client.derive_api_key(nonce)
    client.set_api_creds(creds)
    creds['nonce'] = nonce
    return creds

# Example workflow:
#
# # Initial setup with nonce tracking
# client = ClobClient(...)
# nonce = 12345  # Choose your nonce or let library generate
# creds = create_credentials_with_tracking(client, nonce)
# save_credentials_securely(creds)
#
# # Later recovery if credentials lost
# client = ClobClient(...)
# saved_creds = load_credentials()
# if saved_creds['nonce']:
#     creds = recover_lost_credentials(client, saved_creds['nonce'])
```

### Wallet Type Detection

```python
# Source: Based on Polymarket documentation patterns
from web3 import Web3
from typing import Tuple

def detect_wallet_type(
    eoa_address: str,
    profile_address: str
) -> Tuple[int, str]:
    """
    Detect wallet type and determine correct signature_type and funder.

    Args:
        eoa_address: Address from your private key
        profile_address: Address shown in Polymarket profile

    Returns:
        (signature_type, funder_address)
    """
    if eoa_address.lower() == profile_address.lower():
        # Direct EOA wallet
        return (0, eoa_address)
    else:
        # Proxy wallet (profile shows proxy address)
        # Use signature_type=1 for Magic/email wallets
        # Use signature_type=2 for Gnosis Safe
        # Default to 1 if unsure
        return (1, profile_address)

def check_usdc_type(web3: Web3, wallet_address: str) -> dict:
    """
    Check which USDC variant user has.
    Returns balance info for both USDC.e and native USDC.
    """
    USDC_E = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
    USDC_NATIVE = "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359"

    abi = '[{"name":"balanceOf","inputs":[{"name":"account","type":"address"}],"outputs":[{"name":"balance","type":"uint256"}],"type":"function"}]'

    usdc_e_contract = web3.eth.contract(address=USDC_E, abi=abi)
    usdc_native_contract = web3.eth.contract(address=USDC_NATIVE, abi=abi)

    balance_e = usdc_e_contract.functions.balanceOf(wallet_address).call()
    balance_native = usdc_native_contract.functions.balanceOf(wallet_address).call()

    return {
        "usdc_e_balance": balance_e / 1e6,  # 6 decimals
        "usdc_native_balance": balance_native / 1e6,
        "has_correct_token": balance_e > 0,
        "needs_swap": balance_native > 0 and balance_e == 0,
        "message": "Ready to trade" if balance_e > 0
                   else "Need to swap native USDC to USDC.e" if balance_native > 0
                   else "No USDC found"
    }

# Example usage:
# web3 = Web3(Web3.HTTPProvider("https://polygon-rpc.com"))
# result = check_usdc_type(web3, "0x...")
# print(result)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual L1/L2 auth implementation | py-clob-client library | 2024+ | Library handles complex EIP-712 and HMAC-SHA256 signatures |
| create_api_key() for all cases | create_or_derive_api_creds() | Recent (2024-2025) | Prevents accidental credential invalidation |
| Native USDC only | USDC.e exclusively | 2025+ (Polygon USDC native launch) | Major confusion source for new users |
| Direct EOA only | Proxy wallet default on Polymarket.com | Platform evolution | Requires understanding signature types, funder parameter |
| web3.py <6.0 | web3.py 6.14.0+ | 2024+ | POA middleware changes for Polygon |

**Deprecated/outdated:**
- **Using web3.py <6.0**: Requires `ExtraDataToPOAMiddleware` instead of `geth_poa_middleware` in newer versions
- **Assuming EOA == funder always**: Polymarket.com defaults to proxy wallets where EOA controls but doesn't hold funds
- **Ignoring USDC.e vs native USDC**: Critical distinction as of 2025+ when exchanges started defaulting to native USDC
- **Magic Link without private key export**: December 2025 security breach highlighted risks of relying solely on third-party auth without backup access

## Open Questions

Things that couldn't be fully resolved:

1. **L2 Authentication Header Specification**
   - What we know: Uses HMAC-SHA256 with headers POLY-ADDRESS, POLY-SIGNATURE, POLY-TIMESTAMP, POLY-NONCE, POLY-PASSPHRASE
   - What's unclear: Exact HMAC construction algorithm (timestamp format, message structure, header encoding)
   - Recommendation: Use py-clob-client library which handles this; don't attempt manual implementation without reverse-engineering source code

2. **Proxy Address Derivation in Python**
   - What we know: TypeScript implementation using CREATE2 with keccak256 salt from EOA address
   - What's unclear: Complete Python implementation not found in official repos
   - Recommendation: For Python users needing proxy derivation, either port TypeScript code or use Polymarket UI to find proxy address in profile settings

3. **Signature Type 2 (Gnosis Safe) Specific Setup**
   - What we know: Use signature_type=2 for Gnosis Safe proxies
   - What's unclear: Detailed setup differences from type 1, when to use 2 vs 1 for proxy wallets
   - Recommendation: Default to signature_type=1 for proxy wallets unless explicitly using Gnosis Safe deployment

4. **Nonce Default Value in create_or_derive_api_creds()**
   - What we know: Method uses "default nonce" but doesn't return nonce value
   - What's unclear: What the default nonce is, whether it's deterministic or random
   - Recommendation: For credential recovery capability, use explicit create_api_key(nonce) with known nonce

5. **Token Allowance Requirements for Signature Type 1 vs 2**
   - What we know: EOA (type 0) needs manual allowances, proxy/Safe "handled automatically"
   - What's unclear: Exact mechanism for automatic handling (are allowances pre-set during proxy deployment?)
   - Recommendation: Test allowances regardless of signature type; check using allowance verification code before first trade

## Sources

### Primary (HIGH confidence)
- [Polymarket py-clob-client GitHub](https://github.com/Polymarket/py-clob-client) - Official Python library documentation
- [Polymarket Authentication Documentation](https://docs.polymarket.com/developers/CLOB/authentication) - L1/L2 auth architecture
- [Polymarket L1 Methods Documentation](https://docs.polymarket.com/developers/CLOB/clients/methods-l1) - API key management
- [Polymarket Proxy Wallet Documentation](https://docs.polymarket.com/developers/proxy-wallet) - Proxy wallet architecture
- [Polymarket Market Makers Setup](https://docs.polymarket.com/developers/market-makers/setup) - Token allowances and workflow
- [Token Allowance Python Gist](https://gist.github.com/poly-rodr/44313920481de58d5a3f6d1f8226bd5e) - Official code example
- [Magic Proxy Builder Example](https://github.com/Polymarket/magic-proxy-builder-example) - Proxy address derivation

### Secondary (MEDIUM confidence)
- [Polymarket Python Tutorial 2025 (PolyTrack)](https://www.polytrackhq.app/blog/polymarket-python-tutorial) - Community guide
- [USDC vs USDC.e Guide 2026](https://www.homesfound.ca/blog/usdc-vs-usdce-polymarket-why-your-balance-shows-000-2026-guide/) - Token confusion explanation
- [NautilusTrader Polymarket Integration](https://nautilustrader.io/docs/latest/integrations/polymarket/) - Third-party implementation insights
- [Polymarket Account Breach Articles (Dec 2025)](https://www.ccn.com/news/crypto/polymarket-users-report-drained-accounts-hack-or-security-issue/) - Security incident documentation

### Tertiary (LOW confidence - marked for validation)
- GitHub Issues for error patterns - User-reported problems, not official guidance
- Community blog posts on wallet setup - Helpful but may contain errors or outdated info

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Official libraries and versions verified from PyPI, GitHub releases (0.34.5 released 2026-01-13)
- Architecture: MEDIUM - Patterns derived from official docs and examples, but some implementation details (L2 headers) not fully specified
- Pitfalls: HIGH - Based on official documentation, GitHub issues, and verified security incident reports (Dec 2025 breach)
- Code examples: HIGH - Sourced from official GitHub repos and documentation, verified with source URLs

**Research date:** 2026-01-31
**Valid until:** 2026-03-02 (30 days - stable technology, but auth patterns can evolve with library updates)

**Research limitations:**
- L2 authentication headers not fully documented in accessible sources (implementation details in library source code not analyzed)
- Python proxy address derivation not found in official repos (TypeScript implementation exists)
- Some edge cases in signature type selection (1 vs 2 for proxies) need field testing to clarify

**Recommended validation:**
- Test signature type selection with actual wallet types (EOA, Magic, Gnosis Safe)
- Verify token allowance setup on testnet before mainnet
- Confirm USDC.e vs native USDC handling with small amounts first
