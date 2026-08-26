# Trustless Work Constitution

The non-negotiable laws of the Trustless Work protocol. This file exists so that AI agents and integrators never invent permissions, skip lifecycle steps, or design flows the smart contract will reject. Every rule here comes from the [official documentation](https://docs.trustlesswork.com/trustless-work); when in doubt, the official docs and the deployed contract behavior prevail over this file.

---

## Article I — Foundational Principles

### 1. Non-custodial, always

Funds live in a Soroban smart-contract escrow on Stellar — never in Trustless Work's or the platform's custody. The API never holds private keys: every write endpoint returns an **unsigned XDR transaction** that must be signed client-side by the correct role's wallet and submitted via `POST /helper/send-transaction`. Any design that asks users to hand over secret keys, or signs on their behalf server-side, violates this constitution.

### 2. Roles are the only source of authority

Authority comes exclusively from holding a role address in the escrow. The smart contract enforces this — a UI cannot grant what the contract denies. Never build flows that assume an actor can perform an action their role does not permit: the transaction will fail on-chain.

### 3. The chain is the source of truth

Indexer queries may serve cached data. Before critical operations (release, dispute, resolve), query with `validateOnChain=true`.

---

## Article II — Roles: Powers and Prohibitions

| Role | CAN | CANNOT |
|------|-----|--------|
| `approver` | Approve milestones; raise a dispute | Mark milestones complete; release funds; resolve disputes |
| `serviceProvider` | Change milestone status and attach evidence; raise a dispute | Approve its own work; release funds; resolve disputes |
| `releaseSigner` | Release funds (`release-funds` / `release-milestone-funds`); raise a dispute at the release stage | Approve milestones; mark milestones complete; resolve disputes |
| `disputeResolver` | Resolve disputes — the **only** address authorized to intervene once a dispute is raised; call `withdraw-remaining-funds` (multi-release) | Raise disputes; act while no dispute is active |
| `receiver` | Receive funds when release executes (passive role) | Any active operation — **including raising disputes** |
| `platformAddress` | Receive the platform fee at release; update the escrow (see Article III §3) | Be replaced — the platform address of an escrow cannot be changed |

Actors without a role slot:

- **Deploy signer** (`signer`): signs the deploy transaction; gains no escrow permissions from it.
- **Depositor**: *anyone* holding the asset and its trustline may fund an escrow; funding grants no permissions inside it.

One address may hold multiple roles, but `serviceProvider` and `approver` should not be the same address — self-approval of one's own work defeats the purpose of escrow.

---

## Article III — Lifecycle Laws

1. **The order is fixed**: Deploy → Fund → Milestone updates → Approval → Release. Disputes may interrupt at any active stage.
2. **Deploy constraints**: `engagementId` must be unique (reuse → "Escrow already initialized"); at least 1 and at most 50 milestones; amounts cannot be zero; `platformFee` cannot exceed 99%; all flags (`approved`, `disputed`, `released`) must be false. Single-release milestones carry **only** `description` (no `status`, no `approvedFlag`). Multi-release has **no top-level `amount`**: each milestone defines its own `amount` and `receiver`, and `receiver` does not exist in its `roles` object.
3. **Update laws**: only `platformAddress` may call `update-escrow`, and only while all flags are false and no milestone is approved. **Once the escrow has funds, the only permitted change is adding more milestones** — every other property is frozen. The `platformAddress` itself can never be changed.
4. **Completion ≠ approval**: the `serviceProvider` marking a milestone complete moves no funds. The `approver` must approve it in a separate action.
5. **Single-release**: **all** milestones must be approved and no dispute active before `release-funds`; the contract then releases the entire balance minus fees in one payment.
6. **Multi-release**: each milestone is approved and released independently; disputes and resolutions are per-milestone; `withdraw-remaining-funds` may be called only by the `disputeResolver` and only when every milestone is already released or dispute-resolved.
7. **Dispute resolution is terminal**: the resolver's `distributions` must sum exactly to the current escrow balance (post-fees). Once resolved, a dispute cannot be reopened.

---

## Article IV — API Laws

1. `x-api-key` header on **every** request — including read-only indexer queries (`401` without it). Never `Authorization: Bearer`.
2. Every write operation is 3 steps: **build** (API returns unsigned XDR) → **sign** (correct role's wallet) → **submit** (`POST /helper/send-transaction`). A new escrow's `contractId` exists only after step 3.
3. `amount` is a **number** everywhere — deploy, `fund-escrow`, milestone amounts, dispute distributions. Never a string.
4. `milestoneIndex` is a **string** (`"0"`, `"1"`, …) — even though it looks numeric.
5. `trustline.address` is the token **issuer** address (starts with `G`), never the Soroban contract address (starts with `C`); the companion field is `symbol`, not `code`.
6. Rate limit: **50 requests per 60 seconds** per client (`429` beyond it).

---

## Article V — Economic Laws

1. **Protocol fee**: fixed **0.3%** on mainnet, deducted automatically at release.
2. **Platform fee**: set at deploy (cannot exceed 99%), paid to `platformAddress` at release.
3. **Lockup**: single-release funds `amount + platformFee`; multi-release funds the sum of all milestone amounts + platform fee.
4. The `receiver` gets the remaining balance after both fees.

---

## Article VI — Network Laws

1. Testnet (`https://dev.api.trustlesswork.com`) and mainnet (`https://api.trustlesswork.com`) are fully separate: separate API keys, separate USDC issuers. **Never mix networks** — a testnet issuer on mainnet (or vice versa), or a wrong network passphrase, makes transactions fail.
   - USDC testnet issuer: `GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5`
   - USDC mainnet issuer: `GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN`
2. Every participant address must hold the asset's trustline **before** the escrow can be funded (each trustline requires a 0.5 XLM reserve).
3. Always develop and test on testnet first.

---

## Article VII — Agent Conduct

1. **No law here may be worked around.** If a requested feature violates one of these laws — a dispute button for the `receiver`, editing a funded escrow's amount, string amounts, server-side signing — flag the conflict to the user instead of implementing it.
2. When this file and the official docs disagree, **the official docs win**: verify at [docs.trustlesswork.com](https://docs.trustlesswork.com/trustless-work) (or through the `trustless-work` MCP tools) and propose an amendment here.
3. Do not invent endpoints, fields, roles, or behavior that the official documentation does not describe.
4. API keys are secrets: environment variables only — never hardcoded, logged, or committed.

---

**Sources**: [Roles in Trustless Work](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/roles-in-trustless-work) · [Escrow Lifecycle](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle) · [Dispute Resolution](https://docs.trustlesswork.com/trustless-work/introduction/technology-overview/escrow-lifecycle/dispute-resolution) · [API Introduction](https://docs.trustlesswork.com/trustless-work/api-rest/introduction) · [Update Escrow](https://docs.trustlesswork.com/trustless-work/api-rest/deploy/update-escrow-properties) · [Withdraw Remaining Funds](https://docs.trustlesswork.com/trustless-work/api-rest/deploy-1/withdraw-remaining-funds) · [Trustlines](https://docs.trustlesswork.com/trustless-work/introduction/stellar-and-soroban-the-backbone-of-trustless-work/trustlines)

**Version**: 1.0.0 — based on **v1** of the official documentation (docs.trustlesswork.com) as of 2026-08-26. Documentation v2 is under development; this file must be re-validated and amended against v2 when it is published.
