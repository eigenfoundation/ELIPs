| Author(s) | Created | Status | References | Discussions |
| :---- | :---- | :---- | :---- | :---- |
| Matt Nelson, Alex Wade, Yash Patil, Matt Curtis | 2025/02/10 | Testing | [Repository](https://github.com/Layr-Labs/eigenlayer-contracts/tree/dev/src/contracts/pods) | [Discussion Forum](https://forum.eigenlayer.xyz/t/elip-004-slashing-aware-eigenpods/14420) |

# ELIP-004: Slashing-Aware EigenPods

--- 

# Executive Summary

The slashing upgrade is a key milestone on the step to a complete EigenLayer. As part of the larger slashing release, EigenPods are upgrading to account for a number of under-the-hood changes that implement the Unique Stake model, Operator Sets, and, as a product of those two, AVS slashing. More about slashing and the mechanics enabling it can be found in [ELIP-002](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-002.md).

This proposal introduces targeted changes to the EigenPod contracts to make them slashing-aware and Pectra compatible. Upgrading Pods keeps natively restaked ETH functionally on-par with ERC-20s when used as slashable stake. The scope of this upgrade introduces breaking changes to EigenPods and updates several interfaces. Broadly, this upgrade brings:

* Slashing-aware EigenPod contracts that enable proper accounting for Native ETH slashings,  
* The introduction of the Beacon Chain Slashing factor that propagates external Ethereum slashings correctly across EigenLayer,  
* A new checkpoint structure,  
* A *pause on checkpointing* around the time of the associated upgrade,  
* A modified withdrawal process for EigenPods to apply Ethereum and/or AVS slashings to assets withdrawn from EigenPods. 

Together, these upgrades properly account for the interactions between Ethereum *and* AVS slashings and ensure a smooth transition during the upgrade and withdrawal process. 

In addition, the upcoming Pectra hard fork of Ethereum includes breaking changes to the proof generation library and adds execution-layer triggered validator withdrawals. This proposal includes upgrades that ensure that EigenPods remain functional after the hard fork.

# Motivation

Slashing has required many changes to the accounting system used in EigenLayer. In the cases of *reductions* of stake for Operators and Stakers, slashing accounting must correctly propagate to stake in the various stages of its lifecycle, like a pending withdrawal. It also needs to be consistent across the various places the data lives and is displayed. 

While ERC-20s are very flexible by design, natively restaked ETH brings some different challenges, as it lives on the consensus layer of Ethereum and is not immediately accessible to the EigenLayer execution environment. Additionally, the consensus layer may also slash this ETH, and EigenPods must account for this reduction in stake on EigenLayer, even when that stake is allocated to an AVS’s slashable Operator Set.

Under the hood, EigenLayer handles the native ETH Strategy differently than the rest of the ERC-20 Strategies. As Strategies are tightly coupled with the slashing design proposed in ELIP-002, corner cases emerged in the EigenPod design. These necessitate upgrades to EigenPods that will make them forward-compatible with slashing, set the contracts up for further improvements following the Pectra upgrade on L1, and ensure necessary security properties when using native ETH as Unique Stake. All assets are crucial to securing AVSs, and this proposal ensures they are all equally functional. 

# Features & Specifications

## Goals

Native ETH restaking on EigenLayer is a valuable security dimension for AVSs, with restaked ETH representing a collective \~4% of the staked beacon chain ETH supply. This upgrade seeks to:

* **Keep EigenLayer parity for native ETH and ERC-20s**, to ensure AVSs have multiple asset classes to use as Unique Stake,  
* **Introduce the `beaconChainSlashingFactor`** for EigenPods to be slashing-aware for accounting and upon withdrawal,  
* Enable forward-compatibility with **post-Pectra slashing**. 

The above goals are achieved via updates to contracts and interfaces in the `EigenPod`, `EigenPodManager`, and the `DelegationManager`. 

## EigenPod Accounting Changes

This proposal changes the beacon chain accounting system to treat natively restaked ETH the same as other tokens for more appropriate handling during a slashing. It is recommended that you understand [Slashing Magnitudes and Unique Stake as described in ELIP-002](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-002.md#unique-stake-allocation--deallocation) before you continue. These `EigenPod` changes are included in the broader slashing release and are not orthogonal.  

Today, any EigenPod balance updates are accounted for using the [checkpoint system](https://docs.eigenlayer.xyz/eigenlayer/restaking-guides/restaking-user-guide/native-restaking/). Checkpoints allow the balance changes of an Ethereum validator to be properly accounted for across Stakers and Operators. The balance of an Ethereum validator is constantly changing, either via an increase from Ethereum rewards distributions, or from a decrease when a validator incurs penalties or a slashing. You can learn about Ethereum penalties and rewards [here](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/rewards-and-penalties/). 

Checkpoints accumulate these changes and node operators check them in periodically to update the Pod’s balance on EigenLayer. This also updates corresponding stake values, which are propagated across delegated Operators and any of their Native ETH allocations. In rare cases, like a beacon chain slashing, others that do not control the Pod may checkpoint a Pod’s balance to ensure proper accounting across EigenLayer. 

EigenPods are unique in that they have to account for slashings that are internal to EigenLayer, i.e. initiated by an AVS, *and* beacon chain validator slashing. For each individual `EigenPod`, the `EigenPodManager` tracks a new `beaconChainSlashingFactor`. This integer is a monotonically decreasing value that represents an accumulation of penalties for each Native Restaker’s underlying Beacon Chain stake. It is represented in `wads` and is updated with every `EigenPod` checkpoint. 

When an EigenPod's checkpoint is completed, a negative balance delta no longer reduces stake accounting directly. Instead, the proportional change in the EigenPod owner’s new and previous balances is accumulated in the `beaconChainSlashingFactor`. This is now used to keep track of how many assets the Pod's shares actually correspond to ([link to code](https://github.com/Layr-Labs/eigenlayer-contracts/blob/slashing-magnitudes/src/contracts/pods/EigenPodManager.sol#L104-L131)):

```solidity
/**
 * @notice Returns the historical sum of proportional balance decreases a pod owner has experienced when
 * updating their pod's balance.
 */
function beaconChainSlashingFactor(
  address staker
) external view returns (uint64);
```

The `beaconChainSlashingFactor` is initialized at `INITAL_BEACON_CHAIN_SLASHING_FACTOR` for each `EigenPod` contract. This factor is treated similarly to the Operator’s `INITIAL_TOTAL_MAGNITUDE` of a given ERC-20 Strategy. The beaconChainSlashingFactor is reduced (e.g. from 1.0 to 0.8) with any decreases in the ETH of an EigenPod and attached validators proven in completed checkpoints. This decrease in the beaconChainSlashingFactor is proportional to the decrease in the proven aggregate balance (a 20% reduction in the previous example).

A "balance decrease" specifically means the sum of the Pod’s native ETH balance *plus* the sum of its validator balances is *strictly less* than it was the last time a checkpoint or withdrawal from the EigenPod was completed. The following formula gives Pod Balance: 

```math
\Delta \text{podShares} = \Delta \text{podBalance} + \sum_{i=0}^{n} \Delta \text{beaconBalance}[ v(i) ]
```


In practice, the `beaconChainSlashingFactor` should only be impacted if any of the validators suffer from inactivity penalties or are slashed on the beacon chain. Any functions using the `beaconChainSlashingFactor` are atomic. They will properly handle any accounting of both full and partial withdrawals *on the beacon chain*. As the `beaconBalance` in the above formula will decrease as a function of any validator exit, the increase in the `podBalance` will mirror this change, keeping the slashing factor consistent if no penalties occurred. 

While native Restakers are primarily impacted by these changes, there are some caveats Operators and AVSs should be aware of:

* Native ETH is handled the same as other Strategies when used to secure an Operator Set,  
* Operators should treat allocations of Native ETH the same as other allocations when considering their use,  
* AVSs may choose to value the security of Native ETH differently than other assets, but it will act the same as any other Unique Stake in an Operator Set. However, external risk factors, like a validator slashing, *may* reduce available Unique Stake. In certain ways, this risk could be thought of as analogous to a LST de-peg. 

By using the `beaconChainSlashingFactor`, balance changes can be easily and correctly propagated across the protocol. It is incorporated into all view functions used by both offchain and onchain entities to determine a Staker's actual stake in the `DelegationManager`. The beaconChainSlashingFactor applies slashing penalties to all of an Operator's shares in the Native ETH strategy, including uncompleted withdrawals. 

## EigenPod Withdrawals

Withdrawals will remain largely the same when functionality is presented to node operators. Upon withdrawal by a Staker, the beaconChainSlashingFactor is applied to Native ETH. If an AVS has slashed a delegated Operator, any changes in the Strategy Magnitudes will be applied to withdrawn assets. [ELIP-002 breaks the process down in greater detail.](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-002.md#deposits-delegations--withdrawals) 

Below is the new withdrawal interface. Stake introspection is handled with some modifications below. The function `getWithdrawableShares` has been moved from the `EigenPodManager` to the `DelegationManager`:

```solidity
/**
 * @notice Given a staker and a set of strategies, return the shares they can queue for withdrawal and the
 * corresponding depositShares.
 * This value depends on which operator the staker is delegated to.
 * The shares amount returned is the actual amount of Strategy shares the staker would receive (subject
 * to each strategy's underlying shares to token ratio).
 */
function getWithdrawableShares(
  address staker,
  IStrategy[] memory strategies
) external view returns (uint256[] memory withdrawableShares, uint256[] memory depositShares);

 /**
     * @notice Returns the number of shares in storage for a staker and all their strategies
     */
    function getDepositedShares(
        address staker
    ) external view returns (IStrategy[] memory, uint256[] memory);
```

This function should be considered the source of truth when reviewing the amount of withdrawable shares a Pod has available. The returned `uint256[] memory depositShares` in this `getWithdrawableShares` function also returns the same `depositShares`. `podOwnerShares` introspection in the `EigenPodManager` is deprecated. `getDepositedShares` will return the raw amount a Staker has deposited, not applying slashings and penalties.

### Queueing Withdrawals

The event when a withdrawal is queued is now named `SlashingWithdrawalQueued` . The updated Withdrawal event and struct are below:

```solidity
/**
 * @notice Emitted when a new withdrawal is queued.
 * @param withdrawalRoot Is the hash of the `withdrawal`.
 * @param withdrawal Is the withdrawal itself.
 * @param sharesToWithdraw Is an array of the expected shares that were queued for withdrawal corresponding to the strategies in the `withdrawal`.
 */
event SlashingWithdrawalQueued(
	bytes32 withdrawalRoot, 
	Withdrawal withdrawal, 
	uint256[] sharesToWithdraw
);

/// @notice Emitted when a queued withdrawal is completed
event SlashingWithdrawalCompleted(bytes32 withdrawalRoot);

/**
 * Struct type used to specify an existing queued withdrawal. Rather than storing the entire struct, only a hash is stored.
 * In functions that operate on existing queued withdrawals -- e.g. completeQueuedWithdrawal`, the data is resubmitted and the hash of the submitted
 * data is computed by `calculateWithdrawalRoot` and checked against the stored hash in order to confirm the integrity of the submitted data.
 */
struct Withdrawal {
    // The address that originated the Withdrawal
    address staker;
    // The address that the staker was delegated to at the time that the Withdrawal was created
    address delegatedTo;
    // The address that can complete the Withdrawal + will receive funds when completing the withdrawal
    address withdrawer;
    // Nonce used to guarantee that otherwise identical withdrawals have unique hashes
    uint256 nonce;
    // Blocknumber when the Withdrawal was created.
    uint32 startBlock;
    // Array of strategies that the Withdrawal contains
    IStrategy[] strategies;
    // Array containing the amount of staker's scaledShares for withdrawal in each Strategy in the `strategies` array
    // Note that these scaledShares need to be multiplied by the operator's maxMagnitude and beaconChainScalingFactor at completion to include
    // slashing occurring during the queue withdrawal delay. This is because scaledShares = sharesToWithdraw / (maxMagnitude * beaconChainScalingFactor)
    // at queue time. beaconChainScalingFactor is simply equal to 1 if the strategy is not the beaconChainStrategy.
    // To account for slashing, we later multiply scaledShares * maxMagnitude * beaconChainScalingFactor at the earliest possible completion time
    // to get the withdrawn shares after applying slashing during the delay period.
    uint256[] scaledShares;
}
```

### Completing Withdrawals

Queuing a withdrawal emits an event with a `withdrawal` struct that *currently* must be indexed and passed into the `completeQueuedWithdrawal` function after the `WITHDRAWAL_DELAY` period. The frontend can be used to supply this calldata but is not strictly mandatory. With the slashing update:

1. The event emitted when queuing a withdrawal changes (above).  
2. The `completeQueuedWithdrawal` parameters change (to remove an unused parameter).

The new complete withdrawal interface is below. Specifically, we are removing the unused `uint256` parameter (`middlewareTimesIndex`) from both complete methods. By using the `getQueuedWithdrawals` function to introspect withdrawals, one can complete withdrawals just from RPC calls, **eliminating the need to run an indexer to complete withdrawals.**

```solidity
// One withdrawal, which is obtained by indexing the withdrawal struct or calling `getQueuedWithdrawals`. 
function completeQueuedWithdrawal(
      Withdrawal calldata withdrawal,
      IERC20[] calldata tokens,
      bool receiveAsTokens
)

// Many withdrawals, which are obtained by indexing the withdrawals when queued or calling `getQueuedWithdrawals`. 
function completeQueuedWithdrawals(
      Withdrawal[] calldata withdrawals,
      IERC20[][] calldata tokens,
      bool[] calldata receiveAsTokens
)

// Introspect currently queued withdrawals. Only returns withdrawals queued, but not completed, post slashing upgrade. 
function getQueuedWithdrawals(
      address staker
) external view returns (Withdrawal[] memory withdrawals, uint256[][] memory shares)
```

## Checkpoints 

The introduction of Beacon Chain Slashing requires a new checkpoint structure and definition. In the struct below, `prevBeaconBalanceGwei` was added in order to calculate the proportional differences needed to update the `beaconChainScalingFactor`. Additionally, `balanceDeltasGwei` has also been reduced to `int64`.   

Below is the new checkpoint structure in the `EigenPod` contract:

```solidity
struct Checkpoint {
  bytes32 beaconBlockRoot;
  uint24 proofsRemaining;
  uint64 podBalanceGwei;
  int64 balanceDeltasGwei;
  uint64 prevBeaconBalanceGwei;
}
```

**This change is not backward compatible with existing uncompleted checkpoints.** **Checkpoint creation will be paused before the slashing upgrade.** All outstanding checkpoints will be completed before the upgrade. After the upgrade occurs, checkpoint creation will be resumed. Timelines will be communicated carefully with the community and with ample time. The procedure will look like the following, and begin ~6 hours before each network forks on L1: 

1. Pause new checkpoint creation & credential proof generation
2. Upgrade the core contracts (on each network)
3. Set the Pectra fork timestamp after it has elapsed. This must be done in our contracts in the next available block we can include a transaction
4. Unpause checkpoint creation & credential proof generation

All EigenPods with negative balances must complete their withdrawals to ensure non-negative balances before processing new checkpoints. This will impact a very small number of EigenPods, as this state can only currently occur if validators are slashed/penalized after a withdrawal is queued. To check if you need to complete a withdrawal prior to the release of the upgrade, query `EigenPodManager.podOwnerDepositShares`. If this query returns a negative value, you will need to complete the withdrawal. After the upgrade, any EigenPods with negative balance will need to complete withdrawals with the flag `receiveAsTokens == false` to resolve the negative balance. This will leave any remaining ETH in the EigenPod and re-enable checkpointing, which must be completed prior to queuing a new withdrawal.

## Post-Pectra Slashing 

### Delivered with the Slashing Release…

Pectra causes a breaking change to the [EigenPod Proof Generation Library](https://github.com/Layr-Labs/eigenpod-proofs-generation), necessitating an update to the on-chain proof processing and the off-chain library. Existing validators (`0x01`) and validators with the new `0x02` [compounding withdrawal credentials](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7804.md#abstract) will be supported in the library release when generating proofs. 

Prior to the Pectra hard fork, starting checkpoint and credential proofs on EigenPods will be paused. During the paused period, checkpoints that were started are still completable. Post hard-fork, the EigenPods will be upgraded, after which credential and checkpoint proofs will become unpaused. 

The upgrade to support this breaking change is dependent on the timing of the Pectra hard fork. Libraries will be updated in advance of the various forks on test and main networks. The timestamp at which these changes take effect will be incorporated into the upgrade once released by the Ethereum Foundation. 

Before the Pectra upgrade, slashed beacon chain Ether will be indefinitely locked in the `EigenPod` contracts. There are no plans to make this recoverable. 

### Following in a Separate, Pending Release…

Pectra provides a key piece of functionality within Ethereum that will enable more robust handling of slashing in EigenPods when it is released. Namely, EigenPods will be able to make use of [EIP-7002](https://eips.ethereum.org/EIPS/eip-7002), Execution layer triggerable withdrawals. **Following Ethereum’s Pectra hardfork *and a separate EigenLayer protocol upgrade***, the mechanics of withdrawals will change. Natively restaked ETH may become slashable by force-exiting slashed beacon chain validators into the associated `EigenPod` contract using EIP-7002, and then sending the slashed portion of exited ETH to `0x00…000e16e4` address. This design is intended to be indicative and may change. A subsequent ELIP will outline these changes proposed to take advantage of Pectra. 

# Rationale

## Why do EigenPods need to upgrade?

The current EigenPods accounting system allows shares of stakers to take on negative values by attempting to decrement pod owners by a number of reduced shares proven in a checkpoint. Keeping this logic after slashing is problematic because:

1. It requires us to implement variables that can take on negative values in other parts of the system, making the protocol harder to understand and implement.  
2. It treats slashing native ETH differently from LSTs and other tokens…  
   1. Suppose the total rETH supply is 100 ETH on the beacon chain, the conversion ratio is 1:1, and all of it is staked in EigenLayer. If 50% is slashed on EigenLayer and then 50 ETH is slashed on the beacon chain, this leaves stakers 50 rETH which is worth 25 ETH on the beacon chain  
   2. Suppose an EigenPod has 100 native ETH shares proven on the beacon chain. If an AVS slashes this Native ETH Strategy by 50% on EigenLayer, the Operator’s magnitude for that strategy is proportionally decreased. Now another 50% is slashed on the beacon chain. Without a slashing factor, a withdrawal would propagate two 50% reductions to 100 ETH proven in the last checkpoint, netting 0 ETH at withdrawal.   
* With the separately recorded Strategy Magnitude and `beaconChainSlashingFactor`, withdrawals apply both reductions proportionally. ETH strategy is reduced 50%, with 50 ETH left. Next, the .5 `beaconChainSlashingFactor` is applied on 50 ETH, netting a 25 ETH withdrawal.  
  3. Having a difference in the treatment of the two strategies makes the system harder to reason about

An upgrade makes the system more consistent and reduces implementation complexity.

### Why must checkpoint creation be paused during the upgrade?

The checkpoint code is modified post-upgrade and is not compatible with existing uncompleted checkpoints. In order to accommodate the changes to EigenPods, we will be performing a migration just before the slashing release. The process roughly comprises:

* \~6 hours before the slashing release is live on mainnet, the `EigenPod.startCheckpoint` method will be paused for all EigenPods.  
* When the slashing release is executed, `EigenPod.startCheckpoint` will be unpaused and normal operations can resume.

### Why must checkpoint completion be blocked until shares are nonnegative?

The checkpoint completion logic after the upgrade is not compatible with negative shares, so this proposal will make completion of checkpoints in the context of negative shares revert. Instead, pending withdrawals must be completed until shares are nonnegative, and then only can checkpoints be completed.

If you have negative stakerDepositShares as of the slashing release, you will be unable to complete a checkpoint or add new validators to your pod. See the relevant [check and explanation here](https://github.com/Layr-Labs/eigenlayer-contracts/blob/slashing-magnitudes/src/contracts/pods/EigenPodManager.sol#L95-L102). To address this case, you will need to complete any existing withdrawals in the `DelegationManager` with `receiveAsTokens == false`.

This checkpoint pause is not problematic from a security perspective since EigenPods that have negative shares are already viewed by AVSs to have 0 funds at stake and receive no rewards.

# Security Considerations

There exists a case where EigenPod contracts become fully inoperable when a checkpoint is completed after all attached validators are fully slashed on the beacon chain (100%) and there are *no other assets* on the EigenPod. It will become “bricked” or non-functional, meaning even if new assets are added or validators attached to the Pod, it will still remain non-functional.

This is considered an open security issue, but due to the nature of Beacon Chain slashings, a full 100% slash has not yet occurred and is a very rare case. 

# Impact Summary

## Native Restakers (`EigenPod` Stakers)

`EigenPod` stakers should primarily be aware of the pause in checkpoints. If you are interfacing with your Pod through the front-end, these changes will be reflected at app.eigenlayer.xyz. 

As slashing goes live, it is worthwhile to review the delegation and withdrawal process for your `EigenPod`. If you have your stake delegated to an Operator, they will have the ability to allocate that stake as slashable. If there is any change needed to reflect a new risk posture, make sure you have the appropriate withdrawal credentials and that you have understood the broader withdrawal and redelegation process. You can learn more about what slashing means for stakers in the [ELIP.](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-002.md)

If you plan on using Smart Contract interfaces to interact with your Pod, review the outlined changes above. 

## LRTs

LRTs should carefully review the new contract interfaces, primarily as they relate to:

* Stake introspection,  
* and withdrawals…

which both have breaking changes associated with them. With stake introspection, `getWithdrawableShares` has moved from the `EigenPodManager` to the `DelegationManager`. 

Withdrawals have changed to accommodate both the new accounting for EigenPods and some quality of life improvements. Existing code paths will work, when passing the new withdrawal struct. The changes to the withdrawal flow are outlined in the [Withdrawal section of the Slashing ELIP](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-002.md#completing-withdrawals).

# References & Relevant Discussions

[https://hackmd.io/@-HV50kYcRqOjl\_7du8m1AA/rJOX08AG1x](https://hackmd.io/@-HV50kYcRqOjl_7du8m1AA/rJOX08AG1x)  
