| Author(s) | Created | Status | References | Discussions |
|-------------|-----------|---------|------|----------|
| [Matt Nelson](mailto:matt.nelson@eigenlabs.org), [0xClandestine](https://github.com/0xClandestine) | 2025-04-30 | `Draft` | [List of relevant work/PRs, if any] | [Discussion Forum Post](https://forum.eigenlayer.xyz/t/elip-006-redistributable-slashing/14553) |

# ELIP-006: Redistributable Slashing

---

# Executive Summary

[Slashing (ELIP-002)](./ELIP-002.md) is a key piece of EigenLayer's vision; it enables enforcement of crypto-economic commitments made by Service builders to their consumers and users. When leveraging slashing on EigenLayer today, security funds are always burned or locked when penalizing Operators. This creates a challenge for builders of use-cases that involve lending, insurance, risk hedging, or, broadly, commitments with a need to compensate harmed parties or amortize risk.

Redistributable Slashing is a feature that gives Service Builders a means to not just burn, but repurpose slashed funds. Redistribution represents an expansion of the types of use-cases builders can launch on EigenLayer, by expanding the expressivity of slashing on the platform. A new type of Operator Set with strict configuration controls allows for specifying a redistribution recipient by the AVS that receives slashed funds. This new feature requires, and is shipped with, adjustments to the EigenLayer security model and stake guarantees for AVSs to support this new slashing paradigm.

# Motivation

Slashings within EigenLayer cause an immediate and irreversible burn of funds. The existing slashing mechanism is restrictive in terms of capital expressivity; funds are either permanently destroyed or necessitate off-protocol methods to repurpose. This limitation significantly narrows the scope of potential applications—particularly in key areas like insurance and lending. Meanwhile, out-of-protocol or competing solutions embrace redistributable slashing. The absence of redistribution impacts Service Builders exploring use-cases like DeFi or insurance.

Introducing redistributable slashing significantly expands the expressivity and practicality of slashing on EigenLayer. More sophisticated slashing unblocks valuable protocol applications such as insurance, lending, bridging, and DeFi services on EigenLayer. Enabling Service Builders to design protocols around redistributable slashing improves capital efficiency for AVSs, Operators, and Stakers. These slashing changes represent high-impact opportunities with only modest engineering effort.

Collectively, Redistributable slashing promises expanded use-case diversity, greater AVS participation, increased value accrual for both security assets and AVS tokens, and ultimately, stronger revenue growth in the EigenLayer ecosystem.

# Features & Specification

## Overview

As of today, when slashed, ERC-20 funds are burned at the `0x0...00316e4` address; EigenPod Native ETH funds are permanently locked when slashed. This is done asynchronously following the `slashOperator` function. There is more detail in [ELIP-002](./ELIP-002.md#slashing-of-unique-stake) on slashing mechanics. The same `slashOperator` mechanics apply, in large part. Redistributable Slashing requires minimal changes to the core protocol...

- to create a new type of Redistributable Operator Set,
- to handle a `redistributionRecipient`, that replaces the burn address when `burnFunds` is called to transfer funds out of the protocol,
- to better decorate each slash with an identifier (`slashId`) that helps in downstream programmatic redistribution and accounting,
- and to modify some permission and withdrawal handling to strengthen guarantees of `slashOperator` (in both the burn and redistribute case).

These changes are externally facing in the `AllocationManager` interface. This is accompanied by changes to the storage of the `AllocationManager` as well. Internally, we have modified the `ShareManager` and `StrategyManager` interfaces, as well as some storage and internal logic.

This proposal outlines a new core contract, the `SlashEscrowFactory` that brings new guarantees to `slashOperator` and protocol outflows. In the case of an implementation bug that would allow an AVS to slash *beyond its allocated unique stake* (e.g. a total protocol TVL drain), the `SlashEscrowFactory` exists to enable a governance pause and intervention. The `SlashEscrowFactory` creates contracts that hold and apply a delay on all slashed funds exiting the protocol to allow for this intervention, whether burned or redistributed.

## Specifications

### Redistributing Operator Sets

To take advantage of redistributable slashing, an AVS must instantiate a new `RedistributingOperatorSet`. These sets specify a `redistributionRecipient` address that CANNOT be changed later on. AVSs may set whatever contracts they like upstream, but should make strong guarantees about the way they function in order to attract and retain Stakers and Operators.

New getter and setter functions are provided in the `AllocationManger` interface:

```solidity
interface IAllocationManager {
    /// @dev Existing struct in `IAllocationManager`.
    struct CreateSetParams {
        uint32 operatorSetId;
        IStrategy[] strategies;
    }
    
    /// EVENT

    /// @notice Emitted when a redistributing operator set is created by an AVS.
    event RedistributionAddressSet(OperatorSet operatorSet, address redistributionRecipient);
    
    /// WRITE

    /**
     * @notice Called by an AVS to slash an operator in a given operator set. The operator must be registered
     * and have slashable stake allocated to the operator set.
     *
     * @param avs The AVS address initiating the slash.
     * @param params The slashing parameters, containing:
     *  - operator: The operator to slash.
     *  - operatorSetId: The ID of the operator set the operator is being slashed from.
     *  - strategies: Array of strategies to slash allocations from (must be in ascending order).
     *  - wadsToSlash: Array of proportions to slash from each strategy (must be between 0 and 1e18).
     *  - description: Description of why the operator was slashed.
     *
     * @dev For each strategy:
     *      1. Reduces the operator's current allocation magnitude by wadToSlash proportion.
     *      2. Reduces the strategy's max and encumbered magnitudes proportionally.
     *      3. If there is a pending deallocation, reduces it proportionally.
     *      4. Updates the operator's shares in the DelegationManager.
     *
     * @dev Small slashing amounts may not result in actual token burns due to
     *      rounding, which will result in small amounts of tokens locked in the contract
     *      rather than fully burning through the burn mechanism.
     * @return slashId The operator set's unique identifier for the slash.
     * @return shares The number of shares to be burned or redistributed for each strategy that was slashed.
     */
    function slashOperator(
        address avs,
        SlashingParams calldata params
    ) external returns (uint256 slashId, uint256[] memory shares);

    /// READ

    /**
     * @notice Returns the address where slashed funds will be sent for a given operator set.
     * @param operatorSet The Operator Set to query.
     * @return For redistributing Operator Sets, returns the configured redistribution address set during Operator Set creation.
     *         For non-redistributing operator sets, returns the `DEFAULT_BURN_ADDRESS`.
     */
    function getRedistributionRecipient(
        OperatorSet memory operatorSet
    ) external view returns (address);

    /**
     * @notice Returns whether a given operator set supports redistribution
     * or not when funds are slashed and burned from EigenLayer.
     * @param operatorSet The Operator Set to query.
     * @return For redistributing Operator Sets, returns true.
     *         For non-redistributing Operator Sets, returns false.
     */
    function isRedistributingOperatorSet(
        OperatorSet memory operatorSet
    ) external view returns (bool);

    /**
     * @notice Returns the number of slashes for a given operator set.
     * @param operatorSet The operator set to query.
     * @return The number of slashes for the operator set.
     * @dev Slash counter will only increment after redistribution is live (not inclusive of previous slashes).
     */
    function getSlashCount(
        OperatorSet memory operatorSet
    ) external view returns (uint256);

    /**
     * @notice Returns whether an operator is slashable by a redistributing operator set.
     * @param operator The operator to query.
     */
    function isOperatorRedistributable(
        address operator
    ) external view returns (bool);
}
```

`RedistributingOperatorSets` act the same way in the core as normal Operator Sets. The [logic and control flow for Operator Set creation and registration](./ELIP-002.md#creation-registration--deregistration) as it relates to the protocol and `AVSRegistrar` contract remain the same. We store an additional field in storage that maps that Operator Set ID to a given `redistributionRecipient`. [Stake allocation mechanics](ELIP-002#allocating-and-deallocating-to-operator-sets) act the same as other Operator Sets.

### Slash Identifiers

Slashes are largely handled the same as in [the existing protocol](./ELIP-002.md#burning-of-slashed-funds). The `slashOperator` function has been updated to return a `slashId` (acting as a nonce), the `strategies` slashed, and the amount slashed in `shares`. This addition of the `slashId` in return data is to aid in programmatic handling of funds in redistribution logic by upstream (and out-of-protocol) contracts. Accounting of stake has not changed. The new storage model is below:

```solidity
abstract contract AllocationManagerStorage {
    /// @notice Returns the number of slashes for a given operator set.
    /// @dev This is also used as a unique slash identifier.
    /// @dev This tracks the number of slashes after the redistribution release.
    mapping(bytes32 operatorSetKey => uint256 slashId) internal _slashCount;

    /// @notice Returns the address where slashed funds will be sent for a given operator set.
    /// @dev For redistributing Operator Sets, returns the configured redistribution address set during Operator Set creation.
    ///      For non-redistributing or non-existing operator sets, returns `address(0)`.
    mapping(bytes32 operatorSetKey => address redistributionAddr) internal _redistributionRecipients;
}
```

All core contracts handling a slash, the `DelegationManager`, `ShareManager`, `StrategyManager`, and `EigenPodManager` have certain inputs updated to accept the `slashId`. This is for proper downstream accounting during burn or redistribution of funds. Below the `ShareManager` is modified for `slashId`

```solidity
interface IShareManager {
    /// @notice Used by the DelegationManager to remove a Staker's shares from a particular strategy when entering the withdrawal queue
    /// @dev strategy must be beaconChainETH when talking to the EigenPodManager
    /// @return updatedShares the staker's deposit shares after decrement
    function removeDepositShares(
        address staker,
        IStrategy strategy,
        uint256 depositSharesToRemove
    ) external returns (uint256);

    /// @notice Used by the DelegationManager to award a Staker some shares that have passed through the withdrawal queue
    /// @dev strategy must be beaconChainETH when talking to the EigenPodManager
    /// @return existingDepositShares the shares the staker had before any were added
    /// @return addedShares the new shares added to the staker's balance
    function addShares(
        address staker, 
        IStrategy strategy, 
        uint256 shares
    ) external returns (uint256, uint256);

    /// @notice Used by the DelegationManager to convert deposit shares to tokens and send them to a staker
    /// @dev strategy must be beaconChainETH when talking to the EigenPodManager
    /// @dev token is not validated when talking to the EigenPodManager
    function withdrawSharesAsTokens(
        address staker, 
        IStrategy strategy, 
        IERC20 token, uint256 shares) external;

    /// @notice Returns the current shares of `user` in `strategy`
    /// @dev strategy must be beaconChainETH when talking to the EigenPodManager
    /// @dev returns 0 if the user has negative shares
    function stakerDepositShares(address user, 
        IStrategy strategy
    ) external view returns (uint256 depositShares);

    /**
     * @notice Increase the amount of burnable/redistributable shares for a given Strategy. This is called by the DelegationManager
     * when an operator is slashed in EigenLayer.
     * @param operatorSet The operator set to burn shares in.
     * @param slashId The slash id to burn shares in.
     * @param strategy The strategy to burn shares in.
     * @param addedSharesToBurn The amount of added shares to burn.
     * @dev This function is only called by the DelegationManager when an operator is slashed.
     */
    function increaseBurnOrRedistributableShares(
        OperatorSet calldata operatorSet,
        uint256 slashId,
        IStrategy strategy,
        uint256 addedSharesToBurn
    ) external;
}
```

### Burn & Distribution Mechanics

The flow and code-paths for exiting slashed funds from the protocol have changed. Previously, ERC-20 funds flowed out of the protocol through withdrawals or a burn (transfer to `0x00...00316e4`) at a regular cadence. Native ETH was withdrawn or locked in EigenPods permanently during a slash. Following this upgrade, when a slash occurs, funds are exited in two steps. In order to maintain the protocol guarantee that `slashOperator` should never fail, outflow transfers are non-atomic.

When a single slash occurs...
-Similar to the original burning implementation, burnable shares are first increased in `StrategyManger` storage.
-The `StrategyManager` now interfaces directly with the `SlashEscrowFactory` to setup and initiate the escrow.
-A child `SlashEscrow` contract is then deployed; one is created per `slashId`. These contracts are stateless and immutable.

In another call, funds are transferred to the `SlashEscrow` contracts. This escrow is to allow intervention in the case of slashing bugs. ***Funds are no longer exited via functions on the `ShareManager`.*** After the escrow delay period, a call to `clearBurnOrRedistributableShares` is required to delete the `StrategyManager` storage and transfer slashed funds out of the EigenLayer protocol.

 The new flow is illustrated in the below diagram:

```mermaid
sequenceDiagram
    title Redistribution & Burn Flow

    participant AVS as AVS
    participant ALM as Allocation Manager
    participant DM as Delegation Manager
    participant SM as Strategy Manager
    participant SEF as Slash Escrow Factory
    participant STR as Strategy Contract
    participant CL as Slash Escrow Clone
    participant BR as Burn Address or Redistribution Recipient

    Note over AVS,BR: Slashing Initiation
    AVS->>ALM: slashOperator<br>(avs, slashParams)
    ALM-->>DM: *Internal* <br>slashOperatorShares<br>(operator, strategies,<br> prevMaxMags, newMaxMags)
    Note over DM,SM: Share Management
    DM-->>SM: *Internal*<br>increaseBurnOrRedistributableShares<br>(operatorSet, slashId, strategy, addedSharesToBurn)
    Note over SM,SEF: Escrow Setup (Initiates 4-day escrow period)
    SM-->>SEF: *Internal* initiateSlashEscrow()
    SEF-->>CL: *Internal* Deploy child proxy
    
    Note over SM,CL: Transfers underlying tokens to the `SlashEscrow` clone (Second Transaction)
    SM->>CL: clearBurnOrRedistributableShares(operatorSet, slashId)
    SM-->>STR: *Internal*<br>withdraw<br>(slashEscrow, token, underlyingAmount)
    
    Note over SEF,SEF: Wait for max(globalDelay, strategyDelay) to elapse.
    SEF->>SEF: getStrategyEscrowDelay()
    
    Note over SEF,CL: Final Distribution
    SEF->>CL: releaseSlashEscrow()
    SEF-->>CL: *Internal* clearBurnOrRedistributableShares()
    CL-->>BR: *Internal* <br>releaseTokens()
    Note right of BR: Final protocol fund outflow
```

The rationale for this new contract, process, and delay is [outlined in the rationale](./ELIP-006.md#outflow-delay). A global minimum escrow period is introduced that is set by governance. This is a constant value, set at a minimum of four days. Strategies (staked assets) can have larger delays as well; this includes EIGEN, which will use a larger (14 day) delay to accommodate upcoming security features. The delay can be set via governance; Strategy confuguration is reserved for future protocol compatability and need.

Previously, funds would be slashed and exited in a single step, with funds being marked to burn and a continuously running cron job executing fund exits. This was done non-atomically with slashing to maintain the guarantee that a slash should never fail, in the case where a token transfer or some other upstream issue of removing funds from the protocol may fail.

The interface for the `SlashEscrowFactory` is provided below:

```solidity
interface ISlashEscrowFactoryErrors {
    /// @notice Thrown when a caller is not the strategy manager.
    error OnlyStrategyManager();

    /// @notice Thrown when a caller is not the redistribution recipient.
    error OnlyRedistributionRecipient();

    /// @notice Thrown when a escrow is not mature.
    error EscrowNotMature();

    /// @notice Thrown when the escrow delay has not elapsed.
    error EscrowDelayNotElapsed();
}

interface ISlashEscrowFactoryEvents {
    /// @notice Emitted when a escrow is initiated.
    event StartEscrow(OperatorSet operatorSet, uint256 slashId, IStrategy strategy, uint32 startBlock);

    /// @notice Emitted when a escrow is released.
    event EscrowComplete(OperatorSet operatorSet, uint256 slashId, IStrategy strategy, address recipient);

    /// @notice Emitted when a escrow is paused.
    event EscrowPaused(OperatorSet operatorSet, uint256 slashId);

    /// @notice Emitted when a escrow is unpaused.
    event EscrowUnpaused(OperatorSet operatorSet, uint256 slashId);

    /// @notice Emitted when a global escrow delay is set.
    event GlobalEscrowDelaySet(uint32 delay);

    /// @notice Emitted when a escrow delay is set.
    event StrategyEscrowDelaySet(IStrategy strategy, uint32 delay);
}

interface ISlashEscrowFactory is ISlashEscrowFactoryErrors, ISlashEscrowFactoryEvents {
    /**
     * @notice Initializes the initial owner and paused status.
     * @param initialOwner The initial owner of the router.
     * @param initialPausedStatus The initial paused status of the router.
     * @param initialGlobalDelayBlocks The initial global escrow delay.
     */
    function initialize(address initialOwner, uint256 initialPausedStatus, uint32 initialGlobalDelayBlocks) external;

    /**
     * @notice Locks up a escrow.
     * @param operatorSet The operator set whose escrow is being locked up.
     * @param slashId The slash ID of the escrow that is being locked up.
     * @param strategy The strategy that whose underlying tokens are being redistributed.
     */
    function initiateSlashEscrow(
        OperatorSet calldata operatorSet, 
        uint256 slashId, 
        IStrategy strategy
    ) external;

    /**
     * @notice Releases an escrow for all strategies in a slash.
     * @param operatorSet The operator set whose escrow is being released.
     * @param slashId The slash ID of the escrow that is being released.
     * @dev The caller must be the redistribution recipient, unless the redistribution recipient
     * is the default burn address in which case anyone can call.
     * @dev The slash escrow is released once the delay for ALL strategies has elapsed.
     */
    function releaseSlashEscrow(
        OperatorSet calldata operatorSet, 
        uint256 slashId
    ) external;

    /**
     * @notice Releases an escrow for a single strategy in a slash.
     * @param operatorSet The operator set whose escrow is being released.
     * @param slashId The slash ID of the escrow that is being released.
     * @param strategy The strategy whose escrow is being released.
     * @dev The caller must be the redistribution recipient, unless the redistribution recipient
     * is the default burn address in which case anyone can call.
     * @dev The slash escrow is released once the delay for ALL strategies has elapsed.
     */
    function releaseSlashEscrowByStrategy(
        OperatorSet calldata operatorSet,
        uint256 slashId,
        IStrategy strategy
    ) external;

    /**
     * @notice Pauses a escrow.
     * @param operatorSet The operator set whose escrow is being paused.
     * @param slashId The slash ID of the escrow that is being paused.
     */
    function pauseEscrow(OperatorSet calldata operatorSet, uint256 slashId) external;

    /**
     * @notice Unpauses a escrow.
     * @param operatorSet The operator set whose escrow is being unpaused.
     * @param slashId The slash ID of the escrow that is being unpaused.
     */
    function unpauseEscrow(OperatorSet calldata operatorSet, uint256 slashId) external;

    /**
     * @notice Sets the delay for the escrow of a strategies underlying token.
     * @dev The largest of all strategy delays or global delay will be used.
     * @param strategy The strategy whose escrow delay is being set.
     * @param delay The delay for the escrow.
     */
    function setStrategyEscrowDelay(
        IStrategy strategy, 
        uint32 delay
    ) external;

    /**
     * @notice Sets the delay for the escrow of all strategies underlying tokens globally.
     * @param delay The delay for the escrow.
     */
    function setGlobalEscrowDelay(
        uint32 delay
    ) external;

    /**
     * @notice Returns the operator sets that have pending escrows.
     * @return operatorSets The operator sets that have pending escrows.
     */
    function getPendingOperatorSets() external view returns (OperatorSet[] memory operatorSets);

    /**
     * @notice Returns the total number of operator sets with pending escrows.
     * @return The total number of operator sets with pending escrows.
     */
    function getTotalPendingOperatorSets() external view returns (uint256);

    /**
     * @notice Returns whether an operator set has pending escrows.
     * @param operatorSet The operator set whose pending escrows are being queried.
     * @return Whether the operator set has pending escrows.
     */
    function isPendingOperatorSet(
        OperatorSet calldata operatorSet
    ) external view returns (bool);

    /**
     * @notice Returns the pending slash IDs for an operator set.
     * @param operatorSet The operator set whose pending slash IDs are being queried.
     */
    function getPendingSlashIds(
        OperatorSet calldata operatorSet
    ) external view returns (uint256[] memory);

    /**
     * @notice Returns the pending escrows and their release blocks.
     * @return operatorSets The pending operator sets.
     * @return isRedistributing Whether the operator set is redistributing.
     * @return slashIds The pending slash IDs for each operator set. Indexed by operator set.
     * @return completeBlocks The block at which a slashID can be released. Indexed by [operatorSet][slashId]
     */
    function getPendingEscrows()
        external
        view
        returns (
            OperatorSet[] memory operatorSets,
            bool[] memory isRedistributing,
            uint256[][] memory slashIds,
            uint32[][] memory completeBlocks
        );

    /**
     * @notice Returns the total number of slash IDs for an operator set.
     * @param operatorSet The operator set whose total slash IDs are being queried.
     * @return The total number of slash IDs for the operator set.
     */
    function getTotalPendingSlashIds(
        OperatorSet calldata operatorSet
    ) external view returns (uint256);

    /**
     * @notice Returns whether a slash ID is pending for an operator set.
     * @param operatorSet The operator set whose pending slash IDs are being queried.
     * @param slashId The slash ID of the slash that is being queried.
     * @return Whether the slash ID is pending for the operator set.
     */
    function isPendingSlashId(
        OperatorSet calldata operatorSet, 
        uint256 slashId
    ) external view returns (bool);

    /**
     * @notice Returns the pending strategies for a slash ID for an operator set.
     * @dev This is a variant that returns the pending strategies for a slash ID for an operator set.
     * @param operatorSet The operator set whose pending strategies are being queried.
     * @param slashId The slash ID of the strategies that are being queried.
     * @return strategies The strategies that are pending strategies.
     */
    function getPendingStrategiesForSlashId(
        OperatorSet calldata operatorSet,
        uint256 slashId
    ) external view returns (IStrategy[] memory strategies);

    /**
     * @notice Returns all pending strategies for all slash IDs for an operator set.
     * @dev This is a variant that returns all pending strategies for all slash IDs for an operator set.
     * @param operatorSet The operator set whose pending strategies are being queried.
     * @return strategies The strategies that are pending strategies.
     */
    function getPendingStrategiesForSlashIds(
        OperatorSet calldata operatorSet
    ) external view returns (IStrategy[][] memory strategies);

    /**
     * @notice Returns the number of pending strategies for a slash ID for an operator set.
     * @param operatorSet The operator set whose pending strategies are being queried.
     * @param slashId The slash ID of the strategies that are being queried.
     * @return The number of pending strategies.
     */
    function getTotalPendingStrategiesForSlashId(
        OperatorSet calldata operatorSet,
        uint256 slashId
    ) external view returns (uint256);

    /**
     * @notice Returns the pending underlying amount for a strategy for an operator set and slash ID.
     * @param operatorSet The operator set whose pending underlying amount is being queried.
     * @param slashId The slash ID of the escrow that is being queried.
     * @param strategy The strategy whose pending underlying amount is being queried.
     * @return The pending underlying amount.
     */
    function getPendingUnderlyingAmountForStrategy(
        OperatorSet calldata operatorSet,
        uint256 slashId,
        IStrategy strategy
    ) external view returns (uint256);

    /**
     * @notice Returns the paused status of a escrow.
     * @param operatorSet The operator set whose escrow is being queried.
     * @param slashId The slash ID of the escrow that is being queried.
     * @return The paused status of the escrow.
     */
    function isEscrowPaused(
        OperatorSet calldata operatorSet, 
        uint256 slashId
    ) external view returns (bool);

    /**
     * @notice Returns the start block for a slash ID.
     * @param operatorSet The operator set whose start block is being queried.
     * @param slashId The slash ID of the start block that is being queried.
     * @return The start block.
     */
    function getEscrowStartBlock(OperatorSet calldata operatorSet, uint256 slashId) external view returns (uint256);

    /**
     * @notice Returns the block at which the escrow can be released.
     * @param operatorSet The operator set whose start block is being queried.
     * @param slashId The slash ID of the start block that is being queried.
     * @return The block at which the escrow can be released.
     */
    function getEscrowCompleteBlock(OperatorSet calldata operatorSet, uint256 slashId) external view returns (uint32);

    /**
     * @notice Returns the escrow delay for a strategy.
     * @param strategy The strategy whose escrow delay is being queried.
     * @return The escrow delay.
     */
    function getStrategyEscrowDelay(
        IStrategy strategy
    ) external view returns (uint32);

    /**
     * @notice Returns the global escrow delay.
     * @return The global escrow delay.
     */
    function getGlobalEscrowDelay() external view returns (uint32);

    /**
     * @notice Returns the salt for a slash escrow.
     * @param operatorSet The operator set whose slash escrow is being queried.
     * @param slashId The slash ID of the slash escrow that is being queried.
     * @return The salt for the slash escrow.
     */
    function computeSlashEscrowSalt(
        OperatorSet calldata operatorSet,
        uint256 slashId
    ) external pure returns (bytes32);

    /**
     * @notice Returns whether a slash escrow is deployed or not.
     * @param operatorSet The operator set whose slash escrow is being queried.
     * @param slashId The slash ID of the slash escrow that is being queried.
     * @return Whether the slash escrow is deployed.
     */
    function isDeployedSlashEscrow(OperatorSet calldata operatorSet, uint256 slashId) external view returns (bool);

    /**
     * @notice Returns whether a slash escrow is deployed.
     * @param slashEscrow The slash escrow that is being queried.
     * @return Whether the slash escrow is deployed.
     */
    function isDeployedSlashEscrow(
        ISlashEscrow slashEscrow
    ) external view returns (bool);

    /**
     * @notice Returns the slash escrow for an operator set and slash ID.
     * @param operatorSet The operator set whose slash escrow is being queried.
     * @param slashId The slash ID of the slash escrow that is being queried.
     * @return The slash escrow.
     */
    function getSlashEscrow(OperatorSet calldata operatorSet, uint256 slashId) external view returns (ISlashEscrow);
}
```

This contract carves out roles and functions for pausing (and unpausing) in-flight slash outflows.

> 🏛️   **Note**
>
> The `pause` functionality in this contract is gated to the `Pauser multi-sig`, as [outlined in Eigen Foundation governance](https://docs.eigenfoundation.org/protocol-governance/technical-architecture). This role can *only* pause (or later unpause) outflows if there are still pending blocks (time) between slash initiation and the end of the escrow period. During a pause, governance and social consensus is the anticipated adjudication mechanism to determine if a rescue is needed for funds due to an [invalid bug](./ELIP-006.md#security-considerations). If funds are to be rescued, the `Community multi-sig` has the authority to upgrade the contract and return the funds to the protocol. The rationale for this is [outlined further below](./ELIP-006.md#governance-design).

For each slash, a child `SlashEscrow` contract is created from the factory. It is a very simple contract intended to hold tokens for the delay period. One contract is instantiated per slash and has a relatively small impact on Ethereum state. The `SlashEscrowFactory` controls the ability to call `releaseTokens`, meaning a pause on the `SlashEscrowFactory` by the `pauser` will freeze calls to the child contracts for certain `slashId`s. There additionally is a global `PAUSEALL` for all outflows, in the case of a widely abused slashing bug. Below is the interface:

```solidity
interface ISlashEscrow {
    /// @notice Thrown when the provided deployment parameters do not create this contract's address.
    error InvalidDeploymentParameters();

    /// @notice Thrown when the caller is not the slash escrow factory.
    error OnlySlashEscrowFactory();

    /// @notice Burns or redistributes the underlying tokens of the strategies.
    /// @param slashEscrowFactory The factory contract that created the slash escrow.
    /// @param slashEscrowImplementation The implementation contract that was used to create the slash escrow.
    /// @param operatorSet The operator set that was used to create the slash escrow.
    /// @param slashId The slash ID that was used to create the slash escrow.
    /// @param recipient The recipient of the underlying tokens.
    /// @param strategy The strategy that was used to create the slash escrow.
    function releaseTokens(
        ISlashEscrowFactory slashEscrowFactory,
        ISlashEscrow slashEscrowImplementation,
        OperatorSet calldata operatorSet,
        uint256 slashId,
        address recipient,
        IStrategy strategy
    ) external;

    /// @notice Verifies the deployment parameters of the slash escrow.
    /// @dev Validates that the provided parameters deterministically generate this contract's address using CREATE2.
    /// - Uses ClonesUpgradeable.predictDeterministicAddress() to compute the expected address from the parameters.
    /// - Compares the computed address against this contract's address to validate parameter integrity.
    /// - Provides a stateless validation mechanism for releaseTokens() inputs.
    /// - Security relies on the cryptographic properties of CREATE2 address derivation.
    /// - Attack vector would require finding a hash collision in the CREATE2 address computation.
    /// @param slashEscrowFactory The factory contract that created the slash escrow.
    /// @param slashEscrowImplementation The implementation contract that was used to create the slash escrow.
    /// @param operatorSet The operator set that was used to create the slash escrow.
    /// @param slashId The slash ID that was used to create the slash escrow.
    /// @return True if the provided parameters create this contract's address, false otherwise.
    function verifyDeploymentParameters(
        ISlashEscrowFactory slashEscrowFactory,
        ISlashEscrow slashEscrowImplementation,
        OperatorSet calldata operatorSet,
        uint256 slashId
    ) external view returns (bool);
}
```

The `StrategyManager` interface has numerous changes; the majority of the functionality has been moved to the `SlashEscrowFactory`, including the previously available `burnShares` functionality.

```solidity
interface IStrategyManager {
    /// @dev Thrown when attempting to add a strategy that is already in the operator set's burn or redistributable shares.
    error StrategyAlreadyInSlash();

    /// @notice Emitted when an operator is slashed and shares to be burned or redistributed are increased
    event BurnOrRedistributableSharesIncreased(
        OperatorSet operatorSet, uint256 slashId, IStrategy strategy, uint256 shares
    );

    /// @notice Emitted when shares marked for burning or redistribution are decreased and transferred to the `SlashEscrow`
    event BurnOrRedistributableSharesDecreased(
        OperatorSet operatorSet, uint256 slashId, IStrategy strategy, uint256 shares
    );

    /// NOTE: We are keeping the original `burnShares` fn so that legacy burns can still be completed.

    /**
     * @notice Removes burned shares from storage and transfers the underlying tokens for the slashId to the slash escrow.
     * @param operatorSet The operator set to burn shares in.
     * @param slashId The slash ID to burn shares in.
     */
    function clearBurnOrRedistributableShares(OperatorSet calldata operatorSet, uint256 slashId) external;

    /**
     * @notice Removes a single strategy's shares from storage and transfers the underlying tokens for the slashId to the slash escrow.
     * @param operatorSet The operator set to burn shares in.
     * @param slashId The slash ID to burn shares in.
     * @param strategy The strategy to burn shares in.
     * @return The amount of shares that were burned.
     */
    function clearBurnOrRedistributableSharesByStrategy(
        OperatorSet calldata operatorSet,
        uint256 slashId,
        IStrategy strategy
    ) external returns (uint256);

    /**
     * @notice Returns the strategies and shares that have NOT been sent to escrow for a given slashId.
     * @param operatorSet The operator set to burn or redistribute shares in.
     * @param slashId The slash ID to burn or redistribute shares in.
     * @return The strategies and shares for the given slashId.
     */
    function getBurnOrRedistributableShares(
        OperatorSet calldata operatorSet,
        uint256 slashId
    ) external view returns (IStrategy[] memory, uint256[] memory);

    /**
     * @notice Returns the shares for a given strategy for a given slashId.
     * @param operatorSet The operator set to burn or redistribute shares in.
     * @param slashId The slash ID to burn or redistribute shares in.
     * @param strategy The strategy to get the shares for.
     * @return The shares for the given strategy for the given slashId.
     * @dev This function will return revert if the shares have already been sent to escrow.
     */
    function getBurnOrRedistributableShares(
        OperatorSet calldata operatorSet,
        uint256 slashId,
        IStrategy strategy
    ) external view returns (uint256);

    /**
     * @notice Returns the number of strategies that have NOT been sent to escrow for a given slashId.
     * @param operatorSet The operator set to burn or redistribute shares in.
     * @param slashId The slash ID to burn or redistribute shares in.
     * @return The number of strategies for the given slashId.
     */
    function getBurnOrRedistributableCount(
        OperatorSet calldata operatorSet,
        uint256 slashId
    ) external view returns (uint256);
}
```

The `EigenPodManager` interfaces are updated to avoid breaking changes in the internal flows.

To recap the new or modified functionality:

- New Operator Set types enable a fixed `redistributionRecipient`.
- Additional meta-data has been added to each slash to help AVSs build programmatic redistribution.
- The `StrategyManager` has a modified flow to accommodate redistribution versus burning.

# Rationale

## Outflow Delay

There are many interactions between the normal code-paths of the protocol and redistribution that are handled carefully with new security mechanisms. With redistributable slashing, the protocol implements what amounts to a new withdrawal path for funds. EigenLayer provides guarantees around withdrawals via the 14-day stake guarantee window. If the protocol provided the same delay, a two week period would detrimentally impact the usability of redistributable slashing, including programmatic fund redistribution use-cases and insurance products.

To this end, this proposal suggests a default slash escrow period amounting to four day, in blocks. This is enough time to ensure oversight by the `Pauser multi-sig`, accounting for coordination (or extraneous) delays and on-chain censorship resistance. This delay exists *only* to allow the `Pauser multi-sig` to execute a pause on slashes that are deemed implementation bugs (e.g. a slash value is greater than the Operator Sets allocated stake). 

The delay exists per Strategy in the protocol. EIGEN will have a larger delay. This delay can be modified via governance and its configuration is reserved for future protocol use, need, and compatibility. 

## Governance Design

The governance design for the `SlashEscrowFactory` in redistributable slashing uses existing protocol governance structures to minimize complexity and oversight. The `Pauser multi-sig` is chosen as the primary control point for pausing buggy slashes, as this aligns with its existing role of pausing the protocol in case of critical bugs. The `Pauser multi-sig` already has established governance controls through Eigen Foundation and maintains the ability to quickly respond to potential implementation issues.

For the rescue of any funds in extreme cases, the `Community multi-sig` is designated as the authority, with a near complete degree of control over contract upgrades in case of emergency. This choice stems from the `Community multi-sig`'s existing position as the social consensus oversight body for the EigenLayer protocol, with its high thresholds (and required signatures) for action to avoid capture. By utilizing these existing governance structures rather than creating new ones, the design maintains simplicity while preserving the strong security guarantees already present in the protocol.

Governance will *only* interface with the `SlashEscrow` contracts in the case of a "catastrophic bug" as defined in the [Security Considerations](./ELIP-006.md#security-considerations). The governance process will not intervene in individual AVS slashings when they are the product of faulty implementations, bugs, or lost keys. The EigenLayer middleware contracts provide [templates for vetoable slashing designs](https://github.com/Layr-Labs/eigenlayer-middleware/blob/dev/src/slashers/VetoableSlasher.sol). The [documentation contains guidance on how to design controls](https://docs.eigenlayer.xyz/developers/HowTo/build/slashing/slashing-veto-committee-design) for these cases for AVS builders.

## Redistributing AVS Guarantees & Legibility  

Redistributable slashing is a modest upgrade in code, but has broad ramifications to the incentives and guarantees of the EigenLayer system. The majority of the design in this proposal is to ensure as much safety as possible for Stakers as redistribution creates a direct increase in the incentive to slash Operators by AVSs.

As funds are released from the protocol to an address specified by the AVS, it is important that Stakers have the right legibility to understand the risk of allocating to any Operators running an AVS with redistributable slashing enabled. This is the primary reason for the `redistributionRecipient` being an immutable address that must be set at the instantiation of the Operator Set. This provides a few guarantees:

- The AVS cannot change this address. While they may use an upstream proxy or pass-through contract, the immutability of this address in EigenLayer means an AVS can layer additional guarantees by guarding the upgradability of the upstream contract via controls such as governance, timelocks, immutability, etc.
- The capability to redistribute cannot be modified. An Operator Set must be redistributable at its creation. As a result, the protocol can make guarantees to Stakers and Operators over the lifetime of that Operator Set. A standard Operator Set cannot suddenly redistribute. And one that redistributes cannot remove that property.
- Legibility on the front-end and in metadata. By forcing immutability of the above properties, EigenLayer can better differentiate on its application and in chain meta-data where redistributable slashing is enabled for its users (and any implied or associated risk and reward tradeoffs of interaction).

These guarantees provide a hedge to increased slashing risk and changed incentives.

## Native ETH Redistribution

Native ETH will not be included in the scope of this proposal. Its exclusion is primarily tied to mechanics of the beacon chain. With redistributable slashing on the ETH strategy, exiting validators from the beacon chain to compensate the `redistributionRecipient` is required. We cannot make guarantees that the effective balance is high enough or that a partial withdrawal covers the balance of a slash without dipping validator balance below the 32 ETH minimum.

Consistently exiting Ethereum validators creates some problems for users and the network:

- Exits require waiting in the validator exit queue, which can range in length from hours to weeks.
- Funds have to wait to be swept before they will arrive at the specified withdrawal addresses in EigenPods. AVSs will not have access to any funds until this queue is cleared, which can impact programmatic designs.
- Impacts Ethereum network security, in some adversarial cases.

Together, these are enough to forgo this scope in the initial implementation of redistributable slashing. With the increase in the [max effective balance of validators](https://eips.ethereum.org/EIPS/eip-7251) enabled in Ethereum's Pectra upgrade, there are possible designs that can alleviate the above concerns (like partial withdrawals above the minimum required balance of 32 ETH). These are being actively explored as part of improvements to EigenPods, including exploration of forced withdrawal mechanisms with regard to slashing on EigenLayer and ETH validators.

# Security Considerations

The original slashing design, launched alongside [ELIP-002](./ELIP-002.md), did not account for the protocol's ability to handle a catastrophic slashing bug. A catastrophic slashing bug is considered to be one where an AVS (or malicious party) can...

- slash more stake than is allocated to a given Operator in an Operator Set (or more than the sum of the set, up to the entire protocol TVL),
- slash the unique stake of an Operator that is not registered to an AVS's Operator Set,
- trigger a slash without being a registered AVS.

These cases generally represent where governance will intervene to prevent a malicious party from slashing too much or slashing someone they shouldn't. These interventions are especially important with redistributable slashing, as the feature, coupled with these bugs, could enable theft of all assets from EigenLayer. These cases are the only cases in which governance will interface with the escrow contracts. Governance will not intervene in other cases, as stated in the [governance design](./ELIP-006.md#governance-design).

As always this update requires very careful management of keys, as a compromised AVS key can lead to theft of user funds when redistributable slashing is in use causing reputational and monetary harm.

# Impact Summary

## AVSs & Service Builders

AVSs will gain access to a powerful new primitive in redistributable slashing upon which to develop use-cases. This comes with an even heavier emphasis on proper key management and op-sec requirements. An attacker that gains access to AVS keys on the `slasher` and `redistributionRecipient` can drain the entirety of Operator and Staker allocated stake for a given Operator Set. This will have heavy repercussions on the AVSs reputation and continued trust.

Because redistribution may allow AVSs to benefit from a theft related to slashing, additional design care must be taken to consider the incentives of all parties that can interact with it. When handled appropriately, AVSs will have new use-case opportunities but must consider the higher risk and slash incentive for those running their code.

## Operators

Operators must similarly ensure focus is given to key management and op-sec when running *any* redistributable AVS. A compromise in an Operator key could cause a malicious actor to register for a malicious AVS, and slash and redistribute allocated Staker funds to some address. Operators would suffer potentially irreparable reputational damage and distrust from Stakers.

Operators should be aware that meta-data will identify them as `Redistributable` when participating in any redistributing Operator Sets. This is to aid in Staker risk legibility. Operators may wish to avoid this change in their presented profile when seeking to attract Stake (or may chose to engage protocols of various risks with higher rewards). Running an Operator Set with redistributable slashing will always remain opt-in for Operators.

## Stakers

This proposal has the largest impact to the risk, reward, and incentive model for Stakers. In general, there is a larger incentive to slash user funds when redistribution is enabled. Stakers should carefully consider the protocols that their delegated Operators are running, and consider the risk and reward trade-offs. Redistributable Operator Sets may offer higher rewards, but these should be considered against the increased slashing risks.

Additionally, Stakers are potentially at risk from malicious AVSs and malicious Operator. If the AVSs governance or its slashing functionality is corrupted, an attacker may be able to drain Operator-delegated funds. If an Operator itself is compromised, it may stand-up its own AVS to steal user funds. Stakers should carefully consider the reputation and legitimacy of Operators when making delegations. These attack scenarios are outlined in more detail [here](https://forum.eigenlayer.xyz/t/risks-of-an-in-protocol-redistribution-design/14458).

# Action Plan

This proposal will move to a testing phase on testnet with the above interfaces and approaches. EigenLayer is actively soliciting community feedback on the feature before moving it to Mainnet. As this is primarily an opt-in feature for Operators (and, by proxy Stakers).

1. Implementation and public comment period (now)
2. Public testing period and iteration of the code-base, with a public audit and associated patches (next)
3. Evaluation by Protocol Council and release to Mainnet, following acceptance (later)

This proposal is intended to be completed within a two to three month period.

# References & Relevant Discussions

- [Risks of an In-Protocol Redistribution Design](https://forum.eigenlayer.xyz/t/risks-of-an-in-protocol-redistribution-design/14458)
- [Out-of-Protocol Redistribution](https://forum.eigenlayer.xyz/t/redistribution-vault-with-an-overloaded-erc20/14434)
