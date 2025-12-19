| Author(s) | Created | Status | References | Discussions |
|-------------|-----------|---------|------|----------|
| [Seri Choi](mailto:seri.choi@eigenlabs.org), [Matt Nelson](mailto:matt.nelson@eigenlabs.org) | 2025-12-12 | `Draft` | [Eigenlayer-contracts PR #1659](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1659), [Sidecar PR #474](https://github.com/Layr-Labs/sidecar/pull/474), [Sidecar PR #476](https://github.com/Layr-Labs/sidecar/pull/476), [Sidecar PR #481](https://github.com/Layr-Labs/sidecar/pull/481) | N/A |

# ELIP-014: Rewards v2.2 - Operator Set Rewards with Unique & Total Stake

---

# Executive Summary

This proposal introduces **Rewards v2.2**, a comprehensive extension to EigenLayer's Rewards v2 system that brings **automatic stake-weighted distribution** to Operator Sets. Building upon the foundation established in [ELIP-001](./ELIP-001.md), this upgrade enables AVSs using Operator Sets via the `AllocationManager` to create both retroactive and forward-looking reward distributions (up to 2 years) based on **unique stake** or **total stake** within Operator Sets.

Rewards v2.2 introduces two new reward mechanisms:

1. **Unique Stake Weighted Rewards** (`createOperatorSetUniqueStakeRewardsSubmission()`):
   - Distributes based on allocated unique stake within operator sets
   - First prospective reward mechanism for AllocationManager-based AVSs
   - Supports both retroactive and future commitments (up to 2 years)

2. **Total Stake Weighted Rewards** (`createOperatorSetTotalStakeRewardsSubmission()`):
   - Operator set analog to Rewards v1 (scoped instead of AVS-wide)
   - Distributes based on total delegated stake within operator sets
   - Supports both retroactive and future commitments (up to 2 years)

This enhancement maintains full backward compatibility with existing Rewards v2 functionality while enabling `AllocationManager`-based AVSs to offer predictable, multi-period incentive structures.

# Motivation

EigenLayer's current rewards architecture has a significant limitation: **AVSs using the `AllocationManager` (required for Operator Sets) cannot create prospective, auto-weighted rewards.**

While Rewards v1 provides forward-looking capabilities (up to 30 days) with automatic distribution, it only works with `AVSDirectory`-registered AVSs and distributes to the total stake weight of the AVS, ignoring factors like Operator performance or slashing risk. Rewards v2.1 supports Operator Sets but only retroactively, requiring AVSs to manually calculate per-operator amounts without stake-weight indexing.

This creates significant limitations for Operator Sets:

**Operator Investment Risk**: Without committed revenue streams, operators cannot justify infrastructure investments for long-term commitments, which can be particularly problematic for AVSs needing Unique Stake or with costly operations.

**Constrained Incentive Design**: AVSs cannot reward unique or total stake automatically, align with quarterly/annual cycles, or support insurance use cases longer duration commitments (v1's 30-day limit is insufficient).

**Competitive Disadvantage**: Platforms offering predictable, auto-distributed rewards attract higher-quality operators and capital, while EigenLayer's modern `AllocationManager` AVSs lack this capability.

This impacts all AVSs seeking sustainable operator relationships, especially those with longer business cycles (quarterly planning, insurance markets, seasonal demand).

## Design Goals

Rewards v2.2 is designed as an additive upgrade to Rewards v2 with the following properties:

1. **Enable Operator Set Rewards**: Bring forward-looking, auto-weighted distribution to AllocationManager-based AVSs, achieving feature parity with AVSDirectory while leveraging existing operator set infrastructure.

2. **Dual Stake Models**: Support both unique stake (allocated) and total stake (delegated) distribution methods for flexible incentive design.

3. **Automatic Distribution**: AVS specifies total amount; system calculates per-operator shares based on stake weight, eliminating manual calculations.

4. **Extended Time Horizons**: Support retroactive and prospective commitments up to 2 years (vs 30 days in v1) for insurance and long-term use cases.

5. **Backward Compatibility**: All existing v2/v2.1 functionality remains unchanged, enabling seamless integration without migration.

6. **Queue-Aware Rewards**: Account for withdrawal and deregistration queues when distributing rewards, ensuring fair compensation during slashable periods.

> 📚  **Note**
>
> If you need to familiarize yourself with concepts related to slashing like Operator Sets, Unique stake, and the `AllocationManager`, refer to [ELIP-002](./ELIP-002.md)

# Features & Specification

## High Level Design

Rewards v2.2 adds two new reward distribution mechanisms to the existing Rewards v2 system:

### Architecture Flow

The high level flow is as follows:

1. **AVS creates Operator Set rewards submission**:
   1. AVS gives an ERC20 token approval to the `RewardsCoordinator` for the total `amount` across all `rewardsSubmissions[]`.
   2. AVS calls either `createOperatorSetUniqueStakeRewardsSubmission()` or `createOperatorSetTotalStakeRewardsSubmission()` on the `RewardsCoordinator` with:
      - `operatorSet`: The operator set (AVS address + operator set ID)
      - `rewardsSubmissions[]`: Array containing `token`, `amount`, `duration`, `startTimestamp`, `strategiesAndMultipliers`, and `description`
   3. The `RewardsCoordinator` performs submission-time validation:
      - Validates timing constraints (`startTimestamp` can be retroactive or up to 2 years future, must be multiple of `CALCULATION_INTERVAL_SECONDS`)
      - Validates `duration` is non-zero and multiple of `CALCULATION_INTERVAL_SECONDS`
      - Validates operator set exists and belongs to calling AVS via `checkCanCall(operatorSet.avs)`
      - For **Unique Stake**: Validates operators have allocated unique stake to the operator set
      - For **Total Stake**: Validates operators are registered to the operator set
   4. Tokens are immediately transferred from AVS to `RewardsCoordinator` and escrowed (no cancellation possible).
   5. An `OperatorSetRewardsSubmissionCreated` event is emitted with the operator set, submission nonce, submission hashes, and full submission details.

2. **Off-chain infrastructure processes rewards**:
   1. The [Sidecar](https://github.com/Layr-Labs/sidecar) listens for `OperatorSetRewardsSubmissionCreated` events and stores them.
   2. The Sidecar generates reward roots on a **daily basis** (aligned to `CALCULATION_INTERVAL_SECONDS` = 1 day) taking into account:
      - Operator set rewards submissions (both unique and total stake variants)
      - Per-AVS operator splits (if set via `setOperatorAVSSplit()` in Rewards v2)
      - Operator registration status to the operator set during each daily snapshot
      - Stake snapshots:
        - **Unique Stake**: Queries `magnitude / max_magnitude` from `operator_allocation_snapshots` via AllocationManager
        - **Total Stake**: Queries total delegated shares from `operator_share_snapshots` via DelegationManager
      - Withdrawal queue adjustments (stakers earn during 14-day withdrawal period)
      - Deregistration queue handling (operators eligible during 14-day deregistration via `slashable_until`)
      - Slashing adjustments (cumulative multipliers applied to queued withdrawals)
   3. For each daily snapshot during the reward `duration`:
      - Calculate: `operator_share = (operator_stake / total_operator_set_stake) × (amount / duration)`
      - Distribute operator share according to per-AVS split (default 10%)
      - Distribute remaining share to stakers proportional to `strategiesAndMultipliers`
   4. The Root Updater retrieves the latest root from the sidecar and posts the root on a **weekly basis** by calling `submitRoot()` on the `RewardsCoordinator`.

3. **Operators and Stakers claim rewards**:
   1. After the root is posted and the 7-day activation delay passes, operators and stakers can claim.
   2. Claims are processed by calling `processClaims()` on the `RewardsCoordinator` contract with merkle proofs.
   3. Earnings are cumulative, so claimants can claim against the latest root without claiming against all prior roots.
   4. Claimed tokens are transferred to the specified recipient address.

**Key Behavioral Notes**:

- **Dynamic Weighting**: Stake weights are queried daily at execution time, not locked at submission. Individual operator shares fluctuate as relative stake changes; total daily rate (`amount/duration`) remains fixed.
- **Refunds**: If operators are not registered to the Operator Set during the reward period, their allocated amounts are refunded to the AVS as distribution leaves (claimable via standard process).
- **No Cancellation**: Once submitted, commitments are binding. Tokens are immediately escrowed and cannot be cancelled.
- **Integration with Rewards v2**: Operator set rewards integrate with existing Rewards v2 infrastructure, including per-AVS operator splits set via `setOperatorAVSSplit()`.

## Low Level Specification

### EigenLayer Protocol

The `RewardsCoordinator` contract is extended with two new functions for operator set rewards:

#### Operator Set Rewards Interface

```solidity
/**
 * @notice Create rewards for an operator set based on unique allocated stake weight
 * @dev First forward-looking rewards mechanism for AllocationManager-registered AVSs
 * @dev Essentially Rewards v1 but for allocated unique stake to operator sets
 * @dev AVS specifies total amount; system automatically calculates distribution
 * @dev Note: Unique stake is always slashable by definition and must be allocated to the operator set
 * @param operatorSet The operator set to create rewards for
 * @param rewardsSubmissions Array of rewards submissions (can be retroactive or forward-looking up to 2 years)
 */
function createOperatorSetUniqueStakeRewardsSubmission(
    OperatorSet calldata operatorSet,
    RewardsSubmission[] calldata rewardsSubmissions
) external onlyWhenNotPaused(PAUSED_OPERATOR_SET_UNIQUE_STAKE_REWARDS) checkCanCall(operatorSet.avs) nonReentrant;

/**
 * @notice Create rewards for an operator set based on total delegated stake weight
 * @dev Operator set analog to Rewards v1 (scoped to operator set instead of AVS-wide)
 * @dev AVS specifies total amount; system automatically calculates distribution
 * @param operatorSet The operator set to create rewards for
 * @param rewardsSubmissions Array of rewards submissions (can be retroactive or forward-looking up to 2 years)
 */
function createOperatorSetTotalStakeRewardsSubmission(
    OperatorSet calldata operatorSet,
    RewardsSubmission[] calldata rewardsSubmissions
) external onlyWhenNotPaused(PAUSED_OPERATOR_SET_TOTAL_STAKE_REWARDS) checkCanCall(operatorSet.avs) nonReentrant;

/**
 * @notice Emitted when operator set rewards submission is created
 * @param operatorSet The operator set receiving rewards
 * @param submissionNonce Current nonce for this AVS
 * @param rewardsSubmissionHashes Array of hashes for each submission
 * @param rewardsSubmissions Array of reward submission details
 */
event OperatorSetRewardsSubmissionCreated(
    OperatorSet indexed operatorSet,
    uint256 indexed submissionNonce,
    bytes32[] rewardsSubmissionHashes,
    RewardsSubmission[] rewardsSubmissions
);
```

Where `RewardsSubmission` struct is:

```solidity
struct RewardsSubmission {
    StrategyAndMultiplier[] strategiesAndMultipliers;
    IERC20 token;
    uint256 amount;
    uint32 startTimestamp;
    uint32 duration;
    string description;
}
```

#### Implementation Requirements

1. **Submission Time Validation**:
   - `startTimestamp` can be in the past (retroactive) or up to 2 years in the future
   - `startTimestamp` cannot exceed `MAX_FUTURE_LENGTH` (upgraded from 30 days to 365 days)
   - `duration` must be non-zero and reasonable
   - `amount` must be > 0

2. **Operator Set Validation**:
   - Operator set must exist and be registered
   - Operator set must belong to the calling AVS
   - For **Unique Stake**: Operators must have allocated unique stake to that specific operator set
   - For **Total Stake**: Operators must be registered to the operator set

3. **Access Control (UAM Compliant)**:
   - All new reward types are UAM-compliant
   - Validates caller has appropriate permissions to create rewards on behalf of the AVS via `checkCanCall(operatorSet.avs)`

4. **Token Handling**:
   - Tokens are immediately escrowed in RewardsCoordinator upon submission
   - Guarantees future distribution regardless of AVS solvency later
   - No cancellation mechanism - commitments are binding once created

5. **Execution Time Behavior**:
   - Stake weights are queried at execution time (daily), not locked at submission time
   - For **Unique Stake**: Uses `magnitude / max_magnitude` from AllocationManager to determine allocated stake
   - For **Total Stake**: Uses total delegated shares from DelegationManager
   - Individual operator shares fluctuate as relative stake weights change during the reward period
   - Total daily distribution rate (`amount/duration`) remains fixed

#### Key Behavioral Notes

1. **No Automatic Indexing for Allocation Changes**: Like v2.1, there is no functionality to automatically index changes in unique stake allocations. AVSs must create separate submissions for each interval between allocation changes if they want to accurately track stake weight fluctuations.

2. **Rate Fluctuation**: While there is visibility into the total amount of rewards that will be paid out per day (`amount/duration`), the actual rate that individual operators receive is a function of their relative stake weight and will fluctuate over time unless coupled with stake duration/capping solutions.

3. **Slashable Stake**: Unique stake is always slashable by definition. Operators must have allocated unique stake to the operator set to be eligible for unique stake rewards.

4. **14-Day Deregistration Queue**: For unique stake rewards, operators remain eligible during the 14-day deregistration queue period (tracked via `slashable_until` field), with rewards adjusted for any slashing events during this period.

### EigenLayer Sidecar

The sidecar has been extended to support Operator Set rewards distribution with the following capabilities:

- **Operator Set Rewards Distribution**: Six new tables (15-20) handle unique stake and total stake reward calculations at the operator set level, including operator distribution, staker distribution, and AVS refunds.

- **Snapshot Generation**: Captures `magnitude/max_magnitude` ratios for unique stake calculations and tracks operator registration periods with `slashable_until` field for 14-day deregistration queue handling.

- **Withdrawal Queue Integration**: Extends rewards to stakers during the 14-day withdrawal period, with cumulative slashing multipliers applied to queued withdrawals.

- **Rounding Management**: Implements allocation/deallocation rounding logic (allocations round UP to next day, deallocations round DOWN to current day) with backward compatibility for pre-Rewards v2.2 behavior.

The implementation merges all reward types (v1, v2, v2.1, v2.2 unique & total) in staging tables before generating final cumulative rewards for merkle tree distribution.

## Example Usage

### Example 1: Unique Stake Rewards (Forward-Looking)

```solidity
// AVS creates forward-looking rewards based on unique stake allocations
// Rewards will be distributed over 90 days starting from April 1st, 2025

OperatorSet memory operatorSet = OperatorSet({
    avs: address(myAVS),
    id: 1  // Operator set ID
});

// Define strategies and multipliers
StrategyAndMultiplier[] memory strategiesAndMultipliers = new StrategyAndMultiplier[](2);
strategiesAndMultipliers[0] = StrategyAndMultiplier({
    strategy: IStrategy(stETHStrategy),
    multiplier: 1e18  // 1x multiplier
});
strategiesAndMultipliers[1] = StrategyAndMultiplier({
    strategy: IStrategy(rETHStrategy),
    multiplier: 1e18
});

// Create rewards submission
RewardsSubmission[] memory submissions = new RewardsSubmission[](1);
submissions[0] = RewardsSubmission({
    strategiesAndMultipliers: strategiesAndMultipliers,
    token: IERC20(rewardToken),
    amount: 100000e18,  // Total pool: 100,000 tokens
    startTimestamp: 1743465600,  // April 1st, 2025
    duration: 90 days,
    description: "Q2 2025 Unique Stake Rewards"
});

// AVS creates the submission - tokens are immediately escrowed
// System will automatically calculate per-operator distribution based on unique stake weight
rewardsCoordinator.createOperatorSetUniqueStakeRewardsSubmission(
    operatorSet,
    submissions
);

// No execution step needed - sidecar automatically processes when startTimestamp is reached
// Daily distribution: 100,000 / 90 days = ~1,111 tokens per day
// Each operator receives: (operator_unique_stake / total_operator_set_stake) × daily_amount
```

### Example 2: Total Stake Rewards (Retroactive)

```solidity
// AVS creates retroactive rewards for past performance based on total delegated stake

OperatorSet memory operatorSet = OperatorSet({
    avs: address(myAVS),
    id: 2
});

StrategyAndMultiplier[] memory strategiesAndMultipliers = new StrategyAndMultiplier[](1);
strategiesAndMultipliers[0] = StrategyAndMultiplier({
    strategy: IStrategy(cbETHStrategy),
    multiplier: 1e18
});

// Retroactive submission for January 2025
RewardsSubmission[] memory submissions = new RewardsSubmission[](1);
submissions[0] = RewardsSubmission({
    strategiesAndMultipliers: strategiesAndMultipliers,
    token: IERC20(rewardToken),
    amount: 50000e18,
    startTimestamp: 1704067200,  // January 1st, 2025 (in the past)
    duration: 31 days,
    description: "January 2025 Total Stake Rewards"
});

// Create retroactive rewards
rewardsCoordinator.createOperatorSetTotalStakeRewardsSubmission(
    operatorSet,
    submissions
);

// Sidecar immediately processes retroactive submissions
// Distribution based on total delegated stake (not unique allocated stake)
```

### Example 3: Multiple Submissions (Insurance Use Case)

```solidity
// Insurance AVS creates 1-year forward commitment with quarterly breakdown

OperatorSet memory operatorSet = OperatorSet({
    avs: address(insuranceAVS),
    id: 1
});

StrategyAndMultiplier[] memory strategiesAndMultipliers = new StrategyAndMultiplier[](1);
strategiesAndMultipliers[0] = StrategyAndMultiplier({
    strategy: IStrategy(wETHStrategy),
    multiplier: 1e18
});

// Create 4 quarterly submissions in one transaction
RewardsSubmission[] memory submissions = new RewardsSubmission[](4);

// Q1 2025
submissions[0] = RewardsSubmission({
    strategiesAndMultipliers: strategiesAndMultipliers,
    token: IERC20(rewardToken),
    amount: 250000e18,
    startTimestamp: 1704067200,  // Jan 1, 2025
    duration: 90 days,
    description: "Insurance Rewards Q1 2025"
});

// Q2 2025
submissions[1] = RewardsSubmission({
    strategiesAndMultipliers: strategiesAndMultipliers,
    token: IERC20(rewardToken),
    amount: 250000e18,
    startTimestamp: 1711929600,  // Apr 1, 2025
    duration: 91 days,
    description: "Insurance Rewards Q2 2025"
});

// Q3 2025
submissions[2] = RewardsSubmission({
    strategiesAndMultipliers: strategiesAndMultipliers,
    token: IERC20(rewardToken),
    amount: 250000e18,
    startTimestamp: 1719792000,  // Jul 1, 2025
    duration: 92 days,
    description: "Insurance Rewards Q3 2025"
});

// Q4 2025
submissions[3] = RewardsSubmission({
    strategiesAndMultipliers: strategiesAndMultipliers,
    token: IERC20(rewardToken),
    amount: 250000e18,
    startTimestamp: 1727740800,  // Oct 1, 2025
    duration: 92 days,
    description: "Insurance Rewards Q4 2025"
});

// Create all quarterly submissions at once
// Total: 1,000,000 tokens escrowed for full year
rewardsCoordinator.createOperatorSetUniqueStakeRewardsSubmission(
    operatorSet,
    submissions
);
```

# Security Considerations

## Access Control & Integrity

**UAM Compliance**: All reward types validate caller permissions via `checkCanCall(operatorSet.avs)` to prevent unauthorized submissions.

**Operator Set Scoping**: Rewards are scoped to specific sets, preventing cross-contamination or unauthorized distribution to non-registered operators.

**Sybil Resistance**: Only registered operators with actual stake (unique or total) receive rewards based on stake weight, preventing zero-stake participants.

## Economic Security

**Token Escrow**: Tokens are immediately escrowed upon submission, guaranteeing distribution regardless of later AVS solvency. No cancellation - commitments are binding.

**Slashing Risk Management**:

- Unique stake is always slashable by definition
- Operators remain slashable during 14-day deregistration queue with proportional reward adjustments
- Stakers earn rewards during 14-day withdrawal queue (commensurate with risk)
- Cumulative slash multipliers applied to queued withdrawals

**Rate Manipulation Resistance**: While individual shares fluctuate with relative stake weight, total daily rate (`amount/duration`) is fixed at submission, preventing inflation attacks.

## System Integrity

**Timing Validation**: `startTimestamp` constraints (up to 2 years, aligned to calculation intervals) prevent excessive future commitments and ensure proper alignment.

**Withdrawal Queue Accounting**: Sophisticated tracking with slashing adjustments prevents double-counting or incorrect attribution during withdrawal periods.

**Fork Safety**: Sabine fork introduces allocation/deallocation rounding logic with backward compatibility for consistent behavior across upgrades.

# Impact Summary

## Pros

1. **Enables AllocationManager Rewards**: Brings forward-looking, auto-weighted distribution to operator sets, achieving feature parity with AVSDirectory-based AVSs.

2. **Dual Stake Model Flexibility**: Supports both unique (allocated) and total (delegated) stake distribution for diverse incentive models.

3. **Reduces AVS Burden**: Eliminates manual per-operator calculations. AVS specifies total; system distributes by stake weight.

4. **Extended Time Horizons**: Up to 2-year commitments (vs 30 days in v1) provide predictable revenue visibility for long-term planning and infrastructure investment.

5. **Full Backward Compatibility**: All existing v2/v2.1 functionality preserved with seamless integration.

6. **Incentivize Security**: Unique stake rewards directly encourage slashable stake allocation to AVS operator sets.

## Cons

1. **Token Lock Period**: Immediate escrow reduces AVS liquidity flexibility, especially for 2-year commitments (e.g., 1M tokens upfront).

2. **No Cancellation**: Binding commitments prevent cancellation if business conditions change.

3. **Rate Fluctuation**: Individual operator shares fluctuate with stake weight changes, complicating per-operator revenue prediction.

4. **No Automatic Indexing**: AVSs must create separate submissions for intervals between allocation changes to track stake fluctuations.

5. **Higher Gas Costs**: Multiple submissions (e.g., quarterly) cost more than single retroactive v2.1 submissions.

# Action Plan

The implementation of this ELIP will follow these key steps:

1. **Upgrade EigenLayer Protocol**: Upgrade the `RewardsCoordinator` Transparent Proxy to implement `createOperatorSetUniqueStakeRewardsSubmission()` and `createOperatorSetTotalStakeRewardsSubmission()` with UAM compliance, increase `MAX_FUTURE_LENGTH` from 30 days to 365 days, and introduce logic specified [for the Eigen Layer Protocol](#eigenlayer-protocol).

2. **Update EigenLayer Sidecar**: Cut a new release for the [EigenLayer Sidecar](https://github.com/Layr-Labs/sidecar) implementing Tables 15-20 for operator set rewards calculation, `magnitude/max_magnitude` snapshot tracking, withdrawal queue integration with slashing adjustments, and Sabine fork allocation/deallocation rounding logic specified [by the Sidecar](#eigenlayer-sidecar). All parties running rewards calculation MUST upgrade before the protocol upgrade block height.

3. **Testing and Deployment**: Conduct security audits of contract extensions and sidecar logic, perform testnet deployment on Holesky/Sepolia with AVS partner testing, and validate performance with large operator sets and multiple submissions.

This process will start on testnet and follow on Mainnet after validation. Success will be tracked by monitoring AVS adoption, total escrowed value in forward-looking commitments, and unique stake allocation increases for operator sets.

---

This proposal establishes comprehensive auto-weighted reward mechanisms for operator sets, enabling AllocationManager-based AVSs to offer predictable, multi-period incentives while preserving the security and trustworthiness of EigenLayer's rewards system.
