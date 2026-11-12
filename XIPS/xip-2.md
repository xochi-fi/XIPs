---
xip: 2
title: Adaptive Settlement Controls
description: Execution config with venue routing, diffusion scheduling, and gas/slippage budgets
author: DROO (@DROOdotFOO)
discussions-to: https://github.com/xochi-fi/XIPs/issues/3
status: Draft
type: Protocol
created: 2026-04-15
requires: 1
---

## Abstract

XIP-1 introduced settlement splitting but left execution parameters to the caller. This XIP adds client-configurable controls for the privacy-cost-speed tradeoff: per-sub-trade venue routing across settlement paths (public, stealth, shielded), time-diffused submission scheduling, and gas/slippage budgets. All changes are SDK-side. The SettlementRegistry from XIP-1 is not modified.

## Motivation

XIP-1's `SplitConfig` controls *how many* pieces a trade splits into. It does not control *where* those pieces settle, *when* they are submitted, or *how much* each costs. For institutional and large-scale users, these are not optional -- they determine whether settlement splitting is usable in practice.

Three specific gaps:

1. **No venue selection.** The Xochi protocol has three settlement paths: public (L1 transfer), stealth (ERC-5564), and shielded (Aztec L2). XIP-1's planner treats all sub-trades identically. A 500 ETH trade split into five 100 ETH sub-trades gains nothing if all five settle publicly. Routing high-value sub-trades to higher-privacy venues and gas-sensitive sub-trades to cheaper venues is the composition point between XIP-1 and the existing settlement infrastructure.

2. **Temporal correlation.** If all sub-settlements land in consecutive blocks, an observer trivially groups them. Time-diffused submission -- spreading sub-trades across a configurable window with jitter -- reduces this correlation.

3. **No execution budgets.** Large split trades can consume significant gas across N sub-settlements, and each sub-settlement may experience different slippage. Without per-sub-trade slippage bounds and a total gas budget, callers cannot control execution cost.

## Specification

### Overview

Adaptive settlement controls are implemented in one layer (SDK-only), with no contract changes:

```
Layer 0 (SDK): ExecutionConfig + VenueRouter
               + DiffusionScheduler + ExecutionOrchestrator
```

The SettlementRegistry from XIP-1 is unmodified. Venue routing is an SDK-side concern. Diffusion timing is caller-controlled.

### ExecutionConfig

`ExecutionConfig` extends XIP-1's `SplitConfig` via composition.

```typescript
interface ExecutionConfig {
  splitConfig: SplitConfig;              // from XIP-1, unchanged
  maxGasBudget: bigint;                  // wei, total across all sub-trades (0 = unlimited)
  maxSlippagePerSubTrade: number;        // basis points per sub-trade
  diffusionWindow: number;               // seconds to spread sub-settlements over
  venuePreference: VenueId[];            // ordered preference list
}

type VenueId = 'public' | 'stealth' | 'shielded';
```

**Default parameters:**

| Parameter               | Default       | Constraint                |
| ----------------------- | ------------- | ------------------------- |
| `maxGasBudget`          | 0 (unlimited) | MUST be >= 0              |
| `maxSlippagePerSubTrade`| 50 bps        | MUST be in [1, 10000]     |
| `diffusionWindow`       | 0 (immediate) | MUST be >= 0              |
| `venuePreference`       | `['public']`  | MUST be non-empty         |

Implementations MUST validate all constraints at plan time and reject invalid configs before proof generation.

### Privacy Model

This XIP does not introduce obfuscation mechanisms (dummy trades, noise, cover traffic). Privacy for split settlements is achieved through venue selection:

- **Public venue** (trust score 0+): No privacy. Sub-trade amounts and timing are fully visible on-chain. Suitable for users who prioritize cost and speed.
- **Stealth venue** (trust score 25+): Recipient privacy via ERC-5564 stealth addresses. Sub-trade amounts are visible but the recipient address is unlinkable.
- **Shielded venue** (trust score 50+): Full privacy via Aztec L2. Sub-trade amounts, recipient, and timing are not visible on-chain. This is the only venue that provides ZK-grade privacy for split settlements.

The privacy guarantee scales with the venue: public provides none, stealth provides recipient unlinkability, shielded provides full confidentiality. There is no intermediate "obfuscation" layer. Users who need trade-size privacy MUST use shielded venues.

