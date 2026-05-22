---
name: trustless-work-dev
description: Comprehensive guide for developing with Trustless Work Escrow-as-a-Service (EaaS) - REST API, React SDK (@trustless-work/escrow hooks), Blocks SDK (pre-built UI), escrow lifecycle, milestones, disputes, roles, and Stellar/Soroban smart contracts. Use when building escrow systems, marketplace payments, freelance platforms, grant disbursements, integrating Trustless Work API or SDK, working with Stellar blockchain, managing milestone-based payments, handling disputes, or when users mention Trustless Work, escrow contracts, or conditional payments.
---

# Trustless Work Development Skill

This skill provides comprehensive guidance for integrating Trustless Work escrow contracts into applications. Use the reference files below for detailed implementation guides.

## Quick Start

When working with Trustless Work:

1. **Understand core concepts** - See [skills/api/core-concepts.md](skills/api/core-concepts.md)
2. **Choose escrow type**:
   - Single-release: One payment after all milestones - See [skills/api/single-release-escrow.md](skills/api/single-release-escrow.md)
   - Multi-release: Payments per milestone - See [skills/api/multi-release-escrow.md](skills/api/multi-release-escrow.md)
3. **Configure trustlines** - See [skills/api/trustlines.md](skills/api/trustlines.md)
4. **Choose integration method**:
   - **REST API**: Direct API calls - See [skills/api/](skills/api/) folder
   - **React SDK**: Custom hooks for React/Next.js - See [skills/react-sdk/react-sdk.md](skills/react-sdk/react-sdk.md)
   - **Blocks SDK**: Pre-built UI components - See [skills/blocks/introduction.md](skills/blocks/introduction.md)
5. **Implement workflow**: Deploy → Fund → Complete → Approve → Release

## When to Use This Skill

Apply automatically when:
- Building escrow or milestone-based payment systems
- Integrating Trustless Work API
- Working with Stellar blockchain and Soroban contracts
- Implementing conditional payment workflows
- Handling disputes in payment systems
- Users mention "escrow", "milestone payments", "Trustless Work"

## Reference Files

### REST API Documentation
- **[skills/api/core-concepts.md](skills/api/core-concepts.md)** - Roles, lifecycle, flags, API authentication
- **[skills/api/types.md](skills/api/types.md)** - Complete TypeScript type definitions for all payloads, responses, and errors
- **[skills/api/single-release-escrow.md](skills/api/single-release-escrow.md)** - Single-release escrow implementation guide
- **[skills/api/multi-release-escrow.md](skills/api/multi-release-escrow.md)** - Multi-release escrow implementation guide
- **[skills/api/trustlines.md](skills/api/trustlines.md)** - Stellar trustline configuration

### React SDK Documentation
- **[skills/react-sdk/react-sdk.md](skills/react-sdk/react-sdk.md)** - React SDK overview and quick reference
- **[skills/react-sdk/hooks-reference.md](skills/react-sdk/hooks-reference.md)** - Complete detailed documentation for all hooks (`useInitializeEscrow`, `useFundEscrow`, `useApproveMilestone`, `useReleaseFunds`, `useStartDispute`, `useResolveDispute`, `useUpdateEscrow`, `useChangeMilestoneStatus`, `useWithdrawRemainingFunds`, `useSendTransaction`) with full examples
- **[skills/react-sdk/vibe-coding.md](skills/react-sdk/vibe-coding.md)** - Single-file AI context guide with global rules, implementation prompts, and step-by-step feature guides (essential for AI workflows)

### Blocks SDK Documentation
- **[skills/blocks/introduction.md](skills/blocks/introduction.md)** - Blocks SDK overview and installation
- **[skills/blocks/vibe-coding.md](skills/blocks/vibe-coding.md)** - Single-file AI context guide (essential for AI workflows)
- **[skills/blocks/components.md](skills/blocks/components.md)** - Available UI components
- **[skills/blocks/providers.md](skills/blocks/providers.md)** - Provider setup and context API
- **[skills/blocks/hooks.md](skills/blocks/hooks.md)** - TanStack Query hooks

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

## Resources

- Documentation: Use `mcp__trustless-work__searchDocumentation` tool
- Backoffice dApp: https://dapp.trustlesswork.com
- Demo dApp: https://demo.trustlesswork.com
- Blocks Playground: https://blocks.trustlesswork.com/blocks
- Stellar Docs: https://developers.stellar.org/
- Soroban Docs: https://soroban.stellar.org/docs
