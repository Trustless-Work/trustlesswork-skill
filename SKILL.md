---
name: trustless-work-dev
description: >
  Use when integrating Trustless Work escrow, deploying single-release or multi-release escrow
  contracts, handling milestone-based payments, releasing funds, managing disputes on Stellar, or
  working with the REST API, React SDK (@trustless-work/escrow hooks), or Blocks SDK (pre-built UI).
  Also use when users mention escrow, conditional payments, non-custodial payments, USDC escrow,
  freelance platforms, marketplace payments, grant disbursements, Soroban contracts, or Stellar
  blockchain payments — even if they don't explicitly mention Trustless Work.
license: Apache-2.0
compatibility: Designed for Claude Code and compatible AI coding assistants. Requires network access to query Trustless Work APIs.
metadata:
  author: Trustless Work
  version: "1.0"
allowed-tools: mcp__trustless-work__searchDocumentation mcp__trustless-work__getPage
---

# Trustless Work Development Skill

[View on skills.sh](https://www.skills.sh/trustless-work/trustless-work-dev-skill)

This skill provides comprehensive guidance for integrating Trustless Work escrow contracts into applications. Use the reference files below for detailed implementation guides.

## Quick Start

### Installation

Install this skill using:

```bash
npx skills add trustless-work/trustless-work-dev-skill
```

### When working with Trustless Work:

1. **Configure MCP (recommended)** — See [MCP Integration](#mcp-integration) below for live docs and escrow tools
2. **Understand core concepts** - See [skills/api/core-concepts.md](skills/api/core-concepts.md)
3. **Choose escrow type**:
   - Single-release: One payment after all milestones - See [skills/api/single-release-escrow.md](skills/api/single-release-escrow.md)
   - Multi-release: Payments per milestone - See [skills/api/multi-release-escrow.md](skills/api/multi-release-escrow.md)
4. **Configure trustlines** - See [skills/api/trustlines.md](skills/api/trustlines.md)
5. **Choose integration method**:
   - **REST API**: Direct API calls - See [skills/api/](skills/api/) folder
   - **React SDK**: Custom hooks for React/Next.js - See [skills/react-sdk/react-sdk.md](skills/react-sdk/react-sdk.md)
   - **Blocks SDK**: Pre-built UI components - See [skills/blocks/introduction.md](skills/blocks/introduction.md)
6. **Implement workflow**: Deploy → Fund → Complete → Approve → Release

## Gotchas

These are the non-obvious facts that the agent will get wrong without being told:

- **`amount` type differs by endpoint**: In `deploy` REST calls `amount` is a `number`. In `fund-escrow` REST calls `amount` is a `string`. React SDK's `FundEscrowPayload` uses `number`. Don't mix them.
- **`milestoneIndex` is always a string**: Pass `"0"` not `0` — even though it looks numeric.
- **Don't include `status` or `approvedFlag` in milestone objects when deploying**: Only `description` is valid on deploy. Adding those fields causes errors.
- **Header is `x-api-key`**: Not `Authorization: Bearer`. This is the single most common auth mistake.
- **Single-release requires ALL milestones approved before any release**: You cannot call release-funds until every milestone is individually approved. Multi-release allows per-milestone releases.
- **Resolve-dispute distributions must sum to post-fee balance**: On mainnet, the 0.3% protocol fee is already deducted before you can distribute. Use the actual escrow balance, not the original deposit amount.
- **Mainnet has a 0.3% protocol fee on top of platform fee**: Total cost = `amount + platformFee + (amount × 0.003)`.
- **After funding, only milestones can be added**: Roles, amount, and platform fee cannot be changed once the escrow has funds.
- **`trustline.address` is always the issuer address (starts with G), never the Soroban contract address (starts with C)**: The API resolves the Soroban contract address internally — you don't need to find it. USDC Testnet: `GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5` — USDC Mainnet: `GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN`. Using a C address or the wrong network's address causes silent funding failures.
- **All parties need trustlines before the escrow can be funded**: Payer, receiver, and platform address must all hold a trustline for the escrow token. Read [skills/api/trustlines.md](skills/api/trustlines.md) if trustline errors occur.
- **Service Provider and Approver should not be the same address**: The API allows it, but it defeats escrow security (self-approval of own work).
- **React SDK: `useSendTransaction` is required after every write hook**: Hooks return an unsigned XDR only. You must sign it and call `sendTransaction` — without this, nothing reaches the blockchain.
- **React SDK: the `type` parameter must match the payload type**: `"single-release"` expects `SingleRelease*Payload`; `"multi-release"` expects `MultiRelease*Payload`. Mismatching causes runtime errors.
- **Use `validateOnChain=true` when querying before critical operations**: Without it you may receive stale cached data. Always use it before release, dispute, or resolve calls.

## Reference Files

Load these on demand — only when the task requires them:

### REST API
- Read **[skills/api/core-concepts.md](skills/api/core-concepts.md)** for roles, lifecycle, flags, and auth details.
- Read **[skills/api/types.md](skills/api/types.md)** when you need TypeScript type definitions for payloads, responses, or errors.
- Read **[skills/api/single-release-escrow.md](skills/api/single-release-escrow.md)** when implementing any single-release endpoint (deploy, fund, approve, release, dispute, resolve, update).
- Read **[skills/api/multi-release-escrow.md](skills/api/multi-release-escrow.md)** when implementing any multi-release endpoint.
- Read **[skills/api/trustlines.md](skills/api/trustlines.md)** when trustline errors occur, when setting up new accounts, or when the user asks about tokens/assets.

### React SDK
- Read **[skills/react-sdk/react-sdk.md](skills/react-sdk/react-sdk.md)** for SDK setup, provider config, and hook overview.
- Read **[skills/react-sdk/hooks-reference.md](skills/react-sdk/hooks-reference.md)** for complete parameter docs and examples of any specific hook (`useInitializeEscrow`, `useFundEscrow`, `useApproveMilestone`, `useReleaseFunds`, `useStartDispute`, `useResolveDispute`, `useUpdateEscrow`, `useChangeMilestoneStatus`, `useWithdrawRemainingFunds`, `useSendTransaction`).
- Read **[skills/react-sdk/vibe-coding.md](skills/react-sdk/vibe-coding.md)** when scaffolding a full React/Next.js integration from scratch (contains global rules, step-by-step guides, and AI workflow context).

### Blocks SDK (pre-built UI)
- Read **[skills/blocks/introduction.md](skills/blocks/introduction.md)** for installation and SDK overview.
- Read **[skills/blocks/vibe-coding.md](skills/blocks/vibe-coding.md)** when scaffolding a Blocks integration from scratch.
- Read **[skills/blocks/components.md](skills/blocks/components.md)** for available UI components.
- Read **[skills/blocks/providers.md](skills/blocks/providers.md)** for provider setup and context API.
- Read **[skills/blocks/hooks.md](skills/blocks/hooks.md)** for TanStack Query hooks.

## API Base URLs

- **Mainnet**: `https://api.trustlesswork.com`
- **Testnet**: `https://dev.api.trustlesswork.com`
- **Swagger (Mainnet)**: `https://api.trustlesswork.com/docs`
- **Swagger (Testnet)**: `https://dev.api.trustlesswork.com/docs`

All endpoints require: `x-api-key: YOUR_API_KEY` header

Rate limit: **50 requests per 60 seconds**

## Common Transaction Flow

1. Call API endpoint → Get unsigned XDR transaction
2. Sign with wallet → Create signed XDR
3. Submit via `/helper/send-transaction` → Broadcast to Stellar
4. Verify on-chain → Query with `validateOnChain=true`

## MCP Integration

Pair this skill with the Trustless Work MCP servers for live documentation and escrow operations. Setup guide: https://docs.trustlesswork.com/trustless-work/ai/mcp

Add both servers to `mcp.json` in the project root (Cursor → Settings → MCP → Add New MCP Server):

```json
{
  "mcpServers": {
    "trustlesswork-docs": {
      "type": "streamable-http",
      "url": "https://docs.trustlesswork.com/trustless-work/~gitbook/mcp",
      "headers": {}
    },
    "trustlesswork": {
      "type": "streamable-http",
      "url": "https://mcp.trustlesswork.com/mcp",
      "headers": {}
    }
  }
}
```

| Server | When to use |
| --- | --- |
| `trustlesswork-docs` | Search docs, answer questions, generate SDK code when local reference files are insufficient |
| `trustlesswork` | Trigger escrow actions and live operations from the editor |

**Docs MCP tools** (use in Agent Mode when reference files don't cover a topic):
- `mcp__trustless-work__searchDocumentation` — search documentation for any topic
- `mcp__trustless-work__getPage` — retrieve a specific documentation page

Prefer local reference files in `skills/` first; fall back to MCP when you need the latest docs or a topic not covered here.

If MCP tools are unavailable, remind the user to install both servers, enable Agent Mode, and confirm **Connected** status under Settings → MCP.

## Resources

- [Trustless Work Documentation](https://docs.trustlesswork.com)
- [MCP Setup Guide](https://docs.trustlesswork.com/trustless-work/ai/mcp)
- [Backoffice](https://dapp.trustlesswork.com)
- [Escrow Lab](https://demo.trustlesswork.com)
- [Escrow Blocks](https://blocks.trustlesswork.com/blocks)
- [Swagger (Mainnet)](https://api.trustlesswork.com/docs)