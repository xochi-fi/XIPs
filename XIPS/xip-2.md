---
xip: 2
title: Split Settlement Execution Controls
description: SDK planning for XIP-1 split trades: per-leg venue routing, diffusion scheduling, gas and slippage budgets
author: DROO (@DROOdotFOO)
discussions-to: https://github.com/xochi-fi/XIPs/issues/7
status: Draft
type: Interface
created: 2026-04-15
requires: 1
---

## Abstract

XIP-1 splits a large trade into N legs but leaves execution to the caller. This XIP specifies the `@xochi/sdk` planning surface for those legs: a settlement venue per leg (public, stealth, shielded), an advisory submission schedule, a gas budget in gas units covering the whole trade, and slippage bounds. All changes are SDK-side except one settlement-server check (Trust boundaries); there are no contract changes. Nothing adapts automatically: every input is fixed at plan time, and the caller re-plans between legs with the same pure functions. The planners ship in `@xochi/sdk` 0.5.0; requirements the shipped code does not meet are marked "Not yet implemented".

## Motivation

XIP-1's `SplitConfig` controls how many legs a trade splits into. It does not control where each leg settles, when it is submitted, or what the trade costs.

1. **Venue selection.** Xochi has three settlement paths: public (L1 transfer), stealth (ERC-5564 on L1), and shielded (Aztec L2). XIP-1 treats every leg the same. Routing legs to higher-privacy venues hides leg amounts or recipients that a public leg exposes.
2. **Temporal correlation.** Legs landing in consecutive blocks are trivially grouped. Spreading submissions over a window with jitter weakens that signal for observers who do not already link the legs (see Privacy Model: on-chain registry calls link them anyway).
3. **Execution cost.** Every leg carries an Oracle compliance submission (about 3.2M gas) and a registry call, and each leg can slip. Callers need a whole-trade gas budget and a bound on total price deviation, not only per-leg checks.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

