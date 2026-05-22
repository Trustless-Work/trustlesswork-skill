# Core Concepts

## Overview

Trustless Work is **Escrow-as-a-Service (EaaS)** for stablecoin escrow. It enables trust-minimized conditional payments on Stellar blockchain using Soroban smart contracts. Build **non-custodial** flows with milestones, approvals, and disputes. Ideal for freelancing, marketplaces, grant disbursements, and any milestone-based payment flow.

## Escrow Lifecycle

### Single-Release Escrow Flow

1. **Deploy**: Initialize escrow with roles, milestones, and configuration
2. **Fund**: Lock funds (escrow amount + platform fee) in escrow account
3. **Update Milestone Status**: Service provider marks milestone(s) as complete, adds evidence
4. **Approve**: Approver verifies and approves milestone(s)
5. **Release**: Release Signer releases all funds at once to Receiver
6. **Dispute** (optional): Service Provider, Approver, or Release Signer can raise a dispute
7. **Resolve**: Dispute Resolver decides how to distribute funds

### Multi-Release Escrow Flow

1. **Deploy**: Initialize escrow with roles and milestones (each with its own amount and receiver)
2. **Fund**: Lock total funds (sum of all milestone amounts + platform fee)
3. **Update Milestone Status**: Service provider marks a milestone as complete
4. **Approve**: Approver verifies and approves milestone
5. **Release Milestone**: Release Signer releases funds for that specific milestone
6. **Repeat**: Steps 3-5 for each milestone
7. **Withdraw Remaining**: Dispute Resolver can withdraw remaining funds after completion

## Key Roles

### Core Roles

| Role | Responsibility |
|------|---------------|
| **Service Provider** | Delivers work, updates milestone status, adds evidence |
| **Approver** | Validates completion, approves milestones, can raise disputes |
| **Release Signer** | Executes fund releases after approvals, can raise disputes |
| **Receiver** | Final recipient of released funds (defaults to Service Provider) |
| **Dispute Resolver** | Resolves disputes by redirecting funds |
| **Platform Address** | Receives platform fees; can update escrow before funding |

### Role Capability Matrix

| Role | Update status | Approve | Raise dispute | Resolve | Release | Receive payout | Receive fee |
|------|-------------|---------|--------------|---------|---------|---------------|------------|
| Service Provider | Yes | No | Yes | No | No | Sometimes | No |
| Approver | No | Yes | Yes | No | No | Usually no | No |
| Release Signer | No | No | Yes | No | Yes | Usually no | No |
| Receiver | No | No | No | No | No | Yes | No |
| Dispute Resolver | No | Case-specific | No | Yes | Case-specific | No | No |
| Platform Address | No (before funding: Yes) | No | No | No | No | No | Yes |

### Important Distinctions

- **Status update** = communicates progress (Service Provider)
- **Approval** = validates completion (Approver)
- **Release** = executes payment movement (Release Signer)
- The same address can hold multiple roles (e.g., Approver + Release Signer), but avoid Service Provider + Approver (can approve own work)

## Escrow Flags

Status tracked via boolean flags:

- **approved**: Milestone(s) approved for release
- **disputed**: Escrow is in dispute
- **released**: Funds have been released
- **resolved**: Dispute has been resolved

## API Authentication

All API requests require an API key header:

```
x-api-key: YOUR_API_KEY
```

### Getting an API Key

1. Connect wallet to https://dapp.trustlesswork.com
2. Click wallet address (bottom left)
3. Go to Settings → API Keys tab
4. Complete profile (name, email, use case — **required**)
5. Choose network (Testnet or Mainnet) and generate API key
6. **Copy immediately** — it cannot be viewed again after closing the dialog

## Base URLs

```
Mainnet:  https://api.trustlesswork.com
Testnet:  https://dev.api.trustlesswork.com
```

**Swagger UI:**
- Mainnet: `https://api.trustlesswork.com/docs`
- Testnet: `https://dev.api.trustlesswork.com/docs`

## Rate Limits

**50 requests per 60 seconds** per client.

## Fees

Mainnet charges a **0.3% protocol fee** on top of your platform fee.

## Common Error Types

```typescript
enum ApiErrorTypes {
  NOT_FOUND = "NOT_FOUND",
  UNAUTHORIZED = "UNAUTHORIZED",
  UNKNOWN_ERROR = "UNKNOWN_ERROR",
  WALLET_ERROR = "WALLET_ERROR",
}
```

## HTTP Status Codes

- **200/201**: Success
- **400**: Bad request (missing/invalid parameters)
- **401**: Unauthorized (invalid/missing API key)
- **429**: Too many requests (rate limiting)
- **500**: Server error (escrow not found, unexpected errors)

## Transaction Pattern

All escrow write operations follow this pattern:

1. **Call API endpoint** → Returns unsigned XDR transaction
2. **Sign transaction** → Use wallet to sign XDR with the correct role wallet
3. **Submit transaction** → POST to `/helper/send-transaction` with signed XDR
4. **Verify on-chain** → Query escrow with `validateOnChain=true`

### Example Transaction Flow

```typescript
// 1. Get unsigned transaction
const response = await fetch('https://api.trustlesswork.com/deployer/single-release', {
  method: 'POST',
  headers: {
    'x-api-key': apiKey,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(deployPayload)
});

const { unsignedTransaction } = await response.json();

// 2. Sign with wallet
const { signedTxXdr } = await signTransaction(unsignedTransaction, {
  address,
  networkPassphrase: WalletNetwork.TESTNET,
});

// 3. Submit transaction
const submitResponse = await fetch('https://api.trustlesswork.com/helper/send-transaction', {
  method: 'POST',
  headers: {
    'x-api-key': apiKey,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ signedXdr: signedTxXdr })
});

// 4. Verify on-chain
const verifyResponse = await fetch(
  `https://api.trustlesswork.com/helper/get-escrows-by-signer?signer=${signerAddress}&validateOnChain=true`,
  { headers: { 'x-api-key': apiKey } }
);
```

## Best Practices

### Security

1. **Never expose API keys** in client-side code or public repos
2. **Use environment variables**: `NEXT_PUBLIC_API_KEY` for frontend (read-only acceptable), server-side for write flows
3. **Validate on-chain** when displaying escrow data (`validateOnChain=true`)
4. **Verify transaction signatures** before submitting
5. **Handle errors gracefully** with user-friendly messages

### Error Handling

```typescript
import axios from "axios";

const http = axios.create({
  baseURL: "https://dev.api.trustlesswork.com", // or https://api.trustlesswork.com for mainnet
  timeout: 10000,
  headers: {
    "Content-Type": "application/json",
    "x-api-key": process.env.NEXT_PUBLIC_API_KEY,
  },
});

async function callTrustlessWorkAPI(endpoint: string, options: RequestInit) {
  try {
    const response = await http.post(endpoint, options);
    return response.data;
  } catch (error: any) {
    const status = error.response?.status;
    switch (status) {
      case 401:
        throw new Error('Invalid API key. Check your API key in settings.');
      case 404:
        throw new Error('Escrow not found');
      case 429:
        throw new Error('Rate limit exceeded. Please try again later.');
      default:
        throw new Error(error.response?.data?.message || `API error: ${status}`);
    }
  }
}
```

### State Management

- Track escrow status locally but always verify with API
- Use `validateOnChain=true` for critical operations
- Poll for status updates during active workflows
- Cache escrow data but refresh before important actions
