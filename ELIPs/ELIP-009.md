| Author(s) | Created | Status | References | Discussions |
| :---- | :---- | :---- | :---- | :---- |
| [Matt Nelson](mailto:matt.nelson@eigenlabs.org), [Alex Wade](mailto:alex@eigenlabs.org), [Matt Curtis](mailto:matt.curtis@eigenlabs.org) | 2025-06-28 | `testing` | [EigenLayer Contracts PR #1425](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1425), [CLI PR \#177](https://github.com/Layr-Labs/eigenpod-proofs-generation/pull/177) | Discussion Forum Post |

# ELIP-009: MOOCOW: Massively Optimized Operations for Consolidation and Withdrawals

---

# Executive Summary

**MOOCOW (Massively Optimized Operations - Consolidation and Withdrawals)** is the latest EigenPod upgrade that leverages new Ethereum Pectra features to improve usability and dramatically reduce gas costs for native ETH restakers.

This upgrade introduces support for validator consolidation, providing up to **64x reduction** in proof size and gas costs for restakers with multiple validators. Operators may convert existing validators to compounding `0x02` credentials, allowing Ethereum consensus layer rewards to compound directly without any user intervention or network fees.

In addition, MOOCOW provides support for execution layer triggerable validator withdrawals. This feature solidifies the EigenPod owner’s control over validator assets, opening new opportunities in staker-operator relationships. This feature is also crucial to and forwards-compatible with the planned implementation of redistribution for native ETH (See [ELIP-006](./ELIP-006.md)).

MOOCOW is fully backwards-compatible with existing EigenPod interfaces and the PEPE checkpoint proof system. The upgrade adds four new methods to EigenPods and enhances existing event systems without altering any existing method ABIs, making it completely opt-in for current integrations.

# Motivation

The Ethereum Pectra upgrade introduced two EIPs that provide the execution layer owner (withdrawal address) of the validator powerful new capabilities:

- [**EIP-7251**](https://eips.ethereum.org/EIPS/eip-7251): Enables consolidation, allowing multiple validators to combine into single validators with up to 2,048 ETH. Additionally, existing validators with 0x01 withdrawal credentials may convert to compounding `0x02` credentials that automatically compound their consensus layer rewards.  
- [**EIP-7002**](https://eips.ethereum.org/EIPS/eip-7002): Enables execution layer triggerable withdrawals, allowing pod owners to manage validators directly from their pods

Consolidation and withdrawal requests are required to come from the withdrawal credentials, and the withdrawal credentials (the pod) don't yet have a means to create those requests. This proposal upgrades the pod to give access to these new features of Ethereum. They address multiple existing pain points in the EigenPod system:

1. **Excessive Gas Costs**: Large restakers require numerous transactions to checkpoint validators, with costs scaling linearly with validator count. Validator consolidation will allow up to 64x reduction of validator count and associated cost.  
2. **Onerous Validator Consolidation Path**: Restakers with `0x01` validators seeking to compound consensus layer rewards on the beaconchain must complete checkpoints, send multiple transactions, and wait 14 days to completely withdraw ETH from EigenLayer. Converting the validator to `0x02` credentials causes these rewards to compound on the beacon chain automatically.  
3. **EigenPod Control of Validators**: While EigenPods account for and lock beacon chain assets in EigenLayer, there is no mechanism for the EigenPod to retrieve beacon chain ETH in the case that the validator’s signing keys are lost, inaccessible, or otherwise uncooperative. Execution layer triggerable withdrawals enable this directly.

Additionally, the slashing of native ETH from EigenPods has incomplete support: while the Staker is unable to withdraw ETH from the EigenPod system until the slash is accounted for, they can simply choose to continue running validators on beacon chain until sufficient ETH rewards have accumulated to pay off that debt. We anticipate that the ability to trigger validator withdrawals from the execution layer will be critical for the future implementation of a proper native ETH slashing and burn /  redistribution solution.

This proposal upgrades the EigenPod system with pass-through functions that grant EigenPod owners and proof submitters the ability to consolidate validators and trigger withdrawals from the execution layer, providing significant operational improvements while maintaining full compatibility with existing EigenLayer infrastructure.

# Features & Specification

## Overview

MOOCOW adds EigenPod support for validator consolidation and execution layer withdrawals through two new methods:

1. **`requestConsolidation()`**:
   1. If `target == source`: updates a validator’s withdrawal credentials from `0x01` to `0x02`  
   2. Otherwise, consolidates multiple validators into fewer, larger validators  
2. **`requestWithdrawal()`**: Triggers validator exits or partial withdrawals directly from the pod

In addition, the predeploy requires a fee that is sent with each request. Two new methods have been exposed to query the current fee for these requests:

3. **`getConsolidationRequestFee()`**: Returns the fee required to add a consolidation request to the EIP-7251 predeploy for the head block.  
4. **`getWithdrawalRequestFee()`**`:` Returns the fee required to add a withdrawal request to the EIP-7002 predeploy.

**Access Control**: Both consolidation and withdrawal requests can only be initiated by a Pod’s owner or proof submitter via `onlyOwnerOrProofSubmitter` modifier, maintaining security while enabling operational flexibility.

## Core Functionality

### Validator Consolidation (EIP-7251)

Consolidation allows you to take multiple validators and combine them into a single validator with a maximum effective balance of 2048 ETH. Consolidations consist of a “source validator” and a “target validator.” The target validator must have `0x02 credentials`, and will receive the source validator’s balance when the consolidation is complete. Source validators are those whose balance you wish to migrate into the target, and will be automatically exited from the beacon chain when the consolidation is complete.

A quirk of the implementation of EIP-7251 is that there is no separate operation for converting a validator from `0x01` to `0x02` withdrawal credentials. A “ConsolidationRequest” with the same source and target validator will update the validator from `0x01` to `0x02` credentials, while a “ConsolidationRequest” with different source and target validators will send the ETH from the source validator to the target validator, then exit the source validator. This proposal uses the same convention. The structs, interfaces, and events are below:

```solidity
/**
 * @param srcPubkey the pubkey of the source validator for the consolidation
 * @param targetPubkey the pubkey of the target validator for the consolidation
 * @dev Note that if srcPubkey == targetPubkey, this is a "switch request," and will
 * change the validator's withdrawal credential type from 0x01 to 0x02.
 * For more notes on usage, see `requestConsolidation`
 */
struct ConsolidationRequest {
    bytes srcPubkey;
    bytes targetPubkey;
}

interface IEigenPod {
    /**
     * @notice Requests consolidation of validators
     * @param requests Array of consolidation requests
     * @dev Payable function requiring predeploy fee
     * @dev Only callable by pod owner or proof submitter
     */
    function requestConsolidation(
        ConsolidationRequest[] calldata requests
    ) external payable onlyOwnerOrProofSubmitter;

    /**
     * @notice Returns the fee required to add a consolidation request to the EIP-7251 predeploy this block.
     * @dev Note that the predeploy updates its fee every block according to https://eips.ethereum.org/EIPS/eip-7251#fee-calculation
     * Consider overestimating the amount sent to ensure the fee does not update before your transaction.
     */
    function getConsolidationRequestFee() external view returns (uint256);
}

/// @notice Emitted when a consolidation request is initiated where source == target
event SwitchToCompoundingRequested(bytes32 indexed validatorPubkeyHash);

/// @notice Emitted when a standard consolidation request is initiated
event ConsolidationRequested(bytes32 indexed sourcePubkeyHash, bytes32 indexed targetPubkeyHash);
```

#### Conversion (Switch Request)

To request a conversion from `0x01` to `0x02` credentials, an EigenPod Owner or Proof Submitter must call `requestConsolidation()` with identical source and target pubkeys.

When the request is successfully completed, the updated validator index remains the same, but now has a 2,048 ETH max effective balance and automatically compounds consensus-layer rewards on the beacon chain (up to that max effective balance).

***Note:*** this update cannot be reversed. A validator with `0x02` credentials will not regularly sweep consensus layer rewards until it has accumulated over 2,048 ETH. If you want to withdraw some amount of ETH before that time, you need to use the new `requestWithadawal` method on the EigenPod interface.  

#### Consolidation

Consolidation requests can be made by the EigenPod Owner or Proof Submitter by calling `requestConsolidation()` with different source and target pubkeys. In this case, the target validator must have **verified** withdrawal credentials pointed at the EigenPod to ensure that ETH cannot be transferred outside of the EigenPod’s control. In addition, the target validator must have `0x02` withdrawal credentials, or the request will fail on the beacon chain.

A successfully completed consolidation request will set the source validator's `exit_epoch` and `withdrawable_epoch`, similar to an exit. When the exit epoch is reached, an epoch sweep will process the consolidation request and transfer balance from the source(s) to the target validator. At this stage, the source validator will no longer participate in consensus. Note that consolidation transfers `min(srcValidator.effective_balance, state.balance[srcIndex])` to the target. This may not be the entirety of the source validator's balance; any remainder will be moved to the target when hit by a subsequent withdrawal sweep.

#### Checkpoint Behavior

Consolidation requests do not interact with or change the EigenPod’s restaked balance. This is because verification of the target validator’s withdrawal credentials is required for consolidation requests. While `0x02` withdrawal credentials allow consensus layer rewards to compound on the beacon chain, the restaked balance of ETH on EigenLayer will only be updated after a checkpoint is completed.

#### Consolidation Request Failures

Note that consolidation requests CAN FAIL for a variety of reasons. Failures occur when the request is processed on the beacon chain, and are invisible to the pod. The pod and predeploy cannot guarantee a request will succeed; it's up to the pod owner to determine this for themselves. If your request fails, you can retry by initiating another request via this method.

Also note that while source validators that are not pointed at the EigenPod can be consolidated into EigenPod validators with 0x02 credentials, the requests to do so must be signed by the source validator’s withdrawal address and do not require these functions.

Some requirements that are NOT checked by the Pod itself, which may cause an ignored request on the beacon chain (while consuming the predeploy fee):

- If `request.srcPubkey == request.targetPubkey`, the validator MUST have `0x01` credentials  
- If `request.srcPubkey != request.targetPubkey`, the target validator MUST have `0x02` credentials  
- Both the source and target validators MUST be active on the beacon chain and MUST NOT have initiated exits  
- The source validator MUST NOT have pending partial withdrawal requests (via `requestWithdrawal()`)  
- If the source validator is slashed after requesting consolidation (but before processing), the consolidation will be skipped

For further reference, see consolidation processing at block and epoch boundaries:  

- [Ethereum Beacon Block Spec](https://github.com/ethereum/consensus-specs/blob/dev/specs/electra/beacon-chain.md#new-process_consolidation_request)  
- [Ethereum Epoch Spec](https://github.com/ethereum/consensus-specs/blob/dev/specs/electra/beacon-chain.md\#new-process\_pending\_consolidations)

### Execution Layer Withdrawals (EIP-7002)

EIP-7002 grants the withdrawal address of a validator capability to initiate withdrawals from the execution layer. This proposal passes that capability to the EigenPod owner and proof submitter. This enables EigenPods to trigger exits and partial withdrawals from validators without requiring beacon chain voluntary exit messages. The new structs, interfaces, and events are below:

```solidity
/**
 * @param pubkey the pubkey of the validator to withdraw from
 * @param amountGwei the amount (in gwei) to withdraw from the beacon chain to the pod
 * @dev Note that if amountGwei == 0, this is a "full exit request," and will fully exit
 * the validator to the pod.
 * For more notes on usage, see `requestWithdrawal`
 */
struct WithdrawalRequest {
    bytes pubkey;
    uint64 amountGwei;
}

interface IEigenPod {
    /**
     * @notice Requests withdrawal from validators  
     * @param requests Array of withdrawal requests
     * @dev Payable function requiring predeploy fee
     * @dev Only callable by pod owner or proof submitter
     */
    function requestWithdrawal(
        WithdrawalRequest[] calldata requests
    ) external payable onlyOwnerOrProofSubmitter;

    /**
     * @notice Returns the current fee required to add a withdrawal request to the EIP-7002 predeploy.
     * @dev Note that the predeploy updates its fee every block according to https://eips.ethereum.org/EIPS/eip-7002#fee-update-rule
     * Consider overestimating the amount sent to ensure the fee does not update before your transaction.
     */
    function getWithdrawalRequestFee() external view returns (uint256);
}

/// @notice Emitted when a withdrawal request is initiated where request.amountGwei == 0
event ExitRequested(bytes32 indexed validatorPubkeyHash);

/// @notice Emitted when a partial withdrawal request is initiated
event WithdrawalRequested(bytes32 indexed validatorPubkeyHash, uint64 withdrawalAmountGwei);
```

Withdrawal requests have two types: full exit request and partial exit requests.

#### Full Exit Requests

If `request.amount == 0`, this is a FULL exit request. A full exit request initiates a standard validator exit. This type of request can be made of both `0x01` and `0x02` validators.

#### Partial Exit Requests

Any other value of request amount will initiate a partial exit request. This requires that the validator have `0x02` withdrawal credentials. It is important to note that a partial exit request cannot leave a validator with less than 32 ETH. The actual amount withdrawn for a partial exit is given by the formula: `min(request.amount, state.balances[vIdx] - 32 ETH - pending_balance_to_withdraw)` where `pending_balance_to_withdraw` is the sum of any outstanding partial exit requests. This means that a partial exit request may return **less than the amount requested**. For example, if a validator has 40 ETH, and a partial exit request is submitted for 20 ETH, only 8 ETH will be sent to the EigenPod. If 20 ETH is required, a Full Exit Request must be submitted, sending all 40 ETH to the EigenPod.

As with Consolidation requests, withdrawal requests can fail when processed on the beacon chain for a variety of reasons. A failed withdrawal request cannot be seen by the EigenPod and must be evaluated by the user and retried. You can check the status by validator index with tools like [beaconcha.in](http://beaconcha.in).

Some requirements that are NOT checked by the pod:

- `request.pubkey` MUST be a valid validator pubkey  
- `request.pubkey` MUST belong to a validator whose withdrawal credentials are this pod  
- If `request.amount` is for a partial exit, the validator MUST have `0x02` withdrawal credentials  
- If `request.amount` is for a full exit, the validator MUST NOT have any pending partial exits  
- The validator MUST be active on the beacon chain and MUST NOT have initiated exit

For further reference: [https://github.com/ethereum/consensus-specs/blob/dev/specs/electra/beacon-chain.md\#new-process\_withdrawal\_request](https://github.com/ethereum/consensus-specs/blob/dev/specs/electra/beacon-chain.md#new-process_withdrawal_request)

### Predeploy Fee Mechanism

Both `requestConsolidation()` and `requestWithdrawal()` are payable methods, accepting a "request fee." This is a new concept introduced in Pectra: both predeploys must be sent a fee along with any requests. This means that when calling either function, you must supply a sufficient fee as msg.value.

The new interface exposes getters for this: `getConsolidationRequestFee()` and `getWithdrawalRequestFee()` return the fee for one request, for the current block. The correct amount to send is `fee * requests.length`, i.e. multiply the return value by the number of requests you intend to send.

Note that depending on how many requests are processed by the network, this fee might increase or decrease at the end of each block. We recommend overestimating the fee sent, as any excess will be sent back to `msg.sender` after all requests have been sent to the predeploy. If you are submitting requests across multiple transactions, check the fee between requests. Fees change using an exponential formula that can quickly become very expensive.

### Consolidation Process Flow Examples

#### Maximum Consolidation Example

For a pod with 1,000 validators:

1. **Select Targets**: Choose 16 validators as consolidation targets: `ceil(1000 * 32 / 2048) == 16`  
2. **Switch Credentials**: Request consolidation with `srcPubkey == targetPubkey` for each target to switch to 0x02 credentials  
3. **Consolidate Sources**: Group remaining 984 validators into 64-validator batches and consolidate each batch to a target  
4. **Monitor Completion**: Beacon chain processes 2 consolidation requests per block (64 per epoch)

#### Beacon Chain Processing

- **Request Lifecycle**: Initiated → Queued in predeploy → Included in beacon block → Validated → Processed  
- **Processing Limits**: 2 consolidation requests per block, 16 withdrawal requests per block  
- **Validation**: Beacon chain validates all requests; invalid requests are skipped without notification (i.e. beacon and execution blocks are valid)  
- **Completion**: Switch requests complete immediately; other requests processed during epoch transitions

# Rationale

The features added in this proposal simply expose features added to validators by the Pectra upgrade to EigenPod users.

## No Support for Cross-Pod Consolidation

MOOCOW will *not* support the consolidation of validators pointed to multiple different EigenPods. While Pectra would allow this on the beacon chain, transferring staked balance between stakers is not currently supported in the EigenLayer state model and would require significant additional complexity to implement. By limiting the consolidation target to validators with withdrawal credentials pointed at the same EigenPod, we are guaranteed that there is no change in the ownership of EigenPod funds.

Stakers seeking to consolidate between EigenPods must still fully withdraw funds from EigenLayer to ensure that proper slashability guarantees are upheld.

## Updates to Existing Functionality

### Enhanced Event System

**Validator Reference Standardization**: All EigenPod events now emit `pubkeyHash` when referencing validators, replacing the previous mixed usage of validator indices and full pubkeys:

```solidity
/// @notice Updated event definitions for consistency
event EigenPodStaked(bytes32 pubkeyHash);
event ValidatorRestaked(bytes32 pubkeyHash);
event ValidatorBalanceUpdated(bytes32 pubkeyHash, uint64 balanceTimestamp, uint64 newValidatorBalanceGwei);
event ValidatorCheckpointed(uint64 indexed checkpointTimestamp, bytes32 indexed pubkeyHash);
event ValidatorWithdrawn(uint64 indexed checkpointTimestamp, bytes32 indexed pubkeyHash);
```

**Benefits**: Provides consistent validator identification across all events, improving indexing and monitoring capabilities.

### Checkpoint Storage Enhancement

**Updated Behavior**: When finalizing a checkpoint, the contract now stores the finalized checkpoint in storage for query access via `EigenPod.currentCheckpoint()`.

**Context**: Prior to v1.3.0, checkpoints were deleted on completion. v1.3.0 changed this to mark completion with `currentCheckpointTimestamp == 0` for gas savings. MOOCOW further enhances this by preserving finalized checkpoint data.

**Benefits**: Enables better state introspection and historical checkpoint querying.

### Method Deprecation

**Removed**: `GENESIS_TIME()` method has been removed from EigenPods as it has been unused for over a year.

**Impact**: No functional impact as this method was already deprecated in practice.

## Compatibility & Integration

### Backwards Compatibility

- **Existing Methods**: All current EigenPod methods remain unchanged  
- **PEPE Integration**: Full compatibility with existing checkpoint proof system  
- **ABI Stability**: No changes to existing method signatures  
- **Opt-in Adoption**: Current integrations continue functioning without modification

### Forwards Compatibility

MOOCOW is designed to be fully forwards-compatible with future EigenLayer upgrades, including:

- Native ETH burning mechanisms  
- ETH redistribution systems  
- Additional Pectra features

# Security Considerations

## Protocol Security

### Request Processing Risks

**Risk**: Invalid consolidation/withdrawal requests could be submitted and processed incorrectly.

**Mitigation**:

- Beacon chain performs comprehensive validation before processing any requests  
- Invalid requests are automatically skipped without affecting pod state  
- EigenPod state updates only occur through verified checkpoint proofs  
- Manually checking the status of beacon chain requests using external tools

### Fee Griefing Vectors

**Risk**: Malicious actors could attempt to exploit fee mechanics by submitting tons of requests.

**Mitigation**:

- The CLI will ask you to check the fee between each transaction  
- Fees are handled by established Ethereum predeploy contracts  
- No custom fee handling logic in EigenPod contracts

# Impact Summary

## Restaker & LRT Impact

- **Massive Cost Reduction**: Up to 98% reduction in checkpoint gas costs for large restakers  
- **Operational Simplification**: Consolidation results in fewer validators to manage  
- **Improved Economics**: Lower operational overhead improves net yields  
- **Automatic Compounding:** Beacon chain rewards compound on the beacon chain  
- **Validator Control:** Withdrawals from the beacon chain can be triggered before 2048 ETH is accumulated instead of receiving continuous sweeps from the beacon chain. This may be more relevant for large entities like LRTs  
- **No Negative Impact**: Opt-in nature ensures no disruption

## Operator Impact

### EigenLayer Operators

- **Restakers Impacting Validators:** In situations where restakers have delegated running Ethereum validators to Operators, EigenLayer participants *may* consolidate, change, or stop validators.  
- **Indirect Benefits**: More efficient restaking ecosystem may increase overall participation

# Action Plan

## Implementation Timeline

### Phase 1: Smart Contracts & CLI Tools

- **Status**: Complete  
- **Deliverables**: Core contract implementation, comprehensive testing, documentation, CLI release

### Phase 2: Audits

- **Timeline**: June 23-30, 2025  
- **Status:** Complete  
- **Scope**: Security audit of new consolidation and withdrawal functionality  
- **Deliverables**: Audit report and remediation of any identified issues

### Phase 3: Deployment

- **Dependencies**: Successful audit completion, Ethereum Pectra activation  
- **Timeline**: Post-Pectra mainnet activation  
- **Status:** Testnet complete; mainnet pending
- **Process**: Testnet deployment → community testing → mainnet upgrade

# References & Relevant Discussions

- **EIP-7251**: [Increase the MAX\_EFFECTIVE\_BALANCE](https://eips.ethereum.org/EIPS/eip-7251)  
- **EIP-7002**: [Execution layer triggerable withdrawals](https://eips.ethereum.org/EIPS/eip-7002)  
- **EigenLayer Contracts PR**: [\#1425](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1425)  
- **CLI Implementation PR**: [\#177](https://github.com/Layr-Labs/eigenpod-proofs-generation/pull/177)  
- **PEPE (Previous EigenPod Upgrade)**: Foundation for forwards-compatible design  
- **Ethereum Pectra Upgrade**: [Ethereum roadmap documentation](https://ethereum.org/en/roadmap/pectra/)