Code references are to `origin/main` of [xochi-fi/xochi-sdk](https://github.com/xochi-fi/xochi-sdk) (`packages/sdk/src/`), [xochi-fi/ERC-8262](https://github.com/xochi-fi/ERC-8262) (`src/`), and [xochi-fi/xochi](https://github.com/xochi-fi/xochi) (`packages/worker-lib/src/`).

### Implementation status

| Requirement                                                       | Status              | Where                                                                                       |
| ----------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------- |
| `planExecution`, `assignVenues`, `scheduleDiffusion`, `provePlan` | Implemented         | `execution-orchestrator.ts`, `venue-router.ts`, `diffusion-scheduler.ts`, `batch-prover.ts` |
| Venue gates from `tiers.ts` (stealth 0, shielded 50)              | Implemented         | `venue-router.ts` `VENUE_MIN_SCORES`                                                        |
| Router never assigns outside `venuePreference`                    | Implemented         | `venue-router.ts` `assignVenues`                                                            |
| Router feasibility reserve                                        | Not yet implemented | `venue-router.ts` is greedy without a reserve                                               |
| `maxGasUnits` (whole-trade gas, overhead included)                | Not yet implemented | shipped field is `maxGasBudget`, venue gas only                                             |
| `VenueConstraints.overheadGas`, `DEFAULT_OVERHEAD_GAS`            | Not yet implemented |                                                                                             |
| Reserve-the-gaps stratified scheduler, index order preserved      | Implemented         | `diffusion-scheduler.ts`                                                                    |
| `diffusionWindow <= MAX_DIFFUSION_WINDOW`                         | Not yet implemented | no upper bound in `validateExecutionConfig`                                                 |
| `maxTotalDeviationBps`, `referenceOut`                            | Not yet implemented |                                                                                             |
| Per-leg slippage check                                            | Caller-enforced     | SDK validates the range only                                                                |
| Server refuses shielded below score 50                            | Not yet implemented | `quote.ts` only advertises it (see Trust boundaries)                                        |

### Overview

```
planExecution()                     execution-orchestrator.ts, synchronous, pure except CSPRNG
  1. planSplit()                    XIP-1: N legs (N == 1 when no split is needed)
  2. assignVenues()                 venue per leg
  3. scheduleDiffusion()            advisory targetTimestamp per leg
then, per leg, by the caller:       prove, submitCompliance, settle on venue, recordSubSettlement
```

The `SettlementRegistry`, `ERC8262Oracle` and `ERC8262Verifier` contracts are unmodified. Venue choice is not recorded on-chain.

### ExecutionConfig

Shipped (`execution-orchestrator.ts`):

```typescript
interface ExecutionConfig {
  splitConfig: SplitConfig; // XIP-1, unchanged
  maxGasBudget: bigint; // gas units, venue settlement gas only (0 = unlimited)
  maxSlippagePerSubTrade: number; // basis points per leg
  diffusionWindow: number; // seconds
  venuePreference: VenueId[]; // allowed venues; order is ignored
}

type VenueId = "public" | "stealth" | "shielded";

const DEFAULT_EXECUTION_CONFIG: ExecutionConfig = {
  splitConfig: DEFAULT_SPLIT_CONFIG,
  maxGasBudget: 0n,
  maxSlippagePerSubTrade: 50,
  diffusionWindow: 0,
  venuePreference: ["public"],
};
```

Required changes (Not yet implemented):

```typescript
interface ExecutionConfig {
  // maxGasBudget is renamed and widened:
  maxGasUnits: bigint; // gas units, whole trade including overhead (0 = unlimited)
  maxTotalDeviationBps: number; // bound on cumulative shortfall vs referenceOut
  // other fields unchanged
}
// DEFAULT_EXECUTION_CONFIG gains maxGasUnits: 0n, maxTotalDeviationBps: 100.
```

| Parameter                | Default       | Constraint                                         | Enforced by                                                                    |
| ------------------------ | ------------- | -------------------------------------------------- | ------------------------------------------------------------------------------ |
| `maxGasUnits`            | 0 (unlimited) | MUST be >= 0                                       | SDK (`maxGasBudget >= 0` today)                                                |
| `maxSlippagePerSubTrade` | 50            | MUST be an integer in [1, 10000]                   | SDK checks the range; integer and NaN not checked                              |
| `maxTotalDeviationBps`   | 100           | MUST be an integer in [1, 10000]                   | Not yet implemented                                                            |
| `diffusionWindow`        | 0 (immediate) | MUST be finite, >= 0 and <= `MAX_DIFFUSION_WINDOW` | SDK (`scheduleDiffusion` for finite and >= 0); upper bound Not yet implemented |
| `venuePreference`        | `["public"]`  | MUST be non-empty, known `VenueId`s only           | SDK (`assignVenues`)                                                           |

`MAX_DIFFUSION_WINDOW = TRADE_TTL - MAX_PROOF_AGE = 604800 - 3600 = 601200` seconds, from `SettlementRegistry.sol` (`TRADE_TTL = 7 days`) and `ERC8262Oracle.sol` (`MAX_PROOF_AGE = 1 hours`). It leaves one hour after the last scheduled leg to record it and to prove, submit and finalize the pattern proof before the trade expires.

`planExecution` MUST validate the config before returning a plan, and therefore before any proof is generated.

### Gas model

All gas quantities in this XIP are gas units, not wei. The wei cost is gas units times the gas price the caller pays at submission; gas price is outside this XIP.

Whole-trade estimate for a plan with N legs:

```
N == 1:  total = submitCompliance + venueGas[0]
N >= 2:  total = overhead(N) + sum(venueGas[i])
         overhead(N) = registerTrade + submitPattern + finalizeTradeBase
                     + N * (submitCompliance + recordSubSettlement + finalizeTradePerLeg)
```

`venueGas[i] = constraints.gasEstimates[venue(i)]`. When `maxGasUnits > 0`, `planExecution` MUST compute `venueBudget = maxGasUnits - overheadFor(N)` (the N == 1 or N >= 2 overhead above) and MUST fail if `venueBudget < N * cheapestQualified` (see VenueRouter). It then passes `venueBudget` to `assignVenues`. It MUST NOT pass a `venueBudget <= 0`, which `assignVenues` would read as unlimited. Not yet implemented: shipped `planExecution` passes `maxGasBudget` to `assignVenues` unchanged, so the budget covers venue gas only.

Overhead defaults (Not yet implemented as `DEFAULT_OVERHEAD_GAS`; callers MUST be able to override them through `VenueConstraints.overheadGas`):

| Field                 | Default   | Source                                                                                           |
| --------------------- | --------- | ------------------------------------------------------------------------------------------------ |
| `submitCompliance`    | 3,246,000 | ERC-8262 `.gas-snapshot` `test_gas_submitCompliance_COMPLIANCE` 3,079,987, real UltraHonk proof  |
| `recordSubSettlement` | 180,000   | forge gas report of `SettlementRegistryTest` on ERC-8262 `origin/main`, max 156,792              |
| `registerTrade`       | 139,000   | same gas report, max 116,108                                                                     |
| `submitPattern`       | 3,004,000 | `.gas-snapshot` `test_gas_submitCompliance_PATTERN` 2,836,418                                    |
| `finalizeTradeBase`   | 144,000   | same gas report, max 117,728 at N = 2; base + 2 * per-leg = 145,000 covers it                    |
| `finalizeTradePerLeg` | 500       | `computeSettlementRoot` reads one slot per leg; about 455 gas per leg measured from N = 2 to 100 |

Each default is the measured execution gas plus 21,000 intrinsic gas plus calldata at 16 gas per byte (a 9,060-byte `submitCompliance` call for 0x01; 9,124 bytes for 0x03), rounded up to the next 1,000. The compliance figure is for proof type 0x01; 0x07 and 0x09 were not benchmarked and callers using them SHOULD override it. With these defaults `overhead(N) = 3,287,000 + 3,426,500 * N`.

Venue gas defaults, shipped as `DEFAULT_GAS_ESTIMATES` (`venue-router.ts`):

| Venue    | Gas per leg |
| -------- | ----------- |
| public   | 65,000      |
| stealth  | 150,000     |
| shielded | 400,000     |

These are unmeasured reference values for the venue settlement transaction only. Callers SHOULD pass current estimates in `VenueConstraints.gasEstimates`. At N = 5 the overhead is 20,419,500 gas; venue choice moves the total by at most 5 * 335,000 = 1,675,000.

### Privacy Model

This XIP adds no obfuscation (no dummy legs, no cover traffic). What each venue hides is limited to the value transfer of a leg:

| Venue    | Gate (`tiers.ts`)                                  | Value transfer of the leg                                           |
| -------- | -------------------------------------------------- | ------------------------------------------------------------------- |
| public   | score >= 0                                         | amount, recipient and timing visible on L1                          |
| stealth  | score >= 0 (`tiers.ts:199`)                        | amount and timing visible; recipient is a one-time ERC-5564 address |
| shielded | score >= 50 (`SHIELDED_MIN_SCORE`, `tiers.ts:170`) | amount and recipient hidden on Aztec L2                             |

On every venue, including shielded, the compliance and registry transactions of a split trade are public L1 transactions sent from the subject's address (the Oracle requires the proof's `submitter == msg.sender`):

