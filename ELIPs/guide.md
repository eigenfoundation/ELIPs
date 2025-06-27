# External

## Blog / Release Copy

Today, Service Builders are constrained on where they can launch their AVSs. Consumers of AVS services (apps and dapps) can only integrate on Ethereum if they want to inherit the strong security guarantees of EigenLayer. It is costly for AVSs to develop, maintain, and execute (gas) a multi-chain solution today as all of the relevant verification data is stuck on L1.

**Enter Multi-Chain Verification.** EigenLayer today uses the maximally secure, decentralized Ethereum L1 for stake accounting. We are launching a public Preview program that brings EigenLayer Operator Sets and stake weights to many target chains. Starting in July, AVSs will be able to launch their services multi-chain, meeting their customers where they are. New EigenLayer contracts will allow AVSs to verify and integrate their services on available chains, with the same protocol guarantees as on Ethereum. Alongside these, new middleware contracts are provided for AVSs to configure their multi-chain experience.

All AVSs have to do is opt-in to the program and choose a verification path from provided options, with out of the box defaults and deeper customizations available. The integration experience is consistent across all target chains, now and into the future, to remove duplicated work and simplify AVS go-to-market. Our goal is to make it easier than ever to launch verifiable services across web3. 

The Preview program is targeting Testnet in July and Mainnet later this year. The EigenLayer multi-chain protocol will be free to use during the Preview. We want to encourage developers to build and test new service and app integrations across chains to get to market fast. We will support Base, Mainnet, and testnets (Sepolia and Base Sepolia) to start, with plans to expand supported chains over time. Stake data will be updated once a week by the protocol during the preview, with more customizability available to AVSs later on with flexibility around gas cost and functionality. Certain updates like a slash or ejection of an Operator will trigger immediate updates across chains to ensure consumer safety. 

Even though this is a Preview, the live stake data is verifiable against Testnet (Sepolia) and Mainnet stake values. Service builders should have the confidence and comfort to explore downstream customer integrations with *real EigenLayer stake against which to verify their services and apps.* For the Preview period, EigenLabs will be running the infrastructure to generate and transport stake data to target chains. This data will be verifiable against Ethereum stake data stored in Core Contracts. 

EigenLayer Multi-Chain Verification is built atop open standards built into new core contracts and middleware. The standards-based model makes uniform existing AVS multi-chain solutions and accelerates net-new ones. Builders can easily and cheaply launch and consume verifiable services with a code-once, deploy-anywhere integration across their supported chains. Projects on Layer 2s can integrate AVSs into their protocols with minimal additional trust assumptions and low dev and gas costs. 

After Opting-In, Operator Set information is transported by EigenLabs to each supported chain. Customization options are available to AVSs for:

- Stake weighting of different security assets according to the needs of the AVS,
- Verification conditions for Operator outputs, like accepting outputs signed over a proportional or nominal amount of stake weight.

We aim to keep integrations simple, yet flexible. We opt for a standards-based approach to enable modular AVS architectures where possible. You can learn more about the verification interfaces in the ELIP, in our integration guides, and docs. This `CertificateVerifier` is the key entry point for AVSs to customize the integration experience of their services for their apps and users. You can learn about the multi-chain stake weighting here, and how AVS builders can decorate and customize verification components to suit their use-case. 

We are excited to see the creativity that is unlocked when AVSs can integrate with new chains and new customers! As with any Preview, we are very keen to hear your direct feedback. Contact us today to get involved in the Preview program and launch your AVS multi-chain!  

*Disclaimer: This experience is intended for testnet and mainnet production consumption. All features and terms may change over time.* 

---

## Integration Guide

EigenLayer has launched **multi-chain verification** for its AVS developers! This new feature let's developers verify their services and applications on supported chains with the same trust and security of restaked assets on Ethereum. For an overview of the feature and preview program, read our announcement blog here.

---

### **1. What you need to know —**

