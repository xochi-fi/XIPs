---
xip: 1
title: Settlement Splitting for Large Trades
description: Split large trades into multiple sub-settlements with independent compliance proofs
author: DROO (@DROOdotFOO)
discussions-to: https://github.com/xochi-fi/XIPs/issues/2
status: Draft
type: Protocol
created: 2026-04-15
requires: 0
---

## Abstract

Large trades settled as a single compliance proof create concentration risk, reduce privacy, and are easier to front-run. This XIP introduces settlement splitting: large trades are divided into multiple sub-settlements, each with its own ZK compliance proof, linked by a shared trade identifier. The design adds a standalone `SettlementRegistry` contract and SDK-side orchestration without modifying the existing Oracle or Verifier contracts.

## Motivation

A single large settlement has four problems:

1. **Concentration risk.** If a proof is invalidated (key compromise, oracle config update, verifier upgrade), the entire settlement is affected. Splitting distributes this risk across independent proofs.

2. **Privacy reduction.** Large on-chain settlements are conspicuous. A 500 ETH transfer stands out; five 100 ETH transfers across a time window do not.

3. **MEV exposure.** Single large settlements are high-value sandwich targets. Smaller sub-settlements reduce per-trade MEV incentive below the profitability threshold for many searchers.

4. **Proof isolation.** Each sub-trade carries its own compliance proof. If a jurisdiction's requirements change or a provider config is revoked between sub-settlements, only the affected sub-trade needs reproving. The remaining sub-trades retain valid, independently verifiable attestations.

The current system has no mechanism to link multiple proofs to a single trade. The Oracle stores one `ComplianceAttestation` per `(subject, jurisdictionId)` pair and **overwrites** on each submission. Sub-settlements to the same jurisdiction would clobber each other's attestations. The Verifier supports batch proof verification (`verifyProofBatch`, max 100), but the Oracle has no batch submission.

## Specification

### Overview

Settlement splitting is implemented in three layers:

```
Layer 0 (SDK):   SplitPlanner + BatchProver -- pure computation, no contract changes
Layer 1 (Contract): SettlementRegistry -- standalone contract linking sub-proofs
Layer 2 (Optimization): Oracle.submitComplianceBatch -- optional gas optimization
```

Layer 0 and Layer 1 are normative. Layer 2 is OPTIONAL and deferred until gas profiling justifies it.

### Layer 0: SDK Split Planning

#### SplitPlanner

The `SplitPlanner` is a pure function that computes sub-trade amounts from a total trade amount.

```typescript
interface SplitConfig {
  splitThreshold: bigint;    // minimum trade size to trigger splitting
  maxSubTrades: number;      // maximum number of sub-settlements (2-100)
  minSubTradeSize: bigint;   // minimum per-sub-trade amount (dust prevention)
}

interface SplitPlan {
  tradeId: Hex;              // keccak256(totalAmount, jurisdictionId, nonce, submitter)
  subTrades: SubTrade[];
  totalAmount: bigint;
  splitConfig: SplitConfig;
}

interface SubTrade {
  index: number;             // 0-based position in the split
  amount: bigint;            // sub-trade amount
}

function planSplit(
  totalAmount: bigint,
  jurisdictionId: JurisdictionId,
  submitter: Address,
  config?: Partial<SplitConfig>,
): SplitPlan;
```

**Splitting algorithm:**

1. If `totalAmount <= splitThreshold`, return a single-trade plan (no split).
2. Compute `n = ceil(totalAmount / splitThreshold)`, clamped to `[2, maxSubTrades]`.
3. Compute `baseAmount = totalAmount / n` (integer division).
4. Assign `baseAmount` to each of the first `n-1` sub-trades.
5. Assign the remainder (`totalAmount - baseAmount * (n-1)`) to the last sub-trade.
6. If `baseAmount < minSubTradeSize`, reduce `n` by 1. If `n` becomes 1, return a single-trade plan (no split). Otherwise repeat from step 3.
7. Generate `tradeId = keccak256(abi.encodePacked(totalAmount, jurisdictionId, nonce, submitter))` where `nonce` is a unique 256-bit value and `submitter` is the caller's Ethereum address from the base input. Implementations MUST ensure `nonce` uniqueness per submitter. Implementations SHOULD use `crypto.randomBytes(32)`; implementations MUST NOT use `Date.now()` or other low-entropy sources.

