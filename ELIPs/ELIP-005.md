| Author(s) | Created | Status | References | Discussions |
| :---- | :---- | :---- | :---- | :---- |
| [Bowen Li](mailto:bowen.li@eigenlabs.org) [Jeff Commons](mailto:jeff@eigenfoundation.org)  | 2025-02-28 | `merged` | | |

# ELIP-005: Simplification and Enhancement Of EIGEN and BackingEIGEN Token Contracts 


---

# Executive Summary

This proposal aims to simplify the EIGEN and BackingEIGEN token contracts by removing deprecated transfer restrictions, consolidating minting functionality, and enhancing event logging for token wrapping operations.

These changes will improve contract maintainability, reduce gas costs, and provide better transparency and legibility for token operations.

# Motivation

The current implementation of EIGEN and BackingEIGEN (aka bEIGEN) token contracts includes several features that are no longer necessary or could be optimized:

1. Transfer restrictions were initially implemented for the token launch phase but are no longer needed for normal operations.
2. Minting functionality is currently split between contracts, creating unnecessary complexity.
3. Token wrapping operations lack observability, making it harder to track and monitor these important operations.

These issues increase contract complexity and decrease maintainability, inflate gas costs, and make it harder to monitor token operations effectively.

# Features & Specification

The proposal introduces the following changes to both Eigen.sol and BackingEigen.sol contracts:

## Transfer Restrictions Removal

- Remove all external APIs related to transfer restrictions
- Deprecate allowedTo / allowedFrom storage mappings
- Mark deprecated mappings as internal for compatibility
- Remove associated events and modifiers

## Minting Consolidation

- Remove mint function and related storage from Eigen contract
- Consolidate all minting functionality into BackingEigen, as it already is process-wise. 
    - E.g. programmatic incentives are already minted in BackingEigen, thus will experience no change
- Maintain existing minter role system in BackingEigen

## New Events

- Add new events to EIGEN contract:
- event TokenWrapped(address indexed user, uint256 amount);
- event TokenUnwrapped(address indexed user, uint256 amount);

## Modified Functions

Update wrap/unwrap functions to emit new events:

```solidity
function wrap(uint256 amount) external {
    require(bEIGEN.transferFrom(msg.sender, address(this), amount));
    _mint(msg.sender, amount);
    emit TokenWrapped(msg.sender, amount);
}

function unwrap(uint256 amount) external {
    _burn(msg.sender, amount);
    require(bEIGEN.transfer(msg.sender, amount));
    emit TokenUnwrapped(msg.sender, amount);
}
```

# Rationale

1. Transfer Restrictions:
    - These were primarily needed during the initial token distribution phase
    - Removing them simplifies the contract and reduces gas costs
    - Keeping mappings as internal/deprecated, instead of removing them, maintains storage backward compatibility
2. Minting Consolidation:
    - the minting code in the EIGEN token was only needed for initial deployment, so is effectively dead code now. This reduces potential confusion for code readers
    - Single source of minting logic improves security and auditability
    - BackingEigen is the logical place for minting as it represents the base token
    - Reduces potential for confusion or errors in permission management
3. New Events:
    - Provides better visibility into wrap/unwrap operations
    - Enables more effective monitoring and analytics
    - Improves UX for applications building on top of Eigen

# Security Considerations

In summary, there is no change to existing security guarantees, context or processes.

1. Backward Compatibility:
    - Deprecated mappings are kept to maintain state compatibility
    - No impact on existing token balances or permissions
2. Minting Security:
    - Minting remains controlled by the minter role system
    - No changes to core permission logic
    - Reduced attack surface by consolidating minting logic
3. Event Implementation:
    - New events don't affect contract logic
    - Added at natural points in existing functions
    - No new attack vectors introduced
4. Security Audits:
    - Security auditing report will be shared once the code change is ready, before the upgrade happens


# Impact Summary

## Positive Impacts

- Reduced gas costs for transfer operations
- Simplified contract code and maintenance
- Better tracking of wrap/unwrap operations
- Clearer minting responsibility and control

## Stakeholder Impact

- Token Holders: Lower gas costs for transfers
- Researchers, Analysts: Better legibility of EIGEN/bEIGEN tokens
- EigenLayer Protocol Developers: Simplified codebase to maintain
- Protocol Governance: Reduced attack surface and less governance risks with minting function removed from EIGEN which is dead code
- Auditors: Simplified codebase to review
- Operators/AVS: No direct impact

# Action Plan

1. Implementation Phase:
    - Implement contract modifications
    - Add comprehensive tests, adapt existing tests
    - Internal review
2. Review Phase:
    - Community feedback period
    - Conduct external audit
    - Share and review auditing report
    - Final adjustments
3. Deployment Phase:
    - Internal testing
    - Deploy to testnet
    - Community testing period
    - Deploy to mainnet

