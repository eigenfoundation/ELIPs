# EigenLayer Improvement Proposal-009: MOOCOW - Massively Optimized Operations for Consolidation and Withdrawals

| Author(s) | Created | Status | References | Discussions |
|-------------|-----------|---------|------|----------|
| [Matt Nelson](mailto:matt.nelson@eigenlabs.org), [Yash Patil](mailto:yash@eigenlabs.org) | 2025-01-03 | `draft` | [EigenLayer Contracts PR #1425](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1425), [CLI PR #177](https://github.com/Layr-Labs/eigenpod-proofs-generation/pull/177) | Discussion Forum Post |

---

# Executive Summary

**MOOCOW (Massively Optimized Operations - COnsolidation and Withdrawals)** is the latest EigenPod upgrade that leverages new Ethereum Pectra features to dramatically reduce gas costs for native ETH restakers. This upgrade introduces support for validator consolidation and execution layer triggerable withdrawals, providing up to **64x reduction** in proof size and gas costs for restakers with multiple validators.

MOOCOW is fully backwards-compatible with existing EigenPod interfaces and the PEPE checkpoint proof system. The upgrade adds four new methods to EigenPods and enhances existing event systems without altering any existing method ABIs, making it completely opt-in for current integrations. Additionally, MOOCOW implements ELIP-5 enhancements to improve observability of Eigen token wrap/unwrap operations.

For large restakers like Renzo (with 1,775 validators representing 56,800 ETH TVL), MOOCOW enables consolidation from 24 transactions consuming ~99 million gas down to just 2 transactions consuming ~1.8 million gas - a **98% reduction** in operational costs.

# Motivation

Native ETH restakers with multiple validators face significant operational challenges and costs under the current EigenPod system:

## Current Pain Points

1. **Excessive Gas Costs**: Large restakers require numerous transactions to checkpoint validators, with costs scaling linearly with validator count
2. **Operational Complexity**: Managing hundreds or thousands of individual 32 ETH validators creates substantial operational overhead
3. **Proof Size Bloat**: Checkpoint proofs grow proportionally with validator count, making large operations prohibitively expensive

## Concrete Impact

Real-world examples demonstrate the severity of these issues:
- **Renzo**: 1,775 validators require 24 transactions and ~99 million gas to checkpoint
- **Operational Frequency**: Checkpointing is required for processing validator exits and accrued beacon chain rewards
- **Cost Scaling**: Linear scaling means costs become prohibitive as validator counts grow

## Ethereum Pectra Opportunity

The Ethereum Pectra upgrade introduces two EIPs that directly address these challenges:
- **EIP-7251**: Enables validator consolidation, allowing multiple 32 ETH validators to combine into single validators with up to 2,048 ETH
- **EIP-7002**: Enables execution layer triggerable withdrawals, allowing pod owners to manage validators directly from their pods

MOOCOW leverages these new capabilities to provide massive operational improvements while maintaining full compatibility with existing EigenLayer infrastructure.

# Features & Specification

## Overview

MOOCOW adds EigenPod support for validator consolidation and execution layer withdrawals through four new methods:

1. **`requestConsolidation()`**: Consolidates multiple validators into fewer, larger validators
2. **`requestWithdrawal()`**: Triggers validator exits or partial withdrawals directly from the pod
3. **`getConsolidationRequestFee()`**: Returns current consolidation request fee from EIP-7251 predeploy
4. **`getWithdrawalRequestFee()`**: Returns current withdrawal request fee from EIP-7002 predeploy

**Access Control**: All state-changing methods are restricted to pod owner or proof submitter via `onlyOwnerOrProofSubmitter` modifier, maintaining security while enabling operational flexibility.

## Core Functionality

### Validator Consolidation

**Purpose**: Combine multiple 32 ETH validators into single validators with up to 2,048 ETH effective balance.

**Key Benefits**:
- Up to 64x reduction in validator count (64 × 32 ETH = 2,048 ETH max)
- Proportional reduction in checkpoint gas costs
- Simplified validator management

**Requirements**:
- Target validators must have verified withdrawal credentials pointed at the pod
- Target validators must have 0x02 withdrawal credentials for consolidation
- Source validators are automatically exited after consolidation

### Execution Layer Withdrawals

**Purpose**: Enable pod owners to trigger validator exits or partial withdrawals without requiring beacon chain voluntary exit messages.

**Key Benefits**:
- Direct validator management from smart contracts
- Automated withdrawal processing
- Integration with DeFi protocols and automated strategies

**Types**:
- **Full Exit**: Completely exit a validator and withdraw all funds
- **Partial Withdrawal**: Withdraw excess balance above 32 ETH while keeping validator active

## Technical Specification

### New Methods

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
     * @notice Requests consolidation of validators
     * @param requests Array of consolidation requests
     * @dev Payable function requiring predeploy fee
     * @dev Only callable by pod owner or proof submitter
     */
    function requestConsolidation(
        ConsolidationRequest[] calldata requests
    ) external payable onlyOwnerOrProofSubmitter;

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
     * @notice Returns the fee required to add a consolidation request to the EIP-7251 predeploy this block.
     * @dev Note that the predeploy updates its fee every block according to https://eips.ethereum.org/EIPS/eip-7251#fee-calculation
     * Consider overestimating the amount sent to ensure the fee does not update before your transaction.
     */
    function getConsolidationRequestFee() external view returns (uint256);

    /**
     * @notice Returns the current fee required to add a withdrawal request to the EIP-7002 predeploy.
     * @dev Note that the predeploy updates its fee every block according to https://eips.ethereum.org/EIPS/eip-7002#fee-update-rule
     * Consider overestimating the amount sent to ensure the fee does not update before your transaction.
     */
    function getWithdrawalRequestFee() external view returns (uint256);
}
```

### New Events

MOOCOW introduces several new events to provide better observability of consolidation and withdrawal operations:

```solidity
/// @notice Emitted when a consolidation request is initiated where source == target
event SwitchToCompoundingRequested(bytes32 indexed validatorPubkeyHash);