Implementations MUST use deterministic splitting (uniform with remainder). Implementations MUST NOT add random noise to sub-trade amounts. The rationale is given in the Rationale section.

**Default parameters:**

| Parameter        | Default | Constraint              |
| ---------------- | ------- | ----------------------- |
| `splitThreshold` | 100 ETH | MUST be >= 10 ETH       |
| `maxSubTrades`   | 10      | MUST be in [2, 100]     |
| `minSubTradeSize`| 1 ETH   | MUST be > 0             |

#### BatchProver

The `BatchProver` generates compliance proofs for all sub-trades sequentially.

```typescript
interface BatchProveResult {
  tradeId: Hex;
  proofs: Array<{
    index: number;
    amount: bigint;
    proofResult: ProofResult;
  }>;
}

async function proveBatch(
  prover: XochiProver,
  plan: SplitPlan,
  baseInput: ComplianceInput | RiskScoreInput,
): Promise<BatchProveResult>;
```

Implementations MUST generate proofs sequentially. The Barretenberg backend is not concurrency-safe; parallel proof generation causes undefined behavior.

Each sub-trade proof MUST use the same `submitter`, `jurisdictionId`, `configHash`, and `providerSetHash` as the base input.

Note: the compliance and risk_score circuits do not take a trade amount as input -- they prove that a risk score (derived from screening signals) meets a jurisdiction threshold. The `SubTrade.amount` field is used by the settlement layer (DEX, AMM, or application contract) to determine how much value to transfer per sub-settlement. The compliance proof attests that the *user* is compliant, not that a specific *amount* is compliant. Each sub-trade carries its own proof so that if any single proof is invalidated, only that sub-trade's settlement is affected.

**Compliance proof scope:** The per-sub-trade compliance proof does NOT attest that the split transaction structure is legitimate or non-structuring. It attests only that the submitting user's aggregate risk score is below the jurisdiction threshold. Split settlements MUST be accompanied by a pattern detection proof (circuit 0x03) to demonstrate that the split does not exhibit structuring characteristics. See the anti-structuring requirement below.

### Layer 1: SettlementRegistry Contract

A standalone contract that links sub-settlement proofs to a trade identifier. The Oracle and Verifier contracts are NOT modified.

#### Deployment

The SettlementRegistry is deployed as an immutable contract with no admin functions, no pause mechanism, and no upgrade path. The oracle address is set at construction and cannot be changed. See Failure Modes > Registry contract upgrade for the migration strategy.

```solidity
constructor(address oracle_);
```

The `oracle_` parameter MUST be a deployed `IXochiZKPOracle` contract. The address is stored as `immutable` and exposed via a public getter:

```solidity
function oracle() external view returns (address);
```

#### Interface