- `registerTrade` emits `TradeRegistered(tradeId, subject, jurisdictionId, subTradeCount)` before leg 0, so the leg count is public.
- Each leg's `submitCompliance` records an attestation for the subject with its block time.
- Each leg's `recordSubSettlement` emits `SubSettlementRecorded(tradeId, index, proofHash)` with its block time.

Leg count, per-leg timing to the block, the subject address and the linkage between legs are therefore public on every venue. No venue hides that a split trade happened or when its legs were recorded. Callers who need the trade size hidden MUST route every leg to shielded; that hides leg amounts, not the leg count.

The per-leg compliance proof attests that the subject's aggregate risk score is below the jurisdiction threshold. It does not bind the leg amount, the venue, or the value transfer; the registry stores proof hashes, not transfers. What the XIP-1 pattern proof (0x03) does and does not establish is specified in [XIP-1](./xip-1.md).

### VenueRouter

Shipped signature (`venue-router.ts`):

```typescript
interface VenueAssignment {
  index: number;
  venue: VenueId;
  estimatedGas: bigint; // gas units
}

interface VenueConstraints {
  trustScore: number; // caller-supplied; not bound to anything (see Trust boundaries)
  gasEstimates: Record<VenueId, bigint>;
}

function assignVenues(
  subTrades: SubTrade[],
  venuePreference: VenueId[],
  constraints: VenueConstraints,
  maxGasBudget: bigint, // venue gas budget in gas units; 0 = unlimited
): VenueAssignment[]; // synchronous, sorted by index
```

Required change (Not yet implemented): `VenueConstraints` gains `overheadGas?: OverheadGas`, an object with the six fields of the overhead table, defaulting to `DEFAULT_OVERHEAD_GAS`.

**Normative constraints:**

1. A leg MUST NOT be assigned a venue whose minimum trust score exceeds `constraints.trustScore`. Minimums are `VENUE_MIN_SCORES`, derived from `tiers.ts`: `public` 0, `stealth` 0, `shielded` 50. Enforced by SDK (advisory; the score is caller-supplied).
2. A leg MUST NOT be assigned a venue outside `venuePreference`. The router MUST NOT fall back to a venue the caller did not list. Enforced by SDK.
3. When `maxGasBudget > 0`, the sum of `estimatedGas` MUST NOT exceed `maxGasBudget`. Enforced by SDK.
4. Every leg MUST receive an assignment, or the call MUST throw. Enforced by SDK.
5. The router MUST throw if and only if no assignment satisfies constraints 1-3: no listed venue qualifies, or `maxGasBudget > 0` and `N * cheapestQualified > maxGasBudget`, where `cheapestQualified` is the lowest `gasEstimates` value among listed venues the score qualifies for. Not yet implemented: the shipped greedy router also throws on some feasible budgets (Test Cases, R4).
6. `assignVenues` MUST be a pure function of its inputs: no hidden state, no network calls. Enforced by SDK.

**Algorithm (normative):**

1. Let `Q` be the venues in `venuePreference` that the score qualifies for, ordered shielded, stealth, public. `venuePreference` order is ignored. If `Q` is empty, throw.
2. Sort legs by amount, descending; ties keep index order (stable sort).
3. If `maxGasBudget == 0`, assign every leg `Q[0]` and return, sorted by index.
4. Let `c = min(gasEstimates[v] for v in Q)`. If `N * c > maxGasBudget`, throw.
5. Walk the sorted legs with `remaining = maxGasBudget`. For the k-th leg (k from 1), with `r = N - k` legs still to place, assign the first `v` in `Q` with `gasEstimates[v] + r * c <= remaining`, then subtract `gasEstimates[v]` from `remaining`. Step 4 guarantees such a `v` exists.
6. Return assignments sorted by index.

Shipped `assignVenues` implements steps 1-3 and 6, and in step 5 picks the first `v` with `gasEstimates[v] <= remaining`, with no reserve and no step 4.

### DiffusionScheduler

Shipped signature and algorithm (`diffusion-scheduler.ts`):

```typescript
interface ScheduledSubTrade extends SubTrade {
  venue: VenueId;
  targetTimestamp: number; // whole seconds after T0
}

function scheduleDiffusion(
  subTrades: SubTrade[],
  venues: VenueId[], // venues[i] belongs to subTrades[i]; attached, not used for timing
  diffusionWindow: number,
): ScheduledSubTrade[]; // synchronous, index order
```

**Algorithm (normative):**

1. If `subTrades` is empty, return `[]`. If `venues.length != subTrades.length`, or `diffusionWindow` is not finite or is negative, throw.
2. If `diffusionWindow == 0`, every `targetTimestamp` is 0. Return.
3. Let `n = subTrades.length` and `minWindow = (n - 1) * 12`. If `diffusionWindow < minWindow`, throw.
4. `slack = floor(diffusionWindow) - minWindow`.
5. Draw `n` 32-bit words `r_i` from `crypto.getRandomValues`; `u_i = r_i / 2^32`, in [0, 1).
6. `targetTimestamp_i = floor((i + u_i) * slack / n) + 12 * i`.