/// @notice Emitted when a standard consolidation request is initiated
event ConsolidationRequested(bytes32 indexed sourcePubkeyHash, bytes32 indexed targetPubkeyHash);

/// @notice Emitted when a withdrawal request is initiated where request.amountGwei == 0
event ExitRequested(bytes32 indexed validatorPubkeyHash);

/// @notice Emitted when a partial withdrawal request is initiated
event WithdrawalRequested(bytes32 indexed validatorPubkeyHash, uint64 withdrawalAmountGwei);
```

**Event Types**:
- **Switch Request**: When `srcPubkey == targetPubkey`, switches validator from 0x01 to 0x02 withdrawal credentials
- **Consolidation**: Standard consolidation combining source validator balance into target
- **Full Exit**: When `amountGwei == 0`, completely exits validator from beacon chain
- **Partial Withdrawal**: Withdraws specific amount while keeping validator active

### Predeploy Fee Mechanism

Both methods require payment of a "predeploy fee" introduced in Pectra:

- **Dynamic Pricing**: Predeploy updates fees every block based on network request volume
- **Per-Request Fee**: Total fee = `fee × requests.length`
- **Overpayment Refund**: Excess fees are automatically refunded to `msg.sender`
- **Fee Queries**: Use `getConsolidationRequestFee()` and `getWithdrawalRequestFee()` for current rates
- **Fee Estimation**: Consider overestimating amounts sent as fees may update before transaction inclusion

### Consolidation Process Flow

#### Maximum Consolidation Example

For a pod with 1,000 validators:

1. **Select Targets**: Choose 16 validators as consolidation targets (1000 ÷ 64 = ~16)
2. **Switch Credentials**: Request consolidation with `srcPubkey == targetPubkey` for each target to switch to 0x02 credentials
3. **Consolidate Sources**: Group remaining 984 validators into 64-validator batches and consolidate each batch to a target
4. **Monitor Completion**: Beacon chain processes 2 consolidation requests per block (64 per epoch)
5. **Checkpoint**: Update pod accounting once consolidations complete on beacon chain

#### Beacon Chain Processing

- **Request Lifecycle**: Initiated → Queued in predeploy → Included in beacon block → Validated → Processed
- **Processing Limits**: 2 consolidation requests per block, 16 withdrawal requests per block  
- **Validation**: Beacon chain validates all requests; invalid requests are skipped without notification
- **Completion**: Switch requests complete immediately; other requests processed during epoch transitions

### Updates to Existing Functionality

#### Enhanced Event System

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

#### Checkpoint Storage Enhancement

**Updated Behavior**: When finalizing a checkpoint, the contract now stores the finalized checkpoint in storage for query access via `EigenPod.currentCheckpoint()`.

**Context**: Prior to v1.3.0, checkpoints were deleted on completion. v1.3.0 changed this to mark completion with `currentCheckpointTimestamp == 0` for gas savings. MOOCOW further enhances this by preserving finalized checkpoint data.

**Benefits**: Enables better state introspection and historical checkpoint querying.

#### Method Deprecation

**Removed**: `GENESIS_TIME()` method has been removed from EigenPods as it has been unused for over a year.

**Impact**: No functional impact as this method was already deprecated in practice.

### ELIP-5 Integration: Eigen Token Observability

MOOCOW also implements ELIP-5 enhancements to the Eigen token contract, adding observability events for wrap/unwrap operations:

```solidity
/// @notice Emitted when bEIGEN tokens are wrapped into EIGEN
event TokenWrapped(address indexed account, uint256 amount);