```solidity
interface ISettlementRegistry {
    struct Settlement {
        bytes32 tradeId;
        address subject;
        uint8 jurisdictionId;
        uint256 subTradeCount;
        uint256 settledCount;
        uint256 createdAt;
        uint256 expiresAt;
        bool finalized;
    }

    struct SubSettlement {
        uint256 index;
        bytes32 proofHash;
        uint256 settledAt;
    }

    /// @notice Register a new trade for split settlement.
    /// @param tradeId Unique trade identifier (computed by SDK).
    /// @param jurisdictionId Jurisdiction for all sub-settlements.
    /// @param subTradeCount Total number of expected sub-trades.
    function registerTrade(
        bytes32 tradeId,
        uint8 jurisdictionId,
        uint256 subTradeCount
    ) external;

    /// @notice Record a sub-settlement after the user has submitted the
    ///         proof directly to the Oracle. The registry verifies the
    ///         attestation exists and the subject matches.
    /// @param tradeId The trade this sub-settlement belongs to.
    /// @param index The sub-trade index (0-based).
    /// @param proofHash The proof hash returned by Oracle.submitCompliance.
    function recordSubSettlement(
        bytes32 tradeId,
        uint256 index,
        bytes32 proofHash
    ) external;

    /// @notice Finalize a trade after all sub-settlements are submitted.
    /// @param tradeId The trade to finalize.
    /// @param patternProofHash Proof hash of a pattern detection (0x03) attestation
    ///        for the trade's subject, used to satisfy the anti-structuring requirement.
    function finalizeTrade(bytes32 tradeId, bytes32 patternProofHash) external;

    /// @notice Expire a trade that was not finalized before its deadline.
    ///         Callable by anyone after expiresAt has passed.
    /// @param tradeId The trade to expire.
    function expireTrade(bytes32 tradeId) external;

    /// @notice Query settlement status.
    function getSettlement(bytes32 tradeId)
        external view returns (Settlement memory);

    /// @notice Query all sub-settlements for a trade.
    function getSubSettlements(bytes32 tradeId)
        external view returns (SubSettlement[] memory);

    event TradeRegistered(
        bytes32 indexed tradeId,
        address indexed subject,
        uint8 indexed jurisdictionId,
        uint256 subTradeCount
    );

    event SubSettlementRecorded(
        bytes32 indexed tradeId,
        uint256 indexed index,
        bytes32 indexed proofHash
    );

    event TradeFinalized(bytes32 indexed tradeId, uint256 timestamp);
    event TradeExpired(bytes32 indexed tradeId, uint256 timestamp);

    error TradeAlreadyExists(bytes32 tradeId);
    error TradeNotFound(bytes32 tradeId);
    error SubTradeIndexOutOfBounds(uint256 index, uint256 max);
    error SubTradeAlreadySettled(bytes32 tradeId, uint256 index);
    error NotTradeSubject(address caller, address subject);
    error TradeAlreadyFinalized(bytes32 tradeId);
    error TradeNotComplete(bytes32 tradeId, uint256 settled, uint256 expected);
    error AttestationNotFound(bytes32 proofHash);
    error SubjectMismatch(address expected, address actual);
    error JurisdictionMismatch(uint8 expected, uint8 actual);
    error TradeExpiredError(bytes32 tradeId);
    error TradeNotExpired(bytes32 tradeId);
    error PatternProofRequired(bytes32 tradeId);
}
```

#### Storage

```solidity
mapping(bytes32 tradeId => Settlement) internal _settlements;
mapping(bytes32 tradeId => mapping(uint256 index => SubSettlement)) internal _subSettlements;
```

#### Behavior

1. `registerTrade` creates a `Settlement` record. The caller becomes the `subject`. The `subTradeCount` MUST be in `[2, 100]`. Sets `expiresAt = block.timestamp + 7 days`. Reverts if `tradeId` already exists.

2. `recordSubSettlement` MUST:
   a. Verify `msg.sender == settlement.subject`.
   b. Verify `index < settlement.subTradeCount`.
   c. Verify the sub-settlement at `index` has not already been settled.
   d. Call `IXochiZKPOracle(oracle).getHistoricalProof(proofHash)` to retrieve the attestation. MUST revert if the attestation does not exist (timestamp == 0).
   e. Verify `attestation.subject == settlement.subject`. Reverts with `SubjectMismatch` otherwise.
   f. Verify `attestation.jurisdictionId == settlement.jurisdictionId`. Reverts with `JurisdictionMismatch` otherwise.
   g. Store `SubSettlement{index, proofHash, block.timestamp}`.
   h. Increment `settlement.settledCount`.

2 (cont). `recordSubSettlement` MUST also revert if `block.timestamp > settlement.expiresAt` with `TradeExpiredError`.

3. `finalizeTrade` MUST revert unless `msg.sender == settlement.subject`. MUST revert unless `settledCount == subTradeCount`. MUST revert if trade is expired. MUST verify the `patternProofHash` attestation exists, belongs to the subject, and was created after trade registration (see Anti-Structuring Requirement). Sets `settlement.finalized = true`.

4. `expireTrade` MUST revert unless `block.timestamp > settlement.expiresAt` and `!settlement.finalized`. Sets `settlement.finalized = true` (prevents further recording). Callable by anyone -- expiry is permissionless. Already-submitted sub-settlements remain valid Oracle attestations.

#### Anti-Structuring Requirement

Split settlements MUST be accompanied by a pattern detection proof (circuit 0x03) to demonstrate that the split does not exhibit structuring, velocity, or round-amount anomalies. This requirement exists because the per-sub-trade compliance proofs attest to user-level risk scores, not to the legitimacy of the split structure itself.

