---
xip: 1
title: Settlement Splitting for Large Trades
description: Split large trades into multiple sub-settlements with independent compliance proofs
author: DROO (@DROOdotFOO)
discussions-to: https://github.com/xochi-fi/XIPs/issues/6
status: Review
type: Protocol
created: 2026-04-15
---

## Abstract

Large trades settled under a single compliance proof concentrate risk on that proof. This XIP splits a large trade into sub-settlements. Each sub-settlement has its own ZK compliance proof, and all of them are linked by a shared trade identifier. The design adds a standalone `SettlementRegistry` contract and SDK-side planning and proving. It does not modify the ERC-8262 Oracle or Verifier. All of it is shipped: [ERC-8262](https://github.com/xochi-fi/ERC-8262) for the contract and [xochi-sdk](https://github.com/xochi-fi/xochi-sdk) for the SDK (see Reference Implementation).

## Motivation

A single large settlement has four problems:

1. **Concentration risk.** If a proof is invalidated (key compromise, config revocation, verifier revocation), the entire settlement is affected. Splitting spreads this risk across independent proofs.

2. **Privacy reduction.** Large on-chain transfers are conspicuous. Smaller transfers are less so, but this XIP's registry links them publicly (see Information leakage from split count). The privacy gain is limited to the transfer layer.

3. **MEV exposure.** Single large settlements are high-value sandwich targets. Smaller legs lower the value per leg. As shipped, `registerTrade` announces the subject and leg count before the first leg, which partly offsets this (see MEV from trade registration).

4. **Proof isolation.** Each leg carries its own compliance proof. If a jurisdiction's requirements change or a provider config is revoked between legs, only legs not yet submitted need re-proving. Legs already recorded keep their own attestations.

Before this XIP, nothing linked several proofs to one trade. The ERC-8262 Oracle keeps the latest attestation per `(subject, jurisdictionId)` in its compliance slot and per `(subject, jurisdictionId, proofType)` in its typed slot, and overwrites both on each submission. Earlier attestations remain retrievable only by `proofHash`. `submitComplianceBatch` submits up to `MAX_BATCH_SIZE = 10` proofs atomically but records no trade linkage.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

### Terms

- **Oracle**: `ERC8262Oracle` (interface `IERC8262Oracle`), ERC-8262 `src/ERC8262Oracle.sol`.
- **Verifier**: `ERC8262Verifier`, ERC-8262 `src/ERC8262Verifier.sol`.
- **Prover**: `ERC8262Prover`, xochi-sdk `packages/sdk/src/prover.ts`.
- **Leg**: one sub-settlement of a split trade.
- **Amount**: an integer in the base units of the traded asset (wei for ETH). All amounts in this XIP are `bigint` / `uint256` base units unless stated otherwise.

Proof types (ERC-8262 `src/libraries/ProofTypes.sol`, xochi-sdk `PROOF_TYPES` in `constants.ts`):

| Id     | Name                      | Circuit                   | Role in this XIP   |
| ------ | ------------------------- | ------------------------- | ------------------ |
| `0x01` | `COMPLIANCE`              | `compliance`              | Leg proof          |
| `0x02` | `RISK_SCORE`              | `risk_score`              | Rejected as a leg  |
| `0x03` | `PATTERN`                 | `pattern`                 | Finalization proof |
| `0x04` | `ATTESTATION`             | `attestation`             | Rejected as a leg  |
| `0x05` | `MEMBERSHIP`              | `membership`              | Rejected as a leg  |
| `0x06` | `NON_MEMBERSHIP`          | `non_membership`          | Rejected as a leg  |
| `0x07` | `COMPLIANCE_SIGNED`       | `compliance_signed`       | Leg proof          |
| `0x08` | `RISK_SCORE_SIGNED`       | `risk_score_signed`       | Rejected as a leg  |
| `0x09` | `COMPLIANCE_MULTI_SIGNED` | `compliance_multi_signed` | Leg proof          |

PATTERN `analysis_type` values (Oracle `PATTERN_STRUCTURING = 1`, `PATTERN_VELOCITY = 2`, `PATTERN_ROUND_AMOUNT = 3`). Only `1` finalizes a trade.

### Overview

```
Layer 0 (SDK):      planSplit + proveBatch -- off-chain planning and proving
Layer 1 (Contract): SettlementRegistry -- standalone contract linking leg proofs to a trade
Layer 2 (Oracle):   submitComplianceBatch -- shipped batch submission, OPTIONAL to use
```

Layers 0 and 1 are normative. Layer 2 is an existing Oracle function; using it is OPTIONAL.

### Layer 0: Split planning

#### planSplit

`planSplit` is a pure function apart from drawing a random nonce (xochi-sdk `split.ts`).

```typescript
interface SplitConfig {
  splitThreshold: bigint; // trades above this amount are split
  maxSubTrades: number; // upper bound on legs, in [2, 100]
  minSubTradeSize: bigint; // lower bound on the base leg amount, > 0
}

interface SplitPlan {
  tradeId: Hex;
  subTrades: SubTrade[];
  totalAmount: bigint;
  splitConfig: SplitConfig;
}

interface SubTrade {
  index: number; // 0-based position in the split
  amount: bigint; // leg amount in base units
}

function planSplit(
  totalAmount: bigint,
  jurisdictionId: JurisdictionId,
  submitter: Address,
  config?: Partial<SplitConfig>,
): SplitPlan;
```

Algorithm. Implementations MUST produce exactly this output:

1. Merge `config` over the defaults. Reject (throw) if `splitThreshold < MIN_SPLIT_THRESHOLD`, if `maxSubTrades` is outside `[2, 100]`, if `minSubTradeSize <= 0`, or if `totalAmount <= 0`.
2. Draw a 32-byte `nonce` from a CSPRNG (shipped: `crypto.getRandomValues`). Implementations MUST NOT derive it from `Date.now()` or other low-entropy sources. Set `tradeId = keccak256(abi.encodePacked(uint256 totalAmount, uint8 jurisdictionId, bytes32 nonce, address submitter))`.
3. If `totalAmount <= splitThreshold`, return one leg of `totalAmount` (N = 1).
4. `n = ceil(totalAmount / splitThreshold)`, clamped to `[2, maxSubTrades]`.
5. `baseAmount = totalAmount / n` (integer division). If `baseAmount < minSubTradeSize`, set `n = n - 1`; if `n < 2`, return one leg of `totalAmount`; otherwise repeat this step.
6. Legs `0 .. n-2` get `baseAmount`. Leg `n-1` gets `totalAmount - baseAmount * (n-1)`.

Splitting MUST be deterministic (uniform with remainder on the last leg). Implementations MUST NOT add random noise to leg amounts (see Rationale). Every leg of a split plan is at least `splitThreshold / 2`: `totalAmount > splitThreshold * (n-1)` for the initial `n`, and clamping or reducing `n` only raises `baseAmount`. Row 5 of the Test Cases hits the bound exactly.

Default parameters (`DEFAULT_SPLIT_CONFIG`):

| Parameter         | Default             | Constraint (SDK-enforced)        |
| ----------------- | ------------------- | -------------------------------- |
| `splitThreshold`  | `100 * 10^18` (wei) | MUST be `>= MIN_SPLIT_THRESHOLD` |
| `maxSubTrades`    | `10`                | MUST be in `[2, 100]`            |
| `minSubTradeSize` | `1 * 10^18` (wei)   | MUST be `> 0`                    |

The default `maxSubTrades` equals the Oracle's `MAX_BATCH_SIZE`, so a default plan fits one `submitComplianceBatch` call.

#### Single-leg plans (N = 1)

When `planSplit` returns one leg, clients MUST NOT call `registerTrade`; the registry reverts `InvalidSubTradeCount(1)`. The single proof is submitted to the Oracle directly and no PATTERN proof is required by this XIP. No registry record exists for such a trade.

#### Minimum split threshold

`MIN_SPLIT_THRESHOLD` is enforced by the SDK (`split.ts` `validateConfig`) only. The registry never sees amounts, so nothing on-chain enforces it, and a client that does not use `planSplit` is not bound by it.

The floor is defined per asset, in that asset's base units:

| Asset    | `MIN_SPLIT_THRESHOLD` |
| -------- | --------------------- |
| ETH/WETH | `10 * 10^18` (10 ETH) |

Implementations MUST NOT split a trade in an asset with no row in this table. Not yet implemented: `planSplit` takes no asset argument and applies `10 * 10^18` base units to every asset. For an 18-decimal USD stablecoin this is a floor of 10 USD; for 6-decimal USDC it is `10^13` USDC, so USDC trades never split.

#### Known limits (remedy pending)

- **Pattern circuit capacity.** The `pattern` circuit analyzes at most `MAX_TRANSACTIONS = 16` transactions (ERC-8262 `circuits/shared/src/constants.nr`). `maxSubTrades` and the registry allow up to 100 legs. A single PATTERN proof cannot cover a trade with more than 16 legs.
- **Pattern amount range.** The circuit takes `amounts: [u64; 16]` (ERC-8262 `circuits/pattern/src/main.nr`). The largest `u64` is `18446744073709551615`, about 18.45 ETH in wei, so a 100 ETH leg in wei cannot be an input. The SDK's `PatternInput.amounts` is `number[]`, exact only up to `2^53 - 1` (about 0.009 ETH in wei). No unit or scaling from leg amounts to circuit amounts is defined, and the registry does not check one.

#### proveBatch

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
  prover: ERC8262Prover,
  plan: SplitPlan,
  baseInput: ComplianceInput | RiskScoreInput,
): Promise<BatchProveResult>;
```

Behaviour as shipped (`batch-prover.ts`):

- It proves one leg per `plan.subTrades` entry, in index order, sequentially. Implementations MUST NOT prove in parallel: the Barretenberg backend (`@aztec/bb.js`) is not concurrency-safe.
- Every leg uses the same `baseInput`, so every leg has the same `submitter`, `jurisdictionId`, `configHash` and `providerSetHash`. Proofs are ZK-flavour UltraHonk proofs with a masking polynomial, so identical inputs still give distinct proof bytes and distinct Oracle `proofHash` values.
- A `ComplianceInput` produces `COMPLIANCE` (`0x01`) proofs. Signed variants (`0x07`, `0x09`) are proved per leg by the caller, outside `proveBatch`.
- On a proving error at leg K it throws and returns nothing; the K proofs already generated are discarded. A caller that needs per-leg recovery SHOULD prove each leg itself with `prover.proveCompliance` just before submitting it.

Leg proofs MUST be a compliance variant (`0x01`, `0x07`, `0x09`); the registry enforces this (`NonComplianceProofType`). Not yet implemented: `proveBatch` still accepts `RiskScoreInput` and produces `RISK_SCORE` proofs, which the Oracle accepts and the registry then rejects.

Compliance proofs attest that the subject's risk score is below the jurisdiction threshold. They take no amount input. `SubTrade.amount` tells the settlement layer (DEX, AMM, application contract) how much to transfer per leg; no proof or contract in this XIP binds a leg's proof to its amount.

#### Oracle constraints on leg proofs

Leg proofs are subject to these shipped Oracle rules (`ERC8262Oracle.sol`):

1. **Freshness.** A proof's `timestamp` public input MUST NOT be after `block.timestamp` (`ProofTimestampInFuture`) and MUST NOT be more than `MAX_PROOF_AGE = 1 hours` before it (`ProofTimestampStale`).
2. **Ordering ratchet.** For each compliance variant the Oracle requires non-decreasing proof timestamps per `(subject, jurisdictionId, proofType)`. Types that write the compliance slot (`COMPLIANCE_MULTI_SIGNED` always; `COMPLIANCE` and `COMPLIANCE_SIGNED` where the jurisdiction's multi-provider floor is 1) are also ratcheted per `(subject, jurisdictionId)`. An older proof reverts `ProofTimestampNotMonotonic`. Equal timestamps are allowed. PATTERN proofs are not ratcheted.
3. **Replay.** Each `proofHash` is accepted once (`ProofAlreadyUsed`).
4. **Attestation TTL.** Each attestation expires `attestationTTL` after submission: 24 hours by default, configurable in `[1 hours, 30 days]`. The registry rejects an expired leg at record time.
5. **Submitter binding.** The proof's `submitter` public input MUST equal `msg.sender` (`SubmitterMismatch`).

Consequences for clients:

- A leg MUST be submitted within 1 hour of its proof timestamp. Clients SHOULD date proofs from the chain (`ERC8262Prover` with a `client`, see xochi-sdk README "Proof timestamps") and SHOULD prove each leg just before submitting it when submissions span more than a few minutes.
- Re-proving a failed leg gives it a later timestamp. Once it is submitted, every unsubmitted proof of the same key with an earlier timestamp is dead and MUST be re-proved.
- Any other compliance proof the subject submits in the same jurisdiction (another split, or an unsplit trade) advances the same ratchet. Clients SHOULD keep at most one split per `(subject, jurisdictionId)` between proving and submission. Not yet implemented: the SDK does not track in-flight splits; the Oracle rejects the late legs.
- A leg MUST be recorded before its attestation expires (24 hours by default), even though the trade itself lives 7 days.

### Layer 1: SettlementRegistry

A standalone contract that links leg proofs to a trade identifier. The Oracle and Verifier are not modified.

#### Deployment

The registry is immutable: no owner, no pause, no upgrade path. The Oracle address is set at construction.

```solidity
constructor(address oracle_); // reverts ZeroAddress() if oracle_ == address(0)
```

`oracle_` MUST be a deployed `ERC8262Oracle`. The registry calls `isAttestationRevoked`, which is on `ERC8262Oracle` but not on `IERC8262Oracle`. The address is exposed as `oracle()`.

#### Interface

Copied from ERC-8262 `src/interfaces/ISettlementRegistry.sol` (imports `IERC8262Oracle`). Implementations MUST match this ABI.

```solidity
interface ISettlementRegistry {
    /// @notice A registered trade awaiting sub-settlement completion
    /// @dev pending -> finalized (`finalizeTrade`) or pending -> expired (`expireTrade`);
    ///      terminal flags are mutually exclusive.
    struct Settlement {
        bytes32 tradeId;
        address subject; // address that registered the trade
        uint8 jurisdictionId;
        uint8 subTradeCount; // total sub-trades expected
        uint8 settledCount; // sub-trades settled so far
        uint256 createdAt;
        uint256 expiresAt;
        bool finalized; // true only after a successful finalizeTrade
        bool expired; // true only after expireTrade closed the trade unfinalized
    }

