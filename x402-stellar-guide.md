# DO NOT USE - WORKING DOC - Using x402 on Stellar with Freighter Wallet

x402 is an open protocol that embeds payments directly into HTTP using the long-dormant `402 Payment Required` status code. On Stellar, it uses Soroban smart contracts and SEP-41 tokens to enable instant, per-request micropayments — without processing fees and with ~2-second settlement times. This makes it ideal for payment-gated APIs, AI agent services, and content paywalls.

This guide walks through building a payment-protected API server and a browser client that pays using the Freighter wallet.

---

## How It Works

The x402 flow on Stellar has six steps:

```
Client                    Server                   Facilitator (OpenZeppelin)
  │                          │                              │
  │──── GET /api/data ───────▶                              │
  │                          │                              │
  │◀─── 402 + requirements ──│                              │
  │                          │                              │
  │  [Freighter signs        │                              │
  │   Soroban auth entry]    │                              │
  │                          │                              │
  │──── GET /api/data ───────▶                              │
  │     X-PAYMENT: <payload> │──── verify ─────────────────▶
  │                          │◀─── verified ────────────────│
  │                          │──── settle ─────────────────▶
  │                          │     (fee-bump tx submitted)  │
  │◀─── 200 + resource ──────│                              │
```

**Key Stellar-specific details:**