`finalizeTrade` accepts a `patternProofHash` parameter. The registry MUST verify:

1. `patternProofHash != bytes32(0)`.
2. Call `IXochiZKPOracle(oracle).getProofType(patternProofHash)` and verify the result equals `PATTERN (0x03)`. This prevents a compliance or other proof type from being substituted.
3. Call `IXochiZKPOracle(oracle).getHistoricalProof(patternProofHash)` to retrieve the attestation. MUST revert if the attestation does not exist.
4. Verify `attestation.subject == settlement.subject`.
5. Verify `attestation.timestamp >= settlement.createdAt` (the proof was generated after trade registration).

If any check fails, `finalizeTrade` MUST revert with `PatternProofRequired(bytes32 tradeId)`.

This ensures that settlement splitting cannot be used for transaction structuring without the protocol's anti-structuring circuit validating the pattern. The pattern detection circuit analyzes up to 16 transactions and flags clustering below reporting thresholds, excessive velocity, and round-amount patterns.

```solidity
error PatternProofRequired(bytes32 tradeId);
```

#### Minimum Split Threshold

To prevent split settlements from being used to target jurisdiction-specific reporting thresholds, the `splitThreshold` parameter has a protocol-enforced floor:

| Parameter        | Default | Constraint                     |
| ---------------- | ------- | ------------------------------ |
| `splitThreshold` | 100 ETH | MUST be >= 10 ETH              |

The 10 ETH floor (~$30,000-40,000 at current prices) ensures that sub-trade amounts remain meaningfully above common reporting thresholds. Implementations MUST reject split plans where `splitThreshold < 10 ETH`.

#### Submitter Binding

The Oracle enforces `submitter == msg.sender` for compliance and risk_score proofs. This XIP uses **direct submission with bookkeeping**: the user calls `oracle.submitCompliance` directly for each sub-trade (satisfying the submitter check), then calls `registry.recordSubSettlement(tradeId, index, proofHash)` to record the linkage. The registry verifies the attestation exists and belongs to the trade's subject via `oracle.getHistoricalProof`.

The alternative -- having the registry call `oracle.submitCompliance` as a proxy -- was rejected. It would make the registry's address the `msg.sender`, requiring proofs to use `submitter = registryAddress`. This breaks the Oracle's fundamental property that `attestation.subject == the entity that proved compliance`. See the Rationale section for full discussion.

### Layer 2: Oracle Batch Submission (OPTIONAL)

If gas profiling demonstrates that N individual `submitCompliance` calls impose unacceptable overhead compared to a batch path, implementations MAY add:

```solidity
function submitComplianceBatch(
    uint8[] calldata jurisdictionIds,
    uint8[] calldata proofTypes,
    bytes[] calldata proofs,
    bytes[] calldata publicInputs,
    bytes32[] calldata providerSetHashes
) external whenNotPaused returns (ComplianceAttestation[] memory);
```

This is NOT specified normatively. It is listed for forward compatibility. Implementations that add batch submission MUST maintain identical validation and storage semantics per sub-proof.

### SDK Integration

The SDK MUST export the following from `@xochi/sdk`:

```typescript
// Split planning
export { planSplit, type SplitConfig, type SplitPlan, type SubTrade }

// Batch proving
export { proveBatch, type BatchProveResult }

// Settlement registry client (viem-based)
export { SettlementRegistryClient }
```

The `SettlementRegistryClient` follows the same pattern as `XochiOracle` and `XochiVerifier`: a typed viem wrapper around the registry contract ABI.

## Rationale

### Why a separate SettlementRegistry, not Oracle extension?

The Oracle contract is responsible for compliance attestation: "does this address meet the compliance threshold for this jurisdiction?" Settlement splitting is an orchestration concern: "how do we break a large trade into compliant pieces?" These have different lifecycles, different audit scopes, and different upgrade cadences.

Extending the Oracle with `tradeId` and batch submission would:
- Increase the Oracle's attack surface during its initial audit.
- Break the `ComplianceAttestation` struct ABI for all existing consumers.
- Couple settlement policy (thresholds, max sub-trades) to compliance policy (jurisdiction thresholds, TTL).