**Compliance proof scope:** The per-sub-trade compliance proof attests that the submitting user's aggregate risk score is below the jurisdiction threshold. It does NOT attest that the split transaction structure is legitimate or non-structuring. XIP-1's anti-structuring requirement (pattern detection proof at finalization) provides that guarantee separately.

### VenueRouter

The `VenueRouter` assigns a settlement venue to each sub-trade.

```typescript
interface VenueAssignment {
  index: number;
  venue: VenueId;
  estimatedGas: bigint;
}

interface VenueConstraints {
  trustScore: number;                    // user's current trust score
  gasEstimates: Record<VenueId, bigint>; // gas per sub-trade per venue
}

function assignVenues(
  plan: SplitPlan,
  config: ExecutionConfig,
  constraints: VenueConstraints,
): VenueAssignment[];
```

**Normative constraints (MUST):**

1. A sub-trade MUST NOT be assigned to a venue whose minimum trust score exceeds the user's score. Minimum scores: `public` = 0, `stealth` = 25, `shielded` = 50.
2. The sum of `estimatedGas` across all assignments MUST NOT exceed `maxGasBudget`, unless `maxGasBudget = 0` (unlimited).
3. All sub-trades MUST receive a venue assignment.
4. `assignVenues` MUST be a pure function of its inputs. No hidden state, no network calls.

**Reference algorithm (SHOULD, non-normative):**

1. Sort sub-trades by amount descending.
2. For each sub-trade, attempt to assign the highest-privacy venue from `venuePreference` that the user qualifies for and that fits within the remaining gas budget.
3. If no preferred venue fits, fall back to the cheapest venue the user qualifies for.
4. If no venue fits within the gas budget, return an error.

Implementations MAY substitute their own routing logic provided they satisfy the normative constraints.

**Static gas estimates (v1):**

| Venue    | Estimated gas per sub-trade |
| -------- | --------------------------- |
| public   | 65,000                      |
| stealth  | 150,000                     |
| shielded | 400,000                     |

These are reference values. Implementations SHOULD allow runtime override via `VenueConstraints.gasEstimates`.

### DiffusionScheduler

The `DiffusionScheduler` computes target submission timestamps for each sub-trade.

```typescript
interface ScheduledSubTrade extends SubTrade {
  venue: VenueId;
  targetTimestamp: number;               // seconds relative to T0
}

function scheduleDiffusion(
  assignments: VenueAssignment[],
  plan: SplitPlan,
  diffusionWindow: number,
): ScheduledSubTrade[];
```

**Algorithm:**

1. If `diffusionWindow = 0`, set all `targetTimestamp = 0` (immediate execution). Return.
2. Compute `meanSpacing = diffusionWindow / N`.
3. For each sub-trade i, compute `baseTime = i * meanSpacing`.
4. Add jitter: `targetTimestamp = baseTime + randomUniform(-0.5 * meanSpacing, 0.5 * meanSpacing)`.
5. Clamp all timestamps to `[0, diffusionWindow]`.
6. Sort by `targetTimestamp` ascending.
7. Enforce minimum spacing: walk the sorted list and push any timestamp that is < 12 seconds after its predecessor forward to `predecessor + 12`.

**Normative constraints:**

1. Minimum spacing between any two consecutive timestamps MUST be >= 12 seconds (one Ethereum L1 block). If the diffusion window is too short to accommodate this for all N sub-trades, the scheduler MUST return an error rather than violating the constraint.
2. The schedule is advisory. The SDK MUST emit the schedule but MUST NOT call `sleep()`, `setTimeout()`, or block execution. The caller controls actual submission timing.
3. For N >= 4, the schedule SHOULD NOT be perfectly uniform. The jitter in step 4 is normative, not optional.
4. Implementations MUST use `crypto.getRandomValues` or equivalent CSPRNG for jitter generation.

**Anti-correlation property:**

An observer who sees sub-settlements from the same address within a time window may correlate them. The +/- 50% jitter significantly disrupts uniform spacing patterns but does not eliminate correlation for a sufficiently motivated observer who monitors address activity over the full window. For strong timing privacy, users should route to shielded venues where submission timing is not observable.

### ExecutionOrchestrator

The `ExecutionOrchestrator` composes all planners into a single pipeline.

```typescript
interface ExecutionPlan {
  tradeId: Hex;
  subTrades: ScheduledSubTrade[];
  totalAmount: bigint;
  config: ExecutionConfig;
}

async function planExecution(
  totalAmount: bigint,
  jurisdictionId: JurisdictionId,
  submitter: Address,
  trustScore: number,
  config: ExecutionConfig,
): Promise<ExecutionPlan>;

async function provePlan(
  prover: XochiProver,
  plan: ExecutionPlan,
  baseInput: ComplianceInput | RiskScoreInput,
): Promise<BatchProveResult>;
```