| **Concept** | **Why it matters to you** | **Where it lives** |
| --- | --- | --- |
| **Stake still originates on Ethereum** | Stake guarantees and protections still apply. Security and slashing rules are unchanged. Secured by Ethereum consensus. | Core contracts on L1 |
| **Four new core contracts** | - `KeyRegistrar` – canonical key store (ECDSA / BN254)
- `CrossChainRegistry` – opt-in & config hub for multi-chain
- `OperatorTableUpdater` – ingests stake root on each target chain
- `CertificateVerifier` – *single* integration surface for every app | L1 only:
- `Key Registrar`
- `CrossChainRegistry` 

Supported Target Chains (incl. L1): 
- `OperatorTableUpdater`
- `CertificateVerifier` |
| **One new middleware contract; modifications to others** | - `OperatorTableCalculator` — new, templated contract for decorating stakes with weighting logic; deployed by the AVS.

Connected to the `CrossChainRegistry` via an opt-in registration. | AVS contract deployed on Ethereum only |
| **Operator Table** | Merklized stake weights for *your* OperatorSet (optionally, custom-weighted). 

Created from calling the `OpeatorTableCalculator` with the correct expected return types. | Calculated off-chain by calling your `OperatorTableCalculator`. 

Lives in ever `CertificateVerifier` deployed to each target chain. |
| **Global Stake Root** 
(aka global confirmation root) | Bundles all Operator Tables into a merkle tree / root for simpler transport to target chains; transported weekly (or instantly on slash/eject). | Posted to every target chain by EigenLabs during the Preview 

Generated (and verifiable) by the Eigen Sidecar  |
| **Verification is done using Operator Certificates** | AVSs and their customers / consumers will use certificate-based verification. 

Certificates signed by Operators are verified against the Operator Table in the `CertificateVerifier` contract against default or custom stake weighting rules. | - `CertificateVerifier` contract  |
| **Curves** | Use **ECDSA** for ≤ ~50 operators; **BN254 BLS** for larger sets. 

More curves anticipated over time. | Same APIs in KeyRegistrar & CertificateVerifier |
| **Preview parameters** | - Chains: Ethereum, Base, Sepolia, Base-Sepolia  
- Stake Refresh: weekly, or for certain events  
- Cost: free | Will evolve post-Preview |

*LINK TO Deeper docs: contract ABIs, ELIP-008 spec, Sidecar transporter repo, etc.*

### **How It Works: System Overview**

The multi-chain verification system follows this flow:

1. **Stake Calculation** (L1): Your `OperatorTableCalculator` decorates raw EigenLayer stakes with custom weighting logic
2. **Global Root Generation** (Off-chain): EigenLabs combines all AVS Operator Tables into a merkle tree with global root  
3. **Transport** (Cross-chain): The global root is posted to each target chain's `OperatorTableUpdater`
4. **Verification** (Target chains): Apps use `CertificateVerifier` to verify operator certificates against transported stake tables

---

### **What you need to do —**

#### **For AVS Builders**

| # | **Action** | **Required/Optional** | **Command / Code Snippet** |
| --- | --- | --- | --- |
| **0. Prep** | Decide cryptographic curve & build an OperatorSet (list of operator addresses). | **Required** | Choose ECDSA (≤50 operators) or BN254 BLS (>50 operators) |
| **1. Deploy Table Calculator** | Deploy `OperatorTableCalculator` contract to define stake weighting logic. | **Required** | **Simple**: Deploy `ECDSATableCalculatorBase` or `BN254TableCalculatorBase` as-is for unweighted stakes.<br/><br/>**Advanced**: Override `calculateOperatorTable()` to add:<br/>- Asset weighting (ETH 2x vs stablecoins)<br/>- Stake capping per operator<br/>- Oracle price feed integration<br/>- Custom filtering logic |
| **2. Configure Keys** | Register cryptographic keys for your OperatorSet. | **Required** | **Either**: Operators self-register via `KeyRegistrar.registerKey(operator, operatorSet, pubkey, sig)`<br/><br/>**Or**: AVS admin registers via UAM permissions |
| **3. Opt-in to Multi-Chain** | Register with `CrossChainRegistry` to enable multi-chain verification. | **Required** | `CrossChainRegistry.createGenerationReservation(operatorSet, calculator, config, [chainIDs])`<br/><br/>**Config**: `staleness = 14 days` (must exceed 7-day refresh), `minWeight = 0` |
| **4. Select Target Chains** | Choose which chains to deploy verification to. | **Required** | `CrossChainRegistry.addTransportDestinations(operatorSet, [1, 8453, 11155111, 84532])`<br/>*Ethereum, Base, Sepolia, Base-Sepolia* |
| **5. Wait for Deployment** | EigenLabs generates and transports your stake table. | **Automatic** | Monitor `OperatorTableUpdater.GlobalRootConfirmed` for confirmation |
| **6. Design Integration Pattern** | Choose how your customers will consume your service. | **Required** | **Options**:<br/>- Direct: Customers call `CertificateVerifier` directly<br/>- Wrapped: Deploy custom contract wrapping `CertificateVerifier`<br/>- Hybrid: Offer both patterns |