A separate contract keeps each concern auditable in isolation.

### Why no randomized amounts?

The design doc originally proposed adding noise to sub-trade amounts. This was rejected for three reasons:

1. **ZK contradiction.** The compliance proof attests that a *user's risk score* is below a jurisdiction threshold -- it does not take a trade amount as input. Adding noise to sub-trade amounts is therefore pure obfuscation theater: the ZK proof says nothing about the amount, so noisy amounts add complexity without ZK-grade privacy guarantees.

2. **Summing attack.** An observer who sees all N sub-settlements can sum them to recover the total regardless of per-trade noise. The noise only prevents correlation of individual sub-settlements, which is already handled by the ZK proofs hiding the actual amounts.

3. **Aztec L2 superiority.** For users in the stealth/private/sovereign privacy tiers (score >= 25), settlement happens within the Aztec shielded pool where sub-trade amounts are not visible at all. Noise is irrelevant for the users who need privacy most.

Deterministic uniform splitting is simpler, auditable, and honest about its privacy properties.

### Why no settlement delay in the SDK?

The design doc proposed randomized delays between sub-settlement submissions. This is an application-layer concern:

- The SDK generates proofs. When those proofs are submitted is the caller's decision.
- Baking delays into the SDK couples it to execution timing, making it harder to use in different contexts (immediate settlement, queued settlement, MEV-protected submission via Flashbots).
- Callers who want delays can trivially add `await sleep()` between sub-settlement submissions.

### Why sequential proof generation?

The Barretenberg backend (`@aztec/bb.js`) uses shared WASM state that is not thread-safe. Parallel proof generation produces corrupted witnesses. This is a known constraint documented in the SDK's vitest config (`test.sequence.concurrent: false`). The `BatchProver` enforces sequential execution to prevent silent corruption.

### Why Option A (direct submission) over Option B (registry as proxy)?

Option B would make the SettlementRegistry the `msg.sender` for oracle submissions. This means:
- The proof's `submitter` field would be the registry address, not the user's address.
- The Oracle would record the registry as the `subject` of the attestation.
- The registry would need its own access control to map registry-as-subject back to the actual user.

This breaks the Oracle's fundamental property: `attestation.subject == the entity that proved compliance`. It conflates "who orchestrated the settlement" with "who is compliant." Option A preserves this property by having the user submit directly to the Oracle and only recording the linkage in the registry.

### Why only single-chain splitting?

Cross-chain settlement splitting -- coordinating sub-trades across L1/L2 or across L2s -- introduces bridge trust assumptions orthogonal to compliance proof splitting. Each chain's SettlementRegistry operates independently. Cross-chain coordination is a candidate for a follow-on XIP.

### Why no dynamic re-splitting?

If sub-trade K of N fails, the system does not automatically re-split the remaining amount into new sub-trades. The user retries the failed sub-trade or abandons the trade. Dynamic re-splitting would require the registry to track partial amounts and recompute splits on-chain, adding complexity and gas cost for a failure mode the user can handle client-side.

### Why no relayer support?

The submitter anti-front-running binding (`submitter == msg.sender`) is incompatible with relayer/meta-transaction patterns. Supporting relayers would require binding to a `recipient` instead of `msg.sender`, which requires changes to the Oracle contract and proof-binding semantics -- a separate concern from settlement splitting.

### Relationship to stealth address splitting

The Xochi whitepaper describes a separate form of settlement splitting: multi-account stealth address splitting, where N stealth keys are derived from the same meta-address and the settlement amount is distributed across N counterfactual SimpleAccount addresses (ERC-5564). That mechanism operates at the settlement layer (how value is distributed across addresses for privacy). This XIP operates at the compliance layer (how compliance proofs are structured for concentration risk, MEV, and regulatory alignment). The two are orthogonal and composable: a single large trade could use this XIP's proof splitting AND stealth address splitting simultaneously.

## Backwards Compatibility

No breaking changes.

- The Oracle contract (`XochiZKPOracle`) is not modified.
- The Verifier contract (`XochiZKPVerifier`) is not modified.
- The `ComplianceAttestation` struct is not modified.
- Existing SDK exports are not changed.
- The `SettlementRegistry` is a new standalone contract.
- SDK additions (`planSplit`, `proveBatch`, `SettlementRegistryClient`) are new exports.