**Pipeline order (normative):**

```
1. SplitPlanner.planSplit()           [XIP-1: produces N sub-trades]
2. VenueRouter.assignVenues()         [assigns venue to each of N]
3. DiffusionScheduler.schedule()      [assigns timestamps to each of N]
4. BatchProver.proveBatch()           [XIP-1: proves all N sequentially]
```

This order is normative. DiffusionScheduler MUST run after VenueRouter (different venues have different finality times that may inform scheduling).

**Execution flow (after planning):**

1. Caller receives `ExecutionPlan`.
2. Caller calls `registry.registerTrade(tradeId, jurisdictionId, N)`.
3. For each sub-trade, at or after its `targetTimestamp`:
   a. Check slippage: if current slippage estimate exceeds `maxSlippagePerSubTrade`, the caller SHOULD delay or retry rather than submit at an unfavorable price.
   b. Route to the assigned venue's settlement path.
   c. Call `oracle.submitCompliance(...)` to get `proofHash`.
   d. Call `registry.recordSubSettlement(tradeId, index, proofHash)`.
4. Caller submits a pattern detection proof (circuit 0x03) per XIP-1's anti-structuring requirement.
5. Caller calls `registry.finalizeTrade(tradeId)`.

The caller MAY deviate from the schedule (submit earlier or later). The schedule is advisory.

### SDK Integration

The SDK MUST export the following from `@xochi/sdk`:

```typescript
// Execution config
export { type ExecutionConfig, type ExecutionPlan, type ScheduledSubTrade }
export { type VenueId, type VenueConstraints, type VenueAssignment }

// Planners
export { assignVenues }
export { scheduleDiffusion }

// Orchestrator
export { planExecution, provePlan }
```

XIP-1 exports (`planSplit`, `proveBatch`, `SettlementRegistryClient`) are unchanged.

## Rationale

### Why composition over inheritance for ExecutionConfig?

`ExecutionConfig` contains a `splitConfig` field rather than extending `SplitConfig`. This preserves XIP-1's `planSplit(totalAmount, jurisdictionId, submitter, config?)` signature. Callers who don't need XIP-2 features continue to use `SplitConfig` directly. The `ExecutionOrchestrator` extracts `splitConfig` and passes it to `planSplit`.

### Why no dummy trades?

An earlier design included "dummy sub-trades" -- zero-value sub-settlements with valid compliance proofs, intended to inflate the observable split count and obscure trade size. This was rejected for three reasons:

1. **Fabricated compliance records.** Each dummy generates a real `ComplianceAttestation` stored permanently in the Oracle's `_proofIndex`. These are genuine on-chain attestations for non-existent economic activity. In a regulatory audit, the protocol would need to explain why its compliance oracle contains proof records for transactions that never happened.

2. **Obfuscation, not privacy.** Dummies provide cover traffic, not cryptographic privacy. The dummy amounts (zero) are trivially distinguishable at the execution layer (DEX, AMM). They are only indistinguishable when routed through shielded venues -- but users with access to shielded venues (trust score >= 50) already have full privacy without dummies.

3. **Structuring risk.** A protocol that generates phantom transactions to obscure real transaction patterns is difficult to distinguish from a protocol that facilitates structuring. The compliance proof does not cover the split structure itself (it attests to user risk score, not transaction legitimacy). Dummies would add a layer of obfuscation on top of an unattested structure.

Users who need trade-size privacy MUST use shielded venues, which provide ZK-grade confidentiality for amounts, recipients, and timing. This is honest about its privacy properties: real privacy from cryptography, not fake privacy from noise.

### Why no venueTag in the SettlementRegistry?

Adding a `venueTag` field to `SubSettlement` was rejected:
- It couples the registry to the current venue taxonomy. Adding a fourth venue would require a registry upgrade.
- XIP-1 established that the registry is venue-agnostic bookkeeping. XIP-2 preserves this property.
- Venue selection is an SDK-side execution concern, not a compliance concern.

If auditors need venue distribution data, the SDK can emit signed off-chain audit logs mapping sub-trade indices to venues.

### Why advisory scheduling rather than SDK-enforced delays?

