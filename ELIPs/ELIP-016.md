# EigenLayer Improvement Proposal-016: Redistribution Delay

| Author(s) | Created | Status | References | Discussions |
| :---- | :---- | :---- | :---- | :---- |
| [Matt Curtis](mailto:matt.curtis@eigenlabs.org) | 2026-05-20 | `merged` | [EigenLayer Protocol PR](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1753) | [[Forum discussion link]](https://forum.eigenlayer.xyz/t/draft-elip-16-redistribution-delay/14829) |

# Executive Summary

This proposal aims to add a 7-day delay between the time of slashing by an AVS and the transfer of assets out of EigenLayer in the case of redistribution. An additional pausing function is added to allow the pauser multisig to prevent redistribution in the case of security events. 

# Motivation

Presently, slashing and burning/redistribution of slashed shares can occur within the same transaction when triggered by an AVS. This leaves no window for any actor to react to an erroneous slashing event before funds are moved, which creates risk for Stakers, Operators, and AVSs. This proposal adds a delay between slashing and burning/redistribution as a safety buffer that can allow fund movement to be halted in an emergency.

# Features and Specs

The existing implementation of slashing and burning/redistribution already involves calling two separate functions: 

- an operator is first slashed by calling `slashOperator` on the `AllocationManager`  
- the assets are redistributed by calling `clearBurnOrRedistributableShares` on the `StrategyManager`. 

As presently implemented, these functions can be called atomically. This proposal adds a 7-day (50,400 block) slash resolution delay before slashed shares can be burned or redistributed. A dedicated pause flag (`PAUSED_BURN_OR_REDISTRIBUTABLE_SHARES`) allows burn/redistribution to be paused independently of other flows.

In the `increaseBurnOrRedistributableShares` function of the StrategyManager called by the DelegationManager during slashing, a resolution block is set `SLASH_RESOLUTION_DELAY_BLOCKS` (set to 50,400) beyond the current block number for the first strategy a new slashId is recorded. Slashes initiated before this upgrade are grandfathered with a resolution block of 0 such that they all remain immediately clearable.

```solidity
        // Set the resolution block the first time this slashId is recorded for this operator set.
        if (_pendingSlashIds[operatorSet.key()].add(slashId)) {
            uint32 resolutionBlock = uint32(block.number) + SLASH_RESOLUTION_DELAY_BLOCKS;
            _slashResolutionBlock[operatorSet.key()][slashId] = resolutionBlock;
            emit SlashResolutionBlockSet(operatorSet, slashId, resolutionBlock);
        }
```

When redistributing funds via the `clearBurnOrRedistributableShares` or `clearBurnOrRedistributableSharesByStrategy` function, a check is added to ensure this resolution block has passed. These functions also revert if the `PAUSED_BURNING_AND_REDISTRIBUTION` flag is set by the pauser multisig.

A view function, `getSlashResolutionBlock` allows the resolution block to be retrieved by slashId.

# Rationale

As currently implemented, slashing with redistribution allows AVSs to instantaneously transfer assets from the protocol. As noted in ELIP-006, the compromise of an Operator’s private keys could allow a malicious entity to register for a malicious AVS, slash, and redistribute allocated funds to an address of their choice. There were two barriers in place to prevent this possibility:

- All existing Operators were subject to a 17.5 initial allocation delay period in which they established their own allocation delay.  
- Allocation Delays were expected to be set higher than the 14 day escrow period such that stakers delegated to public operators would have time to react to allocation changes.

In the intervening period, a number of operators (including public operators) have set their allocation delays to 0 days, a setting that allows for instantaneous risk profile changes. In addition, multiple public Operators that still have assets delegated to them have ceased operations, raising questions as to the status of their private keys. Finally, malicious attacks on Defi protocols have increasingly targeted private keys with control over assets. 

This has added significant risk to the following threat scenario: If an Operator with 0 Allocation Delay becomes compromised by a malicious entity, assets delegated to it can be instantaneously allocated and redistributed out of EigenLayer. While there is presently no evidence that such a situation has occurred, the systemic risk of private key mishandling will continue to increase. 

# Security Considerations

In the event of a security incident, redistribution can be paused to prevent slashed funds from leaving the protocol. By using an independent pauser flag, EigenLayer can continue providing security to additional protocols while the redistribution is paused; however, there is no mechanism included in this upgrade to allow an erroneous slashing event to be reverted. Redistribution must remain paused and all slashed funds remain inaccessible to all parties until protocol upgrades capable of handling the malicious slashing event are completed.

The use of this pausing system is at the discretion of the [Pauser Multisig](https://etherscan.io/address/0x5050389572f2d220ad927CcbeA0D406831012390).

# Impact Summary

This is a breaking change for any AVSs currently using redistribution by calling both the slash and clear functions in the same transaction. AVSs requiring this functionality for normal operation may require additional changes.