    /// @notice A single sub-settlement linked to a compliance proof
    struct SubSettlement {
        uint8 index; // sub-trade index within the trade
        bytes32 proofHash; // proof hash from the oracle
        uint256 settledAt; // block.timestamp when recorded
    }

    // -------------------------------------------------------------------------
    // Events
    // -------------------------------------------------------------------------

    /// @notice Emitted when a new trade is registered
    event TradeRegistered(
        bytes32 indexed tradeId, address indexed subject, uint8 indexed jurisdictionId, uint8 subTradeCount
    );

    /// @notice Emitted when a sub-settlement is recorded against a trade
    event SubSettlementRecorded(bytes32 indexed tradeId, uint8 indexed index, bytes32 indexed proofHash);

    /// @notice Emitted when all sub-settlements are complete and the trade is finalized
    event TradeFinalized(bytes32 indexed tradeId, uint256 timestamp);

    /// @notice Emitted when a trade expires without finalization
    event TradeExpired(bytes32 indexed tradeId, uint256 timestamp);

    // -------------------------------------------------------------------------
    // Errors
    // -------------------------------------------------------------------------

    error TradeAlreadyExists(bytes32 tradeId);
    error TradeNotFound(bytes32 tradeId);
    error SubTradeIndexOutOfBounds(uint8 index, uint8 subTradeCount);
    error SubTradeAlreadySettled(bytes32 tradeId, uint8 index);
    error NotTradeSubject(address caller, address subject);
    error TradeAlreadyFinalized(bytes32 tradeId);
    error TradeNotComplete(bytes32 tradeId, uint8 settledCount, uint8 subTradeCount);
    error AttestationNotFound(bytes32 proofHash);
    error SubjectMismatch(address expected, address actual);
    error JurisdictionMismatch(uint8 expected, uint8 actual);
    error TradeExpiredError(bytes32 tradeId);
    error TradeNotExpired(bytes32 tradeId);
    error PatternProofRequired(bytes32 tradeId);
    error InvalidSubTradeCount(uint8 count);
    error PatternPublicInputsMismatch(bytes32 expected, bytes32 actual);
    error PatternAnalysisTypeMismatch(uint256 expected, uint256 actual);
    /// @notice Raised when the pattern proof's `settlement_root` public input does
    ///         not match the keccak commitment over this trade's recorded
    ///         sub-settlement proof hashes (audit H-1).
    error SettlementRootMismatch(bytes32 expected, bytes32 actual);
    /// @notice Raised when the same pattern proof is presented to finalize more
    ///         than one trade (audit H-1 reuse-prevention).
    error PatternProofAlreadyUsed(bytes32 patternProofHash);
    /// @notice Raised when a sub-settlement proof is not one of the compliance
    ///         variants (COMPLIANCE / COMPLIANCE_SIGNED / COMPLIANCE_MULTI_SIGNED).
    /// @dev The registry advertises "verified compliance attestation" semantics; a
    ///      MEMBERSHIP/RISK_SCORE/PATTERN/etc. proof for the same subject and
    ///      jurisdiction would otherwise satisfy `recordSubSettlement` and silently
    ///      substitute a non-AML guarantee.
    error NonComplianceProofType(bytes32 proofHash, uint8 proofType);
    /// @notice Leg attestation is past its oracle `expiresAt`.
    error AttestationExpired(bytes32 proofHash, uint256 expiresAt);
    error AttestationRevoked(bytes32 proofHash);
    /// @notice Leg attestation was recorded (Oracle submission time) before the trade was registered.
    error AttestationPredatesTrade(bytes32 proofHash, uint256 attestedAt, uint256 tradeCreatedAt);
    /// @notice The PATTERN proof's own `timestamp` public input predates trade registration.
    /// @dev `proofTimestamp` is public input 7, not the Oracle attestation time, which may be later.
    error PatternProofPredatesTrade(bytes32 proofHash, uint256 proofTimestamp, uint256 tradeCreatedAt);
    /// @notice Proof hash already backs another leg of this trade.
    error DuplicateSubSettlementProof(bytes32 tradeId, bytes32 proofHash);