XIP-1's Rationale explains why delays don't belong in the SDK ("baking delays into the SDK couples it to execution timing"). XIP-2 takes a middle path: the SDK *computes* an optimal schedule but the caller *controls* submission timing. This allows the schedule to be used in different contexts:
- Immediate execution: caller ignores timestamps.
- Manual pacing: caller uses timestamps as guidance.
- Automated execution: a separate orchestration service reads the schedule and submits at the specified times.

### Why 12-second minimum spacing?

Two sub-settlements in the same Ethereum block are trivially correlatable regardless of diffusion scheduling. The 12-second minimum (one L1 block time) ensures each sub-settlement has the opportunity to land in a distinct block. For L2 venues with shorter block times, implementations MAY use a shorter minimum, but the 12-second floor applies to the schedule computation.

### Why +/- 50% jitter?

An earlier design used +/- 10% jitter. At `meanSpacing = 75s`, that produces +/- 7.5 seconds -- too narrow to defeat block-level timing analysis. An analyst who sees N transactions from the same address at roughly uniform intervals trivially identifies them as a batch. The 50% jitter (`+/- 37.5s` at 75s spacing) creates sufficient overlap between adjacent sub-trade windows that the temporal ordering is no longer obvious.

## Backwards Compatibility

No breaking changes.

- The SettlementRegistry contract from XIP-1 is not modified.
- The Oracle contract is not modified.
- The Verifier contract is not modified.
- XIP-1's `SplitConfig`, `planSplit`, and `proveBatch` are not modified.
- `ExecutionConfig` extends `SplitConfig` via composition, not modification.
- Callers using XIP-1 directly (without XIP-2) are unaffected.

XIP-2 is a strict superset of XIP-1's SDK surface. The `ExecutionOrchestrator` calls XIP-1's `planSplit` and `proveBatch` internally.

## Test Cases

### Venue Routing

```
Input:  5 sub-trades, trustScore=60, venuePreference=['shielded', 'stealth', 'public']
        maxGasBudget=0 (unlimited)
Output: All 5 assigned to 'shielded' (user qualifies, no gas constraint)

Input:  5 sub-trades, trustScore=20, venuePreference=['shielded', 'stealth', 'public']
Output: All 5 assigned to 'public' (user only qualifies for public)

Input:  5 sub-trades, trustScore=60, venuePreference=['shielded', 'public']
        maxGasBudget=1,000,000 wei, shielded=400,000/sub, public=65,000/sub
Output: 2 shielded (800,000) + 3 public (195,000) = 995,000 <= budget
        Largest sub-trades assigned to shielded

Input:  5 sub-trades, trustScore=60, maxGasBudget=100,000 wei
        (no venue fits all 5 even at cheapest: 5 * 65,000 = 325,000)
Output: Error -- gas budget insufficient
```

### Diffusion Scheduling

```
Input:  5 sub-trades, diffusionWindow=300 seconds
Output: 5 timestamps in [0, 300], minimum spacing >= 12s
        Mean spacing ~60s, jitter +/- ~30s
        Timestamps are NOT uniformly spaced

Input:  5 sub-trades, diffusionWindow=0
Output: All timestamps = 0

Input:  10 sub-trades, diffusionWindow=100 seconds
        (9 gaps * 12s minimum = 108s > 100s window)
Output: Error -- diffusion window too short for minimum spacing
```

### End-to-End Orchestration

```
1. Alice configures: totalAmount=500 ETH, splitThreshold=100 ETH,
   venuePreference=['shielded', 'stealth'],
   diffusionWindow=600, trustScore=60

2. SplitPlanner: N=5 sub-trades of 100 ETH each

3. VenueRouter: trustScore=60 qualifies for shielded
   All 5 assigned to 'shielded' (no gas constraint)

4. DiffusionScheduler: 600s / 5 = 120s mean spacing, jitter +/- 60s
   Timestamps: [0, 95, 230, 310, 485] (approx, randomized)

5. BatchProver: 5 proofs generated sequentially

6. Alice registers trade with subTradeCount=5
7. Alice submits each sub-trade near its target timestamp via shielded venue
8. Alice submits pattern detection proof (circuit 0x03)
9. Alice calls finalizeTrade
10. Observer sees: 5 sub-settlements, all shielded
    Amounts, timing, and recipient are NOT visible on-chain (Aztec L2 privacy)
```

## Reference Implementation

Deferred until status = Approved. Implementation will span:

