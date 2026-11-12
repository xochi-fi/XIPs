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

Large trades settled under a single compliance proof concentrate risk on that proof. This XIP splits a large trade into sub-settlements. Each sub-settlement has its own ZK compliance proof, and all of them are linked by a shared trade identifier. The shipped design adds a standalone `SettlementRegistry` contract and SDK-side planning and proving without modifying the ERC-8262 Oracle or Verifier: [ERC-8262](https://github.com/xochi-fi/ERC-8262) for the contract and [xochi-sdk](https://github.com/xochi-fi/xochi-sdk) for the SDK (see Reference Implementation). This XIP also specifies the anti-structuring design that the shipped code does not yet implement: registry and lifecycle rules, a per-jurisdiction reporting threshold table in the Oracle, and (v2) compliance and pattern circuits that bind the structuring analysis to each leg's committed amount.

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
Anti-structuring:   Oracle threshold table, registry and lifecycle rules, leg chain (v2)
```

Layers 0 and 1 are normative. Layer 2 is an existing Oracle function; using it is OPTIONAL. The Anti-structuring section is normative and Not yet implemented; each rule names its enforcement point and tracking issue.

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

When `planSplit` returns one leg, clients MUST NOT call `registerTrade`; the registry reverts `InvalidSubTradeCount(1)`. The single proof is submitted to the Oracle directly, and this XIP asks for no PATTERN proof. No registry record exists for such a trade.

#### Minimum split threshold

`MIN_SPLIT_THRESHOLD` is enforced by the SDK (`split.ts` `validateConfig`) only. The registry never sees amounts, so nothing on-chain enforces it, and a client that does not use `planSplit` is not bound by it.

The floor is defined per asset, in that asset's base units:

| Asset    | `MIN_SPLIT_THRESHOLD` |
| -------- | --------------------- |
| ETH/WETH | `10 * 10^18` (10 ETH) |

Implementations MUST NOT split a trade in an asset with no row in this table. Not yet implemented: `planSplit` takes no asset argument and applies `10 * 10^18` base units to every asset. For an 18-decimal USD stablecoin this is a floor of 10 USD; for 6-decimal USDC it is `10^13` USDC, so USDC trades never split.

#### Leg cap and amount units

- **Leg cap.** The shipped `pattern` circuit analyzes at most `MAX_TRANSACTIONS = 16` transactions (ERC-8262 `circuits/shared/src/constants.nr`). Until the v2 leg chain ships, `maxSubTrades` MUST NOT exceed `MAX_SUB_TRADES`, which MUST equal the deployed circuit's `MAX_TRANSACTIONS` (rule R2). Not yet implemented: `planSplit` accepts up to 100 and the shipped registry accepts up to 100. A trade larger than 16 legs is split into several registered trades.
- **Amount units (v1).** The shipped circuit takes `amounts: [u64; 16]` and a `u64` `reporting_threshold` capped at `MAX_REPORTING_THRESHOLD = 204962515653461695`, so neither wei nor 18-decimal base units fit. Under v1 the circuit's amounts and the jurisdiction threshold are whole units of the jurisdiction's reporting currency (rule O3), and converting a leg's base-unit amount into them is the prover's choice. The SDK's `PatternInput.amounts` is `number[]`, exact only up to `2^53 - 1`. The v2 leg chain replaces this with committed `u128` 18-decimal base units (rule V7).

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

Compliance proofs attest that the subject's risk score is below the jurisdiction threshold. In the shipped circuits they take no amount input. `SubTrade.amount` tells the settlement layer (DEX, AMM, application contract) how much to transfer per leg; in the shipped design no proof or contract binds a leg's proof to its amount. The v2 leg chain makes each leg's compliance proof commit to an amount (rule V2), which still has to be checked against the value actually transferred (rule V8).

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

#### Target interface (Not yet implemented)

The Anti-structuring rules change the registry ABI and add to the Oracle. Members not listed are unchanged from the shipped interface; the constructor stays `constructor(address oracle_)`. Tracked in [ERC-8262 #38](https://github.com/xochi-fi/ERC-8262/issues/38) (registry rules), [#39](https://github.com/xochi-fi/ERC-8262/issues/39) (lapsed-trade lifecycle) and [#40](https://github.com/xochi-fi/ERC-8262/issues/40) (leg chain v2).

```solidity
interface ISettlementRegistry {
    struct Settlement {
        bytes32 tradeId; // computeTradeId(subject, tradeRef)
        address subject;
        uint8 jurisdictionId;
        uint8 subTradeCount;
        uint8 settledCount;
        uint64 lastSettledAt; // block time of the latest recorded leg, 0 before any
        uint256 reportingThreshold; // copied from the Oracle at registration
        uint256 createdAt;
        uint256 expiresAt;
        bool finalized; // true only after a successful finalizeTrade
        bool expired; // true only after expireTrade
        bool abandoned; // true only after a successful abandonTrade
        bool flagged; // true only after flagUnresolvedTrade: permanently unresolved
        bytes32 planTxSetHash; // tx_set_hash of the plan proof, zero without one
    }

    event TradeRegistered(
        bytes32 indexed tradeId,
        address indexed subject,
        uint8 indexed jurisdictionId,
        uint8 subTradeCount,
        bytes32 tradeRef,
        uint256 reportingThreshold,
        bytes32 planProofHash // zero without a plan proof
    );
    event TradeAbandoned(bytes32 indexed tradeId, uint8 settledCount, bytes32 patternProofHash);
    event TradeFlaggedUnresolved(bytes32 indexed tradeId, address indexed subject, uint8 settledCount);

    error UnsupportedJurisdiction(uint8 jurisdictionId);
    error OpenTradeLimit(address subject);
    error UnresolvedTrade(bytes32 tradeId);
    error TradeAlreadyAbandoned(bytes32 tradeId);
    error TradeAlreadyFlagged(bytes32 tradeId);
    error TradeNotFlaggable(bytes32 tradeId);
    error ReportingThresholdMismatch(uint256 expected, uint256 actual);
    error PatternProofPredatesLastLeg(bytes32 proofHash, uint256 proofTimestamp, uint256 lastSettledAt);
    error PatternWindowExcludesTrade(bytes32 proofHash, uint256 windowStart, uint256 tradeCreatedAt);
    error PatternTxSetMismatch(bytes32 expected, bytes32 actual);
    error PlanProofStale(bytes32 proofHash, uint256 proofTimestamp);
    // Changed: DuplicateSubSettlementProof(tradeId, proofHash) names the trade that already holds the proof.
    // Removed: PatternProofPredatesTrade (replaced by PatternProofPredatesLastLeg).

    function registerTrade(bytes32 tradeRef, uint8 jurisdictionId, uint8 subTradeCount)
        external
        returns (bytes32 tradeId);
    function registerTradeWithPlan(
        bytes32 tradeRef,
        uint8 jurisdictionId,
        uint8 subTradeCount,
        bytes32 planProofHash,
        bytes calldata planPublicInputs
    ) external returns (bytes32 tradeId);
    function abandonTrade(bytes32 tradeId, bytes32 patternProofHash, bytes calldata patternPublicInputs) external;
    function flagUnresolvedTrade(bytes32 tradeId) external;

    function computeTradeId(address subject, bytes32 tradeRef) external pure returns (bytes32 tradeId);
    function computePlanRoot(bytes32 tradeId, uint8 jurisdictionId, uint8 subTradeCount)
        external
        view
        returns (bytes32 root);
    function isLegAuthorized(bytes32 tradeId, uint8 index) external view returns (bool authorized);
    function openTrades(address subject) external view returns (bytes32[] memory tradeIds);
    function flaggedTradeCount(address subject) external view returns (uint256 count);
    function legProofTrade(bytes32 proofHash) external view returns (bytes32 tradeId); // zero when unused
}
```

v2 (leg chain) additions and replacements:

```solidity
    event TradeFinalized(bytes32 indexed tradeId, uint256 timestamp, bool aggregateReportable); // replaces the 2-field event
    event TradeAbandoned(bytes32 indexed tradeId, uint8 settledCount, bytes32 patternProofHash, bool aggregateReportable);
    error LegPublicInputsMismatch(bytes32 expected, bytes32 actual);
    error SettlementLinkMismatch(bytes32 tradeId, bytes32 expected, bytes32 actual);
    error SettlementSeedMismatch(bytes32 expected, bytes32 actual);
    // replaces the 3-argument form
    function recordSubSettlement(bytes32 tradeId, uint8 index, bytes32 proofHash, bytes calldata legPublicInputs) external;
    function settlementSeed(bytes32 tradeId) external view returns (bytes32 seed);
    function isAggregateReportable(bytes32 tradeId) external view returns (bool reportable);
```

Oracle additions (`ERC8262Oracle` and `IERC8262Oracle`):

```solidity
event JurisdictionReportingThresholdSet(uint8 indexed jurisdictionId, uint256 previous, uint256 threshold);
function setJurisdictionReportingThreshold(uint8 jurisdictionId, uint256 threshold) external; // REGISTRAR_ROLE
function jurisdictionReportingThreshold(uint8 jurisdictionId) external view returns (uint256 threshold);
```

#### Constants and storage (shipped)

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

#### Behavior (shipped)

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

#### Anti-structuring check (shipped)

As shipped, `finalizeTrade` requires a STRUCTURING PATTERN proof (`0x03`, `analysis_type == 1`) bound to the recorded leg `proofHash` values (checks 3.3 to 3.11). It runs only when the subject finalizes; a trade that is never finalized is never checked. The circuit leaves `settlement_root` unconstrained, the analyzed amounts and timestamps are private, unsigned prover inputs, and `reporting_threshold` can be any value registered in the Oracle. The STRUCTURING analysis flags a set in which at least `ceil(num_transactions / 2)` amounts fall in `[reporting_threshold * 90 / 100, reporting_threshold)`. The target design is in Anti-structuring below.

#### Submitter binding

The Oracle enforces `submitter == msg.sender` for every proof type. This XIP uses direct submission with bookkeeping: the subject submits each leg to the Oracle, then calls `recordSubSettlement(tradeId, index, proofHash)` from the same address. The registry looks the attestation up with `getHistoricalProof`.

Having the registry call the Oracle as a proxy was rejected. The registry would become `msg.sender` and the attestation subject, which breaks `attestation.subject == the entity that proved compliance`. See Rationale.

### Anti-structuring

Every rule in this section is normative and Not yet implemented, unless marked shipped. Shipped means ERC-8262 `f018627`. Tracking: [ERC-8262 #37](https://github.com/xochi-fi/ERC-8262/issues/37) (epic), [#38](https://github.com/xochi-fi/ERC-8262/issues/38) (Oracle threshold table and registry rules), [#39](https://github.com/xochi-fi/ERC-8262/issues/39) (lapsed-trade lifecycle), [#40](https://github.com/xochi-fi/ERC-8262/issues/40) (leg chain v2).

The goal is that splitting does not change whether a trade is reportable, and that a split which moved value does not disappear without a public record. Under these rules a registered split closes only through `finalizeTrade` or `abandonTrade` with a STRUCTURING PATTERN proof bound to its legs and to the trade jurisdiction's threshold, or by being publicly flagged unresolved.

| Layer                               | Rules               | Status                                     |
| ----------------------------------- | ------------------- | ------------------------------------------ |
| Reporting threshold table (Oracle)  | O1-O3               | Not yet implemented (#38)                  |
| Registration, leg and closing rules | R1-R5, G1-G3, C1-C6 | Not yet implemented (#38)                  |
| Lifecycle                           | L1-L6               | Not yet implemented (#39)                  |
| Plan proof                          | P1-P3               | OPTIONAL; Not yet implemented              |
| Leg chain (v2 circuits)             | V0-V8               | REQUIRED target; Not yet implemented (#40) |

Until the leg chain ships, the PATTERN proof is a self-attestation: its amounts are private, unsigned prover inputs. The registry and lifecycle rules fix when, against which threshold and for which recorded legs a proof is presented, and make abandonment public. They do not make the analysis honest. The leg chain makes the analysis cover exactly the amounts the legs committed to.

#### Definitions and constants

- **Closed**: `finalized`, `abandoned` or `flagged` is true.
- **Lapsed**: not closed and `block.timestamp > expiresAt`. `expireTrade` need not have been called.
- **Unresolved trade**: lapsed with `settledCount >= 1`. Flagging (L3) closes it permanently as unresolved.
- **Open**: not closed and not lapsed with `settledCount == 0`. A lapsed trade with no leg moved no value and frees its slot.

| Name                 | Value                                                                 |
| -------------------- | --------------------------------------------------------------------- |
| `MAX_SUB_TRADES`     | `MAX_TRANSACTIONS` of the deployed pattern circuit: 16 (v1), 100 (v2) |
| `MAX_OPEN_TRADES`    | `4`                                                                   |
| `TRADE_TTL`          | `7 days` (shipped)                                                    |
| `RESOLUTION_GRACE`   | `30 days`                                                             |
| `PLAN_PROOF_MAX_AGE` | `1 hours`                                                             |

`RESOLUTION_GRACE` is 30 days because a confession has to stay provable for the whole grace period. A closing proof needs `time_window >= timestamp - createdAt` (C3) and the Oracle caps `time_window` at `MAX_TIME_WINDOW = 90 days`, so a closing proof exists only until `createdAt + 90 days`. The last moment to abandon is `createdAt + 7 days + 30 days = createdAt + 37 days`, leaving 53 days of margin for a longer TTL or a shorter Oracle window. Thirty days also covers an operational outage, and it bounds how often one address can lapse trades: at most `MAX_OPEN_TRADES` per 37 days, each publicly flagged.

`MAX_OPEN_TRADES` is 4. Lazy lapse detection makes `registerTrade`, `recordSubSettlement` (G3) and `isLegAuthorized` scan the subject's open trades, and a fixed table bounds that scan. Four is the value the lifecycle prototype used with every honest flow passing, including two concurrent splits and a lapsed split alongside an in-flight one. The registry-hardening prototype's limit of 3 counted only part-paid trades and needed a separate release call; counting every open trade is simpler once lapses are detected lazily.

#### Reporting threshold (Oracle)

- **O1.** The Oracle MUST hold one reporting threshold per jurisdiction, read with `jurisdictionReportingThreshold(jurisdictionId)` and written with `setJurisdictionReportingThreshold(jurisdictionId, threshold)` under `REGISTRAR_ROLE`. In timelocked deployments (`script/Deploy.s.sol` with `USE_TIMELOCK`) that role is exercised through `src/Timelock.sol`. Each write MUST emit `JurisdictionReportingThresholdSet`. Zero means the jurisdiction is unsupported for splitting.
- **O2.** A non-zero threshold MUST already be in the Oracle's registered reporting-threshold set (`isValidReportingThreshold(bytes32(threshold))`); the setter MUST revert otherwise. PATTERN proofs at that value then pass the Oracle's own validation.
- **O3.** The threshold is in the unit of the deployed pattern circuit's amounts: whole units of the jurisdiction's reporting currency under v1 (it MUST NOT exceed the v1 circuit's `MAX_REPORTING_THRESHOLD = 204962515653461695`), 18-decimal base units of the reporting currency under v2. The switch to v2 MUST change the table in the same timelocked change set as the verifier upgrade. Threshold values are a governance decision; this XIP does not fix them.

#### Registration rules

- **R1.** `registerTrade(tradeRef, jurisdictionId, subTradeCount)` MUST key the trade by `tradeId = keccak256(abi.encode(msg.sender, tradeRef))` (`computeTradeId`) and return it. A copied `tradeRef` registered by another address creates a different trade. Clients MUST pass `planSplit`'s random value as `tradeRef` and use the returned `tradeId` (SDK: Not yet implemented).
- **R2.** `subTradeCount` MUST be in `[2, MAX_SUB_TRADES]` (`InvalidSubTradeCount`).
- **R3.** The registry MUST read `oracle.jurisdictionReportingThreshold(jurisdictionId)`, revert `UnsupportedJurisdiction` if it is zero, and copy it into `Settlement.reportingThreshold`. Later Oracle changes do not affect the trade.
- **R4.** It MUST revert `OpenTradeLimit` while the subject holds `MAX_OPEN_TRADES` open trades.
- **R5.** It MUST revert `UnresolvedTrade` while the subject holds an unresolved trade. Detection is lazy: the registry compares `block.timestamp` with each open trade's `expiresAt` during the call.

#### Leg rules

- **G1.** A compliance attestation MUST back at most one leg of one trade, across all trades (`DuplicateSubSettlementProof`, naming the trade that holds it; view `legProofTrade`).
- **G2.** `recordSubSettlement` MUST set `lastSettledAt = block.timestamp`, and MUST revert `TradeAlreadyAbandoned` or `TradeAlreadyFlagged` on a trade closed that way. Shipped checks 2.1 to 2.13 remain.
- **G3.** `recordSubSettlement` SHOULD revert `UnresolvedTrade` while the subject holds an unresolved trade, so legs stop on the subject's live trades too. The lifecycle prototype measured about 9.4k gas per leg for this scan.

#### Closing proof rules

These apply to `finalizeTrade` and to `abandonTrade` with at least one recorded leg. Shipped checks 3.3 to 3.8, 3.10 and 3.11 remain: non-zero, unused, PATTERN type, same subject, not revoked, public inputs hash, STRUCTURING, `settlement_root`. A closing proof is used once.

- **C1.** `settlement_root` (word 6) MUST equal `computeSettlementRoot(tradeId)`. Under v1 that is the shipped encoding over indices `[0, subTradeCount)`, with `bytes32(0)` for an unrecorded index, so an abandon proof binds to exactly the legs recorded. Under v2 it is the stored chain root (V4).
- **C2.** The proof's `timestamp` (word 7) MUST be at or after `lastSettledAt` (`PatternProofPredatesLastLeg`). This replaces shipped check 3.9. The Oracle rejects future timestamps, so the proof was submitted after the last leg was recorded.
- **C3.** `timestamp - time_window` (words 7 and 3) MUST be at or before `createdAt` (`PatternWindowExcludesTrade`). Under v2 leg timestamps are leg proof timestamps, which can precede `createdAt` by up to `MAX_PROOF_AGE`, so provers SHOULD use `time_window >= timestamp - createdAt + 1 hours`.
- **C4.** `reporting_threshold` (word 2) MUST equal `Settlement.reportingThreshold` (`ReportingThresholdMismatch`).
- **C5.** v1 only: if the trade was registered with a plan proof, `tx_set_hash` (word 4) MUST equal `planTxSetHash` (`PatternTxSetMismatch`). Under v2 `tx_set_hash` is the chain root and C5 MUST NOT be applied.
- **C6.** v2 only: the checks in V6.

#### Lifecycle rules

- **L1.** `finalizeTrade` keeps its shipped checks, applies C1 to C6, MUST revert on an abandoned or flagged trade, and frees the trade's open slot.
- **L2.** `abandonTrade(tradeId, patternProofHash, patternPublicInputs)` is subject-only (`NotTradeSubject`). It MUST revert if the trade is finalized, abandoned or flagged, and is callable before or after `expiresAt` until the trade is flagged. With no leg recorded the proof arguments are ignored. With at least one leg recorded it MUST apply the closing proof rules. It sets `abandoned = true`, MUST NOT set `finalized`, frees the open slot and emits `TradeAbandoned`. Under v2 it also records `aggregate_reportable` (V6).
- **L3.** `flagUnresolvedTrade(tradeId)` is callable by anyone. It MUST revert `TradeNotFlaggable` unless the trade is an unresolved trade and `block.timestamp > expiresAt + RESOLUTION_GRACE`. It sets the permanent `flagged` field, MUST NOT set `finalized` or `abandoned`, increments `flaggedTradeCount(subject)`, frees the open slot (which releases R5 for the subject) and emits `TradeFlaggedUnresolved`. `abandonTrade` and `finalizeTrade` MUST revert on a flagged trade.
- **L4.** `expireTrade` is unchanged (shipped) and OPTIONAL to call. Lapse detection MUST NOT depend on it. An expired trade with recorded legs stays an unresolved trade until abandoned or flagged.
- **L5.** `finalized` is true only after a successful `finalizeTrade`; `expired`, `abandoned` and `flagged` never imply it (shipped for `expired`). Consumers MUST read `finalized`.
- **L6.** `isLegAuthorized(tradeId, index)` MUST return true only if the trade is registered and not closed, `block.timestamp <= expiresAt`, `index < subTradeCount` and is unrecorded, and the subject holds no unresolved trade. Venues SHOULD check it before executing a leg. The registry cannot see a leg a venue executes without asking.

#### Plan proof (OPTIONAL)

- **P1.** `registerTradeWithPlan(tradeRef, jurisdictionId, subTradeCount, planProofHash, planPublicInputs)` applies R1 to R5 and then MUST check the plan proof: PATTERN type, same subject, not revoked, unused (then marked used), public inputs hash, `analysis_type == 1`, `reporting_threshold == Settlement.reportingThreshold`, `settlement_root == computePlanRoot(tradeId, jurisdictionId, subTradeCount)`, and `timestamp >= block.timestamp - PLAN_PROOF_MAX_AGE` (`PlanProofStale`). `computePlanRoot` is `bytes32(uint256(keccak256(abi.encode(address(this), tradeId, jurisdictionId, subTradeCount))) % BN254_FR_MODULUS)`.
- **P2.** It stores the plan's `tx_set_hash` as `planTxSetHash` (C5 then applies under v1) and emits it in `TradeRegistered.planProofHash`.
- **P3.** Cost: one extra PATTERN submission, 2,836,418 gas with the real verifier (`GasBenchmarkTest:test_gas_submitCompliance_PATTERN`), plus about 60.6k registry gas in the lifecycle prototype. It stops a wallet whose own analysis of the planned split reports structuring from paying the first leg (`test_attack_legsSettleBeforeAnalysis`), and under v1 it stops the analysed set being chosen after the legs paid (`test_attack_analysedSetChosenAfterLegs`). It does not stop a prover who lies about amounts, since both the plan set and the closing set are prover-chosen under v1, and it does nothing about splitting across wallets.

#### Leg chain (v2, REQUIRED target)

- **V0.** The leg chain is the REQUIRED target and Not yet implemented ([ERC-8262 #40](https://github.com/xochi-fi/ERC-8262/issues/40); prototype `research/circuit-binding`). Until it ships, `MAX_SUB_TRADES` MUST equal the deployed circuit's `MAX_TRANSACTIONS` (16).
- **V1.** Link function: `settlement_link(prev, amount, timestamp, salt) = poseidon2_permutation([prev, amount * 2^40 + timestamp, salt, DOMAIN_SETTLEMENT_LINK])[0]`, one Poseidon2 permutation with `t = 4`. `amount` is `u128`, `timestamp < 2^40`, and `DOMAIN_SETTLEMENT_LINK = 0x584950312e6c696e6b * 2^64 + 3` ("XIP1.link" with the absorbed length).
- **V2.** The `compliance`, `compliance_signed` and `compliance_multi_signed` circuits MUST take private `leg_amount: u128` and `leg_salt: Field`, MUST append public `settlement_link_in` and `settlement_link_out` as their last two words, and MUST constrain `settlement_link_out == settlement_link(settlement_link_in, leg_amount, timestamp, leg_salt)` with `timestamp` the proof's own timestamp. Signed payloads are unchanged. Public input counts become 8, 11 and 16 (`ProofTypes.expectedPublicInputCount` and the Oracle validators change with them).
- **V3.** At registration the registry MUST set the trade's root to `seed = bytes32(uint256(keccak256(abi.encode(SETTLEMENT_SEED_DOMAIN, block.chainid, address(this), tradeId, subject, jurisdictionId, subTradeCount))) % BN254_FR_MODULUS)`, with `SETTLEMENT_SEED_DOMAIN = keccak256("ERC8262.XIP1.settlementSeed")`, exposed as `settlementSeed(tradeId)`.
- **V4.** `recordSubSettlement(tradeId, index, proofHash, legPublicInputs)` MUST require `keccak256(legPublicInputs) == attestation.publicInputsHash` (`LegPublicInputsMismatch`) and `settlement_link_in == root` (`SettlementLinkMismatch`), then set `root = settlement_link_out`. Legs are therefore recorded in chain order. `computeSettlementRoot` returns the stored root.
- **V5.** The `pattern` circuit MUST use `MAX_TRANSACTIONS = 100`, private `amounts: [u128; 100]`, `timestamps: [u64; 100]`, `salts: [Field; 100]` and `num_transactions`, and public words 0 to 7 as in v1 (with `reporting_threshold: u128`) plus `[8] settlement_seed` and `[9] aggregate_reportable` (10 words). It MUST constrain: folding the first `num_transactions` legs from `settlement_seed` equals `settlement_root`; `tx_set_hash == settlement_root`; `aggregate_reportable == (sum of amounts >= reporting_threshold)` with checked `u128` arithmetic; every active timestamp lies in `[timestamp - time_window, timestamp]`; inactive slots are zero; the shipped analyses. The threshold is a public input checked by the registry (C4) against the Oracle table; the circuit MUST NOT compile in a threshold table.
- **V6.** `finalizeTrade` and `abandonTrade` MUST require `settlement_seed == settlementSeed(tradeId)` (`SettlementSeedMismatch`) and C1 against the stored root, MUST record `aggregate_reportable` (view `isAggregateReportable`) and MUST emit it in `TradeFinalized` or `TradeAbandoned`.
- **V7.** Committed amounts MUST be 18-decimal base units of the jurisdiction's reporting currency and MUST NOT be rescaled or truncated.
- **V8.** A leg's committed `leg_amount` MUST equal the value the leg transfers, converted to the reporting currency. Nothing in this XIP enforces this: it needs a settlement-layer check (the venue compares the commitment with the transfer, or the leg circuit is the transfer circuit) and a trusted price. Venues SHOULD perform it.

The prototype's measured costs: the 100-slot pattern circuit is 25,045 gates (about 0.47 s to prove), PATTERN verification 2,436,504 gas, COMPLIANCE verification 2,432,134 gas, and the two extra public inputs add about 4.3k gas per leg submission. The prototype compiled the threshold table into the circuit and carried a `jurisdiction_id` public input; this XIP takes the threshold from the Oracle instead, so that input is dropped and the seed binds the jurisdiction.

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

Target flow (Not yet implemented): step 2 becomes `tradeId = registerTrade(tradeRef, jurisdictionId, N)` (or `registerTradeWithPlan`); in step 5 the closing proof follows C1 to C5 (`time_window` reaching back to `createdAt`, `reporting_threshold` equal to the copied value); a trade that cannot be finalized is closed with `abandonTrade`. Under v2 each leg is proved on the previous leg's `settlement_link_out` and recorded in chain order with its public inputs.

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

Not yet implemented in the SDK: the target `registerTrade` and `registerTradeWithPlan` ABI, `abandonTrade`, `flagUnresolvedTrade`, `isLegAuthorized`, a `planSplit` cap of `MAX_SUB_TRADES`, and (v2) `leg_amount`, `leg_salt` and chain-ordered leg proving.

### Trust boundaries

| Requirement                                                                                        | Enforced by                                        | Status                                         |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------- |
| Per-asset `MIN_SPLIT_THRESHOLD`; no split for an asset without a row                               | SDK (`planSplit`)                                  | Not yet implemented (one value for all assets) |
| `maxSubTrades` in `[2, MAX_SUB_TRADES]`                                                            | SDK (`planSplit`)                                  | Not yet implemented (SDK allows 100)           |
| `minSubTradeSize > 0`, deterministic split, CSPRNG nonce                                           | SDK (`planSplit`)                                  | Shipped                                        |
| Sequential proving                                                                                 | SDK (`proveBatch`)                                 | Shipped                                        |
| N = 1 skips the registry                                                                           | Client; registry reverts `InvalidSubTradeCount`    | Shipped                                        |
| Leg proofs are compliance variants; leg subject, jurisdiction, liveness, revocation, not predating | Registry                                           | Shipped                                        |
| Closing proof type, subject, revocation, inputs hash, STRUCTURING, `settlement_root`, single use   | Registry                                           | Shipped                                        |
| `expired` never reads `finalized`                                                                  | Registry                                           | Shipped                                        |
| `submitter == msg.sender`; proof freshness (1 h) and timestamp ratchet                             | Oracle                                             | Shipped                                        |
| `reporting_threshold` is a registered value                                                        | Oracle                                             | Shipped (any jurisdiction's value)             |
| PATTERN result matches the analyzed set                                                            | Circuit                                            | Shipped                                        |
| Threshold table per jurisdiction (O1-O3)                                                           | Oracle, `REGISTRAR_ROLE` through Timelock          | Not yet implemented                            |
| `tradeId` keyed by subject (R1); leg cap (R2); threshold copied (R3)                               | Registry                                           | Not yet implemented                            |
| Open-trade limit (R4); lazy lapse block (R5)                                                       | Registry                                           | Not yet implemented                            |
| Leg attestation single-use across trades (G1)                                                      | Registry                                           | Not yet implemented (shipped: per trade)       |
| Legs frozen while an unresolved trade exists (G3, SHOULD)                                          | Registry                                           | Not yet implemented                            |
| Closing proof after the last leg (C2), window to `createdAt` (C3), threshold (C4)                  | Registry; Oracle no-future and 90-day window rules | Not yet implemented                            |
| `abandonTrade` (L2), `flagUnresolvedTrade` (L3), `abandoned`/`flagged` never read `finalized` (L5) | Registry                                           | Not yet implemented                            |
| Venue checks `isLegAuthorized` (L6, SHOULD)                                                        | Venue                                              | Not yet implemented                            |
| Plan proof checks (P1, OPTIONAL)                                                                   | Registry                                           | Not yet implemented                            |
| Analyzed set is exactly the committed legs (V2-V5)                                                 | Circuits and registry                              | Not yet implemented (v2)                       |
| `aggregate_reportable` computed (V5) and recorded (V6)                                             | Circuit; registry                                  | Not yet implemented (v2)                       |
| One split in flight per `(subject, jurisdictionId)` (SHOULD)                                       | Nothing; late legs revert in the Oracle            | -                                              |
| Committed amount equals the transferred amount (V8)                                                | Nothing in this XIP; venue SHOULD                  | -                                              |
| Conversion of a token amount into the reporting currency                                           | Nothing; needs a trusted price                     | -                                              |

Caller- and prover-supplied inputs under the target design, and what binds them:

- `tradeRef`: caller. Combined with `msg.sender` into `tradeId` (R1).
- `jurisdictionId`: caller. It selects the copied threshold (R3), but nothing binds it to the subject's real jurisdiction.
- `subTradeCount`: caller. Bounded by R2. Under v1 nothing binds it to the circuit's private `num_transactions`; under v2 the chain does.
- Leg `proofHash`: caller. Bound by the Oracle lookup and single use (G1). Under v2, `legPublicInputs` is bound by hash to the attestation and its link to the stored root (V4).
- Closing `reporting_threshold`: prover. Bound to the copied value (C4).
- Closing `time_window` and `timestamp`: prover. Bound by the Oracle's freshness and `[1 hours, 90 days]` window range, and by C2 and C3.
- Closing `settlement_root`: prover. Checked against the registry's value (C1).
- v1 `amounts`, `timestamps`, `num_transactions`: prover, private and unsigned. Unbound.
- v2 `leg_amount` and `leg_salt`: chosen by the subject per leg and committed in that leg's compliance proof before it is recorded. Leg timestamps are the leg proofs' own timestamps. Nothing binds `leg_amount` to the transferred value (V8).

## Rationale

### Why a separate SettlementRegistry, not an Oracle extension?

The Oracle answers "does this address meet the compliance threshold for this jurisdiction?". Splitting is orchestration: how a large trade is broken into proven pieces. The two have different lifecycles, audit scopes and upgrade cadences.

Extending the Oracle with `tradeId` would:

- Widen the Oracle's attack surface.
- Break the `ComplianceAttestation` struct ABI for existing consumers.
- Couple settlement policy (leg counts, trade TTL) to compliance policy (jurisdiction thresholds, attestation TTL).

### Why no randomized amounts?

The design originally proposed noise on leg amounts. It was rejected for three reasons:

1. **No ZK benefit.** Noise on transferred amounts gives no proof-level privacy: the shipped compliance proofs take no amount, and v2 commits each leg's real amount inside the proof, where it is already hidden.
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

### Why a confess path, not a permanent block?

Closing a trade with paid legs needs a proof whose window reaches back to `createdAt`, and the Oracle caps windows at 90 days. A rule that blocked the subject until the trade was resolved would become a permanent ban about 90 days after registration, for a subject who might only have suffered an outage. The confess path keeps resolution open during `RESOLUTION_GRACE` (`abandonTrade` with a bound STRUCTURING proof); after it, anyone can flag the trade. The flag is permanent and public, which is the record the anti-structuring goal needs, and it releases the subject's registration block so the penalty is a public record, not lost access.

### Why the threshold table lives in the Oracle

The registry is immutable and ownerless, so a table inside it could change only by redeploying. The Oracle already governs verifiers, configs and the registered-threshold set under roles behind the Timelock, and the registry already depends on that governance, so a per-jurisdiction mapping there adds no new trust. Copying the value at registration keeps a trade's rules fixed for its life. A threshold compiled into the circuit, as in the leg chain prototype, would need a verifier upgrade for every change.

### Why a per-trade hash chain (v2)

Each leg's compliance proof extends a chain seeded by the registry, and the pattern circuit refolds it, so the analysis covers exactly the recorded legs with their committed amounts and timestamps: no omitted legs, no fake legs, no rescaled or truncated amounts, no relabelled timestamps. Poseidon2 costs one permutation per link; the prototype's Pedersen control was about 4.6 times larger (114,873 against 25,045 gates) and keccak costs about 20k gates per permutation. Folding the chain on-chain instead would allow parallel leg proving at an estimated 15k to 20k gas per leg; it was not built.

### Why the plan proof is not mandatory

It costs about 2.8M gas per trade and, under v1, analyses a prover-chosen set, so it adds nothing against a subject who lies. It only stops an honest wallet from paying the first leg of a split its own analysis reports as structured. Making it REQUIRED would raise the cost of every split for that case alone.

### Relationship to stealth address splitting

The [Xochi whitepaper](https://xochi.fi/whitepaper) (Appendix, "Settlement Splitting") describes multi-account stealth address splitting: N stealth keys derived from one meta-address, with the amount spread across N ERC-5564 accounts. That works at the transfer layer. This XIP works at the compliance-proof layer. The two compose.

## Backwards Compatibility

- The Oracle (`ERC8262Oracle`), Verifier (`ERC8262Verifier`) and `ComplianceAttestation` struct are not modified.
- `SettlementRegistry` is a new standalone contract; `planSplit`, `proveBatch`, `provePlan` and `SettlementRegistryClient` are new SDK exports.
- Consumers that do not split are unaffected.

The registry ABI in this version differs from the text merged in PR #1 and PR #4 (`uint8` counts and indexes, three-argument `finalizeTrade`, the `expired` flag, new errors, `IERC8262Oracle`). No implementation of the earlier text exists; the shipped contract has always matched this version's ABI.

The target design breaks the shipped ABI: `registerTrade` takes `tradeRef` and returns `tradeId`, `Settlement` and `TradeRegistered` gain fields, `PatternProofPredatesTrade` is replaced, and the Oracle gains the threshold table. Under v2, `recordSubSettlement` takes `legPublicInputs`, `TradeFinalized` gains `aggregateReportable`, and the compliance and PATTERN public input counts change (8, 11, 16 and 10), which needs new verifiers through the Oracle's timelocked upgrade path. The registry is immutable, so the target ships as a new deployment; trades on the shipped registry stay readable there.

## Test Cases

### Split planning

Amounts in wei. Defaults unless stated: `splitThreshold = 100000000000000000000`, `maxSubTrades = 10`, `minSubTradeSize = 1000000000000000000`. Every output sums to `totalAmount`.

| #   | `totalAmount`            | Config override                                                                  | Legs | Output (wei)                                                                                             |
| --- | ------------------------ | -------------------------------------------------------------------------------- | ---- | -------------------------------------------------------------------------------------------------------- |
| 1   | `500000000000000000000`  | none                                                                             | 5    | 5 x `100000000000000000000`                                                                              |
| 2   | `350000000000000000000`  | none                                                                             | 4    | 4 x `87500000000000000000`                                                                               |
| 3   | `350000000000000000001`  | none                                                                             | 4    | 3 x `87500000000000000000`, then `87500000000000000001`                                                  |
| 4   | `100000000000000000000`  | none                                                                             | 1    | `100000000000000000000` (equal to threshold, no split)                                                   |
| 5   | `100000000000000000001`  | none                                                                             | 2    | `50000000000000000000`, `50000000000000000001`                                                           |
| 6   | `50000000000000000000`   | none                                                                             | 1    | `50000000000000000000`                                                                                   |
| 7   | `1000000000000000000000` | `maxSubTrades = 5`                                                               | 5    | 5 x `200000000000000000000` (clamped)                                                                    |
| 8   | `25000000000000000001`   | `splitThreshold = 10000000000000000000`, `minSubTradeSize = 9000000000000000000` | 2    | `12500000000000000000`, `12500000000000000001`                                                           |
| 9   | `11000000000000000000`   | `splitThreshold = 10000000000000000000`, `minSubTradeSize = 6000000000000000000` | 1    | `11000000000000000000` (reduced to N = 1)                                                                |
| 10  | `1000000000000000000000` | `splitThreshold = 10000000000000000000`, `maxSubTrades = 100`                    | 100  | 100 x `10000000000000000000` (shipped; rejected under the target until v2, see Leg cap and amount units) |
| 11  | `10000000000000000000`   | `splitThreshold = 3000000000000000000`                                           | -    | Throws: below `MIN_SPLIT_THRESHOLD`                                                                      |
| 12  | `500000000000000000000`  | `maxSubTrades = 101`                                                             | -    | Throws: `maxSubTrades` outside `[2, 100]`                                                                |

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

### Anti-structuring harness (target)

Prototype harness `test/research/StructuringAttacks.t.sol` (ERC-8262 `research/anti-structuring` `d666a71`, extended on `research/registry-hardening` and `research/lifecycle`). A `test_attack_*` passes only when the attack is refused; every `test_honest_*` MUST keep working. Jurisdiction 0's threshold is 10000 (v1 whole units); 999999 is another jurisdiction's registered threshold.

| Test                                         | Attack                                                     | Refused by                   | Expected                                                                 |
| -------------------------------------------- | ---------------------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------ |
| `test_attack_crossTradeLegReuse`             | One leg attestation backs legs of two trades               | G1                           | `DuplicateSubSettlementProof(firstTrade, proofHash)`                     |
| `test_attack_tradeIdFrontrun`                | Bob registers Alice's trade first                          | R1                           | Bob's call creates `keccak256(abi.encode(bob, ref))`; Alice registers    |
| `test_attack_foreignThreshold`               | Closing proof at another jurisdiction's higher threshold   | C4                           | `ReportingThresholdMismatch(10000, 999999)`                              |
| `test_attack_patternPredatesLastLeg`         | Closing proof dated between leg 0 and leg 1                | C2                           | `PatternProofPredatesLastLeg`                                            |
| `test_attack_windowDoesNotCoverLegs`         | Legs over three days, one-hour window                      | C3                           | `PatternWindowExcludesTrade`                                             |
| `test_attack_moreLegsThanCircuitCapacity`    | 17 legs finalized by one proof                             | R2                           | `InvalidSubTradeCount(17)`                                               |
| `test_attack_abandonAndRepeat`               | Fully paid, never finalized, expired, then a new split     | R5                           | `UnresolvedTrade`                                                        |
| `test_attack_abandonWithoutExpireCall`       | Same, nobody calls `expireTrade`                           | R5 (lazy)                    | `UnresolvedTrade`                                                        |
| `test_attack_abandonPartialAndRepeat`        | Part-paid splits abandoned ten times                       | R5                           | `UnresolvedTrade` on the second round                                    |
| `test_attack_abandonWithoutBoundProof`       | Abandon with no proof, then with a proof bound to nothing  | L2, C1, R5                   | `PatternProofRequired`, `SettlementRootMismatch`, then `UnresolvedTrade` |
| `test_attack_legsOnLiveTradeWhileUnresolved` | Legs keep settling on a live trade while one is unresolved | G3 (SHOULD)                  | `UnresolvedTrade`                                                        |
| `test_attack_expiredReadsFinalized`          | Expired trade reads finalized                              | L5 (shipped)                 | `finalized == false`                                                     |
| `test_attack_patternReuse`                   | One closing proof for two trades                           | Shipped 3.4                  | `PatternProofAlreadyUsed`                                                |
| `test_attack_wrongAnalysisType`              | VELOCITY proof stands in for STRUCTURING                   | Shipped 3.10                 | `PatternAnalysisTypeMismatch(1, 2)`                                      |
| `test_attack_unboundSettlementRoot`          | Proof not bound to this trade's legs                       | Shipped 3.11                 | `SettlementRootMismatch`                                                 |
| `test_attack_legsSettleBeforeAnalysis`       | Structured plan, first leg paid without any analysis       | P1 only (OPTIONAL)           | Refused only with `registerTradeWithPlan`                                |
| `test_attack_analysedSetChosenAfterLegs`     | Closing set swapped after the legs paid                    | C5 (plan proof, v1); V5 (v2) | `PatternTxSetMismatch`; under v2 the proof cannot be made                |

Honest flows that MUST pass: `test_honest_twoLegs`, `test_honest_sixteenLegs`, `test_honest_legsOverTwoDays`, `test_honest_secondTradeAfterFinalized`, `test_honest_seventeenLegsAsTwoTrades`, `test_honest_recoverAfterFailedLeg`, `test_honest_resolveLapsedTrade`, `test_honest_lapsedEmptyTradeDoesNotBlock`, `test_honest_inFlightTradeResumesAfterResolve`, `test_honest_concurrentTrades`.

Not covered by the prototypes, and needed under #38 and #39: the Oracle threshold table (O1-O3), the copy at registration (R3), and `flagUnresolvedTrade` (refused at `expiresAt + RESOLUTION_GRACE`, accepted one second later, permanent flag, block released, `abandonTrade` refused afterwards).

Leg chain (v2) circuit tests, `circuits/pattern/src/research.nr` on `research/circuit-binding`. Each attack MUST fail to prove: `test_attack_result_lie`, `test_attack_unrelated_amounts`, `test_attack_unit_rescale`, `test_attack_u64_truncation`, `test_attack_more_legs_than_capacity`, `test_attack_leg_omission`, `test_attack_fake_dilution`, `test_attack_real_dilution`, `test_attack_relabel_into_window`, `test_attack_fractional_truncation`. `test_attack_foreign_threshold` is refused by the prototype's compiled table; under this XIP it is refused by C4. Each honest case MUST prove: `test_honest_two_legs_clean`, `test_honest_sixteen_legs_clean`, `test_honest_structuring_reported`, `test_honest_large_split`, `test_honest_legs_over_two_days`, `test_honest_fractional_amounts`, `test_honest_hundred_legs_clean`.

## Reference Implementation

Pinned to the commits this version was checked against:

- ERC-8262 [`f018627`](https://github.com/xochi-fi/ERC-8262/tree/f018627ce498839308a0403ca06bcbbdf1140127): [`src/SettlementRegistry.sol`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/src/SettlementRegistry.sol), [`src/interfaces/ISettlementRegistry.sol`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/src/interfaces/ISettlementRegistry.sol), [`test/SettlementRegistry.t.sol`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/test/SettlementRegistry.t.sol), [`circuits/pattern/src/main.nr`](https://github.com/xochi-fi/ERC-8262/blob/f018627ce498839308a0403ca06bcbbdf1140127/circuits/pattern/src/main.nr).
- xochi-sdk [`068c301`](https://github.com/xochi-fi/xochi-sdk/tree/068c301eb4bc3bd783858e5e172f102389357f7c): [`packages/sdk/src/split.ts`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/src/split.ts), [`packages/sdk/src/batch-prover.ts`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/src/batch-prover.ts), [`packages/sdk/src/settlement-registry.ts`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/src/settlement-registry.ts), [`packages/sdk/docs/onchain.md`](https://github.com/xochi-fi/xochi-sdk/blob/068c301eb4bc3bd783858e5e172f102389357f7c/packages/sdk/docs/onchain.md#settlement-splitting-xip-1).

Anti-structuring prototypes (research code, not production; each passes its own forge suite). Tracked for production in [ERC-8262 #37](https://github.com/xochi-fi/ERC-8262/issues/37):

- [`research/anti-structuring` `d666a71`](https://github.com/xochi-fi/ERC-8262/tree/d666a719d04454486e41d555563a70132a24a5dd): the shared attack harness.
- [`research/registry-hardening` `6fa31ca`](https://github.com/xochi-fi/ERC-8262/tree/6fa31ca6b730572d1cd8ae30fc5b2e18d57656bc): R1, R2, G1, C2, C3, a constructor threshold table, and a part-paid release variant ([#38](https://github.com/xochi-fi/ERC-8262/issues/38)).
- [`research/lifecycle` `62b75af`](https://github.com/xochi-fi/ERC-8262/tree/62b75af8b4c329d9387b3946a3e7879288faa0af): open-trade table, lazy lapse, `abandonTrade`, `isLegAuthorized`, G3, and the plan proof ([#39](https://github.com/xochi-fi/ERC-8262/issues/39)).
- [`research/circuit-binding` `c25d150`](https://github.com/xochi-fi/ERC-8262/tree/c25d1503e41ed4ce21fdb94df4c15a772e0b4794): the leg chain circuits and registry ([#40](https://github.com/xochi-fi/ERC-8262/issues/40)).

The prototypes differ from this XIP where it chose differently: thresholds come from the Oracle table (O1), not a constructor or compiled table; lapsed trades are flagged (L3), not permanently blocking or released by count; the plan proof is OPTIONAL (P1).

## Security Considerations

### Front-running leg submissions

Each leg proof carries `submitter` as a public input and the Oracle enforces `submitter == msg.sender`. A copied leg proof cannot be submitted from another address (`SubmitterMismatch`; ERC-8262 `test/ValidatorChecks.t.sol` `test_compliance_revert_submitterMismatch`).

### Trade registration griefing (shipped known issue)

The shipped `registerTrade` makes the first caller the subject and never checks the `tradeId` preimage. An attacker who sees a pending `registerTrade` can register the same `tradeId` first. The victim gets `TradeAlreadyExists`, and every retry with a new `tradeId` is exposed the same way. The attacker pays one registration per grief and gains nothing else; the victim loses time, not funds.

R1 closes this by keying trades by `(msg.sender, tradeRef)` (Not yet implemented). Until then, clients MAY submit `registerTrade` through a private transaction channel.

### MEV from trade registration (known issue)

`TradeRegistered` publishes `subject` and `subTradeCount` before the first leg settles. Splitting is deterministic, so once one leg amount is visible on a public venue the remaining flow is about `(N - 1)` times that amount, from a known address, within the trade's lifetime. Searchers can position against the remaining legs. This works against Motivation 3.

Mitigation options: submit all legs in one `submitComplianceBatch` and settle them in as few blocks as possible; settle on a shielded venue; register through a private transaction channel. Emitting only a commitment, or registering after the legs, needs a contract change (not implemented; registering after the legs also conflicts with `AttestationPredatesTrade`).

### One attestation backing legs of several trades (shipped known issue)

Shipped duplicate-leg tracking is per trade (`_usedLegProofs[tradeId]`), so one compliance attestation can back a leg in each of several trades with the same subject and jurisdiction. G1 closes this (Not yet implemented).

### Partial settlement abandonment

Shipped: a trade with M < N recorded legs stays open until `expiresAt` (7 days); anyone can then call `expireTrade`, which sets `expired`, and nothing stops the subject opening the next split. Under the target, a lapsed trade with a recorded leg blocks the subject's registrations (R5), and with G3 its live legs, until the subject abandons it with a STRUCTURING proof bound to the recorded legs (L2) or anyone flags it after `RESOLUTION_GRACE` (L3). Either outcome is public, and neither sets `finalized`.

### Oracle attestation overwrite

Each leg overwrites the subject's typed slot `(subject, jurisdictionId, proofType)` and, for slot-writing types, the compliance slot `(subject, jurisdictionId)`. Earlier legs remain retrievable through `getHistoricalProof(proofHash)`, and the registry stores every leg's `proofHash`. `checkCompliance` returning the latest leg is correct: the subject is still compliant.

### Leg revocation after recording

`finalizeTrade` does not re-check recorded legs. A leg revoked after it was recorded still counts toward `settledCount`. Consumers that depend on every leg being live SHOULD check `ERC8262Oracle.isAttestationValid(proofHash)` for each recorded leg.

### Gas

The Oracle's own benchmark puts 10 batched compliance proofs at about 24.1M gas (`ERC8262Oracle.sol` `MAX_BATCH_SIZE` note, `test_gas_batch_atMaxSize_fitsBlockGasTarget`), about 2.4M per proof. A 100-leg trade therefore costs about 240M gas in leg submissions, at least 10 transactions, plus 100 `recordSubSettlement` calls, the PATTERN proof and `finalizeTrade`. The cost falls on the subject, not on other users.

Target overhead, from the prototypes (registry calls only, excluding Oracle submissions; shipped baseline 527,508 gas for 2 legs and 2,757,190 for 16): the registry rules added 34,466 and 37,016 gas; the lifecycle prototype, including the plan proof check and G3, totalled 667,147 and 3,034,431. A plan proof adds a 2,836,418-gas PATTERN submission. v2 adds about 4.3k gas per leg submission.

### Structuring risk

Under the target, a registered split closes only with a STRUCTURING proof bound to its recorded legs, dated after the last leg, covering the whole trade and using the jurisdiction's threshold, and a split that lapses with paid legs blocks its subject until it is confessed or publicly flagged. These residual risks remain:

1. **Analysis honesty before v2.** Until the leg chain ships, the closing proof's amounts are private, unsigned prover inputs. The v1 rules fix when and against what a proof is presented, not what it analyzed; a subject who lies passes all of them.
2. **Committed versus paid amount.** v2 binds the analysis to the amount each leg's compliance proof committed, not to the value that moved. Closing that needs the settlement-layer check in V8, which nothing in this XIP performs.
3. **Fiat conversion.** Thresholds are in the reporting currency. Converting a token amount needs a trusted price at leg time; no circuit or contract here provides one, so the subject chooses the conversion.
4. **Weak heuristic.** The STRUCTURING result (at least half of the amounts in `[90%, 100%)` of the threshold) is diluted by adding small real legs (`test_attack_real_dilution`), so it cannot be relied on alone. What matters is `aggregate_reportable` (v2): whether the legs together reach the threshold. Consumers SHOULD act on `isAggregateReportable`, not on the heuristic result.
5. **Several wallets or subjects.** Every rule is per address. Splitting across addresses, or across subjects acting together, is invisible on-chain; it needs identity-level aggregation off-chain.
6. **Unsplit and unregistered trades.** Trades at or below `splitThreshold`, and legs a venue executes without the registry, are not checked. `isLegAuthorized` helps only at venues that call it.
7. **Separate trades.** Each trade is analyzed alone, so several trades each below the threshold pass. That needs history across a subject's trades.
8. **Jurisdiction choice.** The subject picks `jurisdictionId`, so the most lenient supported threshold applies. That needs a jurisdiction credential bound to the subject.
9. **Confession and flag rate.** A subject can close part-paid trades with `abandonTrade`, or let up to `MAX_OPEN_TRADES` trades lapse and be flagged every 37 days. Each outcome is public (`TradeAbandoned`, `TradeFlaggedUnresolved`, `flaggedTradeCount`), not prevented.
10. **Analysis after payment.** The closing analysis runs after the legs pay unless the OPTIONAL plan proof is used, and under v1 that proof analyzes a prover-chosen set.

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

Under the target, R4 and R5 add per-subject contention in the registry: one subject's unresolved trade blocks that subject's registrations and, with G3, legs; other subjects are unaffected.

### Oracle threshold change mid-trade

The threshold is copied at registration (R3), so a change through `setJurisdictionReportingThreshold` affects only later trades. If governance removes the copied value from the Oracle's registered-threshold set while a trade is open, every PATTERN proof at that value fails Oracle validation (`InvalidReportingThreshold`), so the trade can be neither finalized nor abandoned with legs, and it ends flagged (L3). Governance SHOULD NOT revoke a threshold that open trades still carry.

### Lapse with paid legs

If the subject misses `expiresAt` (outage, lost key), the trade blocks the subject's registrations (R5). The subject resolves it with `abandonTrade` and a STRUCTURING proof bound to the recorded legs, any time before `expiresAt + RESOLUTION_GRACE`. After that, anyone flags it; the flag is permanent and the block is released. A lost key leaves the trade to be flagged.

### Sequential leg proving (v2)

Each v2 leg proof extends the previous leg's link, so legs are proved and recorded in chain order, each submitted within `MAX_PROOF_AGE` of its own timestamp. A client can compute the chain ahead from its own amounts, salts and timestamps, but a leg that fails and is re-proved gets a new timestamp, hence a new link, so every later leg proved on the old link MUST be re-proved. Legs cannot be executed in parallel at different venues beyond what can be proved and recorded in order within the proof-age limit.

## History

- 2026-05-13: Merged as Draft (`XIPS/xip-draft_settlement-splitting.md`), PR #1.
- 2026-05-13: Reverted, PR #3 (mistaken merge).
- 2026-11-12: Re-merged as `XIPS/xip-1.md`, Draft -> Review, PR #4. Text unchanged from PR #1, so it predated the ERC-8262 audit fixes H-1, H-2 and M-1 in the shipped code.
- 2026-11-12: Review, amended in place: resynced to shipped ERC-8262 `f018627` and xochi-sdk `068c301` after a three-reviewer audit; `discussions-to` moved to #6; `requires: 0` removed.
- 2026-11-12: Review, amended in place: added the anti-structuring target design as normative and Not yet implemented (Oracle threshold table, registry and lifecycle rules with a confess-and-flag path, OPTIONAL plan proof, REQUIRED v2 leg chain); tracked in ERC-8262 #37.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