Consumers that do not use settlement splitting are unaffected.

## Test Cases

### Split Planning

```
Input:  totalAmount=500 ETH, splitThreshold=100 ETH, maxSubTrades=10, minSubTradeSize=1 ETH
Output: 5 sub-trades of 100 ETH each

Input:  totalAmount=350 ETH, splitThreshold=100 ETH
Output: 4 sub-trades: [87, 87, 87, 89] ETH (baseAmount=87, remainder=89)

Input:  totalAmount=50 ETH, splitThreshold=100 ETH
Output: 1 sub-trade of 50 ETH (no split, below threshold)

Input:  totalAmount=1000 ETH, splitThreshold=100 ETH, maxSubTrades=5
Output: 5 sub-trades: [200, 200, 200, 200, 200] ETH (clamped to maxSubTrades)

Input:  totalAmount=10 ETH, splitThreshold=3 ETH, minSubTradeSize=2 ETH
Output: 4 sub-trades: [2, 2, 2, 4] ETH
  (ceil(10/3)=4, baseAmount=10/4=2, 2 >= minSubTradeSize, remainder=4)
```

### Sub-Settlement Flow (E2E)

```
1. Alice calls planSplit(500 ETH, EU, aliceAddress) -> SplitPlan{tradeId: 0xabc, subTrades: [100, 100, 100, 100, 100]}
2. Alice calls proveBatch(prover, plan, complianceInput) -> 5 ProofResults
3. Alice calls registry.registerTrade(0xabc, EU, 5)
4. For each sub-trade i in [0..4]:
   a. Alice calls oracle.submitCompliance(EU, COMPLIANCE, proof[i], publicInputs[i], providerSetHash) -> proofHash[i]
   b. Alice calls registry.recordSubSettlement(0xabc, i, proofHash[i])
      Registry verifies: oracle.getHistoricalProof(proofHash[i]).subject == Alice
5. Alice calls registry.finalizeTrade(0xabc)
6. registry.getSettlement(0xabc) -> {subject: Alice, settled: 5/5, finalized: true}
```

### Submitter Mismatch

```
1. Alice generates proof with submitter=Alice for sub-trade 0
2. Bob calls oracle.submitCompliance with Alice's proof
3. Oracle reverts with SubmitterMismatch (submitter=Alice, msg.sender=Bob)
4. Sub-settlement cannot be front-run
```

### Finalization Guard

```
1. Alice registers trade with 5 sub-trades
2. Alice submits 3 of 5 sub-settlements
3. Alice calls finalizeTrade -> reverts with TradeNotComplete(0xabc, 3, 5)
4. Alice submits remaining 2 sub-settlements
5. Alice calls finalizeTrade -> succeeds
```

## Reference Implementation

Deferred until status = Approved. Implementation will span:

- `@xochi/sdk` -- `src/split.ts` (SplitPlanner), `src/batch-prover.ts` (BatchProver), `src/settlement-registry.ts` (client)
- `erc-xochi-zkp` -- `src/SettlementRegistry.sol`, `src/interfaces/ISettlementRegistry.sol`
- `erc-xochi-zkp` -- `test/SettlementRegistry.t.sol` (Foundry tests)

## Security Considerations

### Front-running sub-settlements

Each sub-trade proof includes the `submitter` field as a public input. The Oracle enforces `submitter == msg.sender`. An attacker who observes a pending sub-settlement transaction cannot submit it from a different address. This protection was validated end-to-end in the submitter anti-front-running work (see `consumer.test.ts`: "rejects proof when submitter != msg.sender").

### Trade registration griefing

An attacker could call `registerTrade` with a `tradeId` that the victim intends to use, blocking the victim's registration. Mitigation: the `tradeId` includes the submitter's address in its preimage (`keccak256(totalAmount, jurisdictionId, nonce, submitter)`). Since the registry enforces `msg.sender == settlement.subject`, the attacker would need to register from the victim's address. Additionally, the 256-bit random nonce makes precomputation infeasible.

### Partial settlement abandonment