#### **For Operators**

| # | **Action** | **Required/Optional** | **Command / Code Snippet** |
| --- | --- | --- | --- |
| **1. Register Keys** | Register cryptographic keys for each OperatorSet you join. | **Required** | `KeyRegistrar.registerKey(myAddress, operatorSet, pubkey, signature)`<br/><br/>**Key Types**: ECDSA address or BN254 G1/G2 points |
| **2. Implement Certificate Generation** | Update your operator binary to produce certificates. | **Required** | **ECDSA**: `{referenceTimestamp, messageHash, concatenatedSigs}`<br/><br/>**BLS**: `{referenceTimestamp, messageHash, aggregateSignature, aggregatePublicKey, nonSignerWitnesses}` |
| **3. Monitor Key Health** | Watch for key rotation needs and ejection events. | **Required** | Monitor `AllocationManager.OperatorSlashed` and rotate keys as needed |

#### **For Consuming Applications**

| # | **Action** | **Required/Optional** | **Integration Pattern** |
| --- | --- | --- | --- |
| **1. Choose Integration Model** | Decide how to consume AVS services. | **Required** | **Pull Model**: Request task → Operators fulfill → Verify certificate on-demand<br/><br/>**Push Model**: Operators publish certificates → You pull cached results → Verify freshness<br/><br/>**Hybrid**: Use cached aggregated results + request fresh data when needed |
| **2. Choose Verification Method** | Select verification approach based on your trust requirements. | **Required** | **Direct**: Call `CertificateVerifier` functions directly<br/><br/>**AVS-Wrapped**: Use AVS-provided verification contract<br/><br/>**Custom-Wrapped**: Add your own logic around `CertificateVerifier` |
| **3. Implement Verification Logic** | Add certificate verification to your application. | **Required** | **Proportional**: `CertificateVerifier.verifyCertificateProportion(operatorSet, cert, [6600])` // ≥66%<br/><br/>**Nominal**: `CertificateVerifier.verifyCertificateNominal(operatorSet, cert, [1000000])` // ≥1M units<br/><br/>**Custom**: `(bool valid, uint256[] memory weights) = CertificateVerifier.verifyCertificate(operatorSet, cert)` then apply custom logic |
| **4. Handle Certificate Sources** | Implement certificate acquisition based on your chosen model. | **Required** | **Pull**: Call AVS operator endpoints directly<br/><br/>**Push**: Query AVS caches, IPFS, or other storage<br/><br/>**Hybrid**: Try cache first, fallback to fresh requests |
| **5. Monitor Verification Health** | Track certificate validity and stake table freshness. | **Required** | Monitor `CertificateVerifier.StakeTableUpdated` and implement staleness checks |

### **Integration Models Explained**