**Normative properties:**

1. Consecutive timestamps MUST differ by at least 12 seconds. Enforced by SDK (by construction: offsets are non-decreasing and the 12 s gaps are reserved).
2. Every timestamp MUST be in `[0, floor(diffusionWindow)]`. Enforced by SDK (by construction).
3. `targetTimestamp` MUST be non-decreasing in leg index, so submitting in schedule order is submitting in index order. Enforced by SDK (by construction). The Oracle's timestamp ratchet depends on this (Execution flow, rule 4).
4. The scheduler MUST throw if and only if step 1 or step 3 fails; every window it accepts can be scheduled. Enforced by SDK.
5. Jitter MUST come from `crypto.getRandomValues` or an equivalent CSPRNG. Enforced by SDK.
6. The schedule is advisory. The SDK MUST NOT sleep, set timers, or block. The caller controls submission time and MAY submit late, but MUST submit legs in index order (Execution flow, rule 4).

### ExecutionOrchestrator

Shipped (`execution-orchestrator.ts`, `batch-prover.ts`):

```typescript
interface ExecutionPlan {
  tradeId: Hex;
  subTrades: ScheduledSubTrade[];
  totalAmount: bigint;
  config: ExecutionConfig; // resolved, defaults applied
}

function planExecution(
  totalAmount: bigint,
  jurisdictionId: JurisdictionId,
  submitter: Address,
  constraints: VenueConstraints,
  config?: Partial<ExecutionConfig>, // splitConfig is merged field by field
): ExecutionPlan; // synchronous

async function provePlan(
  prover: ERC8262Prover,
  plan: ExecutionPlan,
  baseInput: ComplianceInput | RiskScoreInput,
): Promise<BatchProveResult>;
```

`planExecution` runs `planSplit`, `assignVenues`, `scheduleDiffusion` in that order. Required change (Not yet implemented): `ExecutionPlan` gains `referenceOut: bigint`, the caller's quoted output for `totalAmount` at plan time, passed in through a new optional `planExecution` argument.

`provePlan` proves every leg up front, sequentially, with the same `baseInput`. It is valid only when every leg is submitted within `MAX_PROOF_AGE` of its proof timestamp, which in practice means `diffusionWindow == 0`. Callers MUST NOT use `provePlan` when `diffusionWindow > 0`; the SDK does not reject it (Not yet implemented). Legs MUST use a compliance proof type (0x01, 0x07 or 0x09); `provePlan` with a `RiskScoreInput` produces 0x02 proofs that `recordSubSettlement` rejects with `NonComplianceProofType` (registry-enforced).

### Execution flow

**N == 1.** `planSplit` returns one leg when `totalAmount <= splitThreshold` or when no split meets `minSubTradeSize`. The caller MUST NOT call `registerTrade`: the registry requires `subTradeCount` in [2, 100] and reverts `InvalidSubTradeCount(1)` (contract-enforced). The caller proves and submits the single leg directly to its venue, with T0 the time `planExecution` returned. No registry entry exists and this flow needs no pattern proof.

**N >= 2.**

1. Caller calls `planExecution` and records `referenceOut`.
2. Caller calls `registry.registerTrade(tradeId, jurisdictionId, N)`. T0 for `targetTimestamp` is the block time of this call. Leg attestations submitted before it revert at record time (`AttestationPredatesTrade`, contract-enforced).
3. For each leg i, in index order, at or after `targetTimestamp_i`:
   1. Pre-leg checks (rules 5-7 below). If any fails, take the re-plan or abort path.
   2. Generate the leg's compliance proof now (just in time), e.g. `prover.proveCompliance(baseInput)`, dated from the chain head (`chainProofTimestamp`).
   3. `oracle.submitCompliance(jurisdictionId, proofType, proof, publicInputs, providerSetHash)` to obtain `proofHash`.
   4. Settle the leg's value transfer on its assigned venue.
   5. `registry.recordSubSettlement(tradeId, i, proofHash)`.
4. Generate and submit a pattern proof (0x03) per XIP-1, bound to `computeSettlementRoot(tradeId)`.
5. `registry.finalizeTrade(tradeId, patternProofHash, patternPublicInputs)`.

**Rules:**

