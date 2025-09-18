---
title: EIP-011

---

# EigenLayer Improvement Proposal ELIP-011: Programmatic Incentives v2.0

| Author(s) | Created | Status | References | Discussions |
|-------------|-----------|---------|------|----------|
| [Robert Drost](mailto:robert@eigenfoundation.org),  [Brandon Curtis](mailto:brandon@eigenlabs.org) | 2025-09-17 | Draft | Listed at end of proposal |  |

## Executive Summary

This proposal builds upon the original programmatic incentives released in 2024, beginning in August of that year. Under the initial scheme, 3% EIGEN incentives were directed to ETH stakers, and 1% to EIGEN stakers, following the eligibility requirements outlined in [rewards v2 (ELIP-001)](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-001.md). Although the initial programmatic incentives were not explicitly versioned and preceded the ELIP process overall, for clarity we retroactively refer to them as **Programmatic Incentives v1 (PIv1)**.

This ELIP proposes a single, focused modification to IIv1: increasing the programmatic incentives allocated to EIGEN stakers from **1%** to **4%**, while leaving eligibility requirements unchanged. This would mark the release of **Programmatic Incentives v2.0 (PIv2.0)**. The versioning indicates our expectation that additional refinements in the v2.x family will follow with no or only minor changes to the overall reward rate.

## Motivation

Feedback around PIv1 indicated that the combined 4% reward rate effectively supported ETH staking through it 3% rewards, but was insufficient for EIGEN staking with only 1% rewards. 

While many protocols start with much higher inflationary rates, e.g. 8%, 9% or even higher, for EigenLayer PIv1's 4% was a conservative choice. The protocol was developing and releasing features such as slashing and redistribution. During this time a lower reward rate both helped with token supply/demand imbalances and was a good risk/reward position during a time when only shared staking was enable. 

Feedback around PIv1 indicated that the combined 4% reward rate effectively supported ETH staking through its 3% rewards. ETH staking levels to date provide an ample security pool for AVSs to draw upon for objective slashable security. On the other had, the 1% incentives to EIGEN were a very low incentive for a native protocol token. Much of the circulating and stakable market capitaliztion of EIGEN are staked, but an appreciable amount is not.

As EigenLayer progresses to verifiable cloud primitives based on EigenDA and EigenVerify, the need for ample intersubjective slashable security becomes critical. EIGEN is the intersubjective work token and it is the only token able to provide intersubjective slashable security.  With the additional risk of intersubjective slashing, the need for more rewards for EIGEN staking becomes only more needed. PIv1's 1% incentive may not adequately compensate for the withdrawal delay and the risk of slashing associated with intersubjective forks. Because staked EIGEN tokens are critical for securing EigenDA, EigenVerify, and other foundational primitives—whose security relies on intersubjective forking and slashing—greater participation of EIGEN token holders to staking is essential.

This proposal seeks to correct the imbalance by raising EIGEN staking incentives from **1%** to **4%**, bringing the total staked token programmatic reward rate for the sum of ETH and EIGEN to **7%**. This better aligns incentives for token EIGEN holders and strengthens the security of EigenLayer’s primitives. 

We note that at least one more follow-up proposal is anticipated in the v2 family, likely termed **PIv2.1**. While that proposal will not alter reward percentages, it will likely aim to adjust distribution criteria to further optimize staker and operator behavior through targeted incentives that improve EigenLayer and AVS capabilities.


## Features & Specification


### Modified code

The implementation of Programmatic Incentives v1 can be found in this commit 9cf6f41 of the EigenHopper repository:

[Deploy_ProgrammaticIncentives_Mainnet.s.sol](https://github.com/Layr-Labs/EigenHopper/commit/9cf6f41c936e4b524aa3e6c8e5441a719a6262a7)

Lines 46 an 47 show the weekly token mint amounts that are currently directed to ETH and EIGEN stakers. 

This ELIP introduces in effect a one-line change to line 46 the file, from
```solidty
uint256 public constant EIGEN_stakers_weekly_distribution = 321_855_128_516_280_769_230_770;
```

to 
```solidty
uint256 public constant EIGEN_stakers_weekly_distribution = 1_287_420_514_065_123_076_923_080;
```

This proposal is deliberately minimal. It adjusts only one parameter in the reward rate of incentives system, requiring no broader design or contract changes.

## Security Considerations
Both internal and external security teams will be consulted to confirm that this change introduces no new vulnerabilities or attack vectors. Since it is purely a numerical update, no risks are currently expected.

## Impact Summary

**EIGEN Token Holders:** Stronger incentives and higher rewards for staking.

**EigenDA:** Greater stake participation enhances its cryptoeconomic security.

**EigenVerify:** Increased stake bolsters its resilience.

**Other AVSs:** More EIGEN stake becomes available to secure additional services.

**Other Effects:** Total reward incentives rise from 4% to 7%, increasing the circulating supply more rapidly and potentially impacting market dynamics.

## Action Plan


1. Implementation Phase
- Modify contract parameters
- Add and adapt tests
- Conduct internal review

2. Review Phase
- Community feedback period
- (Optional) External audit, even for a simple parameter change. Publish and review audit report
- Apply final adjustments

3. Deployment Phase
- Internal testing
- Deploy to testnet
- Community testing period
- Mainnet deployment

## References & Relevant Discussions


[Programmatic Incentives v1](https://blog.eigencloud.xyz/introducing-programmatic-incentives-v1) and [v1 FAQs](https://docs.eigenfoundation.org/programmatic-incentives/programmatic-incentives-faq?utm_source=chatgpt.com)

[EigenLayer Improvement Proposal-001: Rewards v2](https://github.com/eigenfoundation/ELIPs/blob/main/ELIPs/ELIP-001.md)

[Forum discussion of ELIP-001](https://forum.eigenlayer.xyz/t/protocol-council-evaluation-elip-001/14348) and [[MERGED] ELIP-001: Rewards v2](https://forum.eigenlayer.xyz/t/merged-elip-001-rewards-v2/14196)

[Twitter post of proposal outline announcement](https://x.com/eigenfoundation/status/1952400791693897896)

[Forum post: Introducing Programmatic Incentives v2](https://forum.eigenlayer.xyz/t/introducing-programmatic-incentives-v2/14690)