#### **Pull Model (Request-Response)**
```solidity
// 1. Consumer requests task from operator
TaskRequest memory task = TaskRequest({data: inputData, deadline: block.timestamp + 1 hours});
bytes memory result = operator.performTask(task);

// 2. Operator responds with certificate
Certificate memory cert = abi.decode(result, (Certificate));

// 3. Consumer verifies immediately
bool isValid = certificateVerifier.verifyCertificateProportion(operatorSet, cert, [6600]);
require(isValid, "Insufficient stake backing");
```

#### **Push Model (Cached Results)**
```solidity
// 1. Query cached certificate (from AVS contract, IPFS, etc.)
Certificate memory cachedCert = avs.getLatestResult(taskType);

// 2. Check certificate freshness and validity
require(block.timestamp - cachedCert.referenceTimestamp < MAX_STALENESS, "Certificate too old");
bool isValid = certificateVerifier.verifyCertificateProportion(operatorSet, cachedCert, [5000]);
require(isValid, "Insufficient stake backing");

// 3. Use cached result
processResult(cachedCert.messageHash);
```

#### **Custom Verification Logic**
```solidity
// Get raw stake weights for custom logic
(bool validSigs, uint256[] memory weights) = certificateVerifier.verifyCertificate(operatorSet, cert);
require(validSigs, "Invalid signatures");

// Apply custom business logic
uint256 totalStake = 0;
uint256 validOperators = 0;
for (uint i = 0; i < weights.length; i++) {
    if (weights[i] >= MIN_OPERATOR_STAKE) {
        totalStake += weights[i];
        validOperators++;
    }
}

// Custom requirements: need both 60% stake AND 3+ operators
require(totalStake * 10000 >= getTotalOperatorSetStake() * 6000, "Need 60% stake");
require(validOperators >= 3, "Need 3+ qualified operators");
```

### **Understanding Certificate Structures**

Certificates are signed attestations from operators that you verify against stake tables:

**ECDSA Certificate** (for smaller operator sets ≤50):
```solidity
struct ECDSACertificate {
    uint32 referenceTimestamp;  // When certificate was created
    bytes32 messageHash;        // Hash of the signed message/task result
    bytes sig;                  // Concatenated operator signatures
}
```

**BLS Certificate** (for larger operator sets >50, more efficient):
```solidity
struct BN254Certificate {
    uint32 referenceTimestamp;  // When certificate was created
    bytes32 messageHash;        // Hash of the signed message/task result
    BN254.G1Point signature;    // Aggregate signature
    BN254.G2Point apk;         // Aggregate public key
    BN254OperatorInfoWitness[] nonSignerWitnesses; // Proof of non-signers
}
```

### **Stake Weighting Examples**

Your `OperatorTableCalculator` can implement various weighting strategies:

```solidity
// Simple: Equal weighting of all stake
function calculateOperatorTable(OperatorSet calldata operatorSet) 
    external view returns (ECDSAOperatorInfo[] memory) {
    // Return raw stake values without modification
    return getRawStakeValues(operatorSet);
}

// Advanced: Custom weighting with asset preferences
function calculateOperatorTable(OperatorSet calldata operatorSet) 
    external view returns (ECDSAOperatorInfo[] memory) {
    ECDSAOperatorInfo[] memory operators = getRawStakeValues(operatorSet);
    
    for (uint i = 0; i < operators.length; i++) {
        // weights[0] = ETH stake, weights[1] = stablecoin stake
        operators[i].weights[0] *= 2;  // Weight ETH 2x higher
        operators[i].weights[1] *= 1;  // Keep stablecoins at 1x
        
        // Cap any single operator at 10% of total
        uint256 maxWeight = getTotalStake() / 10;
        if (operators[i].weights[0] > maxWeight) {
            operators[i].weights[0] = maxWeight;
        }
    }
    return operators;
}
```

### **System Parameter Reference**

#### **Mutable Parameters (What to Monitor)**

