# Trustless Work Constitution

The compressed, agent-facing representation of the rules that govern Trustless Work escrows — so AI agents never invent permissions, skip enforced preconditions, or design flows the smart contract will reject.

> **Protocol scope:** This constitution describes **Trustless Work V1** (`single-release-main` / `multi-release-main` contract semantics) — the only live/mainnet builder product. **V2 is beta and intentionally excluded** unless a rule is explicitly marked V2. V2 changes the role model (role collections, `admin`, `observers`), approvals (thresholds), update authority, and batch operations — do not import V2 beta rules here. Version-aware skill architecture is tracked in issue #6.

## Source-of-Truth Hierarchy

This file is **not** an independent source of truth. When layers disagree, trust the higher layer and flag the inconsistency to the user:

1. Deployed, audit-bound smart-contract behavior
2. Deployed API schema and current SDK types for the same protocol version
3. Official developer documentation ([docs.trustlesswork.com](https://docs.trustlesswork.com/trustless-work))
4. This file (`constitution.md`)
5. Detailed local skill reference files (`skills/**`)

## How to Read This File

Every statement is tagged:

- **[ENFORCED]** — the contract or API rejects violations. Never design around these.
- **[CANONICAL]** — the recommended integration sequence or pattern. Deviating is allowed when technically valid; do not refuse a valid integration merely because it diverges from the recommended flow.
- **[SECURITY]** — best-practice constraint. Flag deviations to the user before implementing them.
- **[FACT]** — current product/deployment/config value. May change between versions or deployments; verify when critical.

---

## Article I — Foundational Principles

### 1. Non-custodial, always

**[ENFORCED]** Funds live in a Soroban smart-contract escrow on Stellar — never in Trustless Work's or the platform's custody. Write endpoints return an **unsigned XDR transaction**; only a signature from the required/authorized signer makes it valid.

**[SECURITY]** Never design flows that collect users' Stellar secret keys or sign on their behalf server-side.

### 2. State-transition authority is role-gated

**[ENFORCED]** Escrow state-transition authority **after deployment** is role-gated, with one exception: funding, which is authorized by the funding signer (any depositor). Deployment is authorized by the deploy signer but grants no escrow role. The smart contract enforces the role gates — a UI cannot grant what the contract denies. Flows that assume an actor can perform an action their assigned role(s) do not permit will fail on-chain.

### 3. The chain is the source of truth

**[CANONICAL]** Indexer queries may serve cached data. Before critical operations (release, dispute, resolve), query with `validateOnChain=true`.

---

## Article II — Role Assignments and Authority

**[ENFORCED]** Each row describes authority granted by that **role assignment**. The final column means “this role assignment alone does not authorize this action” — it is not a prohibition on an address that also holds another role.

| Role assignment | This assignment authorizes | This assignment alone does not authorize |
|---|---|---|
| `approver` | Approve milestones; raise a dispute | Milestone status/evidence updates; release; dispute resolution |
| `serviceProvider` | Change milestone status and attach evidence; raise a dispute | Approval; release; dispute resolution |
| `releaseSigner` | Release funds (`release-funds` / `release-milestone-funds`); raise a dispute | Approval; milestone status/evidence updates; dispute resolution |
| `disputeResolver` | Execute dispute **resolution** and its distributions — the address assigned to this role is required to resolve | Raising a dispute — the contract explicitly rejects the `disputeResolver` address from doing so |
| `receiver` | Receive funds when release executes; raise a dispute (single-release: the escrow `receiver`; multi-release: each milestone's `receiver`, for that milestone) | Approval; milestone status/evidence updates; release; dispute resolution |
| `platformAddress` | Receive the platform fee at release; update the escrow subject to Article III §3; raise a dispute | Approval; milestone status/evidence updates; release; dispute resolution; replacing the escrow's `platformAddress` |

**Who may raise a dispute** (verified against the V1 contract source, `single-release-main` / `multi-release-main`): `approver`, `serviceProvider`, `platformAddress`, `releaseSigner`, and the (milestone) `receiver`. The `disputeResolver` address is explicitly rejected. Note: docs v1 describes a narrower set (no receiver/platform); the contract source is the higher truth layer here.

Actors without a role slot:

- **Deploy signer** (`signer`): signs the deploy transaction; gains no escrow permissions from it.
- **Depositor**: *anyone* holding the asset and its trustline may fund an escrow; funding grants no permissions inside it.

**[FACT]** One address may hold multiple roles by design. Its effective authority is the combination of those role assignments, except where the contract contains an explicit address-level prohibition (for example, an address assigned as `disputeResolver` cannot raise a dispute even if it also matches another dispute-capable role).

**[CANONICAL]** Use distinct addresses when the product workflow requires separation of duties or independent checks. Role separation is a configurator choice, not a V1 contract requirement.

---

## Article III — Lifecycle Laws

1. **[CANONICAL]** The recommended workflow is Deploy → Fund → Milestone updates → Approval → Release, with disputes as an interruption path. Individual steps have their own enforced preconditions (below); the sequence itself is the canonical integration pattern, not a contract rule.

2. **Deploy and external references**
   - **[FACT]** `engagementId` is a caller-defined external reference used by the configurator/integrator to associate an escrow with its own business object or serial number — for example a contract, sale, invoice, order, grant, or internal record. **V1 does not enforce global uniqueness of `engagementId`.** `EscrowAlreadyInitialized` means that the same escrow contract instance has already been initialized; it does not mean the `engagementId` exists globally.
   - **[ENFORCED]** Deploy constraints include: at least 1 and at most 50 milestones; configured amounts must be positive; `platformFee` cannot exceed the contract/API cap; initial release/dispute/resolution/approval flags must satisfy the escrow type's initialization rules. Single-release API deploy milestones carry **only** `description` (no `status`, no `approvedFlag`). Multi-release has **no top-level `amount`**: each milestone defines its own `amount` and `receiver`, and `receiver` does not exist in its `roles` object.

3. **[ENFORCED] Update laws are state- and escrow-type-aware:**
   - Only the existing `platformAddress` may call `update-escrow`, and the `platformAddress` itself cannot be replaced.
   - **Before funding — Single Release:** escrow properties may be changed by the platform subject to the validator; an already-approved existing milestone blocks property changes, and the updated state cannot introduce release/dispute/resolution or approved milestone flags.
   - **Before funding — Multi Release:** the platform may update escrow properties subject to the validator; existing milestone flags must be preserved, and any newly appended milestone must start with all milestone flags false.
   - **After funding — both V1 types:** existing escrow properties and existing milestones are frozen. The only permitted structural change is appending new milestones that satisfy the initial/unset flag requirements. **Existing approved milestones do not by themselves prevent appending new milestones** as long as the funded escrow's existing state is preserved.

4. **[ENFORCED]** Completion ≠ approval: the `serviceProvider` marking a milestone complete moves no funds. Only the `approver` role assignment's separate approval satisfies the approval precondition.

5. **[ENFORCED]** Single-release: **all** milestones must be approved and no dispute active before `release-funds`; the contract then pays out the **configured escrow amount** minus fees in one payment (release requires the contract balance to cover it — payouts are computed from `escrow.amount`, not from arbitrary excess balance).

6. **[ENFORCED]** Multi-release: each milestone is approved and released independently; disputes and resolutions are per-milestone.

7. **[ENFORCED]** Dispute-resolution invariants differ by escrow type, and resolution is terminal in both:
   - **Single-release** `resolve-dispute`: the escrow must be disputed; distributions (max 50 entries) must be positive and sum **exactly** to the current escrow balance.
   - **Multi-release** `resolve-milestone-dispute`: the milestone must be disputed (not released/resolved); distributions (max 50 entries) must be positive, must **not exceed the milestone's amount**, and must not exceed the current contract balance — they do **not** have to equal the full balance.
   - Once resolved, a dispute cannot be reopened.

8. **[FACT — validation pending] `withdraw-remaining-funds`:** both current V1 contract branches contain a `withdraw_remaining_funds` surface, while public API/product guidance has historically emphasized Multi Release. The intended deployed/API support matrix and exact Single Release semantics are being validated in [`trustlesswork-smart-contract-stellar#117`](https://github.com/Trustless-Work/trustlesswork-smart-contract-stellar/issues/117). Until that validation is resolved, do **not** encode “Multi Release-only” or “Single Release-supported” as a stable public protocol law.

---

## Article IV — API Laws

1. **[ENFORCED]** `x-api-key` header on **every** request — including read-only indexer queries. Verified against the deployed V1 API: reads without a key return `401 Unauthorized`. Never `Authorization: Bearer` for the API-key requirement.
2. **[ENFORCED]** Every write operation is 3 steps: **build** (API returns unsigned XDR) → **sign** (with the signer authorized for that operation) → **submit** (`POST /helper/send-transaction`). A new escrow's on-chain contract exists only after the signed deployment transaction is successfully submitted.
3. **[ENFORCED]** `amount` is a **number** in V1 operate payloads — deploy, `fund-escrow`, milestone amounts, dispute distributions. Never send these operate amounts as strings.
4. **[ENFORCED]** `milestoneIndex` is a **string** (`"0"`, `"1"`, …) at the V1 API layer — even though it looks numeric.
5. **[ENFORCED]** In the V1 API payload shape described by this skill, `trustline.address` is the token **issuer** address (starts with `G`), never the escrow Soroban contract address; the companion field is `symbol`, not `code`.
6. **[FACT]** Rate limit: **50 requests per 60 seconds** per client (`429` beyond it).

---

## Article V — Economic Laws

1. **[CANONICAL]** Funding target: the configured escrow `amount` (single-release) or the sum of milestone amounts (multi-release). `platformFee` is a **fee-rate configuration** set at deploy — not an extra token amount to fund.
2. **[ENFORCED]** `fund-escrow` itself only enforces amount > 0, sufficient signer balance, and escrow-property consistency — it does not require a single deposit to equal the configured amount. What is enforced later: **release requires the contract balance to cover the configured release amount**.
3. **[ENFORCED]** At release, the contract computes deductions from the **configured release amount** — the platform fee (sent to `platformAddress`) and the Trustless Work protocol fee — and transfers the remainder to the receiver for that release.
4. **[FACT]** The protocol fee is currently a fixed **0.3%** on V1 mainnet.
5. **[ENFORCED]** The V1 contract/API rejects `platformFee` values above its configured cap (currently 99% subject also to the protocol-fee basis-point constraint).

---

## Article VI — Network Laws

1. **[ENFORCED]** Testnet (`https://dev.api.trustlesswork.com`) and mainnet (`https://api.trustlesswork.com`) are separate networks with separate assets/issuers and credentials. Mixing a testnet issuer with mainnet or using the wrong network passphrase makes transactions fail.
   - **[FACT]** USDC testnet issuer: `GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5`
   - **[FACT]** USDC mainnet issuer: `GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN`
2. **[ENFORCED — by Stellar]** An account must have the relevant asset trustline and satisfy Stellar reserve requirements before it can hold or receive that classic Stellar asset. Therefore the **depositor** and every address that will **receive tokens** — receiver(s), the `platformAddress` when receiving fees, and dispute-distribution addresses — need the relevant trustline before that transfer reaches them. Authority-only actions (approve, release-sign, resolve) are signatures, not token transfers, and do not by themselves require holding the asset.
3. **[FACT]** The additional base reserve associated with a trustline is a Stellar network parameter (commonly 0.5 XLM at the time of this version); treat the numeric reserve as versioned network configuration, not an immutable Trustless Work protocol law.
4. **[CANONICAL]** Always develop and test on testnet first.

---

## Article VII — Agent Conduct

1. **Never work around an [ENFORCED] law.** If a requested feature violates one — for example a dispute action signed by the `disputeResolver`, changing frozen properties of a funded escrow, or sending string amounts to a V1 operate endpoint — flag the conflict to the user instead of implementing it. A **[CANONICAL]** deviation that remains technically valid is acceptable; note the deviation.
2. When this file disagrees with a higher layer of the Source-of-Truth Hierarchy, **the higher layer wins**. Verify against the matching protocol version's deployed/audit-bound contract, deployed API/Swagger, SDK types, or official docs, then propose an amendment here.
3. **Never invent endpoints, fields, roles, or behavior.** If official documentation is silent or conflicts with a higher source, use the higher source and state which source supports the behavior rather than treating documentation silence as a prohibition.
4. **[SECURITY]** Key model: Trustless Work **API keys** are client-visible application keys in the current V1 SDK pattern (`NEXT_PUBLIC_API_KEY`). Still: never commit them to repositories, and rotate them from the dApp if leaked. Stellar **secret keys** (`S...`) are absolute secrets: they never leave the user's wallet, are never logged, and never touch a server. Server-side signing may satisfy on-chain authorization if it holds the authorized key, but custodial handling of a user's secret key violates the intended non-custodial security model.
5. **Version discipline:** do not import V2 beta SDK/API/contract behavior into a V1 answer. If the user explicitly requests V2, load the V2 profile tracked by issue #6 and label it beta.

---

**Sources**: [Smart-contract source](https://github.com/Trustless-Work/trustlesswork-smart-contract-stellar) (`single-release-main` / `multi-release-main`) · [Roles in Trustless Work](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/roles-in-trustless-work) · [Escrow Lifecycle](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle) · [Dispute Resolution](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle/dispute-resolution) · [Release Phase](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle/release-phase) · [API Introduction](https://docs.trustlesswork.com/trustless-work/api-rest/introduction) · [Update Escrow](https://docs.trustlesswork.com/trustless-work/api-rest/deploy/update-escrow-properties) · [Withdraw Remaining Funds](https://docs.trustlesswork.com/trustless-work/api-rest/deploy-1/withdraw-remaining-funds) · [Trustlines](https://docs.trustlesswork.com/trustless-work/introduction/stellar-and-soroban-the-backbone-of-trustless-work/trustlines)

**Contract references used for this revision:** `single-release-main@6e4d40e25057dc51fcffd9c5cbee05eaf369380a` · `multi-release-main@d5cfc2ee0341141ab0a348388771afe71c8c28d2`.

**Version**: 1.3.0 — V1 constitution revised 2026-08-28. V2 is beta and intentionally outside this file until the version-aware architecture in issue #6 is implemented.