- `@xochi/sdk` -- `src/execution-config.ts` (types), `src/venue-router.ts`, `src/diffusion-scheduler.ts`, `src/execution-orchestrator.ts`
- `@xochi/sdk` -- `test/venue-router.test.ts`, `test/diffusion-scheduler.test.ts`, `test/execution-orchestrator.test.ts`

No contract changes. No modifications to `erc-xochi-zkp`.

## Security Considerations

### Structuring risk

XIP-2 does not introduce new structuring risk beyond XIP-1. Venue routing and diffusion scheduling do not change the split structure (number or size of sub-trades). XIP-1's anti-structuring mitigations (pattern proof requirement, minimum split threshold, on-chain trade linkage) apply to all split settlements regardless of XIP-2 configuration.

Diffusion scheduling spaces sub-settlements over time, which could superficially resemble structuring behavior (spreading transactions to avoid detection). However, the transactions remain linked via the SettlementRegistry's `tradeId`, and finalization requires a pattern detection proof. Unlike ad-hoc structuring, diffused split settlements are explicitly grouped and auditable.

### Venue routing reveals trust score lower bound

If any sub-trade is routed to a shielded venue, an observer knows the user's trust score is >= 50. If any is routed to stealth, the observer knows the score is >= 25. This is not new information leakage -- the same inference applies to any user of those venues without XIP-2. Venue routing does not increase the trust score information available to observers.

### Diffusion timing correlation

Sub-settlements from the same address within a diffusion window may be correlated by an observer regardless of scheduling jitter. The +/- 50% jitter disrupts uniform spacing patterns but does not eliminate correlation for a sufficiently motivated observer who monitors address activity over the full window. The 12-second minimum spacing ensures sub-settlements do not land in the same block (which would be trivially groupable).

For strong timing privacy, users MUST use shielded venues where submission timing is not observable on-chain.

### Gas budget exhaustion during diffused execution

If gas prices spike during execution of a diffused plan (sub-trades submitted over minutes or hours), later sub-trades may exceed the gas budget computed at plan time. Implementations SHOULD re-estimate gas before each submission and warn the caller if the remaining budget is insufficient. Implementations MUST NOT silently skip sub-trades -- either submit or explicitly report the budget shortfall to the caller.

### Front-running diffused sub-settlements

Each sub-trade retains XIP-1's submitter anti-front-running binding (`submitter == msg.sender`). Diffusion does not weaken this protection. An attacker who observes a pending sub-settlement in the mempool cannot submit it from a different address, regardless of timing.

### Compliance proof scope

The per-sub-trade compliance proof attests that the submitting user's aggregate risk score is below the jurisdiction threshold. It does NOT attest that the split structure, venue selection, or timing schedule is legitimate. XIP-1's anti-structuring requirement (pattern detection proof at finalization) provides structural legitimacy attestation separately. Venue routing and diffusion scheduling are execution-layer optimizations that do not affect the compliance proof's meaning.

## Failure Modes

### VenueRouter produces infeasible plan

User has trust score 20 (only `public` eligible) but sets `venuePreference=['shielded']`. No venue qualifies. The VenueRouter MUST NOT silently downgrade -- it MUST return an error indicating no viable venue assignment exists. The caller can then adjust `venuePreference` to include venues they qualify for.

### Partial venue failure

Aztec L2 becomes unreachable mid-execution. Sub-trades assigned to the shielded venue cannot be submitted. The SettlementRegistry does not know about venues, so no registry-level recovery is needed. The caller has three options:
1. Wait for the venue to recover and submit later (within the 7-day trade expiry from XIP-1).
2. Regenerate proofs for the affected sub-trades and submit to a different venue.
3. Abandon the trade and let it expire.

Option 2 requires new proof generation because the caller may want to use a different venue (and thus a different settlement path), but the compliance proof itself is venue-agnostic -- the same proof is valid regardless of where settlement occurs.

### Diffusion window too short

If `diffusionWindow < (N - 1) * 12` seconds, the DiffusionScheduler cannot satisfy the 12-second minimum spacing constraint. It MUST return an error. The caller can either increase the diffusion window or reduce the sub-trade count (by adjusting `splitThreshold`).

### Gas price spike during diffusion

A plan computed with gas estimates at time T0 may become infeasible at time T0 + diffusionWindow if gas prices spike. The `maxGasBudget` is a planning-time constraint only. The SDK does not enforce it at submission time (it has no control over when the caller submits). Implementations SHOULD provide a `checkBudget()` helper that the caller can invoke before each submission to verify the remaining budget is sufficient at current gas prices.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