| **Parameter** | **Controlled By** | **Update Frequency** | **Impact** | **Monitoring Event** |
| --- | --- | --- | --- | --- |
| **Operator Tables** | EigenLabs (Preview) | Weekly + force updates | Certificate verification validity | `CertificateVerifier.StakeTableUpdated` |
| **Operator Keys** | Operators + AVS Admins | On-demand | Certificate signature validation | `KeyRegistrar.KeyRegistered/Deregistered` |
| **Stake Weights** | `OperatorTableCalculator` | Per table update | Verification thresholds | Custom events in your calculator |
| **Operator Registration** | AVS + Operators | On-demand | Available operators for tasks | `AVSRegistrar.OperatorRegistered/Deregistered` |
| **Slashing/Ejections** | EigenLayer Core | On-demand (immediate transport) | Operator validity and weights | `AllocationManager.OperatorSlashed` |

#### **Immutable Parameters**

| **Parameter** | **Set By** | **Description** |
| --- | --- | --- |
| **Operator Set ID** | AVS | Cryptographic curve and operator list hash |
| **Contract Addresses** | EigenLayer Core | `CertificateVerifier`, `OperatorTableUpdater` addresses per chain |
| **Chain Support** | EigenLayer Core | Which chains support multi-chain verification |

#### **Configurable Parameters**

| **Parameter** | **Configured By** | **Options** |
| --- | --- | --- |
| **Staleness Period** | AVS | 1-30 days (must exceed 7-day refresh) |
| **Minimum Stake Weight** | AVS | Any uint256 value |
| **Target Chains** | AVS | Any supported chain IDs |
| **Verification Thresholds** | Consuming Apps | Proportional % or nominal amounts |
| **Custom Stake Weighting** | AVS | Override `calculateOperatorTable()` with any logic |

### **Security Considerations**

**Key security aspects to consider when implementing:**

| **Risk** | **Mitigation** | **Implementation** |
| --- | --- | --- |
| **Stale Stake Data** | Configure appropriate staleness periods | Set `staleness > 7 days` in your `OperatorSetConfig` |
| **Key Compromise** | Monitor for operator ejections and key rotations | Listen for `AllocationManager.OperatorSlashed` and `KeyRegistrar.KeyDeregistered` |
| **Insufficient Stake** | Set minimum thresholds in verification | Use `verifyCertificateNominal()` with minimum stake requirements |
| **Operator Centralization** | Implement stake capping in your calculator | Cap individual operators at 10-20% of total weight |
| **Certificate Replay** | Check certificate freshness | Validate `referenceTimestamp` is recent and within staleness period |

**Emergency Procedures:**
- **Operator Ejection**: Immediately updates across all chains when operators are slashed/ejected
- **Pause Mechanisms**: System-wide pause capabilities for critical vulnerabilities
- **Key Rotation**: Operators can rotate compromised keys with configurable delays

### **Troubleshooting Tips**

| **Symptom** | **Likely Cause** | **Fix** |
| --- | --- | --- |
| `verifyCertificate…` returns `false` | Stake table is stale or wrong curve type | Check `referenceTimestamp`; refresh reservation; ensure operators registered correct curve. |
| Gas cost too high verifying sigs | Large OperatorSet using ECDSA | Switch to BN254 BLS calculator & certificates. |
| Operator keys missing on target chain | Key not in `KeyRegistrar` | Call `isRegistered()`; re-register & wait for the next table update. |
| Certificate verification fails with valid signatures | Operator not in current OperatorSet | Check operator registration status and OperatorSet membership |
| Custom verification logic errors | Incorrect stake weight interpretation | Use `verifyCertificate()` to inspect raw weights before applying custom logic |

---

### **Impact & Benefits**

**For AVS Builders:**
- **Expanded Market Access**: Serve customers on Base, Optimism, and other L2s without custom bridges
- **Reduced Development Overhead**: Single integration pattern across all supported chains  
- **Enhanced Security**: Maintain Ethereum L1 security guarantees across all deployments

**For App Developers:**
- **Simplified Integration**: One `CertificateVerifier` interface works across all AVSs and chains
- **Lower Gas Costs**: Verify services on cheaper L2s while keeping L1 security
- **Flexible Trust Models**: Choose between AVS-provided verification or implement custom logic