    // -------------------------------------------------------------------------
    // Functions
    // -------------------------------------------------------------------------

    /// @notice Register a new multi-leg trade for settlement tracking
    /// @param tradeId Unique identifier for the trade
    /// @param jurisdictionId Jurisdiction for compliance verification
    /// @param subTradeCount Number of sub-trades (must be in [2, 100])
    function registerTrade(bytes32 tradeId, uint8 jurisdictionId, uint8 subTradeCount) external;

    /// @notice Record a sub-settlement by linking it to a verified compliance proof
    /// @dev Attestation: compliance variant for the trade's subject and jurisdiction,
    ///      `block.timestamp <= expiresAt`, `timestamp >= createdAt`, unused in this trade.
    /// @param tradeId The trade to record against
    /// @param index The sub-trade index (0-based, must be < subTradeCount)
    /// @param proofHash The proof hash from a prior oracle submission
    function recordSubSettlement(bytes32 tradeId, uint8 index, bytes32 proofHash) external;

    /// @notice Finalize a trade after all sub-settlements are recorded
    /// @dev Requires a pattern detection proof of analysis_type=STRUCTURING for the subject.
    ///      The caller supplies the original public inputs so the registry can verify
    ///      keccak256(patternPublicInputs) matches the attestation's stored publicInputsHash,
    ///      then enforces analysis_type==1 (anti-structuring). VELOCITY/ROUND-AMOUNT proofs
    ///      are rejected. The proof's `timestamp` public input must be >= `createdAt`
    ///      (`PatternProofPredatesTrade`); the attestation's submission time is not enough.
    /// @param tradeId The trade to finalize
    /// @param patternProofHash Proof hash of a pattern detection proof for the subject
    /// @param patternPublicInputs Original public inputs used to generate the pattern proof
    function finalizeTrade(bytes32 tradeId, bytes32 patternProofHash, bytes calldata patternPublicInputs) external;