1. Every `recordSubSettlement` and the `finalizeTrade` MUST be mined by `createdAt + TRADE_TTL`; after that they revert `TradeExpiredError` (contract-enforced). `diffusionWindow <= MAX_DIFFUSION_WINDOW` keeps the schedule one hour inside that limit.
2. `submitCompliance` MUST land within `MAX_PROOF_AGE` (3600 s) of the proof's timestamp; the Oracle reverts `ProofTimestampStale` otherwise (Oracle-enforced). When `diffusionWindow > 0`, each leg's proof MUST be generated just before its submission (step 3.2). When `diffusionWindow == 0`, `provePlan` MAY prove all legs up front; re-proving leg k then also requires re-proving every later leg (rule 4). This XIP picks just-in-time proving per leg over bounding `diffusionWindow` by the proof age; see Rationale.
3. A proof timestamp MUST NOT be in the future of the target chain; the Oracle reverts `ProofTimestampInFuture`. `chainProofTimestamp` dates proofs from the chain head.
4. Compliance proof timestamps MUST be non-decreasing in submission order per (subject, jurisdiction); the Oracle ratchets on them and reverts `ProofTimestampNotMonotonic` (Oracle-enforced). Callers MUST therefore submit compliance proofs for one (subject, jurisdiction) in the order they were generated, across all of their trades, and the legs of one trade in index order. Two concurrent split trades by the same subject in the same jurisdiction MUST NOT interleave proving and submission.
5. **Per-leg slippage.** A leg MUST NOT be submitted if its quoted slippage exceeds `maxSlippagePerSubTrade`. The caller MAY wait and re-quote. Caller-enforced; the SDK validates the range only.
6. **Total deviation.** Let `E_i = referenceOut * amount_i / totalAmount` (integer division), `R_i` the output actually received for settled leg i, and `Q_k` the minimum output quoted for leg k. Before leg k the caller MUST check `(sum over i < k of (E_i - R_i) + (E_k - Q_k)) * 10000 <= maxTotalDeviationBps * referenceOut`. If it fails, leg k MUST NOT be submitted and the caller MUST abort the remaining legs. Caller-enforced; the fields are Not yet implemented in the SDK.
7. **Venue and budget.** Before leg k the caller MUST re-check that the current trust score still qualifies the leg's venue and that the remaining venue budget covers the remaining legs at current `gasEstimates`. If not, the caller MUST re-plan or abort; it MUST NOT submit the leg to a venue the score no longer qualifies for.
8. The trade cannot be resized after `registerTrade`: `subTradeCount` is fixed on-chain.

**Re-plan.** Call `assignVenues(remainingLegs, venuePreference, freshConstraints, remainingVenueBudget)`, where `remainingVenueBudget` is the plan's `venueBudget` minus the venue gas of legs already settled. The result keeps the leg count and amounts, so the registry is unaffected (it does not know venues). Keep the existing `targetTimestamp`s. If `assignVenues` throws, abort. The SDK does not perform this step for the caller.

**Abort.** Stop submitting legs. Settled legs are not reversed. The trade cannot be finalized; after `createdAt + TRADE_TTL` anyone can call `expireTrade`, which sets `expired`, never `finalized`. A new trade for the unsettled remainder needs a new `tradeId`.

### SDK exports

Shipped in `@xochi/sdk` (`index.ts`):

```typescript
export {
  assignVenues,
  DEFAULT_GAS_ESTIMATES,
  VENUE_MIN_SCORES,
} from "./venue-router.js";
export { scheduleDiffusion } from "./diffusion-scheduler.js";
export {
  planExecution,
  DEFAULT_EXECUTION_CONFIG,
} from "./execution-orchestrator.js";
export { proveBatch, provePlan } from "./batch-prover.js";
export type {
  VenueId,
  VenueAssignment,
  VenueConstraints,
} from "./venue-router.js";
export type { ScheduledSubTrade } from "./diffusion-scheduler.js";
export type {
  ExecutionConfig,
  ExecutionPlan,
} from "./execution-orchestrator.js";
```

Not yet implemented: `DEFAULT_OVERHEAD_GAS` and `type OverheadGas`. No other helper is part of this XIP.

### Trust boundaries

| Requirement                                               | Enforced by                                                                            |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Venue gate on trust score                                 | SDK, advisory only; server partially (below)                                           |
| Only listed venues; gas budget                            | SDK                                                                                    |
| Feasibility reserve; overhead in the budget               | SDK, Not yet implemented                                                               |
| 12 s spacing, timestamps inside the window, index order   | SDK                                                                                    |
| `diffusionWindow <= MAX_DIFFUSION_WINDOW`                 | Not yet implemented (SDK); registry expiry reverts late legs                           |
| N == 1 skips the registry                                 | Contract (`InvalidSubTradeCount`)                                                      |
| Leg proof age, future timestamps, ratchet                 | Oracle (`ProofTimestampStale`, `ProofTimestampInFuture`, `ProofTimestampNotMonotonic`) |
| Leg proof is a compliance type, unused, post-registration | Contract (`recordSubSettlement`)                                                       |
| `provePlan` not used when `diffusionWindow > 0`           | Caller; nothing enforces it                                                            |
| Per-leg slippage, total deviation, re-plan/abort          | Caller; nothing enforces it                                                            |

Server enforcement of the venue gate (xochi `origin/main`): the quote handler resolves the score from its own attestation records and never trusts a client-supplied `trust_score` (`handlers/intents/quote.ts` `resolveQuoteTier`). It refuses every shielded quote while `SHIELDED_SETTLEMENT_LIVE` is off (`settlement-policy.ts` `checkSettlementAllowed`), which is the default in repository configs. With the flag on, refusing a shielded quote for a score below `SHIELDED_MIN_SCORE` is Not yet implemented: the score only sets the advertised `settlement_options` (`quote.ts:927`). The settlement server is the component that MUST enforce the gate; the SDK check is advisory.

Caller-supplied inputs and what binds them:

- `trustScore`: nothing in the SDK. The server binds the score it uses to its own attestation records.
- `gasEstimates`, `overheadGas`: nothing; a wrong estimate only mis-sizes the caller's own budget.
- `referenceOut`, quotes `Q_k`, received `R_i`: nothing; the deviation bound protects the caller and is only as good as these values.
- Leg proof timestamp: bound by the proof (public input) and checked by the Oracle.

