# Trustless Work Constitution

The compressed, agent-facing representation of the rules that govern Trustless Work escrows — so AI agents never invent permissions, skip enforced preconditions, or design flows the smart contract will reject.

> **Protocol scope:** This constitution describes **Trustless Work V1** (`single-release-main` / `multi-release-main` contract semantics) — the only live/mainnet builder product. **V2 is beta and intentionally excluded** unless a rule is explicitly marked V2. V2 changes the role model (role collections, `admin`, `observers`), approvals (thresholds), update authority, and batch operations — do not import V2 beta rules here. Version-aware skill architecture is tracked in issue #6.

## Source-of-Truth Hierarchy

This file is **not** an independent source of truth. When layers disagree, trust the higher layer and flag the inconsistency to the user:

1. Deployed, audit-bound smart-contract behavior
2. Deployed API schema and current SDK types
3. Official developer documentation ([docs.trustlesswork.com](https://docs.trustlesswork.com/trustless-work))
4. This file (`constitution.md`)
5. Detailed local skill reference files (`skills/**`)

## How to Read This File

Every statement is tagged:

- **[ENFORCED]** — the contract or API rejects violations. Never design around these.
- **[CANONICAL]** — the recommended integration sequence or pattern. Deviating is allowed when technically valid; do not refuse a valid integration merely because it diverges from the recommended flow.
- **[SECURITY]** — best-practice constraint. Flag deviations to the user before implementing them.
- **[FACT]** — current deployment/config value. May change between deployments; verify when critical.

---

## Article I — Foundational Principles

### 1. Non-custodial, always

**[ENFORCED]** Funds live in a Soroban smart-contract escrow on Stellar — never in Trustless Work's or the platform's custody. Write endpoints return an **unsigned XDR transaction**; only a signature from the required/authorized signer makes it valid.
**[SECURITY]** Never design flows that collect users' Stellar secret keys or sign on their behalf server-side.

### 2. State-transition authority is role-gated

**[ENFORCED]** Escrow state-transition authority **after deployment** is role-gated, with one exception: funding, which is authorized by the funding signer (any depositor). Deployment is authorized by the deploy signer but grants no escrow role. The smart contract enforces the role gates — a UI cannot grant what the contract denies. Flows that assume an actor can perform an action their role does not permit will fail on-chain.

### 3. The chain is the source of truth

**[CANONICAL]** Indexer queries may serve cached data. Before critical operations (release, dispute, resolve), query with `validateOnChain=true`.

---

## Article II — Roles: Powers and Prohibitions

**[ENFORCED]** unless noted:

| Role | CAN | CANNOT |
|------|-----|--------|
| `approver` | Approve milestones; raise a dispute | Mark milestones complete; release funds; resolve disputes |
| `serviceProvider` | Change milestone status and attach evidence; raise a dispute | Approve its own work; release funds; resolve disputes |
| `releaseSigner` | Release funds (`release-funds` / `release-milestone-funds`); raise a dispute at the release stage | Approve milestones; mark milestones complete; resolve disputes |
| `disputeResolver` | Execute dispute **resolution** (and its distributions) — the only address that can resolve; call `withdraw-remaining-funds` (multi-release) | Raise disputes — the contract rejects it explicitly; resolve when nothing is disputed |
| `receiver` | Receive funds when release executes; raise a dispute (single-release: the escrow `receiver`; multi-release: each milestone's `receiver`, for that milestone) | Approve or complete milestones; release funds; resolve disputes |
| `platformAddress` | Receive the platform fee at release; update the escrow (see Article III §3); raise a dispute | Be replaced — the platform address of an escrow cannot be changed |

**Who may raise a dispute** (verified against the contract source, `single-release-main` / `multi-release-main`): `approver`, `serviceProvider`, `platformAddress`, `releaseSigner`, and the (milestone) `receiver`. Only the `disputeResolver` is rejected. Note: docs v1 describes a narrower set (no receiver/platform); the contract source is the higher truth layer here.

Actors without a role slot:

- **Deploy signer** (`signer`): signs the deploy transaction; gains no escrow permissions from it.
- **Depositor**: *anyone* holding the asset and its trustline may fund an escrow; funding grants no permissions inside it.

**[SECURITY]** One address may hold multiple roles, but `serviceProvider` and `approver` should not be the same address — self-approval defeats the purpose of escrow.

---

## Article III — Lifecycle Laws

1. **[CANONICAL]** The recommended workflow is Deploy → Fund → Milestone updates → Approval → Release, with disputes as an interruption path. Individual steps have their own enforced preconditions (below); the sequence itself is the canonical integration pattern, not a contract rule.
2. **[ENFORCED]** Deploy constraints: `engagementId` must be unique (reuse → "Escrow already initialized"); at least 1 and at most 50 milestones; amounts cannot be zero; `platformFee` cannot exceed 99%; all flags (`approved`, `disputed`, `released`) must be false. Single-release milestones carry **only** `description` (no `status`, no `approvedFlag`). Multi-release has **no top-level `amount`**: each milestone defines its own `amount` and `receiver`, and `receiver` does not exist in its `roles` object.
3. **[ENFORCED]** Update laws: only `platformAddress` may call `update-escrow`, and only while all flags are false and no milestone is approved. **Once the escrow has funds, the only permitted change is adding more milestones** — every other property is frozen. The `platformAddress` itself can never be changed.
4. **[ENFORCED]** Completion ≠ approval: the `serviceProvider` marking a milestone complete moves no funds. Only the `approver`'s separate approval satisfies the release precondition.
5. **[ENFORCED]** Single-release: **all** milestones must be approved and no dispute active before `release-funds`; the contract then pays out the **configured escrow amount** minus fees in one payment (release requires the contract balance to cover it — payouts are computed from `escrow.amount`, not from arbitrary excess balance).
6. **[ENFORCED]** Multi-release: each milestone is approved and released independently; disputes and resolutions are per-milestone; `withdraw-remaining-funds` may be called only by the `disputeResolver` and only when every milestone is already released or dispute-resolved.
7. **[ENFORCED]** Dispute-resolution invariants differ by escrow type, and resolution is terminal in both:
   - **Single-release** `resolve-dispute`: the escrow must be disputed; distributions (max 50 entries) must be positive and sum **exactly** to the current escrow balance.
   - **Multi-release** `resolve-milestone-dispute`: the milestone must be disputed (not released/resolved); distributions (max 50 entries) must be positive, must **not exceed the milestone's amount**, and must not exceed the current contract balance — they do **not** have to equal the full balance.
   - Once resolved, a dispute cannot be reopened.

---

## Article IV — API Laws

1. **[ENFORCED]** `x-api-key` header on **every** request — including read-only indexer queries. Verified against the deployed API: reads without a key return `401 Unauthorized`. Never `Authorization: Bearer`.
2. **[ENFORCED]** Every write operation is 3 steps: **build** (API returns unsigned XDR) → **sign** (correct role's wallet) → **submit** (`POST /helper/send-transaction`). A new escrow's `contractId` exists only after step 3.
3. **[ENFORCED]** `amount` is a **number** everywhere — deploy, `fund-escrow`, milestone amounts, dispute distributions. Never a string.
4. **[ENFORCED]** `milestoneIndex` is a **string** (`"0"`, `"1"`, …) — even though it looks numeric.
5. **[ENFORCED]** `trustline.address` is the token **issuer** address (starts with `G`), never the Soroban contract address (starts with `C`); the companion field is `symbol`, not `code`.
6. **[FACT]** Rate limit: **50 requests per 60 seconds** per client (`429` beyond it).

---

## Article V — Economic Laws

1. **[CANONICAL]** Funding target: the configured escrow `amount` (single-release) or the sum of milestone amounts (multi-release). `platformFee` is a **fee-rate configuration** set at deploy — not an extra token amount to fund.
2. **[ENFORCED]** `fund-escrow` itself only enforces amount > 0, sufficient signer balance, and escrow-property consistency — it does not require a single deposit to equal the configured amount. What is enforced later: **release requires the contract balance to cover the configured release amount**.
3. **[ENFORCED]** At release, the contract computes the deductions from the **configured escrow amount** — the platform fee (sent to `platformAddress`) and the Trustless Work protocol fee — and transfers the remainder to the `receiver`.
4. **[FACT]** The protocol fee is currently a fixed **0.3%** on mainnet.
5. **[ENFORCED]** The API rejects `platformFee` values exceeding 99%.

---

## Article VI — Network Laws

1. **[ENFORCED]** Testnet (`https://dev.api.trustlesswork.com`) and mainnet (`https://api.trustlesswork.com`) are fully separate: separate API keys, separate USDC issuers. Mixing networks — a testnet issuer on mainnet, or a wrong network passphrase — makes transactions fail.
   **[FACT]** USDC testnet issuer: `GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5`
   **[FACT]** USDC mainnet issuer: `GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN`
2. **[ENFORCED — by Stellar]** An account cannot receive or hold an asset without that asset's trustline (0.5 XLM reserve each). Therefore the **depositor** and every address that will **receive tokens** — the `receiver`(s), the `platformAddress` (fee recipient), and any dispute-distribution address — need the trustline. Authority-only actions (approve, release-sign, resolve) are signatures, not token transfers, and do not by themselves require holding the asset.
3. **[CANONICAL]** Always develop and test on testnet first.

---

## Article VII — Agent Conduct

1. **Never work around an [ENFORCED] law.** If a requested feature violates one — a dispute button for the `disputeResolver`, editing a funded escrow's amount, string amounts, server-side signing — flag the conflict to the user instead of implementing it. A **[CANONICAL]** deviation that remains technically valid is acceptable; note the deviation.
2. When this file disagrees with a higher layer of the Source-of-Truth Hierarchy, **the higher layer wins**: verify (docs.trustlesswork.com, the `trustless-work` MCP tools, or the deployed API/Swagger) and propose an amendment here.
3. Do not invent endpoints, fields, roles, or behavior that the official documentation does not describe.
4. **[SECURITY]** Key model: Trustless Work **API keys** are client-visible application keys — the official SDK pattern (`NEXT_PUBLIC_API_KEY`) exposes them to the browser by design. Still: never commit them to repositories, and rotate them from the dApp if leaked. Stellar **secret keys** (`S...`) are absolute secrets: they never leave the user's wallet, are never logged, and never touch a server.

---

**Sources**: [Smart-contract source](https://github.com/Trustless-Work/Trustless-Work-Smart-Escrow) (branches `single-release-main` / `multi-release-main` — dispute authorization, resolution invariants, fund/release/update validators) · [Roles in Trustless Work](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/roles-in-trustless-work) · [Escrow Lifecycle](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle) · [Dispute Resolution](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle/dispute-resolution) · [Release Phase](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle/release-phase) · [API Introduction](https://docs.trustlesswork.com/trustless-work/api-rest/introduction) · [Update Escrow](https://docs.trustlesswork.com/trustless-work/api-rest/deploy/update-escrow-properties) · [Withdraw Remaining Funds](https://docs.trustlesswork.com/trustless-work/api-rest/deploy-1/withdraw-remaining-funds) · [Trustlines](https://docs.trustlesswork.com/trustless-work/introduction/stellar-and-soroban-the-backbone-of-trustless-work/trustlines)

**Version**: 1.2.0 — based on **v1** of the official documentation, the contract source on `single-release-main` / `multi-release-main` (dispute, resolution, fund, release, update validators), and spot-checks against the deployed testnet API, as of 2026-08-26. Documentation v2 is under development; this file must be re-validated and amended against v2 when it is published.