    /// @notice Expire a trade that has passed its expiry window without finalization
    /// @dev Permissionless after expiry. Sets `expired`, never `finalized`.
    /// @param tradeId The trade to expire
    function expireTrade(bytes32 tradeId) external;

    /// @notice Get a settlement record
    /// @param tradeId The trade to query
    /// @return settlement The settlement data
    function getSettlement(bytes32 tradeId) external view returns (Settlement memory settlement);

    /// @notice Get all recorded sub-settlements for a trade
    /// @param tradeId The trade to query
    /// @return subSettlements Array of recorded sub-settlements (excludes unset entries)
    function getSubSettlements(bytes32 tradeId) external view returns (SubSettlement[] memory subSettlements);

    /// @notice The oracle contract used for attestation lookups
    /// @return The oracle address
    function oracle() external view returns (IERC8262Oracle);
}
```

The contract (`src/SettlementRegistry.sol`) MUST also expose these functions, which are not in the interface:

```solidity
error ZeroAddress();
function computeSettlementRoot(bytes32 tradeId) external view returns (bytes32 root); // reverts TradeNotFound
function isPatternProofUsed(bytes32 patternProofHash) external view returns (bool used);
```

#### Constants and storage

| Name               | Value                                                                           |
| ------------------ | ------------------------------------------------------------------------------- |
| `TRADE_TTL`        | `7 days`                                                                        |
| `MIN_SUB_TRADES`   | `2`                                                                             |
| `MAX_SUB_TRADES`   | `100`                                                                           |
| `BN254_FR_MODULUS` | `21888242871839275222246405745257275088548364400416034343698204186575808495617` |

```solidity
mapping(bytes32 tradeId => Settlement settlement) internal _settlements;
mapping(bytes32 tradeId => mapping(uint8 index => SubSettlement subSettlement)) internal _subSettlements;
mapping(bytes32 patternProofHash => bool used) internal _usedPatternProofs;
mapping(bytes32 tradeId => mapping(bytes32 proofHash => bool used)) internal _usedLegProofs;
```

#### Behavior

All checks below are enforced by the registry contract. Each list is in revert order.

1. `registerTrade(tradeId, jurisdictionId, subTradeCount)` MUST revert `TradeAlreadyExists` if `tradeId` has a record. It MUST revert `InvalidSubTradeCount` unless `subTradeCount` is in `[2, 100]`. It stores a `Settlement` with `subject = msg.sender`, `createdAt = block.timestamp`, `expiresAt = createdAt + 7 days`, `finalized = false`, `expired = false`, and emits `TradeRegistered`. `tradeId` is opaque to the registry: its preimage is not checked. `jurisdictionId` is not validated here; a trade with an invalid jurisdiction can never record a leg.

2. `recordSubSettlement(tradeId, index, proofHash)` MUST revert:
   1. `TradeNotFound` if the trade has no record.
   2. `NotTradeSubject` unless `msg.sender == subject`.
   3. `TradeAlreadyFinalized` if `finalized`.
   4. `TradeExpiredError` if `block.timestamp > expiresAt`.
   5. `SubTradeIndexOutOfBounds` unless `index < subTradeCount`.
   6. `SubTradeAlreadySettled` if `index` is already recorded.
   7. `AttestationNotFound` if `oracle.getHistoricalProof(proofHash)` reverts.
   8. `NonComplianceProofType` unless `attestation.proofType` is `0x01`, `0x07` or `0x09`.
   9. `SubjectMismatch` unless `attestation.subject == subject`.
   10. `JurisdictionMismatch` unless `attestation.jurisdictionId == jurisdictionId`.
   11. `AttestationExpired` if `block.timestamp > attestation.expiresAt`.
   12. `AttestationRevoked` if `oracle.isAttestationRevoked(proofHash)`.
   13. `AttestationPredatesTrade` if `attestation.timestamp < createdAt`. `attestation.timestamp` is the Oracle submission time, so a leg proved before `registerTrade` and submitted after it passes.
   14. `DuplicateSubSettlementProof` if `proofHash` already backs a leg of this trade.

   On success it marks `proofHash` used for this trade, stores `SubSettlement{index, proofHash, block.timestamp}`, increments `settledCount`, and emits `SubSettlementRecorded`.

3. `finalizeTrade(tradeId, patternProofHash, patternPublicInputs)` MUST revert:
   1. `TradeNotFound`, `NotTradeSubject`, `TradeAlreadyFinalized`, `TradeExpiredError` as in 2.1 to 2.4.
   2. `TradeNotComplete` unless `settledCount == subTradeCount`.
   3. `PatternProofRequired` if `patternProofHash == bytes32(0)`.
   4. `PatternProofAlreadyUsed` if `patternProofHash` finalized any trade before.
   5. `PatternProofRequired` unless `oracle.getProofType(patternProofHash) == 0x03`. A missing attestation reverts `AttestationNotFound(patternProofHash)` from the Oracle's `getProofType`.
   6. `SubjectMismatch` unless the PATTERN attestation's `subject == subject`.
   7. `AttestationRevoked` if `oracle.isAttestationRevoked(patternProofHash)`.
   8. `PatternPublicInputsMismatch` unless `keccak256(patternPublicInputs) == attestation.publicInputsHash`.
   9. `PatternProofPredatesTrade` if the proof's `timestamp` public input (word 7) `< createdAt`.
   10. `PatternAnalysisTypeMismatch` unless `analysis_type` (word 0) `== 1` (STRUCTURING).
   11. `SettlementRootMismatch` unless `settlement_root` (word 6) `== computeSettlementRoot(tradeId)`.

   On success it marks `patternProofHash` used, sets `finalized = true`, and emits `TradeFinalized`. PATTERN public inputs are 8 words of 32 bytes: `[0] analysis_type, [1] result, [2] reporting_threshold, [3] time_window, [4] tx_set_hash, [5] submitter, [6] settlement_root, [7] timestamp`. The PATTERN attestation's jurisdiction and its `expiresAt` are not checked. Recorded legs are not re-checked for revocation or expiry.

4. `computeSettlementRoot(tradeId)` MUST return

   ```
   bytes32(uint256(keccak256(abi.encode(uint8 subTradeCount, bytes32[] proofHashes))) % BN254_FR_MODULUS)
   ```

   where `proofHashes[i]` is the recorded `proofHash` at index `i` for `i` in `[0, subTradeCount)`. `abi.encode(uint8, bytes32[])` is `96 + 32 * N` bytes: the padded count, the array offset `0x40`, the array length, then the elements. It is not a packed concatenation. Provers MUST call this view (or reproduce it bit for bit) after the last leg is recorded and before proving the PATTERN proof.

5. `expireTrade(tradeId)` is callable by anyone. It MUST revert `TradeNotFound`, `TradeAlreadyFinalized` if `finalized`, `TradeExpiredError` if already `expired`, and `TradeNotExpired` unless `block.timestamp > expiresAt`. It sets `expired = true` and MUST NOT set `finalized`. `finalized` is true only after a successful `finalizeTrade`. It emits `TradeExpired`. Leg attestations stay valid in the Oracle.

6. `getSettlement` and `getSubSettlements` MUST revert `TradeNotFound` for an unknown trade. `getSubSettlements` returns only recorded legs, in index order.

Consumers that release value on registry state MUST read `finalized`, not `!expired` or `settledCount`.

#### Anti-Structuring Requirement

Split settlements MUST be accompanied by a PATTERN proof (`0x03`) with `analysis_type == 1` (STRUCTURING) to finalize. This requirement exists because the per-leg compliance proofs attest to user-level risk scores, not to the split structure. It is enforced by the registry, in `finalizeTrade` only (checks 3.3 to 3.11 above); a trade that is never finalized is never checked.

`finalizeTrade` takes `patternProofHash` and the proof's original `patternPublicInputs`. Any check failure reverts with the error listed in Behavior item 3. `settlement_root` binds the proof to this trade's recorded leg `proofHash` values and to nothing else: the circuit leaves `settlement_root` unconstrained, and the analyzed amounts and timestamps are private, unsigned prover inputs. The STRUCTURING analysis flags a set in which at least `ceil(num_transactions / 2)` amounts fall in `[reporting_threshold * 90 / 100, reporting_threshold)`. `reporting_threshold` can be any value registered in the Oracle; it is not bound to the trade's jurisdiction. See Known limits for the 16-transaction and `u64` bounds.

#### Submitter binding

The Oracle enforces `submitter == msg.sender` for every proof type. This XIP uses direct submission with bookkeeping: the subject submits each leg to the Oracle, then calls `recordSubSettlement(tradeId, index, proofHash)` from the same address. The registry looks the attestation up with `getHistoricalProof`.

Having the registry call the Oracle as a proxy was rejected. The registry would become `msg.sender` and the attestation subject, which breaks `attestation.subject == the entity that proved compliance`. See Rationale.

### Layer 2: Batch submission (shipped, OPTIONAL)

The Oracle ships:

```solidity
function submitComplianceBatch(
    uint8 jurisdictionId,
    uint8[] calldata proofTypes,
    bytes[] calldata proofs,
    bytes[] calldata publicInputs,
    bytes32[] calldata providerSetHashes
) external returns (ComplianceAttestation[] memory attestations);
```

One `jurisdictionId` covers every entry. The batch MUST hold 1 to `MAX_BATCH_SIZE = 10` entries (`EmptyBatch`, `BatchTooLarge`, `BatchLengthMismatch`). Each entry runs the same validation and storage as `submitCompliance`, in input order, and the call is atomic: one failing entry reverts the batch. The ratchet applies entry by entry, so proofs in a batch MUST be in non-decreasing timestamp order; `proveBatch` produces them that way. The SDK wraps it as `ERC8262Oracle.submitBatch({ batch, jurisdictionId, proofType, providerSetHash })`, which uses one proof type for all entries and returns each leg's `proofHash` from the `ComplianceVerified` events. Trades with more than 10 legs need several batches or individual submissions.

### Execution flow

For N >= 2 a client MUST perform these steps in this order:

1. `planSplit` (SDK).
2. `registerTrade(tradeId, jurisdictionId, N)` and wait for inclusion. Legs submitted to the Oracle before this block revert `AttestationPredatesTrade` when recorded.
3. Prove the legs (`proveBatch` or per leg) and submit them to the Oracle (`submitCompliance` or `submitComplianceBatch`) within `MAX_PROOF_AGE` of each proof's timestamp.
4. `recordSubSettlement` for every index, each before its attestation expires.
5. `computeSettlementRoot(tradeId)`; prove a STRUCTURING PATTERN proof with that `settlement_root` and a `timestamp` at or after `createdAt` (SDK: `notBefore: createdAt`); submit it to the Oracle within `MAX_PROOF_AGE`.
6. `finalizeTrade(tradeId, patternProofHash, patternPublicInputs)` before `expiresAt`.

### SDK integration

`@xochi/sdk` exports (`packages/sdk/src/index.ts`):

```typescript
export { planSplit, DEFAULT_SPLIT_CONFIG } from "./split.js";
export { proveBatch, provePlan } from "./batch-prover.js";
export {
  SettlementRegistryClient,
  SETTLEMENT_REGISTRY_ABI,
} from "./settlement-registry.js";
export { ERC8262Oracle, MAX_BATCH_SIZE } from "./oracle.js";
export type { SplitConfig, SplitPlan, SubTrade } from "./split.js";
export type { BatchProveResult } from "./batch-prover.js";
export type { Settlement, SubSettlement } from "./settlement-registry.js";
```

`SettlementRegistryClient` is a typed viem wrapper with the same pattern as `ERC8262Oracle`. Its methods map one to one onto the registry: `registerTrade`, `recordSubSettlement`, `finalizeTrade(tradeId, patternProofHash, patternPublicInputs)`, `expireTrade`, `getSettlement`, `getSubSettlements`, `computeSettlementRoot`, `oracle`. `provePlan` is the XIP-2 adapter over `proveBatch`.

### Trust boundaries

| Requirement                                                                | Enforced by                                     |
| -------------------------------------------------------------------------- | ----------------------------------------------- |
| `MIN_SPLIT_THRESHOLD`, `maxSubTrades` in `[2, 100]`, `minSubTradeSize > 0` | SDK (`planSplit`) only                          |
| Deterministic split, CSPRNG nonce                                          | SDK (`planSplit`) only                          |
| Sequential proving                                                         | SDK (`proveBatch`)                              |
| Leg proofs are compliance variants                                         | Registry (`NonComplianceProofType`)             |
| N = 1 skips the registry                                                   | Client; registry reverts `InvalidSubTradeCount` |
| `subTradeCount` in `[2, 100]`                                              | Registry                                        |
| Leg subject, jurisdiction, liveness, revocation, not predating             | Registry                                        |
| No duplicate leg proof within a trade                                      | Registry (not across trades)                    |
| Finalization checks 3.1 to 3.11                                            | Registry                                        |
| `submitter == msg.sender`                                                  | Oracle                                          |
| Proof freshness (1 h) and timestamp ratchet                                | Oracle                                          |
| `reporting_threshold` is a registered value                                | Oracle (any jurisdiction's value)               |
| PATTERN result matches the analyzed set                                    | Circuit                                         |
| One split in flight per `(subject, jurisdictionId)`                        | Nothing (SHOULD; late legs revert in Oracle)    |
| Leg amounts actually transferred match the plan                            | Nothing                                         |
| Per-asset floor                                                            | Nothing (Not yet implemented)                   |

Caller- and prover-supplied inputs and what binds them:

- `tradeId`: caller. Opaque to the registry; the first caller to register it becomes `subject`.
- `jurisdictionId`, `subTradeCount`: caller. The registry requires legs to match the jurisdiction; nothing binds the count to `totalAmount`.
- Leg `proofHash`: caller. Bound by the Oracle lookup to subject, jurisdiction, type, liveness and submission time.
- Leg proof `timestamp`: prover. Bound by Oracle freshness and the ratchet.
- `patternPublicInputs`: caller. Bound by hash to the PATTERN attestation.
- PATTERN `settlement_root`: prover. Checked by the registry against the recorded leg `proofHash` values.
- PATTERN `reporting_threshold`, `time_window`, `timestamp`: prover. Bound by the Oracle's registered-threshold set, `[MIN_TIME_WINDOW, MAX_TIME_WINDOW]` = `[1 hours, 90 days]`, freshness, and the registry's `>= createdAt`.
- PATTERN `amounts`, `timestamps`, `num_transactions`: prover, private and unsigned. Bound only by `tx_set_hash`, which nothing compares with the legs.

## Rationale

### Why a separate SettlementRegistry, not an Oracle extension?

The Oracle answers "does this address meet the compliance threshold for this jurisdiction?". Splitting is orchestration: how a large trade is broken into proven pieces. The two have different lifecycles, audit scopes and upgrade cadences.

Extending the Oracle with `tradeId` would:

- Widen the Oracle's attack surface.
- Break the `ComplianceAttestation` struct ABI for existing consumers.
- Couple settlement policy (leg counts, trade TTL) to compliance policy (jurisdiction thresholds, attestation TTL).

### Why no randomized amounts?

The design originally proposed noise on leg amounts. It was rejected for three reasons:

1. **No ZK benefit.** Compliance proofs take no amount input, so noisy amounts add complexity without any proof-level privacy.
2. **Summing attack.** An observer who sees every leg can sum them to recover the total regardless of noise.
3. **Shielded settlement covers the users who need it.** Per xochi-sdk `tiers.ts` (`PRIVACY_LEVELS`), the `private` (trust score >= 50) and `sovereign` (>= 75) levels settle on Aztec, where leg amounts are not visible. `stealth` (score >= 0) settles on L1 through ERC-5564 with visible amounts; `open`, `public` and `standard` settle publicly. On every level the leg count stays visible through the registry.

### Why no settlement delay in the SDK?

When legs are submitted is the caller's decision. Baking delays into `planSplit` or `proveBatch` would couple them to execution timing. XIP-2 specifies timing separately, subject to the 1-hour proof age above.

### Why sequential proof generation?

`@aztec/bb.js` uses shared WASM state that is not thread-safe; parallel proving corrupts witnesses. The SDK test config also runs sequentially (`test.sequence.concurrent: false`).

### Why direct submission over the registry as proxy?

With the registry as proxy, the proof's `submitter` and the attestation's `subject` would be the registry, and the registry would need its own access control to map itself back to the user. That conflates "who orchestrated" with "who is compliant". Direct submission keeps `attestation.subject` equal to the user and records only the linkage.

### Why only single-chain splitting?

Cross-chain splitting adds bridge trust assumptions unrelated to proof splitting. Each chain's registry operates independently. Cross-chain coordination is a candidate for a later XIP.

### Why no dynamic re-splitting?

If leg K fails, the user retries leg K or abandons the trade. `subTradeCount` is fixed at registration, so re-splitting would need on-chain amount tracking for a failure the client can handle.

### Why no relayer support?

`submitter == msg.sender` rules out relayers and meta-transactions. Supporting them needs a different binding in the Oracle and circuits, which is out of scope.

### Relationship to stealth address splitting

The [Xochi whitepaper](https://xochi.fi/whitepaper) (Appendix, "Settlement Splitting") describes multi-account stealth address splitting: N stealth keys derived from one meta-address, with the amount spread across N ERC-5564 accounts. That works at the transfer layer. This XIP works at the compliance-proof layer. The two compose.

## Backwards Compatibility

- The Oracle (`ERC8262Oracle`), Verifier (`ERC8262Verifier`) and `ComplianceAttestation` struct are not modified.
- `SettlementRegistry` is a new standalone contract; `planSplit`, `proveBatch`, `provePlan` and `SettlementRegistryClient` are new SDK exports.
- Consumers that do not split are unaffected.

The registry ABI in this version differs from the text merged in PR #1 and PR #4 (`uint8` counts and indexes, three-argument `finalizeTrade`, the `expired` flag, new errors, `IERC8262Oracle`). No implementation of the earlier text exists; the shipped contract has always matched this version's ABI.

## Test Cases

### Split planning

Amounts in wei. Defaults unless stated: `splitThreshold = 100000000000000000000`, `maxSubTrades = 10`, `minSubTradeSize = 1000000000000000000`. Every output sums to `totalAmount`.

| #   | `totalAmount`            | Config override                                                                  | Legs | Output (wei)                                                                                     |
| --- | ------------------------ | -------------------------------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------ |
| 1   | `500000000000000000000`  | none                                                                             | 5    | 5 x `100000000000000000000`                                                                      |
| 2   | `350000000000000000000`  | none                                                                             | 4    | 4 x `87500000000000000000`                                                                       |
| 3   | `350000000000000000001`  | none                                                                             | 4    | 3 x `87500000000000000000`, then `87500000000000000001`                                          |
| 4   | `100000000000000000000`  | none                                                                             | 1    | `100000000000000000000` (equal to threshold, no split)                                           |
| 5   | `100000000000000000001`  | none                                                                             | 2    | `50000000000000000000`, `50000000000000000001`                                                   |
| 6   | `50000000000000000000`   | none                                                                             | 1    | `50000000000000000000`                                                                           |
| 7   | `1000000000000000000000` | `maxSubTrades = 5`                                                               | 5    | 5 x `200000000000000000000` (clamped)                                                            |
| 8   | `25000000000000000001`   | `splitThreshold = 10000000000000000000`, `minSubTradeSize = 9000000000000000000` | 2    | `12500000000000000000`, `12500000000000000001`                                                   |
| 9   | `11000000000000000000`   | `splitThreshold = 10000000000000000000`, `minSubTradeSize = 6000000000000000000` | 1    | `11000000000000000000` (reduced to N = 1)                                                        |
| 10  | `1000000000000000000000` | `splitThreshold = 10000000000000000000`, `maxSubTrades = 100`                    | 100  | 100 x `10000000000000000000` (valid plan; exceeds the 16-leg pattern capacity, see Known limits) |
| 11  | `10000000000000000000`   | `splitThreshold = 3000000000000000000`                                           | -    | Throws: below `MIN_SPLIT_THRESHOLD`                                                              |
| 12  | `500000000000000000000`  | `maxSubTrades = 101`                                                             | -    | Throws: `maxSubTrades` outside `[2, 100]`                                                        |

Row 8: `ceil(25000000000000000001 / 10000000000000000000) = 3`; `25000000000000000001 / 3 = 8333333333333333333 < 9000000000000000000`, so `n = 2`. Row 9: `n = 2`, `5500000000000000000 < 6000000000000000000`, so `n = 1`.

### Settlement root

```
subTradeCount = 2
proofHashes   = [0x1111111111111111111111111111111111111111111111111111111111111111,
                 0x2222222222222222222222222222222222222222222222222222222222222222]