## Rationale

### Why composition for ExecutionConfig?

`ExecutionConfig` holds a `splitConfig` field rather than extending `SplitConfig`, so XIP-1's `planSplit(totalAmount, jurisdictionId, submitter, config?)` is unchanged and callers without XIP-2 needs keep using it.

### Why gas units, and why rename the field?

A wei budget needs a gas price, and the price changes across the diffusion window while the gas units of each call do not. Gas units are fixed at plan time; the caller prices each transaction when it sends it. The shipped `maxGasBudget` was already compared against gas units, but excluded the Oracle and registry calls, which are over 90% of the cost (91% at N = 5 all shielded, 98% all public). Including them changes the meaning of the number, so the field is renamed to `maxGasUnits` rather than silently reinterpreted: a budget sized for venue gas alone would otherwise start failing.

### Why a feasibility reserve?

The greedy router gives the largest legs the most private venue that fits the remaining budget, and can spend the budget before the last legs are placed. Reserving `r * c` for the `r` legs still to place keeps the cheapest qualifying venue affordable for each of them. If `N * c <= budget` the invariant `remaining >= r * c` holds at every step, so the router never fails on a budget some assignment satisfies; if `N * c > budget` no assignment exists. Largest legs still get first choice.

### Why just-in-time proving rather than a window bound?

The Oracle rejects a compliance proof more than one hour old. Bounding `diffusionWindow` by `MAX_PROOF_AGE` minus proving time would cap diffusion at under an hour and still strand every later leg after any retry, because re-proving a failed leg gives it a newer timestamp than the pre-proved legs after it, which the ratchet then rejects. Proving each leg just before it is submitted keeps every proof fresh, keeps timestamps in submission order, and lets a failed leg be re-proved without touching the others. The cost is proving time between legs, which the advisory schedule absorbs.

### Why advisory scheduling?

XIP-1 keeps delays out of the SDK. The SDK computes a schedule and the caller controls submission: immediate (ignore timestamps), manual pacing, or a separate executor service.

### Why 12-second minimum spacing?

Two legs in one L1 block are trivially grouped. 12 seconds is one L1 slot. The floor applies to the schedule for every venue.

### Why reserve-the-gaps stratified jitter?

The earlier draft jittered each slot by +/- 50% of the mean spacing, clamped to the window, sorted, then pushed timestamps forward to restore the 12 s gap. That could push past the window for windows its own check accepted, made the error depend on the random draw, and could reorder legs, which breaks the Oracle ratchet. Reserving the `(n - 1) * 12` seconds first and drawing one offset per stratum of the remaining slack keeps every accepted window schedulable, keeps index order, and still spreads offsets across the whole window.

### Why no dummy trades?

Zero-value dummy legs with real compliance proofs were rejected:

1. **Fabricated compliance records.** Each dummy creates a real attestation in the Oracle for activity that never happened.
2. **No privacy gain.** Dummies are cover traffic. Zero amounts are distinguishable at the execution layer on public and stealth venues, and the leg count they would inflate is already public through `TradeRegistered`.
3. **Structuring risk.** Generating phantom transactions to obscure real patterns is hard to distinguish from facilitating structuring.

### Why no venue tag in the SettlementRegistry?

A `venueTag` on `SubSettlement` would couple the registry to the venue taxonomy (a fourth venue would need a registry upgrade) and is not a compliance concern. If auditors need venue data, the SDK caller can keep signed off-chain logs mapping leg indices to venues.

### Why the retitle?

The earlier title was "Adaptive Settlement Controls". Nothing in the shipped code adapts: trust score, gas estimates and prices are inputs fixed at plan time, and no component reacts to feedback. Re-planning between legs is a caller action using the same pure functions. The title now says what the XIP specifies.

## Backwards Compatibility

- No contract changes: `SettlementRegistry`, `ERC8262Oracle` and `ERC8262Verifier` are unmodified.
- XIP-1's `SplitConfig`, `planSplit` and `proveBatch` are unmodified.
- Against shipped `@xochi/sdk` 0.5.0:
  - Renaming `maxGasBudget` to `maxGasUnits` and adding overhead is breaking. Callers MUST raise budgets by `overhead(N)` when migrating.
  - The feasibility reserve only turns some throws into valid plans. For a given venue budget on which the shipped greedy router succeeds, the reserve router returns the same assignment: each greedy choice leaves enough for the remaining legs, so it also passes the reserve check, and no higher-privacy venue passes either check.
  - `maxTotalDeviationBps` and `overheadGas` are additions with defaults; `referenceOut` is a new optional `planExecution` argument. Existing calls keep working.
- Against the draft merged in #2: `assignVenues`, `scheduleDiffusion` and `planExecution` signatures now match shipped code (synchronous, `VenueConstraints` instead of a bare `trustScore`); `checkBudget` is removed; `finalizeTrade` takes three arguments; stealth is ungated; the scheduler algorithm is replaced.

## Test Cases

Amounts are in wei. `D = DEFAULT_GAS_ESTIMATES`; overheads use the default overhead table, `overhead(5) = 20,419,500`. Unless stated, `totalAmount = 450e18 + 3`, default `SplitConfig` (`splitThreshold = 100e18`), so `planSplit` gives N = 5 with amounts `[90e18, 90e18, 90e18, 90e18, 90e18 + 3]` and leg 4 is the largest.