**For the Ecosystem:**
- **Network Effects**: Standardized interfaces reduce integration complexity between AVSs and apps
- **Innovation Acceleration**: Focus on core business logic instead of cross-chain infrastructure
- **Security Scaling**: EigenLayer's restaking security extends to the broader multi-chain ecosystem

### **Next Steps**

1. Clone the [contracts repo] and pull the **Preview** branch.
2. Add the "multi-chain" checklist above to your CI pipeline.
3. Join the #multichain-preview channel in Discord for support.

*Multi-Chain verification is early-access software – expect iterative updates before GA.*

For AVSs looking to take advantage of multi-chain right away, they will need to meet one (or more) of the following criteria:

- For task-based AVSs:
    - Deploy through DevKit for out-of-the-box integration and configuration via the included TOML file.
- For non task-based AVSs:
    - Deploy a compliant `OperatorTableCalcualtor` on Ethereum
    - Use one of the following means to interact with the `CertificateVerifier`:
        - If using the SDK to aggregate outputs, use the updated signature aggregation features on the SDK.
        - If using a bespoke aggregation mechanism, fit Operator Outputs to the anticipated input type of the `CertificateVerifier` certificates.
        - If Operator Output data and signatures are available onchain, configure integration directly with the stake weight verification functions on the `CertificateVerifier`.



---

# Internal