abi.encode    = 0x0000000000000000000000000000000000000000000000000000000000000002
                  0000000000000000000000000000000000000000000000000000000000000040
                  0000000000000000000000000000000000000000000000000000000000000002
                  1111111111111111111111111111111111111111111111111111111111111111
                  2222222222222222222222222222222222222222222222222222222222222222
keccak256     = 0xac69297b8dc306800c2b3e53630ac6f51d61973038a4efe6e85a3151669e7a95
root          = 0x1b3c3e22ea2e2602e33a6d2fde86bddda4c5de56cb789e331cb45095969e7a92  (keccak mod BN254_FR_MODULUS)
```

### End-to-end (Alice, EU = 0, 500 ETH)

```
1. planSplit(500000000000000000000, 0, alice) -> tradeId T, 5 legs of 100000000000000000000
2. registry.registerTrade(T, 0, 5) -> createdAt C, expiresAt C + 604800
3. proveBatch(prover, plan, complianceInput) -> 5 COMPLIANCE proofs, timestamps non-decreasing, all >= C - 3600
4. oracle.submitComplianceBatch(0, [0x01 x5], proofs, publicInputs, [providerSetHash x5]) within 1 h of the proof timestamps
   -> proofHash[0..4], attestation.timestamp >= C
5. for i in 0..4: registry.recordSubSettlement(T, i, proofHash[i])
6. root = registry.computeSettlementRoot(T)
7. prover.provePattern({ analysisType: 1, settlementRoot: root, submitter: alice, notBefore: C, ... })
   oracle.submitCompliance(0, 0x03, patternProof, patternPublicInputs, providerSetHash) -> P