/// @notice Emitted when EIGEN tokens are unwrapped into bEIGEN
event TokenUnwrapped(address indexed account, uint256 amount);
```

**Purpose**: Improves tracking and monitoring of Eigen token operations, providing better transparency for token wrapping activities.

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

### Cross-Pod Limitations

**Not Supported**: Cross-pod consolidation where validators from different pods consolidate together.

**Reasoning**: Cross-pod consolidation would constitute asset transfers between different stakers, which is not a fundamental operation supported by the EigenLayer protocol.

**Alternative**: Restakers with multiple single-validator pods can exit from EigenLayer and redeposit to 0x02 validators in their target pod.

**Result**: EigenPods will support Pectra features, allowing the pod owner or proof submitter to initiate consolidations and withdrawals via the EIP-7002 and EIP-7251 predeploys.

# Rationale

## Design Decisions

### Opt-in Architecture

**Decision**: Make MOOCOW features completely optional additions to existing EigenPods.

**Rationale**: 
- Preserves backwards compatibility for all existing integrations
- Allows gradual adoption based on individual restaker needs
- Minimizes upgrade risk and complexity

### Predeploy Fee Pass-through

**Decision**: Require users to pay predeploy fees directly rather than abstracting them away.

**Rationale**:
- Maintains transparent cost model aligned with Ethereum's fee structure
- Prevents unexpected costs for EigenLayer protocol
- Allows users to optimize fee payments based on network conditions

### Single-Pod Restriction

**Decision**: Limit consolidation to validators within the same pod.

**Rationale**:
- Avoids complex cross-staker asset transfer logic
- Maintains clear ownership boundaries
- Simplifies security model and reduces attack surface

## Economic Considerations

### Gas Cost Optimization

The 64x improvement comes from fundamental changes in checkpoint operations:
- **Linear Scaling**: Current system requires O(n) operations per validator
- **Logarithmic Scaling**: Consolidated system reduces to O(log n) through validator reduction
- **Proof Efficiency**: Fewer validators mean smaller merkle proofs and reduced verification costs

### Fee Structure

- **Market-Based Pricing**: Predeploy fees adjust dynamically based on network demand
- **Cost Transparency**: Direct fee pass-through ensures users understand true costs
- **Efficiency Incentives**: Lower operational costs encourage consolidation adoption

# Security Considerations

## Protocol Security

### Request Processing Risks

**Risk**: Invalid consolidation/withdrawal requests could be submitted and processed incorrectly.

**Mitigation**: 
- Beacon chain performs comprehensive validation before processing any requests
- Invalid requests are automatically skipped without affecting pod state
- EigenPod state updates only occur through verified checkpoint proofs

### Fee Attack Vectors

**Risk**: Malicious actors could attempt to exploit fee mechanics or overpayment refunds.

**Mitigation**:
- Fees are handled by established Ethereum predeploy contracts
- Overpayment refunds follow standard Ethereum patterns
- No custom fee handling logic in EigenPod contracts

## Operational Security

### Consolidation Target Selection

**Risk**: Improper target validator selection could lead to failed consolidations or fund loss.

**Mitigation**:
- Clear documentation of target validator requirements
- Validation that target validators have proper withdrawal credentials
- Comprehensive error handling for invalid requests

### Beacon Chain Dependencies

**Risk**: Reliance on beacon chain processing introduces external dependencies.

**Mitigation**:
- EigenPod state remains authoritative through checkpoint system
- Failed beacon chain processing doesn't affect pod integrity
- Clear monitoring and retry mechanisms for stuck requests

## Upgrade Security

### Backwards Compatibility

**Risk**: New methods could interfere with existing functionality.

**Mitigation**:
- Strict ABI compatibility maintained for all existing methods
- Comprehensive testing of upgrade scenarios
- Isolated implementation of new functionality

### Emergency Procedures

**Risk**: Critical issues with consolidation or withdrawal features could affect pod operations.

**Mitigation**:
- Existing checkpoint and proof mechanisms remain fully functional
- New features are opt-in and can be avoided if issues arise
- Standard EigenLayer pause and upgrade mechanisms apply

# Impact Summary

## Restaker Impact

### Large Restakers (100+ validators)
- **Massive Cost Reduction**: Up to 98% reduction in checkpoint gas costs
- **Operational Simplification**: Significantly fewer validators to manage
- **Improved Economics**: Lower operational overhead improves net yields

### Small Restakers (1-10 validators)
- **Limited Immediate Benefit**: Minimal gas savings for small validator counts
- **Future Flexibility**: Access to new withdrawal mechanisms
- **No Negative Impact**: Opt-in nature ensures no disruption

### Liquid Restaking Tokens (LRTs)
- **Scalability Unlock**: Enables cost-effective scaling to thousands of validators
- **Competitive Advantage**: Dramatic cost reduction improves value proposition
- **Operational Efficiency**: Simplified validator management reduces complexity

## Operator Impact

### EigenLayer Operators
- **No Direct Impact**: MOOCOW focuses on native ETH restaking
- **Indirect Benefits**: More efficient restaking ecosystem may increase overall participation

## Protocol Impact

### EigenLayer Protocol
- **Enhanced Scalability**: Removes major barrier to large-scale native restaking
- **Competitive Positioning**: Maintains leadership in native ETH restaking solutions
- **Future-Proofing**: Establishes foundation for additional Pectra feature adoption

### Ethereum Ecosystem
- **Validator Efficiency**: Encourages more efficient use of beacon chain validator slots
- **Gas Optimization**: Reduces overall gas consumption for restaking operations

# Action Plan

## Implementation Timeline

### Phase 1: Smart Contracts ✅
- **Status**: Complete
- **Deliverables**: Core contract implementation, comprehensive testing, documentation

### Phase 2: Audits 🔄
- **Timeline**: June 23-30, 2025
- **Scope**: Security audit of new consolidation and withdrawal functionality
- **Deliverables**: Audit report and remediation of any identified issues

### Phase 3: CLI Tools ✅
- **Status**: Complete  
- **Deliverables**: Updated CLI tools for generating consolidation and withdrawal requests

### Phase 4: Deployment 📅
- **Dependencies**: Successful audit completion, Ethereum Pectra activation
- **Timeline**: Post-Pectra mainnet activation
- **Process**: Testnet deployment → community testing → mainnet upgrade

## Coordination Requirements

### Ethereum Protocol Coordination
- **Pectra Activation**: Deployment must wait for Ethereum Pectra fork activation
- **Predeploy Readiness**: Coordination with Ethereum client teams on predeploy availability

### EigenLayer Ecosystem Coordination  
- **Documentation**: Comprehensive guides for restakers on consolidation strategies
- **Tooling**: Integration with existing EigenLayer monitoring and management tools
- **Community Education**: Workshops and materials explaining new capabilities

### Integration Partner Coordination
- **LRT Protocols**: Assistance with integration of new consolidation features
- **Infrastructure Providers**: Coordination on tooling and automation support
- **Monitoring Services**: Updates to support consolidated validator tracking

# References & Relevant Discussions

- **EIP-7251**: [Increase the MAX_EFFECTIVE_BALANCE](https://eips.ethereum.org/EIPS/eip-7251)
- **EIP-7002**: [Execution layer triggerable withdrawals](https://eips.ethereum.org/EIPS/eip-7002)  
- **EigenLayer Contracts PR**: [#1425](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1425)
- **CLI Implementation PR**: [#177](https://github.com/Layr-Labs/eigenpod-proofs-generation/pull/177)
- **PEPE (Previous EigenPod Upgrade)**: Foundation for forwards-compatible design
- **Ethereum Pectra Upgrade**: [Ethereum roadmap documentation](https://ethereum.org/en/roadmap/pectra/)
