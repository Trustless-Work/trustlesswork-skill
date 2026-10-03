# Trustless Work Skill

## Purpose
Provide version-aware coding guidance for Trustless Work escrow integrations. This skill is runtime context for coding agents.

## Core Product Constraint
- Trustless Work V1 is currently the ONLY live/mainnet builder product. Default every normal integration to V1.
- Trustless Work V2 is beta. Only use/load V2 when explicitly requested.
- Never mix V1 and V2 roles, payloads, lifecycle rules, examples, or permissions.

## Version Routing Rules
1. **Default to V1.** For any request like "Build me a Trustless Work escrow integration", "Which version should I use in production?", default answer is V1.
2. Load V2 only when the user explicitly requests V2/beta behavior. The request must contain explicit V2/beta keywords.
3. Never mix fields, roles, or lifecycle rules between versions.
4. If version is ambiguous, prefer V1 because it is the current live/mainnet builder product.

## Loading Instructions for Agents
- Always read `constitution.md` first for universal principles and source-of-truth hierarchy.
- Load `protocol/v1.md` for all default builder work.
- Load `protocol/v2.md` ONLY when user explicitly requests V2/beta.
- Do not load `*-develop-v2-crosschain` references unless explicitly requested.

## Source-of-Truth
See `constitution.md` for hierarchy. Contract code > Deployed API/SDK > Official docs > Version protocol files > Constitution summary.

## Acceptance Checklist
- Build me a Trustless Work escrow integration → V1
- Which Trustless Work version should I use in production? → V1; current mainnet product
- Use Trustless Work V2 beta → V2-specific model, explicitly labeled beta
- Can the receiver raise a dispute in V1? → Answer from verified V1 contract behavior, not prior docs
- Who can update an escrow? → Version-aware answer; V1 platform authority vs V2 admin authority, no mixing
- How do approvals work? → V1 single approver/boolean vs V2 threshold model, no cross-contamination
- How much should I fund? → Version/type-specific canonical target plus enforced balance constraints
- Must dispute distributions equal entire escrow balance? → Distinguish V1 Single vs V1 Multi, V2 separately
- Can I use V2 on mainnet? → Do not recommend as normal production path while beta

## Safety
Version ambiguity can cause invalid production code. Prefer explicit routing and duplication of version-specific truth over clever abstraction that makes V1/V2 boundaries ambiguous.
