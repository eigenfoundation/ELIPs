# EigenLayer Improvement Proposal-017: Duration Vault Fixes

| Author(s) | [Matt Curtis](mailto:matt.curtis@eigenlabs.org) |
| :---- | :---- |
| Created | May 20, 2026 |
| Status | `draft` |
| References | [EigenLayer Protocol PR](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1753) |
| Discussions | [Forum Post]() |

# Executive Summary
This minor proposal aims to correct an oversight that presently prevents the creation of Duration Vaults for multiple ETH LSTs.

# Motivation

Presently, the deployment of Duration Vaults by the `StrategyFactory` checks and prevents deployment if the underlying token matches an internal blacklist. This check was an oversight: ETH LSTs represent an abundant form of economic security that AVSs may wish to make long duration commitments around. To correct this, this proposal removes the blacklist check, instead preventing the deployment of Duration Vaults only for the EIGEN and bEIGEN tokens.

# Features and Specs

Removal of blacklist check from Duration Vault deployment via the Strategy Factory.

Hardcoded blacklisting of EIGEN / bEIGEN Duration Vault deployment.

```solidity
        require(underlyingToken != EIGEN && underlyingToken != bEIGEN, ProhibitedDurationVaultToken());
```

# Rationale

The `StrategyFactory`’s [blacklist](https://github.com/Layr-Labs/eigenlayer-contracts/blob/v1.12.0/docs/core/StrategyManager.md#strategyfactoryblacklisttokens) was implemented exclusively to prevent the permissionless deployment of duplicate strategies for LSTs that had already been implemented manually. The blacklist includes EIGEN, bEIGEN, cbETH, stETH, rETH, ETHx, ankrETH, OETH, osETH, swETH, wBETH, sfrxETH, lsETH, and mETH. The removal of the blacklist check allows deployment of Duration Vaults for these tokens. 

EIGEN and bEIGEN will separately be manually excluded from duration vault deployment, as the specifications necessary to handle EIGEN Forking are not yet clear.

# Security Considerations

This change prevents the addition of new tokens to the blacklist to prevent Duration Vault deployments. This was not a planned use case for the blacklist.   
