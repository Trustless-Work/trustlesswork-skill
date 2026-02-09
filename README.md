# Trustless Work Development Skill

A comprehensive AI skill for developing with the Trustless Work platform - enabling escrow contracts, milestone-based payments, and dispute resolution on the Stellar blockchain.

## Installation

Install this skill using:

```bash
npx skills add https://github.com/Trustless-Work/trustless-work-dev-skill
```

Or manually:

```bash
git clone https://github.com/Trustless-Work/trustless-work-dev-skill.git
cp -r trustless-work-dev-skill ~/.cursor/skills/trustless-work-dev
```

## Structure

```
trustless-work-dev-skill/
├── SKILL.md                    # Main skill definition (required)
├── README.md                   # Project documentation
├── CONTRIBUTING.md             # Contributing guidelines
├── CODE_OF_CONDUCT.md          # Contributor Covenant Code of Conduct
├── package.json                # NPM package configuration
├── LICENSE                     # Apache-2.0 License
├── .gitignore                  # Git ignore file
└── skills/                     # Skill documentation
    ├── api/                    # REST API documentation
    │   ├── core-concepts.md    # Roles, lifecycle, flags, API authentication
    │   ├── types.md            # Complete TypeScript type definitions
    │   ├── single-release-escrow.md # Single-release escrow implementation guide
    │   ├── multi-release-escrow.md  # Multi-release escrow implementation guide
    │   └── trustlines.md       # Stellar trustline configuration
    ├── react-sdk/              # React SDK documentation
    │   ├── react-sdk.md        # React SDK overview and quick reference
    │   ├── hooks-reference.md  # Complete hooks documentation
    │   └── vibe-coding.md      # AI context guide with prompts
    └── blocks/                 # Blocks SDK documentation
        ├── introduction.md     # Blocks SDK overview and installation
        ├── vibe-coding.md      # AI context guide
        ├── components.md       # Available UI components
        ├── providers.md        # Provider setup and context API
        └── hooks.md            # TanStack Query hooks
```

## What This Skill Provides

This skill equips AI assistants with comprehensive knowledge of:

### REST API
- **Core Concepts**: Roles, escrow lifecycle, status flags, API authentication
- **TypeScript Types**: Complete type definitions for all payloads (13+ types), responses, and errors
- **Single-Release Escrows**: Deploy, complete milestones, approve, release, dispute, resolve
- **Multi-Release Escrows**: Incremental payments, per-milestone releases, withdraw remaining funds
- **Trustlines**: Stellar asset configuration, setup, and management

### React SDK
- **Custom Hooks**: `useInitializeEscrow`, `useFundEscrow`, `useApproveMilestone`, `useReleaseFunds`, `useStartDispute`, `useResolveDispute`, `useUpdateEscrow`, `useWithdrawRemainingFunds`, `useChangeMilestoneStatus`, `useSendTransaction`
- **TypeScript Types**: Complete type definitions for all payloads
- **Integration Examples**: Full React component examples
- **AI Context Guide**: Vibe coding guide with implementation prompts

### Blocks SDK
- **UI Components**: Pre-built cards, tables, dialogs, forms for escrow management
- **Providers**: API config, wallet context, dialogs, amount formatting
- **TanStack Query Hooks**: For fetching and mutating escrows
- **Context API**: Global escrow state management
- **AI Context Guide**: Vibe coding guide with project bootstrap

## Use Cases

This skill is automatically applied when working on:

- Escrow or milestone-based payment systems
- Trustless Work API integration
- Stellar blockchain and Soroban smart contracts
- Conditional payment workflows
- Dispute resolution systems
- Freelancing or gig economy platforms
- Building React/Next.js escrow interfaces

## Quick Reference

### REST API Endpoints

**Single-Release Escrow**
- Deploy: `POST /deployer/single-release`
- Fund: `POST /escrow/single-release/fund-escrow`
- Change Status: `POST /escrow/single-release/change-milestone-status`
- Approve: `POST /escrow/single-release/approve-milestone`
- Release: `POST /escrow/single-release/release-funds`
- Dispute: `POST /escrow/single-release/dispute-escrow`
- Resolve: `POST /escrow/single-release/resolve-dispute`
- Update: `PUT /escrow/single-release/update-escrow`

