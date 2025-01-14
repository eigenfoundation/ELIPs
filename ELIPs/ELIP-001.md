# EigenLayer Improvement Proposal-001: Rewards v2

| Author(s) | [Matt Nelson](mailto:matt.nelson@eigenlabs.org), [Rajath Alex](mailto:rajath@eigenlabs.org), [Scott Conner](mailto:scott@eigenlabs.org), [Sean McGary](mailto:sean@eigenlabs.org) |
| :---- | :---- |
| Created | Oct 18, 2024 |
| Status | Final |
| References | [EigenLayer Protocol PR](https://github.com/Layr-Labs/eigenlayer-contracts/pull/837) [EigenLayer Middleware PR](https://github.com/Layr-Labs/eigenlayer-middleware/pull/315) [EigenLayer Sidecar PR](https://github.com/Layr-Labs/sidecar/pull/106) |
| Discussions | [Forum Post](https://forum.eigenlayer.xyz/t/elip-001-rewards-v2/14196) |

# Executive Summary

This proposal introduces an iteration to EigenLayer **Rewards (v2)**, an upgrade designed to enhance the flexibility of rewards within the EigenLayer ecosystem. Through feedback from AVSs and Operators following the release of [the first iteration of Rewards](https://docs.eigenlayer.xyz/eigenlayer/rewards-claiming/rewards-claiming-overview), we are proposing an expansion to platform rewards functionalities. Rewards v2 is a lightweight addition to the rewards structures and contracts that address key rewards use-cases: 

* Performance-based and directed rewards from AVSs to Operators,   
* Variable Operator fee on AVS rewards,  
* Operator split for Programmatic Incentives specifically,  
* Batch rewards claiming for Stakers and Operators.

Together, these changes will help enable broad flexibility in the ecosystem: more AVSs can reward Operators, more Operators can run different AVSs with variable fee rates, and Stakers will have data to discover which services and forms of security AVSs value. 

# Motivation

Rewards v2 addresses key challenges identified in the EigenLayer Rewards MVP (v1), particularly the need for Operator-directed rewards via external logic and flexibility for Operators in setting different fee rates to better cover operating costs or attract more stake when running different AVS types. The growing demand from AVSs for more performance-based rewards prompted this new proposal, ensuring EigenLayer’s rewards protocol can serve a wider variety of scenarios. 

Following discussions with AVSs, this proposal seeks to quickly iterate on EigenLayer Rewards to serve the following use-cases:

* Operator-directed rewards from AVSs, allowing AVSs to segment and reward specific Operators. This gives AVSs the flexibility to set custom logic for rewards to individual Operators, based on work completed or anything else they may design or desire (*e.g.*, more equal distribution of Operator support for decentralization or security reasons).   
  * Variable Operator fee per AVS, set by the Operators, allowing Operators to take less or more than the 10% default fee on rewards. This keeps EigenLayer fee-agnostic as a protocol and unlocks flexibility via a variable take rate for Operators in choosing which AVS to run and in attracting new stake.  
  * This also better serves integrated Staker-Operators like LRTs by allowing them to streamline their reward-claiming process, and unlocks more flexibility in programmatically splitting rewards, which may enable new use cases.  
* Batch rewards claiming for Stakers and Operators, allowing a gas efficient way to claim on behalf of multiple earners in a single transaction. 

The above iteration will enable more flexibility in how rewards flow from AVSs to Operators. 

AVSs have sought ways to add additional incentive functionality to the token lifecycle managed by the EigenLayer protocol, like Staker-directed rewards. This functionality is out-of-scope for this proposal.

## Design Goals

Rewards v2 is designed to be an ***optional*** and additive upgrade to the Rewards MVP (v1) with the following properties:

1. **External reward calculation logic for AVSs**:  
   1. AVSs can utilize their own on- or off-chain reward calculation logic based on things other than scaling linearly with stake (*i.e.*, utilizing performance or work generated), and submit strictly retroactive rewards through the performance-based rewards interface. 

   

2. **Flexibility for AVSs and Operator infrastructure:**  
   1. Operators can set their fee per AVS, and signal to both the AVS and Stakers on how much their services are worth and what fees they’ll be taking in exchange for the work involved. Any fee change is time-delayed for seven days to give Stakers time to adjust their Operator service providers as necessary based on the new take rate.   
   2. AVSs can reward the Operators based on the work they have performed over a certain period, with rewards being priced by AVSs for work provided by Operators. This will create an iterative adjustment process to reach equilibrium.  
   3. AVSs can signal to Stakers the services and form of security they value and by what weight. 

3. **Cheaper batch-claiming:**  
   1. Batch-claiming for multiple claimants in a single transaction will help reduce gas costs.    
         
4. **Utilizing existing infrastructure:**  
   1. Only the `RewardsCoordinator` contract will be upgraded to support functionality for performance-based rewards distribution, setting Operator fees per AVS and batch claiming.  
   2. The existing EigenLayer sidecar will be used to support the new rewards type and will factor that into the rewards calculation using the new Operator fees. The rewards root calculation and rewards root posting infrastructure will remain the same.   
   3. The claim UX experience will remain the same for Stakers and Operators with the added benefit of batch-claiming being included. 

5. **Attribution of Rewards**   
   1. The on-chain nature of performance-based or directed Operator rewards, along with the inclusion of strategies and multipliers and integration with the sidecar, enables attribution and reward rate calculations. This helps in populating user-facing information and attracting stake to Operators and AVSs.  
      1. Attribution is defined as the on-chain guarantee that the rewards are split amongst Stakers according to stake weight. This guarantees attributions of the form "Staker-A earned X amount of Token-B on AVS-C because they staked Y amount of Token-D”

   

6. **Permissionless verifiability**:  
   1. All aspects of Rewards v2 can be independently verified.  
   2. Further explanation in the ‘Security Considerations’ section.

   

7. **Preventing Rewards Distribution Tree bloat:**  
   1. There will be validation in the sidecar to ensure that Operators being rewarded during the specified duration had been registered to the AVS for at least a portion of that duration. This will keep the Rewards Tree free of bloat from non-registered Operators and their respective Stakers.  
   2. Further explained in the ‘Security Considerations’ section.

8. **Enabling Programmatic Incentives**:  
   1. More flexibility in rewards and Operator fees better enables the protocol to allocate incentives in the ecosystem based on inputs from AVSs, Stakers, and Operators.   
   2. Richer information provides more insights into AVS, Staker, and Operator preferences, allowing contributors to better understand the state of the market and potentially design new targeted incentives to encourage more specific behaviors.  
   3. Allowing a permissioned set of addresses to send rewards to any Operator will enable incentivization of a wide range of behaviors within the ecosystem.

# Features & Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## High Level Design

![Rewards v2 Flow Chart](../assets/elip-001/Rewards%20v2.png)

The high level flow is as follows:

1. Operators can set their per-AVS reward fee (called a “split”) by calling `setOperatorAVSSplit()` on the `RewardsCoordinator`. This is between 0% or 100% of AVS rewards. It’s valid after a 7-day activation delay. If they don’t set it, it will remain at the default of 10%. An Operator may only have one pending split configuration at a time. 
2. AVSs calculate off-chain the appropriate rewards to be distributed to their registered Operators:  
   1. They first give an ERC20 approval to their AVSServiceManager for the sum of all Operator rewards.   
   2. They then call `createOperatorDirectedAVSRewardsSubmission()` on the `AVSServiceManager` which proxies the call to the `RewardsCoordinator`. This initiates the performance-based rewards allocations and deposits the sum of all Operator rewards in the allocations to the `RewardsCoordinator`.   
   3. An `OperatorDirectedAVSRewardsSubmissionCreated` event is emitted.  
3. The off-chain infrastructure of the existing Rewards v1 is reused for performance-based rewards as well:  
   1. The [Sidecar](https://github.com/Layr-Labs/sidecar) listens for the  `OperatorDirectedAVSRewardsSubmissionCreated` event and stores it.  
   2. The Sidecar generates reward roots on a daily basis taking into account the Operator-directed rewards and the appropriate Operator split (in addition to the other rewards events it’s already processing).  
   3. The Root Updater retrieves the latest root from the sidecar and posts the root on a weekly basis by calling `submitRoot()` on the `RewardsCoordinator`.  
4. Operators and Stakers then batch-claim all their rewards by calling `processClaims()` on the `RewardsCoordinator` contract, after the existing 7-day activation delay.

More details on the Sidecar can be found below.

## Low Level Specification

### EigenLayer Protocol

The `RewardsCoordinator` Transparent Proxy will be upgraded to point to a new implementation that includes the following logic. 

#### Distribution of Operator-directed Rewards

The design is largely built atop the existing Rewards v1 interface with additional fields that:

1. Enable attributable rewards from AVSs to Operators.  
2. Future-proofs the interface for AVS delegation.

The AVS is free to use on-chain or off-chain data in reward attribution logic to determine the reward amount per Operator. This can be custom to the work performed by Operators during a certain period of time, can be a flat reward rate, or some other structure based on the AVS’s economic model. This would enable AVSs’ flexibility in rewarding different Operators for performance and other variables while maintaining the same easily calculable reward rate for Stakers delegating to the same Operator and strategy. The AVS can submit multiple performance-based rewards denominated in different tokens for even more flexibility. 

##### Interface

The Operator-directed Rewards interface in the `RewardsCoordinator` can be found below.

```solidity
/**
 * @notice A linear combination of strategies and multipliers for AVSs to weigh EigenLayer strategies.
 * @param strategy The EigenLayer strategy to be used for the rewards submission.
 * @param multiplier The weight of the strategy in the rewards submission.
 */
 struct StrategyAndMultiplier {
     IStrategy strategy;
     uint96 multiplier;
 }

/**
 * @notice A reward struct for an operator
 * @param operator The operator to be rewarded
 * @param amount The reward amount for the operator
 */
struct OperatorReward {
    address operator;
    uint256 amount;
}

/**
 * @notice OperatorDirectedRewardsSubmission struct submitted by AVSs when making operator-directed rewards for their operators and stakers.
 * @param strategiesAndMultipliers The strategies and their relative weights.
 * @param token The rewards token to be distributed.
 * @param operatorRewards The rewards for the operators.
 * @param startTimestamp The timestamp (seconds) at which the submission range is considered for distribution.
 * @param duration The duration of the submission range in seconds.
 * @param description Describes what the rewards submission is for.
 */
struct OperatorDirectedRewardsSubmission {
    StrategyAndMultiplier[] strategiesAndMultipliers;
    IERC20 token;
    OperatorReward[] operatorRewards;
    uint32 startTimestamp;
    uint32 duration;
    string description;
}

/**
 * @notice Emitted when an AVS creates a valid `OperatorDirectedRewardsSubmission`
 * @param caller The address calling `createOperatorDirectedAVSRewardsSubmission`.
 * @param avs The avs on behalf of which the operator-directed rewards are being submitted.
 * @param operatorDirectedRewardsSubmissionHash Keccak256 hash of (`avs`, `submissionNonce` and     `operatorDirectedRewardsSubmission`).
 * @param submissionNonce Current nonce of the avs. Used to generate a unique submission hash.
 * @param operatorDirectedRewardsSubmission The Operator-Directed Rewards Submission. Contains the token, start timestamp, duration, operator rewards, description and, strategy and multipliers.
 */
event OperatorDirectedAVSRewardsSubmissionCreated(
    address indexed caller,
    address indexed avs,
    bytes32 indexed operatorDirectedRewardsSubmissionHash,
    uint256 submissionNonce,
    OperatorDirectedRewardsSubmission  
);

/**
 * @notice Creates a new operator-directed rewards submission on behalf of an AVS, to be split amongst the operators and
 * set of stakers delegated to operators who are registered to the `avs`.
 * @param avs The AVS on behalf of which the reward is being submitted.
 * @param operatorDirectedRewardsSubmissions The operator-directed rewards submissions being created.
 */
function createOperatorDirectedAVSRewardsSubmission(
    address avs,
    OperatorDirectedRewardsSubmission[] calldata operatorDirectedRewardsSubmissions
) external;
```

##### Implementation

1. Operator-directed rewards MUST include the above fields; none are OPTIONAL.  
2. It is ONLY callable by the `avs` currently (It is future-proofed to be called by an avs delegated address). There needs to be a `msg.sender == avs` check. Throw an error if the check fails.   
3. For each `operatorDirectedRewardsSubmissions`:  
   1. Validate the following:  
      1. Throw an error if length of `strategiesAndMultipliers` array is 0. These are REQUIRED.  
      2. Throw an error if length of `operatorRewards` array is 0. These are REQUIRED.  
      3. For each `operatorReward`, validate the following:  
         1. Throw an error if `operator` is `address(0)`.  
         2. Throw an error if Operator addresses are not in ascending order. This is to handle duplicates.  
         3. Throw an error if `amount` is 0.  
      4. Sum of all amounts MUST be <= `MAX_REWARDS_AMOUNT`. Throw an error otherwise.  
      5. `duration` MUST be <= `MAX_REWARDS_DURATION`. Throw an error otherwise.  
      6. `duration` MUST be a multiple of `CALCULATION_INTERVAL_SECONDS`. Throw an error otherwise.  
      7. `duration` MUST be a multiple of `CALCULATION_INTERVAL_SECONDS`. Throw an error otherwise.  
      8. `startTimestamp` MUST be a multiple of `CALCULATION_INTERVAL_SECONDS`. Throw an error otherwise.  
      9. `startTimestamp` MUST be > `GENESIS_REWARDS_TIMESTAMP`. Throw an error otherwise.  
      10. `startTimestamp` MUST be within the `MAX_RETROACTIVE_LENGTH` prior to `block.timestamp`. Throw an error otherwise.  
      11. `startTimestamp + duration` MUST be < `block.timestamp`. Throw an error otherwise. Performance-based rewards have to be strictly retroactive.  
      12. For each `strategyAndMultiplier`, validate the following:  
          1. `strategy` has to be whitelisted for deposit. Throw an error otherwise.  
          2. Throw an error if strategy addresses are not in ascending order. This is to handle duplicates.  
   2. Keccak256 hash the ABI encoding of `avs`, `submissionNonce` and `operatorDirectedRewardsSubmission`.  
   3. Store the hash and update the nonce.   
   4. Transfer the total amount of rewards for the Performance Reward Submission token into the `RewardsCoordinator` contract.  
   5. `OperatorDirectedAVSRewardsSubmissionCreated` event will be emitted.

##### Caveats

1. The `avs` is currently the respective `AVSServiceManager`.  
2. The `RewardsCoordinator` contract needs a token approval of the sum of all `operatorRewards` in the `operatorDirectedRewardsSubmissions`, before calling `createOperatorDirectedAVSRewardsSubmission`.  
3. The `createOperatorDirectedAVSRewardsSubmission` function mostly just does on-chain validation, depositing the total amount of rewards into the `RewardsCoordinator` contract and emitting the event.  
4. When implemented along with the sidecar changes, each call to `createOperatorDirectedAVSRewardsSubmission()` would reward each `operator` and their Stakers `amount` of `token` over `duration` since `startTimestamp`, on behalf of the `avs`. The Operator would get their contract-configured split of the rewards, set in the `RewardsCoordinator` and the rest would be distributed to Stakers proportional to the strategies and multipliers.   
5. The rewards calculation is done off-chain in the sidecar. Further explained in the [EigenLayer Sidecar](#eigenlayer-sidecar) section. Check there to understand how Operators and Staker rewards are calculated for the given duration. 

#### Variable Operator Fees

After Rewards v2 is live, Operators will be able to change their take-home fee on a per-AVS granularity in the `RewardsCoordinator`. It’s valid after a 7-day activation delay to give Stakers time to adjust their Operator service providers as necessary based on the new take rate. If they don’t set the per-AVS rewards split, it will default to 10%. 

This variable fee is provided with no constraints on its value between 0% and 100% of the AVS-paid rewards. AVSs may still choose to eject Operators they believe are acting in bad faith by calling [deregisterOperatorFromAVS()](https://docs.eigenlayer.xyz/developers/avs-dashboard-onboarding) on the AVS Directory contract. We hope AVSs and Operators will engage in discussion with each other to discover the correct dynamics for different types of AVSs.

##### Interface

Variable Operator Split Interface in the `RewardsCoordinator` can be found below.

```solidity
/**
 * @notice A split struct for an Operator
 * @param oldSplitBips The old split in basis points. This is the split that is active if `block.timestamp < activatedAt`
 * @param newSplitBips The new split in basis points. This is the split that is active if `block.timestamp >= activatedAt`
 * @param activatedAt The timestamp at which the split will be activated
 */
struct OperatorSplit {
    uint16 oldSplitBips;
    uint16 newSplitBips;
    uint32 activatedAt;
}



/**
 * @notice Emitted when the operator split for an AVS is set.
 * @param caller The address calling `setOperatorAVSSplit`.
 * @param operator The operator on behalf of which the split is being set.
 * @param avs The avs for which the split is being set by the operator.
 * @param activatedAt The timestamp at which the split will be activated. 
 * @param oldOperatorAVSSplitBips The old split for the operator for the AVS.
 * @param newOperatorAVSSplitBips The new split for the operator for the AVS.
 */
event OperatorAVSSplitBipsSet(
    address indexed caller,
    address indexed operator,
    address indexed avs,
    uint32 activatedAt,
    uint16 oldOperatorAVSSplitBips,
    uint16 newOperatorAVSSplitBips
);


/**
 * @notice Sets the split for a specific operator for a specific avs.
 * @param operator The operator who is setting the split.
 * @param avs The avs for which the split is being set by the operator.
 * @param split The split for the operator for the specific avs.
 */
function setOperatorAVSSplit(address operator, address avs, uint16 split) external;


/// @notice the split for a specific `operator` for a specific `avs`
function getOperatorAVSSplit(address operator, address avs) external view returns (uint16);
```

##### Implementation

1. `setOperatorAVSSplit()` MUST include all fields; none are OPTIONAL.   
2. It is ONLY callable by the `operator` currently (It is future-proofed to be called by an Operator delegated address). There needs to be a `msg.sender == operator` check. Throw an error if the check fails.   
3. Throw an error if `split` is strictly greater than `10000` (i.e 100%).  
4. Each call to `setOperatorAVSSplit()` sets the `split` (in Bips) for the `avs` on behalf of the `operator`.   
5. The `split` will be activated after a 7-day activation delay.   
6. `OperatorAVSSplitBipsSet` event will be emitted.

##### Caveats

1. The 7-day activation delay for the split will be taken into account by the sidecar when doing the rewards calculation. Further explained in the [EigenLayer Sidecar](#eigenlayer-sidecar) section.

#### Operator Splits for Programmatic Incentives

Eigen Foundation Programmatic Incentives will have a default Operator split of 10% with a variable split to ensure Operators of all sizes and stakes are rewarded for their participation. It’s valid after a 7-day activation delay. It can be set independently of other AVSs, with the same reward distribution dynamics for Operator-delegated Stakers as v2 rewards. 

##### Interface

Operator Splits for Programmatic Incentives Interface in the `RewardsCoordinator` can be found below.

```solidity
/**
 * @notice A split struct for an Operator
 * @param oldSplitBips The old split in basis points. This is the split that is active if `block.timestamp < activatedAt`
 * @param newSplitBips The new split in basis points. This is the split that is active if `block.timestamp >= activatedAt`
 * @param activatedAt The timestamp at which the split will be activated
 */
struct OperatorSplit {
    uint16 oldSplitBips;
    uint16 newSplitBips;
    uint32 activatedAt;
}


/**
 * @notice Emitted when the operator split for Programmatic Incentives is set.
 * @param caller The address calling `setOperatorPISplit`.
 * @param operator The operator on behalf of which the split is being set.
 * @param activatedAt The timestamp at which the split will be activated. 
 * @param oldOperatorPISplitBips The old split for the operator for Programmatic Incentives.
 * @param newOperatorPISplitBips The new split for the operator for Programmatic Incentives.
 */
event OperatorPISplitBipsSet(
    address indexed caller,
    address indexed operator,
    uint32 activatedAt,
    uint16 oldOperatorPISplitBips,
    uint16 newOperatorPISplitBips
);


/**
 * @notice Sets the split for a specific operator for Programmatic Incentives.
 * @param operator The operator on behalf of which the split is being set.
 * @param split The split for the operator for Programmatic Incentives.
 */
function setOperatorPISplit(address operator, uint16 split) external;


/// @notice the split for a specific `operator` for Programmatic Incentives
function getOperatorPISplit(address operator) external view returns (uint16);
```

##### Implementation

1. `setOperatorPISplit()` MUST include all fields; none are OPTIONAL.   
2. It is ONLY callable by the `operator` currently (It is future-proofed to be called by an Operator delegated address). There needs to be a `msg.sender == operator` check. Throw an error if the check fails.   
3. Throw an error if `split` is strictly greater than `10000` (i.e 100%).  
4. Each call to `setOperatorPISplit()` sets the `split` (in Bips) for Programmatic Incentives on behalf of the `operator`.   
5. The `split` will be activated after a 7-day activation delay.   
6. `OperatorPISplitBipsSet` event will be emitted.

##### Caveats

1. The 7-day activation delay for the split will be taken into account by the Sidecar when doing the rewards calculation. Further explained in the [EigenLayer Sidecar](#eigenlayer-sidecar) section.

#### Batch Reward Claiming 

For a gas-efficient way to claim, Rewards v2 also ships a new interface for batch claiming (i.e `processClaims()`). The v1 interface (i.e `processClaim()`) will still be available. Currently, we only enable claiming per single earner. This interface provides a way to batch claims for multiple earners into a single transaction. 

##### Interface

Batch Reward Claiming Interface in the `RewardsCoordinator` can be found below.

```solidity
/**
 * @notice Internal leaf in the merkle tree for the earner's account leaf.
 * @param earner The address of the earner.
 * @param earnerTokenRoot The merkle root of the earner's token subtree.
 */
struct EarnerTreeMerkleLeaf {
    address earner;
    bytes32 earnerTokenRoot;
}


/**
 * @notice The actual leaves in the distribution merkle tree specifying the token earnings for the respective earner's subtree.
 * @param token The ERC20 token for which the earnings are being claimed.
 * @param cumulativeEarnings The cumulative earnings of the earner for the token.
 */
struct TokenTreeMerkleLeaf {
    IERC20 token;
    uint256 cumulativeEarnings;
}

	
/**
 * @notice A claim against a distribution root.
 * @param rootIndex The index of the root in the list of DistributionRoots.
 * @param earnerIndex The index of the earner's account root in the merkle tree.
 * @param earnerTreeProof The proof of the earner's EarnerTreeMerkleLeaf against the merkle root.
 * @param earnerLeaf The earner's EarnerTreeMerkleLeaf struct, providing the earner address and earnerTokenRoot
 * @param tokenIndices The indices of the token leaves in the earner's subtree
 * @param tokenTreeProofs The proofs of the token leaves against the earner's earnerTokenRoot
 * @param tokenLeaves The token leaves to be claimed.
 */
struct RewardsMerkleClaim {
    uint32 rootIndex;
    uint32 earnerIndex;
    bytes earnerTreeProof;
    EarnerTreeMerkleLeaf earnerLeaf;
    uint32[] tokenIndices;
    bytes[] tokenTreeProofs;
    TokenTreeMerkleLeaf[] tokenLeaves;
}


/**
 * @notice Emitted when earner rewards are claimed against a distribution root. 
 * @param root The distribution root being claimed against.
 * @param earner The earner on behalf of which the rewards are being claimed.
 * @param claimer The claimer of the rewards.
 * @param recipient The address that receives the ERC20 rewards.
 * @param token The ERC20 token for which the earnings are being claimed.
 * @param claimedAmount The amount being claimed.
 */
event RewardsClaimed(
    bytes32 root,
    address indexed earner,
    address indexed claimer,
    address indexed recipient,
    IERC20 token,
    uint256 claimedAmount
);


/**
 * @notice Batch claim rewards against a given distribution root.
 * @param claims The RewardsMerkleClaims to be processed. Contains the root index, earner, token leaves, and required proofs
 * @param recipient The address that receives the ERC20 rewards.
 */
function processClaims(RewardsMerkleClaim[] calldata claims, address recipient) external;
```

##### Implementation

1. `processClaims()` MUST include all fields; none are OPTIONAL.   
2. All logic in the existing `processClaim()` external function will be refactored into a new `_processClaim()` internal function. `processClaim()` will call `_processClaim()`. This keeps the existing interface for `processClaim()` intact.  
3. `processClaims()` is only only callable by the valid claimer, that is if `claimerFor[claim.earner]` is `address(0)` then only the earner can claim, otherwise only `claimerFor[claim.earner]` can claim the rewards.  
4. `processClaims()` will iterate over the multiple `claims` and call `_processClaim()` for each claim.  
5. The rewards merkle tree is structured with the merkle root at the top and `EarnerTreeMerkleLeaf` as internal leaves in the tree. Each earner leaf has its own subtree with `TokenTreeMerkleLeaf` as leaves in the subtree. To prove a claim against a specified `rootIndex` (which specifies the distributionRoot being used),  the claim will first verify inclusion of the earner leaf in the tree against `_distributionRoots[rootIndex].root`. Then for each token, it will verify inclusion of the token leaf in the earner's subtree against the earner's `earnerTokenRoot`.  
6. `RewardsClaimed` event will be emitted for each claim.

##### Caveats

1. Earnings are cumulative, so earners don't have to claim against all distribution roots they have earnings for. They can simply claim against the latest root and the contract will calculate the difference between their `cumulativeEarnings` and `cumulativeClaimed`. This difference is then transferred to the `recipient` address.  
2. Each claim can specify which of the earner's earned tokens they want to claim (i.e not all tokens have to be claimed in a single claim)

#### Updated Calculation Interval Seconds

`CALCULATION_INTERVAL_SECONDS` will be updated from 1 week to 1 day to reflect the daily rewards calculation happening in the sidecar. This will enable AVSs to submit rewards with `startTimestamp` and `duration` that are multiples of 1 day instead of 1 week.

### EigenLayer Middleware

The EigenLayer Middleware SHALL have a mandatory release as part of this ELIP to support performance-based rewards. AVSs MUST upgrade their respective `AVSServiceManager` contracts to inherit the new `ServiceManagerBase` implementation in order to be able to submit performance-based rewards.

##### Interface

The Operator-directed Rewards interface in the `ServiceManagerBase` can be found below:

```solidity
/**
 * @notice Creates a new operator-directed rewards submission, to be split amongst the operators and set of Stakers delegated to operators who are registered to this `avs`.
 * @param operatorDirectedRewardsSubmissions The operator-directed rewards submissions being created
 */
function createOperatorDirectedAVSRewardsSubmission(IRewardsCoordinator.OperatorDirectedRewardsSubmission[] calldata operatorDirectedRewardsSubmissions) external;


/**
 * @notice Forwards a call to Eigenlayer's RewardsCoordinator contract to set the address of the entity that can call `processClaim` on behalf of this contract.
 * @param claimer The address of the entity that can call `processClaim` on behalf of the earner
 */
function setClaimerFor(address claimer) external;
```

##### Implementation

1. `createOperatorDirectedAVSRewardsSubmission()` MUST include the above fields; none are OPTIONAL.   
2. It is ONLY callable by the `rewardsInitiator`. Throw an error otherwise.  
3. For each `operatorDirectedRewardsSubmissions`:  
   1. Transfer the total amount of rewards to the `AVSServiceManager` in that token.  
   2. Approve the `RewardsCoordinator` for the total amount of rewards in that token.  
4. Call `createOperatorDirectedAVSRewardsSubmission` in the `RewardsCoordinator` contract.

##### Caveats

1. The `operatorDirectedRewardsSubmission`object has to adhere to the validations in `RewardsCoordinator.createOperatorDirectedAVSRewardsSubmission`  
2. The `AVSServiceManager` contract needs a token approval of the sum of all `operatorRewards` in the `operatorDirectedRewardsSubmissions`, before calling `createOperatorDirectedAVSRewardsSubmission`.

### EigenLayer Sidecar

The EigenLayer Sidecar is an open source, permissionless, verified indexer enabling anyone (AVS, Operator, etc) to access EigenLayer’s protocol rewards and EigenLayer state in real-time. The EigenLayer Sidecar SHALL have a mandatory release as part of this ELIP to augment performance-based rewards. This is a critical part of the Rewards v2 flow.

The following high-level features will be introduced:

1. Ingest new Rewards v2 events and store them in the sidecar’s internal DB: `OperatorDirectedAVSRewardsSubmissionCreated`, `OperatorAVSSplitBipsSet`, `OperatorPISplitBipsSet`.  
2. New Operator-directed rewards calculation that includes the per-avs Operator split (if set).  
3. Update Rewards MVP (v1) calculation to include the per-avs Operator split (if set).   
4. Update Programmatic Incentives rewards calculation to include the Operator split for Programmatic Incentives (if set).  
   

#### EigenStateModel

The Rewards v2 release will include state models for the 3 new events being supported: `OperatorDirectedAVSRewardsSubmissionCreated`, `OperatorAVSSplitBipsSet` and `OperatorPISplitBipsSet`.

##### Schema

```solidity
// Eigen State Schema for `OperatorDirectedAVSRewardsSubmissionCreated` event
type OperatorDirectedRewardSubmission struct {
	Avs             string
	RewardHash      string
	Token           string
	Operator        string
	OperatorIndex   uint64
	Amount          string
	Strategy        string
	StrategyIndex   uint64
	Multiplier      string
	StartTimestamp  *time.Time
	EndTimestamp    *time.Time
	Duration        uint64
	Description     string
	BlockNumber     uint64
	TransactionHash string
	LogIndex        uint64
}

// Eigen State Schema for `OperatorAVSSplitBipsSet` event
type OperatorAVSSplit struct {
	Operator                string
	Avs                     string
	ActivatedAt             *time.Time
	OldOperatorAVSSplitBips uint64
	NewOperatorAVSSplitBips uint64
	BlockNumber             uint64
	TransactionHash         string
	LogIndex                uint64
}

// Eigen State Schema for `OperatorPISplitBipsSet` event
type OperatorPISplit struct {
	Operator                string
	ActivatedAt             *time.Time
	OldOperatorAVSSplitBips uint64
	NewOperatorAVSSplitBips uint64
	BlockNumber             uint64
	TransactionHash         string
	LogIndex                uint64
}
```

##### Implementation

Each of the new events will be ingested by the sidecar, processed and stored in the sidecar’s internal DB in their own tables according to the schema described above. 

#### Operator-directed Rewards Calculation

Rewards v2 release will include a new Operator-directed Rewards Calculation that takes into account the per-avs Operator split. 

##### Implementation

Operator-directed rewards calculation will follow the existing implementation of rewards calculation as specified [here](https://hackmd.io/Fmjcckn1RoivWpPLRAPwBw).

With the addition of the following things:

1. Operator commission is calculated according to the following logic:  
   1. If an activated `OperatorAVSSplit` row exists in the `operator_avs_split` table for the particular `operator` at the current snapshot time, then use that `split` for the rewards calculation.   
   2. Else, default to a `split` of 10%.  
2. In the edge case of Operator-directed reward submissions including Operators not registered to that specific AVS during the specific snapshot time, the Operator amount for that snapshot is refunded to the AVS as a distribution leaf for that snapshot. The AVS can claim it using the regular claim process to get refunded. Reasoning for this is explained in the [Security Considerations](#security-considerations) section (under Preventing Rewards Distribution Tree bloat)

#### Rewards MVP (v1) Calculation

The Rewards MVP calculation will be updated to include the per-avs Operator split being released as part of Rewards v2. 

##### Implementation

1. If an activated `OperatorAVSSplit` row exists in the `operator_avs_split` table for the particular `operator` at the current snapshot time, then use that `split` for the rewards calculation.   
2. Else, default to a `split` of 10%. 

#### Programmatic Incentives Calculation

The Programmatic Incentives calculation will be updated to include the per-avs Operator split being released as part of Rewards v2. 

##### Implementation

1. If an activated `OperatorPISplit` row exists in the `operator_pi_split` table for the particular `operator` at the current snapshot time, then use that `split` for the rewards calculation.   
2. Else, default to a `split` of 10%.

# Security Considerations

Key security considerations include:

1. **Permissionless Verifiability**:  
   1. Rewards v2 utilizes the same off-chain infrastructure for rewards calculation (i.e sidecar) and root posting (i.e root updater) as the Rewards MVP (v1).  
   2. The sidecar is designed for replayability from EigenLayer genesis so that anyone can spin up a new sidecar in a permissionless manner and verify the calculations.  
   3. The sidecar will witness and verify that the updated root in the RewardsCoordinator is correct using its existing calculations.  
        
2. **Preventing Rewards Distribution Tree bloat:**  
   1. There will be validation in the sidecar to ensure that Operators being rewarded during the specified duration had been registered to the AVS for at least a portion of that duration. This will keep the Rewards Tree free of bloat from non-registered Operators and their respective Stakers.  
   2. In case there is an on-chain reward to a non-registered Operator during the specific snapshot time, the amount for the non-registered Operator for that snapshot is refunded to the AVS as a distribution leaf for that snapshot. The AVS can then claim those funds as part of the regular claim process.

# Impact Summary

The expected impacts of Rewards v2 include:

Pros:

1. **Enhance Flexibility**: AVSs gain more flexible control over reward logic, enabling customized and diverse reward mechanisms that can be attributed on-chain.  
2. **Increase Adoption by AVS:** By supporting Operator-directed reward systems, Rewards v2 will make Eigenlayer more useful to AVSs.  
3. **Encourage Operators to Run more AVSs**: By providing a variable Operator fee structure per AVS, Operators have more economic flexibility when choosing an AVS to run.   
4. **Improve Rewards Release Velocity**: Operator-directed rewards decouples reward logic from core protocol contracts, enabling AVSs to iterate rewards logic faster and reducing bottlenecks in protocol updates.  
5. **Permissionless Verifiability**: All aspects of Rewards v2 can be independently verified.  
6. **Batching gas savings**: By enabling batching for rewards claims, Operators and Stakers with many addresses can save on redemptions.   
   

Cons:

1. **Gas Cost**: As Operator-directed rewards remain on-chain to keep the property of attribution of rewards, gas costs of the reward submission will increase linearly as AVSs reward more Operators.   
2. **No direct-to-Staker rewards or incentive structures**: The trade-off of on-chain attribution removed the wholly flexible structure for distribution. This removes the use-case for EigenLayer as an incentives distribution mechanism and token lifecycle tool for AVSs.

# Action Plan

The implementation of this ELIP will follow these key steps:

1. **Upgrade EigenLayer Protocol**: Upgrade the [`RewardsCoordinator`](https://etherscan.io/address/0x7750d328b314EfFa365A0402CcfD489B80B0adda) Transparent Proxy to point to a new implementation contract that introduces logic specified [here](#eigenlayer-protocol).   
2. **Update EigenLayer Middleware**: Cut a new release for [`ServiceManagerBase`](https://github.com/Layr-Labs/eigenlayer-middleware/blob/mainnet/src/ServiceManagerBase.sol) which introduces logic specified [here](#eigenlayer-middleware). AVSs MUST upgrade their respective `AVSServiceManager` contracts to inherit the new `ServiceManagerBase` implementation in order to be able to submit performance-based rewards.   
3. **Update EigenLayer Sidecar**: Cut a new release for the [EigenLayer Sidecar](https://github.com/Layr-Labs/sidecar) which introduces logic specified [here](#eigenlayer-sidecar). All parties running the rewards calculation and verification MUST upgrade their EigenLayer sidecar to this release before the block height at which the EigenLayer protocol is upgraded for this ELIP. Not doing so will lead to incorrect calculations and possible halting of the sidecar. 

This process will start first on the Eigen testnet and follow later on Mainnet after testing and integration by AVSs to ensure success. Following Mainnet, the up-take of Rewards v2 will be tracked by indexing the events of rewards on-chain. This can help determine the success of this ELIP and track the improvements to the protocol.

# References & Relevant Discussions

[Operator Working Group October 2024](https://drive.google.com/file/d/1OPlpFnIKppYxqTKqvfu4x0QJy7bd5Uxm/view?usp=sharing)  
[AVS Working Group October 2024](https://drive.google.com/file/d/13PKbxbyaqiJHsQExd1yc63hDSBSdeRqN/view?usp=sharing)