8. registry.finalizeTrade(T, P, patternPublicInputs)
9. registry.getSettlement(T) -> { subject: alice, subTradeCount: 5, settledCount: 5, finalized: true, expired: false }
```

### Registry guards

Starting from step 2 above unless stated:

| Action                                                       | Result                                         |
| ------------------------------------------------------------ | ---------------------------------------------- |
| `registerTrade(T2, 0, 1)`                                    | `InvalidSubTradeCount(1)`                      |
| `registerTrade(T2, 0, 101)`                                  | `InvalidSubTradeCount(101)`                    |
| `registerTrade(T, 0, 5)` again                               | `TradeAlreadyExists(T)`                        |
| Record a MEMBERSHIP (`0x05`) or PATTERN proof as a leg       | `NonComplianceProofType(proofHash, type)`      |
| Record `proofHash[0]` at index 0 and again at index 1        | `DuplicateSubSettlementProof(T, proofHash[0])` |
| Record a leg submitted to the Oracle before block C          | `AttestationPredatesTrade(...)`                |
| Record a leg after its attestation's `expiresAt`             | `AttestationExpired(...)`                      |
| `finalizeTrade` with 3 of 5 recorded                         | `TradeNotComplete(T, 3, 5)`                    |
| Finalize with a VELOCITY (`analysis_type = 2`) PATTERN proof | `PatternAnalysisTypeMismatch(1, 2)`            |
| Finalize with a PATTERN proof whose `settlement_root` is 0   | `SettlementRootMismatch(root, 0)`              |
| Finalize with a PATTERN proof whose `timestamp` is `C - 1`   | `PatternProofPredatesTrade(P, C - 1, C)`       |
| Finalize trade T2 with `P` after it finalized T              | `PatternProofAlreadyUsed(P)`                   |
| `expireTrade(T)` at `block.timestamp == C + 604800`          | `TradeNotExpired(T)`                           |
| `expireTrade(T)` at `C + 604801`, unfinalized                | `expired = true`, `finalized = false`          |
| `finalizeTrade` or `recordSubSettlement` after that          | `TradeExpiredError(T)`                         |

### Oracle constraints

| Action                                                                                        | Result                               |
| --------------------------------------------------------------------------------------------- | ------------------------------------ |
| Bob submits Alice's leg proof (`submitter = alice`)                                           | `SubmitterMismatch()`                |
| Submit a leg proof with timestamp `t` at `block.timestamp = t + 3601`                         | `ProofTimestampStale(t, t + 3601)`   |
| Legs 0-4 proved at `t0`; leg 2 re-proved at `t1 > t0` and submitted; then submit leg 3 (`t0`) | `ProofTimestampNotMonotonic(t0, t1)` |

The ERC-8262 Foundry suites cover these behaviours: `test/SettlementRegistry.t.sol`, `test/AttestationRatchet.t.sol`, `test/ValidatorChecks.t.sol`, `test/GasBenchmark.t.sol`.

## Reference Implementation

Pinned to the commits this version was checked against:

- ERC-8262 [`f018627`](https://github.com/xochi-fi/ERC-8262/tree/f018627ce498839308a0403ca06bcbbdf1140127): [`src/SettlementRegistry.sol`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/src/SettlementRegistry.sol), [`src/interfaces/ISettlementRegistry.sol`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/src/interfaces/ISettlementRegistry.sol), [`test/SettlementRegistry.t.sol`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/test/SettlementRegistry.t.sol), [`circuits/pattern/src/main.nr`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/circuits/pattern/src/main.nr).
- xochi-sdk [`068c301`](https://github.com/xochi-fi/xochi-sdk/tree/068c301eb4bc3bd783858e5e172f102389357f7c): [`packages/sdk/src/split.ts`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/src/split.ts), [`packages/sdk/src/batch-prover.ts`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/src/batch-prover.ts), [`packages/sdk/src/settlement-registry.ts`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/src/settlement-registry.ts), [`packages/sdk/docs/onchain.md`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/docs/onchain.md#settlement-splitting-xip-1).

## Security Considerations

### Front-running leg submissions

Each leg proof carries `submitter` as a public input and the Oracle enforces `submitter == msg.sender`. A copied leg proof cannot be submitted from another address (`SubmitterMismatch`; ERC-8262 `test/ValidatorChecks.t.sol` `test_compliance_revert_submitterMismatch`).

### Trade registration griefing (known issue)

`registerTrade` makes the first caller the subject and never checks the `tradeId` preimage. An attacker who sees a pending `registerTrade` can register the same `tradeId` first. The victim gets `TradeAlreadyExists`, and every retry with a new `tradeId` is exposed the same way. The attacker pays one registration per grief and gains nothing else; the victim loses time, not funds. The `submitter` term in the `tradeId` preimage does not help because the registry cannot see the preimage.

Mitigations: keying settlements by `(msg.sender, tradeId)` closes it (contract change, not implemented). Until then, clients MAY submit `registerTrade` through a private transaction channel.

### MEV from trade registration (known issue)

`TradeRegistered` publishes `subject` and `subTradeCount` before the first leg settles. Splitting is deterministic, so once one leg amount is visible on a public venue the remaining flow is about `(N - 1)` times that amount, from a known address, within the trade's lifetime. Searchers can position against the remaining legs. This works against Motivation 3.

Mitigation options: submit all legs in one `submitComplianceBatch` and settle them in as few blocks as possible; settle on a shielded venue; register through a private transaction channel. Emitting only a commitment, or registering after the legs, needs a contract change (not implemented; registering after the legs also conflicts with `AttestationPredatesTrade`).

### One attestation backing legs of several trades (known issue)

Duplicate-leg tracking is per trade (`_usedLegProofs[tradeId]`). One compliance attestation can back a leg in each of several trades with the same subject and jurisdiction. Keying it by `proofHash` alone closes this (contract change, not implemented).

### Partial settlement abandonment

A trade with M < N recorded legs stays open until `expiresAt` (7 days). Then anyone can call `expireTrade`, which sets `expired`. `finalized` stays false, so an abandoned trade is never reported as complete. Recorded legs remain valid Oracle attestations.

### Oracle attestation overwrite

Each leg overwrites the subject's typed slot `(subject, jurisdictionId, proofType)` and, for slot-writing types, the compliance slot `(subject, jurisdictionId)`. Earlier legs remain retrievable through `getHistoricalProof(proofHash)`, and the registry stores every leg's `proofHash`. `checkCompliance` returning the latest leg is correct: the subject is still compliant.

### Leg revocation after recording

`finalizeTrade` does not re-check recorded legs. A leg revoked after it was recorded still counts toward `settledCount`. Consumers that depend on every leg being live SHOULD check `ERC8262Oracle.isAttestationValid(proofHash)` for each recorded leg.

### Gas

The Oracle's own benchmark puts 10 batched compliance proofs at about 24.1M gas (`ERC8262Oracle.sol` `MAX_BATCH_SIZE` note, `test_gas_batch_atMaxSize_fitsBlockGasTarget`), about 2.4M per proof. A 100-leg trade therefore costs about 240M gas in leg submissions, at least 10 transactions, plus 100 `recordSubSettlement` calls, the PATTERN proof and `finalizeTrade` (not separately benchmarked). The cost falls on the subject, not on other users.

### Structuring risk

Settlement splitting could be misused to structure transactions below reporting thresholds. Three measures apply:

1. **PATTERN proof at finalization.** `finalizeTrade` requires a STRUCTURING PATTERN proof bound to the trade's legs through `settlement_root`. It is checked only when the subject finalizes, after every leg has settled, and its analyzed amounts are private prover inputs.
2. **Minimum split threshold.** `planSplit` rejects `splitThreshold < MIN_SPLIT_THRESHOLD`. This is SDK-only, and its value is per asset (see Minimum split threshold). Every leg of a split plan is at least `splitThreshold / 2`.
3. **On-chain linkage.** Every leg's `proofHash` is linked to one `tradeId`. The registry records no amounts and no transfers.

A trade that is never finalized, a trade at or below `splitThreshold`, and splitting done outside the registry are not subject to these checks.

### Information leakage from split count

The leg count is public on every venue: `TradeRegistered` emits it, and every leg's `submitCompliance` and `recordSubSettlement` are public transactions from the subject's address, whatever venue settles the value. When neither `maxSubTrades` nor `minSubTradeSize` reduced N, the total lies in `((N - 1) * splitThreshold, N * splitThreshold]`. Aztec settlement (`private`, `sovereign`) hides leg amounts, not the count or the timing.

## Failure Modes

### Prover failure mid-batch

If proving fails at leg K, `proveBatch` throws and the K legs already proved are discarded. Nothing has been submitted, so re-running is safe; the new proofs carry later timestamps. A client proving per leg keeps legs 0 to K-1 and re-proves only leg K, as long as legs 0 to K-1 are submitted before leg K's re-proof.

### Oracle rejects a leg

- `ProofTimestampStale`: more than 1 hour passed between proving and submission. Re-prove the leg.
- `ProofTimestampNotMonotonic`: a later-stamped compliance proof was submitted for the same key first (a re-proved leg, another split, or an unsplit trade). Re-prove this and every remaining leg.
- In a batch, one failing entry reverts the whole batch. No leg in it is recorded; the batch can be re-sent unchanged while the proofs are fresh.

Recorded legs are unaffected. If the trade cannot be completed before `expiresAt`, it expires.

### Oracle config update between legs

The Oracle accepts any config hash in `_validConfigs`, so `updateProviderConfig` alone does not break pending legs. If a config is revoked (`revokeConfig`), unsubmitted legs using it revert `InvalidConfigHash` and MUST be re-proved with a valid config. Submitted legs keep their attestations.

### Registry replacement

The registry holds no funds and stores only linkage. A replacement is a new deployment; trades on the old registry stay readable and new trades use the new one. The Oracle remains the source of truth for attestations.

### Concurrent splits

The registry has no shared state across trades, so different subjects do not contend. One subject running two splits in the same jurisdiction does contend, through the Oracle's timestamp ratchet: whichever later-stamped proof lands first invalidates the other split's earlier-stamped unsubmitted legs.

## History

- 2026-05-13: Merged as Draft (`XIPS/xip-draft_settlement-splitting.md`), PR #1.
- 2026-05-13: Reverted, PR #3 (mistaken merge).
- 2026-11-12: Re-merged as `XIPS/xip-1.md`, Draft -> Review, PR #4. Text unchanged from PR #1, so it predated the ERC-8262 audit fixes H-1, H-2 and M-1 in the shipped code.
- 2026-11-12: Review, amended in place: resynced to shipped ERC-8262 `f018627` and xochi-sdk `068c301` after a three-reviewer audit; `discussions-to` moved to #6; `requires: 0` removed.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