*Learn more about the technical specifics in the [MultiChain TDD Index](https://www.notion.so/MultiChain-TDD-Index-1f813c11c3e080c0882be0211a9ef256?pvs=21).*

The Preview period is measured in success by the number of multi-chain deployments and integrations for our customer's customers. The Preview releases these features without requiring coupled development of a multi-chain pricing/cost structure or Operator Set structure (and accompanying agreements with EigenDA OR Cloud Operators). This is so we can deliver value for customers in Q2.

## Multi-Chain Contract Architecture

We are shipping new core contracts to reduce complexity for AVS builders and keeping the number of deployed contracts low. This also allows us to hot-swap the back-end infrastructure to use EigenCloud when it is ready, improving DevEx without impacting existing customer integrations. t

AVS's only need to register in this cross-chain system (and their operators too) to get started. 

### Core L1

- `CrossChainRegistry` - Where AVS's register their stake decorations in Core; if nothing is specified, EigenLayer provides a default (non-weighted) *** TO BE EVALUATED
    - Contract for registering AVS-Specified `OperatorTableCalculators` and setting certain configs
    - Houses the global stake table for all AVSs; data is transported from here to destination chains and confirmed against the Operator Set
- `KeyRegistrar`
    - Contract for key material of Operators on L1, supporting key rotation
- `CertificateVerifier`
    - Stake weight verification: entire data structure, proportional, nominal
    - Houses the Operator Tables with decorated weights and key material for use in verification
        - `Certificates` - Operators produce certificates with their key material that can be single or aggregate. These are checked against the contract logic and `Operator Table`
- Public (permissionless) generation ABIs to verify the roots Labs posts and publishes
    - Enables trust but verify of centralized components on the Preview (can re-run the proofs and compare)

### Core L2 - one per supported chain

- `CertificateVerifier` - against which to check `Certificates`

### Vended Middleware

*Learn more about middleware updates and simplification in [Meta TDD: Middleware V2](https://www.notion.so/Meta-TDD-Middleware-V2-1f113c11c3e080aa8d39fa55fc3fc2c9?pvs=21).*

Multi-chain has simplified the middleware and reduced the number of contracts AVS's will need to deploy. We are updating existing required contracts (the `AVSRegistrar` ), moving into Core functionality previously in middleware (the `KeyRegistrar`), and adding the optional `OperatorTableCalculator` for decorating stake weights. 

- `AVSRegistrar v2`  - update to existing contract
    - Op set management / allowlisting
    - Socket Registry
    - *Key functionality has been moved into the core `KeyRegistar`*
- (L1 Only) `OperatorTableCalculator` - template deployed by AVS
    - AVS-customized decoration of stake weights
        - Support in Preview for: slashable stake vs delegated, multipliers
    - *todo: see how default / non-deployed case interacts with the `CrossChainRegistry`*
- (Hourglass tech) For Hourglass AVSs
    - `Aggregator` - integration with mailbox
        - Needed to support the verification of Operator work products, by aggregating signatures and comparing against the on-chain stake-weight in the `CertificateVerifier` and comparing against the Mailbox
        - *to explore: Hugely impactful for Compute GTM*
            - Many offering models possible

### Developer Tools

EigenLayer's existing tools will need to be updated to support the new system. 

**DevKit** 

- TO BE SCOPED - See https://lucid.app/lucidchart/19a5c679-73c6-419c-956d-03bdd4b25ab8/edit?invitationId=inv_db796687-8094-4e6b-a234-d63d997effa9&page=twtgB-w2a0mt# for dependencies and dates
    - Two dependencies:
        - DevKit: Middleware Integration + Hourglass
        - DevKit: Integrate with Cross-chain Registrar

**Operator CLI** 

- Support for registering Operator keys with the `KeyRegistrar` and rotating them.

**AVS SDK** 

- Support for signature aggregation into a format acceptable by the `CertificateVerifier`

---

# AVS Guide

## AVS Multi-Chain Role & Responsibilities

AVSs have to deploy and configure the following on-chain contracts to successfully leverage multi-chain. 

1. Deploy middleware on L1 and configure (can upgrade existing contracts)
    - `AVSRegistrar`
    - `OperatorTableCalculator`
        - Decide how stake weighting works for them and decorating the raw weights from the EigenLayer contracts (i.e. stables are 1.5x in security vs alt coins)
    - Configure permissions in UAM
    - Configure strategies/operatorSets
    - Configure slashing/rewards on the L1 (or via message bridging if on L2)
    - `Aggregator` integration for Hourglass AVSs
2. Design integration patterns built atop the `CertificateVerifier` 
    - AVSs will design and communicate in their go-to-market how to integrate;
        - Leveraging nominal, proportional, or wrapping the entire stake weighting in another contract if more customization is needed
        - The pattern is the same across any supported chain - set-and-forget by the AVS builder / app builder
        - The pattern is the same across AVSs in the default case, streamlining app consumption
    - *App builders can call the `CertificateVerifier` themselves, to verify AVS work or set up custom integration logic if they desire or when AVS's target a self-serve solution for their customers*
3. Set permissions in the `PermissionController` to ensure the proper security posture
4. Configure rewards on L1; this cannot be done on destination chains 

## Migration Story

AVS migration story for middleware support is to redeploy against the new versions. These are forward-compatible with existing middleware modules. This is not ideal, but we will provide significant documentation toward the newly, substantially streamlined middleware. 

**Middleware**

- The current Middleware Contracts will require an upgrade & net-new contract deploy to use multichain functionality
    - Already existing AVSs will have to upgrade their `BLSKeyRegistry` to be compatible with the `OperatorTableCalculator`
    - AVSs can also begin a gradual migration from their own key registry to the core key management contracts when ready
- `OperatorTableCalculator` is a singleton contract that is compliant with the new AVS architecture
    - Existing AVSs will have to make callbacks into the `StakeRegistry` as opposed to the `AllocationManager`
    - This contract requires key material to function
    - The AVS sets the `KeyRegistry` in their `OperatorTableCaulculator`; the AVS can specify the new `KeyRegistrar` core contract or their existing key registries like the `BLSKeyRegistrar`.

**Standards**

- Existing multi-chain solutions from customers may be updated to adhere to our proposed standards in order to meet the single integration pattern and long-term management of their own stake tables. This allows them to user the `CertificateVerifier` with minor customizations (i.e. the stake table)

**Aggregator** 

- If an AVS wants to use existing aggregation solutions or update theirs, the need to fit these expected inputs on the `CertificateVerifier` —>  [Certificate Verification - ECDSA](https://www.notion.so/Certificate-Verification-ECDSA-1f513c11c3e08004b9fef0f563eac249?pvs=21)

---
