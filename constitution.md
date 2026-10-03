# Trustless Work Constitution

## Universal Trustless Work Principles

These principles apply to all protocol versions. They are a compressed representation of verified protocol truth, not an independent authority.

### Non-custodial Architecture
[FACT] Funds are held in on-chain escrow contracts. Trustless Work does not custody funds.

### Authorization / Signing Model
- Deployment is authorized by the deploy signer.
- Funding is authorized by the funding signer/depositor.
- Escrow state-transition operations are role-gated according to the relevant contract version.
- Being deployer/depositor does not itself grant an escrow role.
- Use **authorized signer/account** rather than assuming every transaction must be signed by a role wallet.

### Source-of-Truth Hierarchy
When validating behavior, use this order:
1. Exact deployed/audit-bound smart-contract implementation for the relevant protocol version
2. Deployed API behavior/schema + current SDK types for that same protocol version
3. Official Trustless Work developer documentation
4. Version-specific skill protocol file `protocol/v1.md` or `protocol/v2.md`
5. `constitution.md` compressed universal guidance
6. Other local skill reference files

If two layers disagree: follow the higher layer, record the inconsistency, fix the lower layer in this repository when in scope, never silently reconcile by assumption.

### Protocol Selection Rule
**V1 = production / builder default**
- Trustless Work V1 is currently the only live/mainnet product.
- Unless a builder explicitly asks for V2/beta behavior, the skill MUST recommend V1, generate V1-compatible payloads, roles, flows, and examples, use V1 API/SDK semantics, avoid V2-only fields or behavior.
- Never tell a builder that V2 is the production/default integration path.

**V2 = beta**
- V2 is beta. It may be documented so agents understand the future protocol, but it must be clearly separated from V1 and only loaded/used when the user explicitly asks for V2/beta functionality.
- `*-develop-v2-crosschain` branches are also beta and remain outside normal V1 builder guidance unless explicitly requested.

### API / Network Safety Rules
- All API requests require `x-api-key` per deployed API verification. Read-only calls are not exempt.
- Do not assume a mutable Git branch is automatically identical to the currently deployed contract. Trace deployment metadata, contract IDs, WASM hashes, tags, release commits, audit references, or network deployment manifests. If this cannot be proven, document uncertainty explicitly.

### Normative Labels
- **[ENFORCED]** — contract/API rejects violations.
- **[CANONICAL]** — recommended production integration pattern; technically valid deviations may exist.
- **[SECURITY]** — integration/security best practice.
- **[FACT]** — current/versioned deployment or configuration value that may change.
- **[UNVERIFIED]** — statement pending direct contract/test verification.

Never mark something `[ENFORCED]` solely because documentation recommends it.
