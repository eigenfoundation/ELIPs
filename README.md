# EigenLayer Improvement Proposals (ELIPs)

This repository tracks EigenLayer Improvement Proposals. These are discrete units of protocol governance outlining improvements to EigenLayer [core contracts](https://github.com/Layr-Labs/eigenlayer-contracts?tab=readme-ov-file#deployments).

## What’s an ELIP?
EigenLayer Improvement Proposals (ELIPs) are the primary mechanisms for proposing new features and upgrades to the EigenLayer core contracts. ELIPs are version-controlled design documents that detail the motivation, technical specification, rationale, implementation path, and impact evaluation of specific proposals impacting core contracts. ELIPs are used to:

* Track progress while designing, building, and implementing new features
* Publicly communicate new features, designs, and create space for community input
* Propose new upgrades for approval & execution

The following are examples of actions that require an ELIP:

* Upgrades/modifications to EigenLayer core contracts
* Modifications to multisig governance architecture (i.e. transferring ownership / admin access)
* Modifying minting rights over the bEIGEN token

Once proposals have passed through the ELIP process, they are submitted to the Protocol Council for final approval and execution. The Protocol Council will execute all EigenLayer Improvement Proposals (ELIPs) that have been approved.

## ELIP Process

In the early stages, ideas for ELIPs and early drafts are discussed in the "ELIP" category on [forum.eigenlayer.xyz](https://forum.eigenlayer.xyz/). After the design and scope of an ELIP has been finalized, it is submitted to this repository as a pull request. To submit an ELIP, open a [pull request](https://github.com/eigenfoundation/ELIPs/pulls). Ongoing and accepted proposals can be found in /elips. 

As the ELIP is developed, the corresponding PR is tracked publicly on the repository using **tags**. Each tag designates a stage of ELIP development. To move from stage to stage, ELIPs must meet specific requirements laid out below. Maintainers of the ELIP repository are responsible for 1) verifying that ELIPs meet the set requirements and 2) updating the PR tags as such. 

idea = pre-ELIP stakeholder feedback on potential upcoming features, unstructured 
gates: executive summary, motivation, some of the interfaces
Forum Post + Draft PR / Branch

draft = requirements, interfaces, 
gates: need all ELIP fields filled out (no TBDs)
Accepted PR - 1 or 2 maintainers need to accept & merge PRs
Update tag on Forum

testing = proposal is in testnet, ready for audit
gates: proposal is in testnet, ready for audit
	Update tag on Forum

council review = proposal is ready to move to mainnet, audited [time-boxed]

approved = proposal has been approved by Protocol Council 

ONCE Protocol Council approves ELIP, THEN the ELIP is queued.

merged = proposal has gone live on mainnet

* `idea`: *pre-ELIP proposal is ready for ecosystem feedback*
    Requirements: 
        * Executive Summary
        * Motivation for Changes
        * Interfaces (if any)
* `draft`: *proposal is in pre-production, ready for audit*
    Requirements:
        * All fields in ELIP template are completed (no TBDs)
        * Forum Ppost is published
* `testing`: *proposal is in testnet*
    Requirements:
        * ELIP is in testnet
        * ELIP is ready for audit
        * Corresponding forum post is updated
* `final`: *proposal is ready to move to mainnet (see Proposing to the Protocl Council)*
    Requirements:
        * ELIP has been audited, results published
        * ELIP is ready for mainnet
        * ELIP is ready for Protocol Council review
        * Corresponding forum post is updated
* `approved`: *proposal has been approved by Protocol Council*
    Requirements:
        * ELIP has been approved by Protocol Council
        * Protocol Council has published evaluation on forum
        * Corresponding forum post has been updated
* `merged`: *proposal has gone live on mainnet*

The same tags are used to track the state of the ELIP in the Technical Proposals category on [forum.eigenlayer.xyz](https://forum.eigenlayer.xyz/).

## Proposing to the Protocol Council

Once a proposal has successfully reached the final stage and is ready to move to mainnet, the forum post title is updated to [FINAL], signaling the start of the Protocol Council’s final review process. The Chair convenes the Protocol Council to review the improvement proposal and ensure it meets security expectations. Once an ELIP is moved to `final`, the Protocol Council has 10 days to convene and evaluate the ELIP. 

After evaluating the ELIP, the Protocol Council formally communicates its decision by publishing an evaluation on [forum.eigenlayer.xyz](https://forum.eigenlayer.xyz/). Coordinated by the Council Chair, this response is expected to include a detailed evaluation of the proposal and overview of the Council’s decision-making process. If the Council **approves the ELIP**, the forum post and PR is updated to `approved` by repository maintainers. If the Council **denies the ELIP**, it is reverted to the `testing` stage for further iteration if applicable or rejected entirely.

**Once the evaluation is published, the proposer queues the proposal on-chain in the timelock, and publishes a Transparency Report detailing relevant information regarding the upcoming changes and outlining execution details (e.g. Foundry scripts)**. Operations should not be queued to the timelock before the Protocol Council has communicated their decision. After the timelock delay expires, the Protocol Council executes the queued ELIP operations. Once the proposal is live on mainnet, the forum post is updated to merged and the PR can be merged on GitHub. In the event that the proposer does not have adequate on-chain permissions to queue the proposal itself, the Protocol Council queues the necessary operations in addition to executing them. Once the proposal is live on mainnet, the forum post is updated to merged and the corresponding PR is merged on GitHub.

## Who can submit ELIPs?
In this early stage, [Eigen Foundation](https://eigenfoundation.org/) and [Eigen Labs](https://www.eigenlabs.org/) are the only entities that can accept proposals for improvements to the EigenLayer core contracts however, anyone can suggest improvements and start discussions on [forum.eigenlayer.xyz](https://forum.eigenlayer.xyz/). Suggestions are reviewed, prioritized, and incorporated into ELIPs according to available resources.

The Protocol Council will have the sole power to execute/approve proposed improvements to the EigenLayer core contracts. To provide a check on the Protocol Council’s unilateral ability to execute upgrades, the Eigen Labs multisig ([Operations Multisig](https://docs.eigenfoundation.org/protocol-governance/technical-architecture) holds the role of canceler on the timelock and can cancel the approved proposals of the Protocol Council. The power to propose/veto transactions in the timelock will be further decentralized in the future. 

## Who manages the ELIP process?
The [Eigen Foundation](https://eigenfoundation.org/) maintains the ELIP repository and process. 