- **No XLM required on the client.** The facilitator sponsors network fees via [fee-bump transactions](https://developers.stellar.org/docs/learn/encyclopedia/transactions-specialized/fee-bump-transactions), so users only need to hold the payment token (e.g., USDC).
- **Soroban authorization entries.** Instead of signing a full transaction, Freighter signs a compact `HashIdPreimageSorobanAuthorization` struct. This is faster and requires less user interaction than a full transaction approval.
- **SEP-41 tokens.** Any token implementing the [SEP-41 interface](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md) (including Stellar Asset Contracts for XLM and USDC) can be used for payment.

---

## Prerequisites

- Node.js 22+ and pnpm 10+
- [Freighter browser extension](https://www.freighter.app/) installed and set to **Testnet**
- A Stellar testnet account funded via [Friendbot](https://laboratory.stellar.org/#account-creator?network=test) with testnet USDC (see [Testnet Assets](#testnet-assets))
- An OpenZeppelin facilitator API key (free — see [Getting a Facilitator Key](#getting-a-facilitator-key))

---

## Getting a Facilitator Key

The [OpenZeppelin Relayer x402 Plugin](https://docs.openzeppelin.com/relayer) acts as the Stellar x402 facilitator. It verifies payment signatures and settles transactions on-chain.

Generate a free testnet API key:

```
https://channels.openzeppelin.com/testnet/gen
```

Save the returned key — you'll use it as `OZ_API_KEY` in your server configuration.

For mainnet, generate a key at `https://channels.openzeppelin.com/gen`.

---

## Project Setup

Clone the official Stellar x402 repository, which includes the server-side paywall library and a working example:

```bash
git clone https://github.com/stellar/x402-stellar.git
cd x402-stellar
pnpm install
pnpm build
```

Or install the packages individually into your own project:

```bash
npm install @x402-stellar/paywall
```

---

## Part 1: Protected API Server

### Basic Express Server

Create `server.ts`:

```typescript
import express from "express";
import { paymentMiddleware } from "@x402-stellar/paywall";

const app = express();
app.use(express.json());

// Configure x402 middleware
app.use(
  paymentMiddleware({
    // Your wallet address receives payments
    payTo: "GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5",

    // Payment amount and token
    amount: "0.10",              // 0.10 USDC per request
    asset: "USDC",

    // OpenZeppelin facilitator (testnet)
    facilitator: {
      url: "https://channels.openzeppelin.com/x402/testnet",
      apiKey: process.env.OZ_API_KEY!,
    },

    // Stellar network
    network: "testnet",
  })
);

// This route is now payment-protected
app.get("/api/data", (req, res) => {
  res.json({
    message: "You paid for this!",
    timestamp: new Date().toISOString(),
  });
});

app.listen(3000, () => {
  console.log("Server running on http://localhost:3000");
});
```

Create a `.env` file:

```
OZ_API_KEY=your_openzeppelin_api_key_here
```

### What the Middleware Does

When a request arrives at `/api/data` without a valid payment:

1. The middleware responds with `HTTP 402` and a `PAYMENT-REQUIRED` header containing a base64-encoded payment requirements object:

```json
{
  "scheme": "exact",
  "network": "stellar:testnet",
  "maxAmountRequired": "0.10",
  "asset": {
    "symbol": "USDC",
    "contractAddress": "CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA"
  },
  "payTo": "GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5",
  "facilitatorUrl": "https://channels.openzeppelin.com/x402/testnet"
}
```

2. When a request arrives with a valid `X-PAYMENT` header, the middleware forwards the payload to the facilitator for verification, then settles the payment on-chain before serving the response.

---

## Part 2: Browser Client with Freighter

The client must:

1. Make the initial request and detect the `402` response
2. Parse the payment requirements
3. Use Freighter to sign a Soroban authorization entry
4. Retry the request with the signed payment payload in the `X-PAYMENT` header

### Install Dependencies

```bash
npm install @stellar/freighter-api @stellar/stellar-sdk
```

### Payment Client

Create `client.ts`:

```typescript
import {
  isConnected,
  getAddress,
  signAuthEntry,
  requestAccess,
} from "@stellar/freighter-api";
import { xdr, hash, Networks } from "@stellar/stellar-sdk";

const FACILITATOR_URL = "https://channels.openzeppelin.com/x402/testnet";
const NETWORK_PASSPHRASE = Networks.TESTNET;

/**
 * Makes an HTTP request, handling x402 payment automatically via Freighter.
 */
export async function fetchWithPayment(url: string): Promise<Response> {
  // Step 1: Make the initial request
  const initialResponse = await fetch(url);

  if (initialResponse.status !== 402) {
    return initialResponse;
  }

  // Step 2: Parse payment requirements from the 402 response
  const paymentRequiredHeader = initialResponse.headers.get("PAYMENT-REQUIRED");
  if (!paymentRequiredHeader) {
    throw new Error("Server returned 402 but no PAYMENT-REQUIRED header found");
  }

  const requirements = JSON.parse(
    Buffer.from(paymentRequiredHeader, "base64").toString("utf-8")
  );

  console.log("Payment required:", requirements);

  // Step 3: Ensure Freighter is connected
  const connected = await isConnected();
  if (!connected.isConnected) {
    throw new Error("Freighter wallet is not installed");
  }

  // Request account access if not already granted
  await requestAccess();
  const { address: userAddress } = await getAddress();

  // Step 4: Fetch a payment authorization entry from the facilitator
  // The facilitator builds the Soroban auth entry that the user needs to sign
  const authEntryResponse = await fetch(`${FACILITATOR_URL}/build-auth-entry`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      paymentRequirements: requirements,
      payer: userAddress,
    }),
  });

  if (!authEntryResponse.ok) {
    throw new Error(`Failed to build auth entry: ${authEntryResponse.statusText}`);
  }

  const { authEntryXdr, expirationLedger } = await authEntryResponse.json();

  // Step 5: Sign the authorization entry with Freighter
  // Freighter signs only the auth entry — not the full transaction.
  // The facilitator will wrap it in a fee-bump transaction and submit it.
  const signedAuthEntry = await signAuthEntry(authEntryXdr, {
    networkPassphrase: NETWORK_PASSPHRASE,
    address: userAddress,
  });

  // Step 6: Build the payment payload
  const paymentPayload = {
    scheme: "exact",
    network: `stellar:testnet`,
    payload: {
      authorization: signedAuthEntry,        // Signed Soroban auth entry
      payer: userAddress,
      paymentRequirements: requirements,
      expirationLedger,
    },
  };

  const paymentHeader = Buffer.from(
    JSON.stringify(paymentPayload)
  ).toString("base64");

  // Step 7: Retry the request with the payment header
  return fetch(url, {
    headers: {
      "X-PAYMENT": paymentHeader,
    },
  });
}
```

### React Component Example

```tsx
import React, { useState } from "react";
import { fetchWithPayment } from "./client";

export function PaywalledContent() {
  const [data, setData] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function loadData() {
    setLoading(true);
    setError(null);

    try {
      const response = await fetchWithPayment("http://localhost:3000/api/data");

      if (!response.ok) {
        throw new Error(`Request failed: ${response.status}`);
      }

      const result = await response.json();
      setData(JSON.stringify(result, null, 2));
    } catch (err) {
      setError(err instanceof Error ? err.message : "Payment failed");
    } finally {
      setLoading(false);
    }
  }

  return (
    <div>
      <button onClick={loadData} disabled={loading}>
        {loading ? "Processing payment..." : "Pay 0.10 USDC to access data"}
      </button>

      {error && <p style={{ color: "red" }}>{error}</p>}

      {data && (
        <pre style={{ background: "#f0f0f0", padding: "1rem" }}>{data}</pre>
      )}
    </div>
  );
}
```

---

## Testnet Assets

To test payments, you need testnet USDC in your Freighter wallet.

### Getting Testnet XLM

1. Open Freighter and switch to **Testnet** in Settings
2. Copy your Testnet public key
3. Go to the [Stellar Friendbot](https://laboratory.stellar.org/#account-creator?network=test) and fund your account

### Adding Testnet USDC

The testnet USDC contract address is:

```
CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA
```

In Freighter:
1. Click **Manage Assets**
2. Click **Add an Asset** → **Add manually**
3. Select **Soroban Token**, enter the contract address above
4. Confirm — the USDC token will appear in your asset list

To receive testnet USDC, use a testnet DEX or faucet, or request it via the [Stellar testnet USDC faucet](https://circle.com/usdc) (if available) or swap from XLM on a testnet DEX.

---

## Running the Example

The `x402-stellar` repository includes a complete working example in `examples/simple-paywall`. Run all three components in separate terminals:

```bash
# Terminal 1 — Facilitator service
cd examples/simple-paywall/facilitator
OZ_API_KEY=your_key pnpm dev

# Terminal 2 — Protected API server
cd examples/simple-paywall/server
pnpm dev

# Terminal 3 — React client
cd examples/simple-paywall/client
pnpm dev
```

Then open `http://localhost:5173` in a browser with Freighter installed and set to Testnet.

---

## How Freighter Signs Auth Entries

When Freighter calls `signAuthEntry`, it signs a `HashIdPreimageSorobanAuthorization` — a compact structure that authorizes a specific contract invocation with a specific amount, to a specific address, valid for a limited number of ledgers.

This is lighter-weight than signing a full transaction:

| | `signTransaction` | `signAuthEntry` |
|---|---|---|
| Signs | Full transaction envelope | Single authorization entry |
| User sees | All transaction details | Specific token transfer auth |
| XLM needed | Yes (for fees) | No (facilitator sponsors fees) |
| Use case | General Stellar transactions | x402 micropayments |

Internally, the signed auth entry is included in a `InvokeHostFunctionOp` operation that the facilitator wraps in a fee-bump transaction and submits to the network.

---

## Supported Wallets

Any Stellar wallet that implements [SEP-43](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0043.md) `signAuthEntry` can be used with x402. Currently supported:

| Wallet | Browser Extension | Mobile |
|---|---|---|
| Freighter | ✅ | ❌ (planned) |
| Albedo | ✅ | — |
| Hana | ✅ | — |
| HOT | ✅ | — |
| Klever | ✅ | — |
| OneKey | ✅ | — |

To support multiple wallets in one app, use the [Stellar Wallets Kit](https://github.com/Creit-Tech/Stellar-Wallets-Kit), which provides a unified API across all SEP-43-compatible wallets.

---

## Going to Production

### 1. Switch to Mainnet Facilitator

```typescript
facilitator: {
  url: "https://channels.openzeppelin.com/x402",   // Mainnet
  apiKey: process.env.OZ_API_KEY!,
},
network: "mainnet",
```

### 2. Use Mainnet USDC

The mainnet USDC Stellar Asset Contract address:

```
CCW67TSZV3SSS2HXMBQ5JFGCKJNXKZM7UQUWUZPUTHXSTZLEO7SJMI75
```

### 3. Security Checklist

- [ ] Store `OZ_API_KEY` in environment variables — never commit it to source control
- [ ] Validate the `payTo` address server-side to prevent redirection attacks
- [ ] Set appropriate `maxAmountRequired` — use the minimum necessary for your use case
- [ ] Add idempotency handling if your server-side action has side effects (use the payment signature as an idempotency key)
- [ ] Monitor your facilitator dashboard for failed settlements

### 4. Replay Attack Prevention

Each Soroban authorization entry includes:
- A **nonce** — unique per signature
- An **expiration ledger** — the signature becomes invalid after this ledger closes (~5 seconds per ledger)

These are enforced by the Soroban runtime, so replay attacks are prevented at the protocol level. Your server does not need to track used payment signatures manually.

---

## Troubleshooting

**Freighter returns "User declined authorization"**
The user rejected the Freighter popup. No payment was made. Handle this gracefully in your UI.

**`402` response but Freighter is not installed**
Check `isConnected()` from `@stellar/freighter-api` before attempting payment. Prompt users to install Freighter if it returns `false`.

**"Auth entry expired" error**
Authorization entries are valid for a fixed number of ledgers (~30 seconds). If network latency is high, the entry may expire before the facilitator submits it. Retry the payment flow from step 4.

**Payment succeeds but server returns 500**
The payment was settled but your server-side handler threw an error. The payment is non-refundable. Use idempotency keys to allow clients to retry safely.

**Facilitator returns "insufficient funds"**
The payer's USDC balance is too low. Ensure the Freighter account has enough USDC for the requested amount plus any fees.

---

## Further Reading

- [x402 on Stellar — Official Docs](https://developers.stellar.org/docs/build/apps/x402)
- [Sign Authorization Entries with Freighter](https://developers.stellar.org/docs/build/guides/freighter/sign-auth-entries)
- [stellar/x402-stellar GitHub repository](https://github.com/stellar/x402-stellar)
- [coinbase/x402 Protocol Specification](https://github.com/coinbase/x402)
- [OpenZeppelin Relayer Documentation](https://docs.openzeppelin.com/relayer/stellar)
- [SEP-41: Token Interface](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)
- [SEP-43: Wallet Interface Standard](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0043.md)
- [Stellar Wallets Kit](https://github.com/Creit-Tech/Stellar-Wallets-Kit)
- [x402.org](https://www.x402.org/)