### Venue routing

```
R1  score=60, venuePreference=[shielded, stealth, public], maxGasUnits=0
    -> all 5 shielded                                       (shipped: same)

R2  score=20, venuePreference=[shielded, stealth, public], maxGasUnits=0
    -> all 5 stealth (stealth gate is 0)                    (shipped: same)

R3  score=20, venuePreference=[shielded]
    -> throw: no qualifying venue                           (shipped: same)

R4  score=60, venuePreference=[shielded, public], gasEstimates=D
    maxGasUnits = 20,419,500 + 900,000 = 21,319,500 -> venueBudget = 900,000
    -> leg 4 shielded, legs 0-3 public; venue gas 400,000 + 4*65,000 = 660,000
       total estimate 21,079,500
    (shipped, assignVenues(legs, [shielded, public], {60, D}, 900_000n):
       legs 4 and 0 shielded, leg 1 public spend 865,000; leg 2 throws)

R5  as R4 with maxGasUnits = 20,419,500 + 325,000 = 20,744,500
    -> all 5 public (5 * 65,000 = 325,000 fits exactly)

R6  as R4 with maxGasUnits = 20,419,500 + 324,999 = 20,744,499
    -> throw: budget insufficient (5 * 65,000 > 324,999)

R7  as R4 with maxGasUnits = 20,419,499 (below overhead)
    -> throw: budget insufficient
```

### Diffusion scheduling

Bounds hold for every random draw; the low bound is all `r_i = 0`, the high bound all `r_i = 2^32 - 1`.

```
S1  n=5, window=300  -> slack=252; targetTimestamp_i in [lo_i, hi_i]
    lo = [0, 62, 124, 187, 249]   hi = [50, 112, 175, 237, 299]
S2  n=5, window=48   -> slack=0; exactly [0, 12, 24, 36, 48]
S3  n=5, window=47   -> throw: window < 48
S4  n=10, window=100 -> throw: window < 108
S5  n=10, window=108 -> exactly [0, 12, 24, ..., 108]
S6  n=5, window=0    -> all 0
S7  n=1, window=300  -> targetTimestamp_0 in [0, 299]
S8  n=5, window=601,201 -> throw: above MAX_DIFFUSION_WINDOW (Not yet implemented)
```

### Total deviation

```
referenceOut = 1,350,000,000,000 (USDC, 6 decimals), maxTotalDeviationBps = 100
E = [269,999,999,999 x4, 270,000,000,000]
allowance = referenceOut * 100 / 10000 = 13,500,000,000

leg 0 received 264,999,999,999 (shortfall 5,000,000,000)
leg 1 received 263,999,999,999 (shortfall 6,000,000,000)
leg 2 quote    266,999,999,999 -> cumulative 14,000,000,000 > 13,500,000,000
    -> leg 2 MUST NOT be submitted; abort legs 2-4; trade expires unfinalized
leg 2 quote    267,499,999,999 -> cumulative 13,500,000,000 <= 13,500,000,000
    -> check passes; leg 2 is submitted
```

### No split

```
totalAmount = 100e18 (== splitThreshold) -> N = 1, one leg of 100e18
-> caller does not call registerTrade (it would revert InvalidSubTradeCount(1))
-> gas estimate: 3,246,000 + venue gas
```

### End to end

```
1. Alice: totalAmount = 450e18 + 3, score = 60, venuePreference = [shielded, stealth],
   maxGasUnits = 0, diffusionWindow = 600, diffusionWindow <= 601,200
2. planSplit: N = 5, amounts as above
3. assignVenues: all 5 shielded; gas estimate 20,419,500 + 5 * 400,000 = 22,419,500
4. scheduleDiffusion: slack = 552; lo = [0, 122, 244, 367, 489], hi = [110, 232, 355, 477, 599]
5. registerTrade(tradeId, jurisdictionId, 5)
6. For i = 0..4 at targetTimestamp_i: re-check score, budget, slippage, deviation;
   prove leg i; submitCompliance; settle on Aztec; recordSubSettlement(tradeId, i, proofHash)
7. Prove and submit pattern proof (0x03) over computeSettlementRoot(tradeId)
8. finalizeTrade(tradeId, patternProofHash, patternPublicInputs)
9. Observer on L1 sees: TradeRegistered(tradeId, Alice, jurisdictionId, 5),
   5 Oracle attestations and 5 SubSettlementRecorded events from Alice with block times,
   the pattern attestation and TradeFinalized. Leg amounts and recipients are not on L1.
```

## Reference Implementation

