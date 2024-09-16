
EigenLayer is designed with upgradeable smart contracts, various adjustable parameters, and the ability to pause functionality. The ability and responsibility to make decisions regarding contract upgrades, pausing functionality, and changing parameters have initially been delegated to three main governance multisigs and the Protocol Council. 

## What’s an ELIP?
EigenLayer Improvement Proposals (ELIPs) are the primary mechanisms for proposing new features/upgrades to the EigenLayer core contracts. ELIPs are version-controlled design documents that detail the technical specification, rationale, implementation path, and impact evaluation of new features/upgrades. ELIPs are used to:

  1) Track progress while designing, building, and implementing new features
  2) Publicly communicate new features and create space for community input
  3) Propose new features to the Protocol Council for approval & execution

## ELIP Process
In the early stages, ideas for ELIPs and early drafts can be discussed on research.eigenlayer.xyz. After the design and scope of an ELIP has been finalized, it is submitted to the EigenLayer Improvement Proposals (ELIPs)  GitHub repository as a pull request. 

The PR is tracked publicly on the repository using *tags*:

`draft:` proposal is in pre-production, ready for audit
`review:` proposal is in testnet
`final:` proposal is ready to move to mainnet
`approved:` proposal has been approved by Protocol Council
`merged:` proposal is live on mainnet

The same tags are used to track the state of the ELIP in the Technical Proposals category on forum.eigenlayer.xyz. 

## Proposing to the Protocol Council
Once an ELIP has successfully reached the final stage and is ready to move to mainnet, the corresponding forum post title is updated to [FINAL], signaling the start of the Protocol Council’s final review process. The Protocol Council convenes to review the improvement proposal and ensure it meets the security expectations and criteria outlined below: 

### Mainnet Approval Criteria:

#### Security:
* The proposal has been thoroughly audited, with all reported issues of medium or higher severity either resolved or acknowledged.
* The proposal has undergone comprehensive testing.
* An operational plan, including automated testing and safety checks (e.g., Forge scripts), is in place.
* All potential risks have been identified and provably mitigated.
* An emergency plan exists to address unforeseen issues.

#### Strategic:
* The impact of the proposal has been fully considered and communicated to all relevant ecosystem stakeholders.
* The proposal minimizes complexity, encapsulates new features as much as possible, and maintains credible neutrality.
* The proposal does not compromise the overall security or reliability of the protocol.

#### Operational:
* The rollout plan is conservative and risk-aware, taking into consideration all potential impacts.
* The proposal has adhered to the established protocol governance process.
* The development of the proposal has been conducted transparently alongside the broader EigenLayer ecosystem.

Once the Protocol Council confirms that the ELIP meets the necessary criteria and is ready for execution, it will formally approve it by responding to the corresponding forum post. This response is expected to include a detailed evaluation of the proposal, outlining how it aligns with the established criteria. At this point, the forum post and PR can be updated to merge. This process will be updated with the launch of our Decisions product.

Once the report is published, the proposer queues the proposal on-chain in the timelock, and publishes a Transparency Report detailing relevant information regarding the upcoming changes and outlining execution details (e.g. Foundry scripts). After the report is published, the Protocol Council executes the queued operations in the timelock. 

## Who can submit ELIPs?

In this early stage, the Ops Multisig held by Eigen Labs and the Protocol Council are the only parties that can propose improvements to the EigenLayer core contracts. The Protocol Council will be the sole party that can execute/approve proposed improvements to the EigenLayer core contracts. To provide a check on the Protocol Council’s unilateral ability to execute upgrades, the Ops multisig will hold the role of canceller on the timelock and can cancel the approved proposals of the Protocol Council until this cancellar power is soon extended to EIGEN holders. The power to propose to the timelock will be further decentralized in the future.