**Multi-Release Escrow**
- Deploy: `POST /deployer/multi-release`
- Fund: `POST /escrow/multi-release/fund-escrow`
- Change Status: `POST /escrow/multi-release/change-milestone-status`
- Approve: `POST /escrow/multi-release/approve-milestone`
- Release: `POST /escrow/multi-release/release-milestone`
- Dispute: `POST /escrow/multi-release/dispute-escrow`
- Resolve: `POST /escrow/multi-release/resolve-dispute`
- Update: `PUT /escrow/multi-release/update-escrow`
- Withdraw: `POST /escrow/multi-release/withdraw-remaining-funds`

**Helpers**
- Query: `GET /helper/get-escrow-by-contract-ids`
- Query by Signer: `GET /helper/get-escrows-by-signer`
- Query by Role: `GET /helper/get-escrows-by-role`
- Submit: `POST /helper/send-transaction`

### React SDK Hooks

```tsx
import { 
  useInitializeEscrow,
  useFundEscrow,
  useApproveMilestone,
  useReleaseFunds,
  useStartDispute,
  useResolveDispute,
  useUpdateEscrow,
  useChangeMilestoneStatus,
  useWithdrawRemainingFunds,
  useSendTransaction
} from '@trustless-work/escrow/hooks';
```

### Blocks SDK

```bash
# Install blocks
npm install @trustless-work/blocks

# List available blocks
npx trustless-work list

# Install specific block
npx trustless-work add escrows/escrows-by-signer/table
```

## Documentation Files

### REST API
- **[skills/api/core-concepts.md](skills/api/core-concepts.md)** - Overview, roles, lifecycle, API authentication, error handling
- **[skills/api/types.md](skills/api/types.md)** - Complete TypeScript type definitions for all payloads (13+ types), responses, and errors with usage examples
- **[skills/api/single-release-escrow.md](skills/api/single-release-escrow.md)** - Complete guide for single-release escrows with detailed REST API examples
- **[skills/api/multi-release-escrow.md](skills/api/multi-release-escrow.md)** - Complete guide for multi-release escrows with detailed REST API examples
- **[skills/api/trustlines.md](skills/api/trustlines.md)** - Stellar trustline setup, configuration, and best practices

### React SDK
- **[skills/react-sdk/react-sdk.md](skills/react-sdk/react-sdk.md)** - React SDK overview, setup, and quick reference
- **[skills/react-sdk/hooks-reference.md](skills/react-sdk/hooks-reference.md)** - Complete detailed documentation for all 10 hooks with full usage examples, parameters, return values, and complete code samples
- **[skills/react-sdk/vibe-coding.md](skills/react-sdk/vibe-coding.md)** - Single-file AI context guide with global development rules, implementation prompts for all hooks, and step-by-step feature guides (essential for AI workflows)

### Blocks SDK
- **[skills/blocks/introduction.md](skills/blocks/introduction.md)** - Blocks SDK overview, installation, and context API
- **[skills/blocks/vibe-coding.md](skills/blocks/vibe-coding.md)** - Single-file AI context guide with project bootstrap, provider order, CLI commands, examples, and troubleshooting (essential for AI workflows)
- **[skills/blocks/components.md](skills/blocks/components.md)** - Available UI components and usage examples
- **[skills/blocks/providers.md](skills/blocks/providers.md)** - Provider setup and configuration
- **[skills/blocks/hooks.md](skills/blocks/hooks.md)** - TanStack Query hooks for fetching and mutating escrows

## Resources

- [Trustless Work Documentation](https://docs.trustlesswork.com)
- [Trustless Work Dashboard](https://dapp.trustlesswork.com/dashboard)
- [Blocks Playground](https://blocks.trustlesswork.com/blocks)
- [Stellar Documentation](https://developers.stellar.org/)
- [Soroban Documentation](https://soroban.stellar.org/docs)

## Contributing

Contributions welcome! Please read our [Contributing Guide](CONTRIBUTING.md) for details on how to contribute to this project.

Please note that this project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code.

## License

Apache-2.0