- [`execution-orchestrator.ts`](https://github.com/xochi-fi/xochi-sdk/blob/main/packages/sdk/src/execution-orchestrator.ts), [`venue-router.ts`](https://github.com/xochi-fi/xochi-sdk/blob/main/packages/sdk/src/venue-router.ts), [`diffusion-scheduler.ts`](https://github.com/xochi-fi/xochi-sdk/blob/main/packages/sdk/src/diffusion-scheduler.ts), [`batch-prover.ts`](https://github.com/xochi-fi/xochi-sdk/blob/main/packages/sdk/src/batch-prover.ts), [`tiers.ts`](https://github.com/xochi-fi/xochi-sdk/blob/main/packages/sdk/src/tiers.ts) in xochi-sdk.
- Contracts used unmodified: [`SettlementRegistry.sol`](https://github.com/xochi-fi/ERC-8262/blob/main/src/SettlementRegistry.sol), [`ERC8262Oracle.sol`](https://github.com/xochi-fi/ERC-8262/blob/main/src/ERC8262Oracle.sol).
- Server gating: [`quote.ts`](https://github.com/xochi-fi/xochi/blob/main/packages/worker-lib/src/handlers/intents/quote.ts), [`settlement-policy.ts`](https://github.com/xochi-fi/xochi/blob/main/packages/worker-lib/src/settlement-policy.ts) in xochi.

Items marked Not yet implemented are open in xochi-sdk and xochi.

## Security Considerations

### Leg count, timing and linkage are public

See Privacy Model. Shielded settlement hides leg amounts and recipients, not the existence, count or timing of legs, because the compliance and registry calls are L1 transactions from the subject. Diffusion jitter does not hide timing from an observer of those calls; it only spreads the value transfers.

### Announced flow invites front-running

`TradeRegistered` publishes the subject and leg count before leg 0. Once leg 0 settles on a venue where amounts are visible, the remaining flow (about `(N - 1)` times leg 0 from a known address) is predictable for up to `MAX_DIFFUSION_WINDOW`. A longer diffusion window gives searchers more time to position against it. The total deviation bound (rule 6) caps the caller's loss and forces an abort rather than continued execution into a moved price. Registry-level mitigations belong to XIP-1.

### Caller-supplied trust score

`VenueConstraints.trustScore` is whatever the caller passes. The SDK gate prevents honest mistakes only. The settlement server is the trust boundary: it resolves the score from its own attestation records and refuses shielded while the rail is off. Refusing a shielded quote for a score below 50 while the rail is live is Not yet implemented server-side; until it is, the score gate on shielded is not enforced anywhere.

### Venue reveals a trust score lower bound

A leg routed to shielded reveals that the subject's score is at least 50. Public and stealth reveal nothing, as both are open to score 0. This is the same inference available for any shielded user without XIP-2.

### Structuring

Diffusion spreads one trade's legs over time, which is the shape of structuring. XIP-2 changes neither the number nor the size of legs and adds no enforcement of its own; the registry links only legs the caller registers, and N == 1 trades never touch it. What the pattern proof establishes, and its limits, are in XIP-1.

### Front-running a leg submission

Each compliance proof binds `submitter`, and the Oracle requires `submitter == msg.sender`, so a mempool observer cannot submit a leg's proof from another address. Diffusion does not change this.

### Self-inflicted ratchet failures

Interleaving the legs of two trades by the same subject and jurisdiction, or re-proving one leg while older pre-proved legs wait, makes later submissions revert `ProofTimestampNotMonotonic`. Rule 4 and just-in-time proving prevent it.

## Failure Modes

### Router cannot assign

No listed venue qualifies (score 20, `venuePreference = [shielded]`), or the budget is below `overhead(N) + N * cheapestQualified`. `planExecution` throws before any proof or transaction. The router MUST NOT silently downgrade to an unlisted venue; the caller widens `venuePreference` or raises the budget.

### Venue unavailable mid-execution

The assigned venue becomes unreachable at step 3.4 of leg k. The caller re-plans the remaining legs, leg k included, onto another listed venue (the registry does not know venues), or waits. Leg k's compliance attestation is already submitted; the proof is venue-agnostic, so the same `proofHash` stays valid for `recordSubSettlement` after settling on the new venue, until the attestation's `expiresAt` (`AttestationExpired`, contract-enforced). No re-proving is needed unless that attestation expires first, in which case leg k is re-proved and resubmitted under rule 4. Waiting is bounded by `createdAt + TRADE_TTL`; after that the trade can only expire.

### Stale or out-of-order proof

`submitCompliance` reverts `ProofTimestampStale` or `ProofTimestampNotMonotonic`. Nothing is recorded for that leg; re-prove it and resubmit. Earlier legs are unaffected.

### Diffusion window too short or too long

`diffusionWindow < (N - 1) * 12`: `scheduleDiffusion` throws; increase the window or the `splitThreshold`. `diffusionWindow > MAX_DIFFUSION_WINDOW`: MUST be rejected at plan time (Not yet implemented); without that check, late legs revert `TradeExpiredError` after earlier legs settled.

### Trust score or gas estimates change mid-window

Rule 7 re-checks before each leg. A dropped score or higher venue gas leads to re-plan through `assignVenues` with the remaining venue budget, or abort. Gas price spikes do not change gas units; they change what each transaction costs in wei, which the caller controls per transaction.

### Price moves against the trade

Rule 6 stops further legs once the cumulative shortfall would exceed `maxTotalDeviationBps`. Settled legs stay settled and the trade expires unfinalized; the caller sees a partial fill, not an unbounded loss.

### Registering an unsplit trade

`registerTrade(..., 1)` reverts `InvalidSubTradeCount(1)`. The N == 1 path skips the registry.

## History

- 2026-11-12: Merged as Draft (#2).
- 2026-11-12: Resynced to shipped xochi-sdk, ERC-8262 and xochi on review; type Protocol -> Interface; retitled from "Adaptive Settlement Controls"; `discussions-to` fixed to #7; status stays Draft.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
