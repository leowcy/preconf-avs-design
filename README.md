# Preconfirmation AVS Design

## 📋 Overview
Design an Active Validator Service(AVS) on EigenLayer that provides credible preconfirmations for Eth transactions.

---

## 🎯 Target
- Low lantency: Offer a solution with ~100ms processing time.
- Highly secure: Make sure the preconfirmation AVS service is credible to both user and validator. 
- Incentive: Include reasonable mechanism to reward/punish validators' activites.
- Compatibility: Can be easily integrated with L1 networks and reuse some features from Eigenlayer restaking.

---

## 🎯 Objectives
- [System Architecture](#-system-architecture)
- [Economic Model](#-economic-model)
- [Security Analysis](#-security-analysis)
- [Implementation Considerations](#-implementation-considerations)

---

## 💡 Background

- High level of Users/L1 network/AVS+EigenLayer(L2) diagram. (Happy path)

![Diagram](references/user_l1_avs.png "User L1 AVS relationship")

---

## 💡 System Architecture
- **Protocol design**:
    - High level flow chart for AVS preconf service:

    ![Diagram](references/preconf-highlevel-design.png "preconfirmation avs highlevel")

    0. Eth's validators register and join preconfirmation AVS service.
    1. User send a pre-conf request to a "trusted matchmaker" preconf-share (Similar to Flashbot MEV-Share) with specific requirements. (block inclusion guarantees, timing preferences, etc) This step can be done by submitting a request via a wallet, application or API call. 
    2. Preconf-share will collect the requests and send to the upcoming proposer/preconfirmer. It servers as a matchmaker and can based on user's preference(target slot or max latency) to rank the promise and help user to select the most suitable one.
    3. Validator(via preconf-operators) evaluate the request and issue signed preconfirmation promises.
    4. Preconf-share ranks  the received promises and return the best one to user.
    5. A commit boost here will request the L1 block and ensures preconfirmed transactions are prioritized and included in the block.
    6. The chosen preconf-operator includes the transaction into the promised block and return it.
    7. Commit boost propose the L1 block to network.
    8. AVS manager monitors the response, ensuring the transaction is included and reports any violations. Also deliver the rewards or slashing for any violation.


- **Key entities and roles**:
    1. Preconf-share (offchain service)
        - Serve as a bridge between users and validators, distributing user requests and ranking preconfirmation promises.
        - Receive user request and validate feasibility.
            - Checking the tip
            - Target block
            - Constraints
        - Sends requests to the relevant Preconf-Operators (validators for the next few blocks).
        - Ranking responses:
            - Collects preconfirmation promises (signed by Preconf-Operators) and ranks them based on user-defined criteria(tip size/target block/latency)
        - Returning the Best Promise to user.
    2. Preconf-operator (offchain service)
        - The validator node that interacts with the Preconf-Share system and handles preconfirmation logic.
        - Listen for event stream from prconf-share.
        - Evaluation.
        - Sign promise if feasible.
            - Includes the transaction hash, target block, tip, timing and etc.
        - Execution
            - Ensure the transaction is included in the promised block.
    3. AVS manager (Onchain smart contract)
        - Tracking and monitoring
            - Ensure all transactions are fulfilled.
            - Handle fallback scenarios if promise cannot be fulfilled.
        - Rewards and Slashing
            - Distributed rewards
            - Slashing for violations

- **Preconfirmation considerations**:
    - How should operators create and sign preconfirmations?
        -  Transaction details should include:
            - Transaction hash
            - Target block number
            - Validity window (time or block range)
            - Incentive(tips)
            - Signature
            - Timestamp
        - Validate feasibility
            - Gas estimation. Tx won't exceeding gas limits.
            - Block slot availability. Consider other pending transactions or perconfirmations.
            - Incentive evaluation. Check if tip is sufficient.
            - Transaction validity. Validate tx's format and correct nonce or valid signature.
        - Construct preconfirmation
            - Return an object back with all required details.
            - Sign the preconfirmation. If operator fails to fulfill the promise, this signed preconfirmation can serve as evidence for slashing penalties.
    
    - What mechanisms can we use to verify preconfirmations?
        - Verify the signature that has not been tampered with.
        - The preconfirmation promise contains a hash of the transaction. Compare and verify it.
        - Add a pre-confirmer role to verifying preconfirmation promises.
        - Fault proof mechanisms like Optimistic Rollups for broken promises.
    
    - What mechanisms handle preconfirmation conflicts and revocations?
        - Priority based resolution. Resolves conflicts by ranking preconfirmations based on:
            1. Higher tips
            2. Ealier timestamp if sharing the same tips
            3. User preference
        - Revocations:
            1. If the revocation is within a validity window(before block building phase starts) and no malicious intent, no penalty will required and updated logs for revocation and notified users.
            2. If the revocation is malicious intent and excessive revocations from the validator, then they will lose a portion of their stake as a penalty.
    
    - What happens to preconfirmations during chain reorganizations?
        - If the preconfirmation specifies a range of valid blocks (N to N+2), the promise may still be fulfilled in subsequent blocks.
        - These transactions are typically returned to the mempool unless otherwise excluded or replaced.
        - Applied dynamic solutoin to this: 
            - Transactions from reorged blocks are reprioritized in the mempool to ensure they are included in the replacement blocks.
            Validators assign higher priority to reorged preconfirmed transactions to maintain system integrity.

    - Since proposers are known of 32 blocks in advance, thus a reasonable window should balance predictability, security and useability. Should consider followings:
        - User Expectation. Are they looking for quick feedback? If so, preconfirmation should target next few blocks.
        - Based on use cases. Short-term(1-5 blocks) provides low latency and high reliability. Long-term(11-32 blocks ahead) will be best suited for future transactions that are not time-critical.
        - Implement a dynamic window based on network conditions. (congestion, reorg risks)
            - Shorter window during high congestion
            - Longer window during low activity periods
    
    - Preconfirmation requests need to be sent well before the 8-second build time starts; otherwise, builders won't have enough time to include them in the block. To guarantee inclusion, users should aim to negotiate preconfirmations at least 12-16 seconds before the target block time.
    - Propagation Delays affect:
        - Ensure low-latency communication between the preconfirmation system and the block builder.
        - Use private relays (like Flashbots' system) to guarantee quick and secure delivery of preconfirmed transactions.

- **State management approach**:
    - Preconfirmation request state:
        - [Pending, Confirmed, In Progress, Completed, Expired, Revoked]
    - Validator state:
        - [Active, Under Review, Suspended, InActive]
    - Store state on Layer 2 for scalability, while syncing critical data to Layer1 for transparency.

---

## 💡 Economic Model
- **Design of penalties for safety violations**:
    - Malicious Behavior:
        - Intentional actions that disrupt the system or harm users.
        - Larger penalties, up to complete slashing. Also, depends on the amount of times violations.
        - Temporarily suspended from participating in the preconfirmation system if applicable.
        - Stake higher collateral for future participation. For example, A validator with two violations must double their collateral to remain active.
    - Incorrect Execution:
        - Validator fails to include a preconfirmed transaction in the promised block or block range.
        - Providing false preconfirmation promises (promising a transaction that cannot fit within gas limits or block capacity).
        - Minor violations result in smaller penalties.

- **Design of penalties for liveness violations**:
    - Slashing
        - The severity of slashing depends on the frequency and magnitude of the liveness violation. For example, A validator fails to respond to a preconfirmation request within the required timeframe and loses 0.1% of their staked collateral.
    - Rewards reduction
        - Reduces the validator’s earned rewards (transaction fees, tips, or block rewards) for failing to maintain liveness.
    - Incremental Penalties for Repeated Violations
        - Each subsequent liveness violation within a set timeframe results in progressively higher penalties.
    - Lose future chance
        - Validators who violate liveness requirements are excluded from participating in future opportunities (proposing blocks, earning rewards).
    - User Compensation
        - Penalties collected from liveness violations are redistributed to users whose transactions or requests were delayed.

- **Fee Market Design**: 
    - Base Fee:
        - network wide fee applies to all transactions. Will dynamically adjust based on network congestion.
    - Preconfirmation Fee:
        - Paid by user to incentivize validators to prioritize their transactions. A user can attach a higher tip to ensure their transaction is included faster.
    - Penalty Fee:
        - Compensates validators for issuing signed promises and bearing the risk of slashing if promises are broken.

- **MEV considerations**:
    - MEV (Maximal Extractable Value) refers to the additional value that validators or block proposers can extract by reordering, including, or excluding transactions within a block. Managing MEV effectively is critical for ensuring fairness, reducing economic centralization, and maintaining user trust.
    - Use of Inclusion Lists. Prevents censorship and ensures preconfirmed transactions are included regardless of MEV opportunities.
    - Commit-Reveal Schemes. Makes it difficult for colluding parties to act on sensitive information ahead of time.
    - PBS (Proposer-Builder Separation) Separates block construction (by builders) from block proposing (by validators).

- **Collusion prevention mechanisms**:
    - Slashing for Collusion
        - Penalizes validators who engage in collusion by slashing their staked collateral.
        - A system smart contract monitors validator behavior and slashes stakes if collusion is detected(conflicting promises or excluded preconfirmed transactions).
    - Reward redistribution
        - Redirects rewards from colluding participants to honest validators and users.
    - Applies increasingly severe penalties for repeated collusion attempts.
---

## 💡 Security Analysis
- **Attack vectors**:
    - User level: 
        - Attackers reuse preconfirmation promises or transactions across multiple blocks or chains.
        - Includes unique identifiers (nonces, timestamps) in preconfirmation promises to prevent reuse.
    - Validator level:
        - Collusion: Validators collaborate to manipulate transaction inclusion, reorder transactions, or censor specific transactions.
        - Validators issue conflicting preconfirmation promises for the same transaction or block, creating ambiguity.
    - MEV Exploitation: See above.
        - Commit-Reveal Schemes.

- **Frontrunning or gaming**:
    - Use cryptographic techniques (commit-reveal schemes) to protect sensitive transaction data in preconfirmation requests.

---

## 💡 Implementation Considerations
1. **Builder integration**:
    - Ensure that preconfirmed transactions are included in the promised order or range.
    - Validate the signed preconfirmation promises attached to transactions.
    - Financial incentives to prioritize preconfirmed transactions without compromising on MEV opportunities.
    - Communicate constructed blocks to validators efficiently. Use protocol PBS to enable validators to choose best builder-proposed blocks.
    - For integrating with EigenLayer's restaking system, we should utilize restaked ETH or other tokens as collateral to guarantee the performance and accountability of validators participating in the AVS.
2. **Network timing constraints**:
    - Preconfirmation systems must ensure that validators can process promises and include transactions within ~12s.
    - Use peer-to-peer propagation protocols optimized for low-latency communication.
    - Use high-performance infrastructure (low-latency APIs) to handle requests.
    - Process multiple preconfirmation requests in batches to reduce computational overhead and improve response times.
3. **Scalability approach**:
    - Handle preconfirmation commitments off-chain while anchoring them to Layer 1 for security.
    - Spread validation tasks across multiple nodes to reduce bottlenecks.
    - Validators handle preconfirmation requests in parallel to block-building tasks.
    - Use Merkle trees to batch promises and reduce data size.
4. **Upgrade path**:
    - Design upgrades to be implemented with minimal network interruptions.
    - Ensure upgrades do not disrupt existing preconfirmation promises or validator operations.
    - Deploy upgrades on testnets to identify bugs or performance bottlenecks.

---

## 📌 Risks and Mitigation
- **Risk 1**: Consecutive validators could potentially collude to break preconfirmation promises. How
might you prevent this?  
  - **Mitigation**: 
    - Limit the number of preconfirmed transactions allowed per block.
    - Use a fallback mechanism (roll over unprocessed preconfirmed transactions to subsequent blocks).

- **Risk 2**: What happens if a block containing preconfirmed transactions gets reorged?
    - **Mitigation**: Similar to above. Fallback to a subsequent blocks.

- **Risk 3**: If a proposer receives a preconfirmation request just before the build phase, they might fail to include it, resulting in broken preconfirmation promises and potential slashing.
    - **Mitigation**: Implement a cutoff time for preconfirmation requests (20 seconds before the target block).
Reject or deprioritize requests submitted after this cutoff to reduce the risk of broken promises.

---

## 🔗 References
Include any references, links, or resources for further information:
- [Based Preconfirmation - JustinDrake](https://ethresear.ch/t/based-preconfirmations/17353/1)
- [Puffer Docs](https://docs.puffer.fi/unifi-avs-protocol)
- [Luban](https://docs.luban.wtf/learn/architecture/off_chain_components/overview)
- [Eigen Build your own AVS](https://docs.eigenlayer.xyz/developers/how-to-build-an-avs)

---

## 🚀 Call to Action
Look forward to your feedbacks on this proposal!