If a user registers a trade with N sub-trades but only submits M < N, the trade remains non-finalized until its expiry (7 days after registration). After expiry, anyone can call `expireTrade(tradeId)` to mark it as expired and prevent further sub-settlement recording. The individual sub-settlements that were submitted remain valid attestations in the Oracle regardless of registry expiry state.

### Oracle attestation overwrite

The Oracle stores one active attestation per `(subject, jurisdictionId)`. If a user submits sub-trade 0 for EU, then sub-trade 1 for EU, the Oracle's active attestation for `(Alice, EU)` will reflect sub-trade 1 only. Sub-trade 0's attestation is preserved in `_proofIndex[proofHash]` for historical lookup but is no longer the "active" attestation.

This is acceptable because:
- The SettlementRegistry tracks all sub-settlement proof hashes independently.
- `checkCompliance` returning the latest sub-trade is correct: the user is still compliant.
- Historical sub-trade attestations are retrievable via `getHistoricalProof(proofHash)`.

### Gas griefing via max sub-trades

A user who registers a trade with `subTradeCount=100` and submits 100 sub-settlements imposes ~100 * 70,000 = 7M gas across all submissions. This is paid by the user, not by other protocol participants, so it is self-griefing only. The `maxSubTrades` cap of 100 bounds the worst case.

### Structuring risk

Settlement splitting could be misused to structure transactions below reporting thresholds. Three mitigations are layered:

1. **Pattern detection proof requirement.** `finalizeTrade` requires a valid pattern detection attestation (circuit 0x03) proving the user's recent transaction pattern does not exhibit structuring characteristics (clustering below thresholds, excessive velocity, round amounts). Without this proof, split trades cannot be finalized.

2. **Minimum split threshold floor.** `splitThreshold` MUST be >= 10 ETH (~$30,000-40,000), ensuring sub-trade amounts remain meaningfully above common reporting thresholds. This prevents the planner from being configured to target specific threshold amounts.

3. **On-chain linkage.** All sub-settlements are linked to a single `tradeId` via the SettlementRegistry. Unlike ad-hoc splitting (where a user manually sends multiple transactions), split settlements are explicitly grouped and auditable as a single trade.

These mitigations do not eliminate structuring risk entirely -- a determined actor could split trades outside the protocol. They ensure that the protocol itself does not facilitate or automate structuring.

### Information leakage from split count

The number of sub-trades is visible on-chain (via `TradeRegistered` event). An observer can infer the approximate total trade size: `subTradeCount * splitThreshold`. This is a privacy tradeoff accepted in exchange for compliance and verifiability. For users in the private/sovereign tiers (score >= 50), settlement occurs within the Aztec shielded pool where neither the split count nor sub-trade amounts are visible.

## Failure Modes

### Barretenberg failure mid-batch

If proof generation fails for sub-trade K of N, the user has K-1 valid proofs and no proof for sub-trade K. The user MAY retry proof generation for sub-trade K. If the circuit inputs are identical, the new proof will have a different `proofHash` (WASM randomness in the proving process), so it will not trigger replay protection.

The K-1 already-submitted sub-settlements remain valid Oracle attestations. The trade in the registry remains non-finalized. No rollback is needed.

### Oracle config update between sub-settlements

If the Oracle admin calls `updateProviderConfig` between sub-settlement N and N+1, sub-settlement N+1's proof may use a config hash that has been superseded. The Oracle validates `_validConfigs[configHash]`, not `configHash == currentConfig`, so historical configs remain valid unless explicitly revoked.

If the config is revoked, sub-settlement N+1 will revert with `InvalidConfigHash`. The user must regenerate the proof with the current config. Partially submitted sub-settlements with the old config remain valid.

### Registry contract upgrade

The SettlementRegistry holds no funds and stores only linkage data (tradeId -> proofHash[]). If the registry needs upgrading, a new registry can be deployed. In-flight trades on the old registry remain readable. New trades use the new registry. No migration is needed because the Oracle is the source of truth for compliance attestations; the registry is only bookkeeping.

### Concurrent splitting load

Multiple users registering and settling split trades simultaneously does not create contention. Each trade has a unique `tradeId` and independent storage slots. The SettlementRegistry has no shared mutable state across trades (no global counters, no shared queues). Gas cost per sub-settlement is constant regardless of how many other trades are in flight.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
