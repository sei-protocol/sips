**SIP-6: Autobahn Upgrade**

| SIP Number | 6 |
| ----- | ----- |
| Title | Autobahn Upgrade |
| Author | [Benjamin Marsh](mailto:ben@seinetwork.io), [Wen Xu](mailto:wen@seinetwork.io) , [Masih Derkani](mailto:masih@seinetwork.io) , [Philip Su](mailto:philip@seinetwork.io)  | Type | Standard (Core) |
| Created | 10/27/2026 |
| Status | Draft |
| Comments | https://github.com/sei-protocol/sips/discussions/17 |

## **Abstract**

This SIP replaces Sei’s single proposer block construction with an Autobahn [GSA+24] based multiple concurrent proposer (MCP) architecture. Every validator in the active committee maintains a lane and may continuously propose transaction batches. A rotating, stake weighted consensus leader orders compact references to availability certified lane blocks rather than carrying the full transaction payload through consensus.

The upgrade separates the protocol into three concurrent pipelines:

1. The availability pipeline produces lane blocks and Proofs of Availability, represented in the implementation as `LaneQC`s;

2. The ordering pipeline agrees on cuts of lane histories and produces `CommitQC`s;

3. The execution pipeline retrieves any missing blocks, executes the committed global order, commits application state, and emits application commitments, the attestation path aggregates application votes into `AppQC`s.

This separation allows block dissemination, ordering, and execution to overlap. A `CommitQC` provides ordering finality such that it fixes the canonical order but does not yet prove the execution result. A later `AppQC` provides a stake weighted certificate over the end of range application state commitment. The design increases throughput by allowing validator bandwidth to contribute concurrently, while retaining low ordering latency by keeping bulk data retrieval and execution off the consensus voting path. It also reduces dependence on one proposer for transaction inclusion and limits the global ordering discretion concentrated in one block builder. It allows for probabilistic censorship resistance through multiple copies of a transaction being sent to the network via different validators, and negates some MEV via the tip based ordering mechanism. Autobahn represents the most meaningful upgrade towards Sei Giga [MLJ+26].

## **Motivation**

Autobahn allows Sei to move towards a multiple concurrent proposer architecture where no single block proposer is responsible for the proposal or allocation in a given slot of the system. Each validator is able to produce its own set of blocks to be included in the overall consensus proposal limiting the risk of network conditions slowing the chain or a single validator censoring a transaction, since no single entity is responsible for the allocation some ordering games, or MEV, is partially mitigated. Probabilistic inclusion and some new economic games are studied in [ML26].

### **Aggregate Throughput**

Validators disseminate lane blocks in parallel and the consensus leader proposes a cut of certified lane tips rather than redistributing every transaction in the cut. The protocol can therefore use aggregate validator bandwidth instead of placing the bulk data path behind one proposer.

### **Low Ordering Latency**

Throughput does not need to be purchased with a long, multi-round DAG ordering path. Validators may vote on availability certified references without first fetching every referenced payload. Bulk data retrieval and application execution proceed concurrently with later ordering work, subject to bounded backpressure. This SIP does not make a fixed latency or throughput guarantee. Performance depends on committee topology, workload, payload size, execution cost, protocol parameters, and deployment conditions. Benchmark results in the Autobahn paper and Sei Giga whitepaper are evidence for the design, and not consensus requirements.

### **Censorship Resistance**

MCP can allow transaction ingress through multiple validator lanes rather than exclusively through the current consensus leader. To realize that benefit, the activation routing rule MUST ensure that a transaction submitted to an honest validator can reach an honest lane rather than always being forwarded to one deterministic shard. Once a valid transaction is admitted, proposed in an honest lane, and certified by a `LaneQC`, it can, under partial synchrony and eventual honest leadership, be included in a later committed cut. This is resistance to single proposer censorship, not instant censorship immunity. A Byzantine lane owner may censor transactions submitted only to it, and a Byzantine cut leader may delay a certified lane tip during its view. If the final ingress API supports multi-submission, clients seeking stronger inclusion guarantees may submit to more than one validator.

### **Reduced Sequencer Discretion**

Block construction is distributed across validator lanes, and a deterministic merge rule converts a committed cut into one global order. This reduces the global ordering discretion of a single block builder. Autobahn does not eliminate MEV or provide a general fair ordering guarantee. Lane owners retain control over ordering within their own lanes, public lane data creates opportunities for cross-lane extraction, and the precise merge and duplicate handling rules materially affect those opportunities. Pre-execution privacy and broader MEV mitigation are outside the scope of this SIP.

## **Scope**

This SIP specifies:

- Validator operated lanes and lane block availability certification;

- Stake weighted cut consensus and ordering finality;

- Retrieval of committed payloads outside the consensus voting path;

- Deterministic conversion of committed lane ranges into a global order;

- Asynchronous execution and application state certification;

- Bounded buffering, pruning, and backpressure;

- Execution derived committees and delayed epoch activation; and

- The distinct finality signals exposed to applications and operators.

This SIP does not specify:

- Permissionless lane creation by non-validators;

- Transaction privacy before ordering;

- A complete fair ordering or MEV prevention mechanism;

- The Sedna upgrade;

- Non-Autobahn changes described in the Sei Giga whitepaper;

- A new validator selection or staking mechanism; or

- Application specific rules for acting on ordering finality instead of application certification or state attestation finality.

## **Terminology**

| Term | Meaning |
| --- | --- |
| MCP | Multiple concurrent proposers. Active validators propose lane blocks concurrently while a view leader coordinates agreement on a cut. |
| Lane | A validator owned, position indexed hash chain. |
| Lane block | One signed batch and header in a lane. |
| Proof of Availability (PoA) | A certificate showing that sufficient voting weight received a lane block. PoA does not mean Proof of Authority. |
| LaneQC | A PoA certificate. |
| Cut | A per-lane vector of selected certified tips. Comparing each tip with the lane’s last committed position determines its newly ordered range, unchanged lanes may be encoded implicitly. |
| Road | The ordered sequence of road blocks produced by consensus. |
| Road block | The output of one consensus instance, containing an agreed cut of lane histories and attested by a CommitQC. Currently called a superblock in the implementation. |
| RoadIndex | The height of a road block in the road. |
| Consensus instance | The agreement process for one road block height. An instance may span multiple views before commitment. |
| CommitQC | A quorum certificate committing a road block and finalizing its cut. The cut, canonical parent state, and deterministic merge rule determine the canonical transaction sequence. |
| FullCommitQC | A CommitQC together with the verified lane block headers needed to reconstruct all of its committed ranges. |
| Ordering finality | The point at which a cut is fixed by a `CommitQC` and the protocol merge determines its canonical transaction order. Transaction batches and execution results may not yet be available. |
| Execution finality | The point at which a committed range has been locally executed and persisted, and the calculated `AppHash` matches the `AppHash` certified by its `AppQC`. |
| AppProposal | A proposal binding an executed application state commitment to an ordered global range. |
| AppQC | A stake weighted quorum certificate over an `AppProposal`. |
| Anchor | The highest persisted matching `CommitQC`, local `AppProposal`, and `AppQC` point used for safe pruning. |
| Epoch leash | A condition preventing ordering from crossing an epoch boundary until the required ordering, execution, and committee state is available. |

## **Basic Protocol Model**

Let *W* be the total active committee voting weight and let *F = floor((W-1)/3)*. The protocol assumes at most *F* Byzantine voting weight. The network is partially synchronous for liveness. Safety does not depend on a known message delay bound. Validators authenticate consensus messages and lane proposals using committee keys.

## **Specification**

The words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, and `MAY` are to be interpreted as normative requirements.

### **Concurrent Pipelines**

Nodes MUST operate the following pipelines concurrently:

| Pipeline | Input | Output | May run ahead of |
| --- | --- | --- | --- |
| Availability | Transactions and prior lane tips | Lane blocks, LaneVotes, and `LaneQC`s | Ordering |
| Ordering | Availability certified lane ranges | `CommitQC`s and one global order. Other consensus messages may be outputted where required. | Execution |
| Execution and attestation | Ordered blocks and payloads | `AppHash`, `AppProposal`, `AppVote`, and `AppQC` | N/A |

The resulting pipeline of pipelines is:

```text
lane blocks -> LaneQCs ->Proposal-> CommitQC -> execution -> AppHash/AppProposal/AppVote -> AppQC
```

Each stage MUST have finite buffering. A slower downstream stage MUST eventually apply backpressure to its upstream stages as specified below.

### **Validator Lanes and Availability**

Every active committee validator MUST have at most one non-Byzantine active lane identifier. A lane identifier MUST bind the validator identity and the epoch in which that lane joined. Non-validator accounts MUST NOT create consensus lanes under this SIP. A validator MAY produce lane blocks independently of the current consensus leader. Each lane block MUST include, directly or by commitment:

- its lane identifier;

- its continuous monotonically increasing lane block number;

- the hash of its parent lane block;

- a commitment to its payload.

Correct validators MUST process and vote on a lane in block number order. A correct validator MUST, under normal operation, vote for a lane block at position `i` only after receiving and accepting the exact parent at position `i - 1`. Upon a restart the validator SHOULD jump to the position derived from the `CommitQC` meaning it may skip the outdated sequenced  lane blocks. It MUST NOT cast conflicting availability votes for the same lane and position. This sequential voting rule, together with the hash link, makes a certified tip evidence that its lane prefix was available to at least one honest voter. The current implementation enforces sequential block receipt and parent hash matching rather than carrying the parent `LaneQC` inside each child proposal. A `LaneQC` MUST contain signatures whose active stake reaches `F + 1`. Because at most `F` stake is Byzantine, a valid `LaneQC` proves that at least one honest voter possessed the certified block when it voted. Correct availability voters MUST retain and serve certified data until the protocol’s pruning rules permit its deletion. A `LaneQC` proves availability only. It does not assign a global order, certify execution, or prevent a Byzantine lane owner from creating certified forks. Cut consensus determines which certified branch, or set thereof, enters canonical history.

### **Cut Consensus and Ordering Finality**

A road is the ordered sequence of road blocks, currently called superblocks in the implementation. Each road block is the output of one consensus instance, which may span multiple views. Each view MUST have a deterministic, stake weighted leader. The view leader coordinates agreement but is not the exclusive producer of transaction data. When constructing a fresh proposal, a correct leader MUST select, for each active committee lane, the highest observed certified tip that advances the lane’s last committed position and satisfies the configured range bounds. Its LaneQC MUST verify under the proposal’s committee. A correct leader MUST NOT selectively omit a lane with such an advancement. An unchanged lane retains its prior committed position. Every non-empty range MUST begin immediately after the lane’s last committed position. For every non-empty lane range, a fresh proposal MUST identify the lane and the first new lane block number, and MUST attach the complete LaneQC certifying the final header.

Before voting on a fresh proposal, validators MUST verify proposal structure, epoch binding, lane membership, range bounds, attached LaneQCs, and range index continuity with the preceding committed cut. For a reproposal, validators MAY rely on the protocol’s evidence that the same cut was validated in an earlier view of the same consensus instance. They MUST verify that evidence and apply the current view’s locking and voting rules. They need not fetch every intermediate header or payload before voting. Before materializing or executing the committed range, a node MUST reconstruct and verify that the newly ordered suffix is internally gap free and hash linked. For a Byzantine lane, the suffix’s first newly ordered block need not descend from the block previously committed at the preceding position; competing branches at already committed positions are ignored. This is the fork resolution behavior described by Autobahn rather than ordinary single chain parent continuity across cuts. Consensus uses stake weighted prepare and commit quorums. A valid `CommitQC` MUST contain at least `W - F` commit voting weight for one proposal. A `CommitQC` finalizes the cut. Once the protocol fixes one deterministic merge rule, that finalized cut uniquely determines the ordering final global sequence and MUST NOT be reordered by execution or by a later road block.

Consensus MAY continue ordering later road blocks before the payloads and application results of earlier road blocks are locally available, subject to bounded buffers and epoch leashes.

### **Payload Retrieval Outside the Voting Path**

A validator MAY vote on a cut containing valid `LaneQC`s without already possessing every referenced payload. Upon receiving a valid cut proposal, it MAY begin fetching missing data in parallel with consensus and MUST begin no later than commitment. A node missing a certified lane block MUST request it from an available peer. A `FullCommitQC` consists of a `CommitQC` and the lane block headers needed to reconstruct its committed ranges. A node obtains it either by reconstructing it locally from retained LaneVotes, the fast path, or by receiving it from peers through gossip. Before using a `FullCommitQC`, the node MUST verify its `CommitQC` and check that the headers reconstruct the committed ranges and match their certified tips. The recipient MUST then verify each retrieved block against the corresponding header before execution.

At least one honest `LaneQC` voter is expected to possess and serve the certified data before anchor based pruning. Peer selection MAY use signers or any other peer with the verified block, the current implementation uses a shared peer request pool rather than targeting signers directly. “Outside the hot path” means payload possession is not a prerequisite for voting on a fully availability certified cut. It does not mean retrieval has zero cost or constant byte volume. A node cannot execute an ordered block until its payload is present.

### **Canonical Transaction Ordering**

A `CommitQC` finalizes a cut of lane blocks. Implementations MAY enumerate the cut's headers in a separate deterministic lane and block order for certificate verification, payload retrieval, persistence, and block identifiers. That enumeration is not the canonical EVM transaction order. Lane position, lane local transaction order, validator index, leader ordering, and payload arrival time MUST NOT be used as hidden transaction ordering tie breakers. Let *T(C)* be the multiset of transaction occurrences in all newly committed lane blocks of cut *C*. Every correct node MUST derive the same canonical transaction sequence from *T(C)* by the following procedure. An implementation is available at [BM26].

1. Normalize metadata. For each prevalidated transaction occurrence, derive its transaction hash, recovered sender, nonce, and effective priority tip using the common fee context for the ordering batch. For a valid EIP-1559 transaction, the effective tip is:

   ```text
   min(max_priority_fee_per_gas, max_fee_per_gas - base_fee)
   ```

   Fee validity MUST be checked before the subtraction. The ordering input MUST also include the canonical parent state's next executable nonce for every represented sender. Missing or inconsistent metadata is invalid input and MUST NOT be resolved using local observation.

2. Remove exact duplicates. If the same transaction hash occurs more than once, retain one canonical occurrence. Equal hashes MUST have identical canonical metadata under a collision resistant hash function, otherwise the ordering batch is invalid. Because later ordering does not depend on lane identity, selecting one occurrence MUST NOT affect execution order.

3. Resolve nonce slot conflicts. Distinct transaction hashes with the same `(sender, nonce)` are replacement conflicts, not exact duplicates. Retain the transaction with the greater effective tip, if effective tips are equal, retain the transaction with the lexicographically lower hash. This rule MUST be applied identically by every validator.

4. Apply parent state nonces. For each sender, discard transactions whose nonce is lower than the sender's next executable nonce in the canonical parent state. Starting at that next nonce, retain only the contiguous prefix of populated nonce slots. At the first gap, exclude every later transaction from that sender from this ordering batch.

5. Construct nonce dependencies. For every retained sender prefix, transaction nonce *n + 1* depends on transaction nonce *n*. These per-sender chains are the complete dependency graph for this ordering rule.

6. Run priority Kahn ordering. Initialize the ready set with the first transaction in each non-empty sender chain. Repeatedly choose the ready transaction with the greatest effective tip, breaking an equal tip tie by the lexicographically lower transaction hash. Emit it, add its immediate same sender successor to the ready set if one exists, and repeat until the ready set is empty.

The result is the priority maximal topological order of the retained transactions. Step 6 is a dynamic ready set algorithm, not a sort of static topological layers, a newly unlocked high tip successor can execute before a lower tip transaction that was already ready from another sender. For example, if sender A has *(nonce 0, tip 5)* followed by *(nonce 1, tip 1)*, while sender B has *(nonce 0, tip 4)* followed by *(nonce 1, tip 10)*, the tip sequence is `5, 4, 10, 1`.

The result MUST be invariant under permutation of lane enumeration, block retrieval, and transaction occurrence input order. Implementations MAY use an alternative ordering algorithm provided it produces exactly the sequence defined above for every valid input, including nonce dependencies and hash tie breaking. Sorting by original tips and subsequently repairing nonce order is not generally equivalent.

#### Financial Justification

When a builder and proposer control transaction ordering, as they do in a standard blockchain, they can use or sell favorable positions. Let u denote the revenue available from that discretion. Committing a multi-lane cut and then applying a deterministic cross-lane ordering rule removes the producer’s unilateral right to assign positions within that cut. It therefore gives up at least some of the direct ordering revenue represented by *u*. The value of early execution does not disappear. A trader chooses its priority tip before knowing the complete cut and its competitors’ bids. For a transaction that uses gᵢ gas, let *vᵢ* be the value of favorable placement and *qᵢ(pᵢ)* its probability of obtaining that placement when it bids tip *pᵢ*. Its ordering incentive can be expressed as:

```text
expected payoff = vᵢqᵢ(pᵢ) − gᵢpᵢ.
```

Thus, when placement matters, a higher tip can be worthwhile even if it is unnecessary for inclusion. The process partly resembles a sealed bid auction as bidders commit under incomplete information, and the ordering rule determines the outcome.

Let *u′* be the additional priority fee revenue induced by this ordering incentive, relative to bids made for inclusion alone. Competition can make *u′* replace some of the producer revenue *u* that discretionary ordering would have generated. Where searchers compete for a trading opportunity, those fees are paid out of their expected gains from favorable placement, rather than imposed as a direct charge on the transaction that creates the opportunity. The amount recovered depends on competition, inclusion, and where priority fees are paid.

### **Asynchronous Execution**

Execution MUST consume the canonical transaction sequence derived under the ordering rule. It MUST NOT execute transactions in lane header enumeration order unless that order happens to equal the canonical sequence. Any execution MUST be deterministic and MUST NOT reorder transactions. For each committed cut, a node MUST obtain and verify the referenced payloads, derive the canonical sequence against the result of the prior committed execution range, execute that sequence in order, persist the resulting application state at the required boundaries, and only then make that result eligible for an application vote. Ordering and execution MAY proceed concurrently. A `CommitQC` therefore communicates ordering finality but not transaction success, receipts, gas used, or the resulting application state. Applications that require an executed result MUST wait for execution, and applications that require a quorum certified end of range state commitment MUST wait for the corresponding `AppQC`.

### **Lazy State Commitment**

The AppProposal binds:

- the epoch and road;

- the ordered block range;

- the application hash produced after executing the final block of that range.

Intermediate application hashes MAY be persisted for local recovery, but only the application hash at the end of the committed range is proposed for `AppQC` certification. In this section, “lazy state commitment” means delayed and batched quorum certification of state. It does not mean that an implementation may omit the deterministic per-block execution and persistence steps required by the application. A validator MUST cast an application vote only after it has executed and durably persisted the covered range. It MUST NOT sign more than one `AppProposal` for the same epoch and road, including across a restart. An implementation MUST durably record enough signing state before broadcast to enforce that rule. Under the stated Byzantine voting weight assumption, a valid `AppQC` together with a valid validator signature over a conflicting `AppProposal` constitutes protocol fault evidence against the signer of that proposal. Both commitments MUST concern the same chain, epoch, road block height, and canonical execution range, and MUST be verified under the applicable committee. This constitutes an execution fault and does not require the validator to have signed both conflicting results. Signing both results additionally constitutes equivocation. An unsigned local execution mismatch constitutes execution divergence but does not, by itself, provide attributable fault evidence. The propagation and consequences of execution fault evidence MUST be specified separately.

An `AppQC` MUST contain signatures reaching `W - F` for the exact same `AppProposal`. An `AppQC` whose application hash differs from the locally executed result MUST be treated as an execution divergence and MUST NOT cause the node to replace its local state silently.

### **Backpressure, Anchoring, and Pruning**

The anchor is the highest persisted checkpoint for which a valid `AppQC` matches the locally executed application commitment and the corresponding committed range. The anchor MUST advance monotonically. Receiving an `AppQC` alone is insufficient to advance the anchor: the node MUST also have executed and durably persisted the matching state. Availability metadata, including lane blocks and `CommitQC`s strictly below the anchor, MAY be evicted once it is no longer required for live verification or recovery. Committed block data MUST remain available for the configured retention period so that the node can serve lagging or newly joined validators. Pruning MUST preserve the anchor and the committee, commitment, and signing information required for safe restart. History beyond the defined minimum retention period is available only from nodes configured to retain and serve it, such as archival nodes or archival RPC services. A node may choose to retain data itself unilaterally. Operating an RPC endpoint does not, by itself, imply archival retention. When a validator leaves the committee, its lane closes at the applicable epoch boundary. Rejoining creates a new LaneID. Once a node has established the committed history through the closing boundary, it MAY prune the closed lane’s uncommitted blocks that cannot enter any later committed cut. Such blocks do not need to acquire global positions or pass the application anchor before deletion. Committed blocks from a closed lane remain subject to the ordinary anchor and retention requirements. Closure MUST NOT discard headers, LaneVotes, certificates, or signing records still required for `FullCommitQC` reconstruction, historical verification, peer service, fault evidence, or recovery.

Nodes MUST apply backpressure to bound the amount of data awaiting ordering, execution, or pruning:

- A lane MUST pause production once it holds `x` unpruned blocks and MAY resume when pruning releases capacity, where `x` is a chain parameter.
- A `CommitQC` MUST be admitted only when its global start lies within the configured position window measured from the oldest executed block retained in memory.
  - We note that the position must be defined before release.
- Lane block sizes, committed range sizes, and pending queues MUST also be bounded. A limit on a range’s start alone does not bound its total size.

Objects waiting for admission SHOULD be retained within bounded capacity and retried when the blocking condition clears. Nodes MUST continue the retrieval, execution, and application attestation work required to release backpressure.

### **Epochs and Execution Derived Committees**

Committee membership and voting weights are derived from executed staking states. Because execution proceeds asynchronously, validators may reach the same execution height at different times. Committee activation MUST therefore follow a deterministic schedule rather than each validator’s local execution progress. The protocol divides consensus history into epochs. Epoch length is configurable in RoadIndex units in the on-chain params, with an intended duration of approximately 12 hours under normal operation. Actual wall clock duration depends on consensus progress. Staking transactions execute throughout each epoch, but only the application state after the final road of epoch *N* determines the committee for epoch *N + 2*. That committee becomes active at the first road of epoch *N + 2* and remains fixed throughout the epoch. The source state MUST be committed and execution finalized, mening the AppQC must be available, at the node before the derived committee becomes active. All correct validators MUST derive the same membership, keys, and voting weights from that state. The initial committees and epoch schedule MUST be established by the activation requirements. Proposals, consensus votes, and application certificates MUST be verified using the committee of their referenced road. Lane availability certificates MUST be valid under the committee of the road in which their blocks are included.

### **Epoch Transition Leash**

An honest validator MUST NOT enter epoch  *E + 1* until both transition leashes release:

- The committee for epoch *E + 1* has been derived from the canonical execution result at the end of epoch *E - 1* and registered locally. The required source execution result MUST be execution finalized. Possession of an `AppQC` alone does not establish that local execution has caught up or that the node has reconstructed the state needed to derive the committee.
- The local anchor has reached at least epoch *E*, allowing epoch *E - 1* to leave the live availability verification window.

For *E > 0*, the availability module waits to admit the CommitQC for epoch *E*’s final road block until both leashes release. Consensus may already have finalized that road block; admission is the local step that allows it to proceed toward execution. The pruning leash MUST be satisfiable using a matching AppQC for an earlier road block in epoch E that the node has already admitted and executed. Producing an AppQC for the final road block requires validators to admit and execute that block first. If every validator waited for that AppQC before admission, none could produce it. Bootstrap and empty workload behavior MUST allow the earlier checkpoint to be reached. Together, the leashes ensure that a validator has the agreed committee before processing the next epoch and keep the live availability verification window bound to the current epoch and its immediate predecessor. Historical data required for recovery or configured peer service MAY be retained longer. If either leash remains blocked, the validator MUST wait rather than infer missing committee state or advance using an incorrect committee.

## **Impact**

The upgrade distributes transaction production across validator lanes and allows availability, ordering, and execution to overlap. This reduces dependence on a single proposer’s dissemination capacity while introducing additional concurrent networking, storage, and execution work for validators. Applications and infrastructure MUST distinguish ordering finality from execution and application certification. A `CommitQC` finalizes a cut but does not establish transaction success, receipts, or the resulting application state. Consumers requiring quorum certified state MUST wait for the corresponding `AppQC`. Validator membership changes take effect through the two epoch activation schedule. Departing validators remain responsible for outstanding data service, until the subsequent AppQC has been built, and application attestation obligations until the relevant retention and certification conditions are satisfied. The upgrade requires coordinated migration of consensus state and updated behavior in validator software, RPC services, indexers, explorers, and other consumers of block and finality information.

## **Security Impact**

The protocol retains the stated Byzantine voting weight assumption for each active committee. Certificate verification MUST use the correct committee, count each signer once, and verify the exact signed message. An availability certificate establishes an honest availability witness, it does not establish a unique lane branch or a valid execution result. Separating ordering from execution introduces additional dependencies on data retention, payload retrieval, deterministic execution, and application attestation. Validators MUST persist data before availability voting and retain voted for data until safe pruning permits deletion, even if they have not observed the resulting `LaneQC`. Execution divergence MUST be detected and surfaced. A conflicting `AppQC` MUST NOT silently replace local state or advance the local anchor. Restart MUST preserve signing and locking rules and reject inconsistent committed, execution, or committee state. Resource limits and backpressure mitigate exhaustion from excessive outstanding data or slow downstream processing. Epoch leashes prevent progress with unavailable committee state. Lane closure permits pruning of permanently uncommitted lane data only after the committed history through the closing boundary has been established. It MUST NOT release data or evidence still required for committed ranges, retention obligations, or recovery. These mechanisms may delay progress when their conditions are unmet; they MUST NOT be bypassed to restore liveness.

## **Activation Requirements**

- [SIP-3](https://github.com/sei-protocol/sips/blob/main/sips/sip-3.md) [PS25] MUST be complete before activation.
- Activation MUST follow Sei’s governance authorized software upgrade process and the applicable voting requirements.
- The release MUST fix the migration checkpoint, initial committees, epoch schedule, resource limits, certificate encodings, execution rules, and externally exposed finality semantics. Said parameters MUST be defined as on-chain parameters.
- Multiple independent security firms MUST audit the release. All critical and high severity findings MUST be resolved before activation.
- Updated validator hardware, bandwidth, storage, retention, and recovery requirements MUST be validated and published.
- The upgrade MUST be rehearsed against representative existing state, preserving the last agreed application state and establishing an unambiguous first Autobahn road.
- The required test results and operator procedures MUST be available before activation.

## **Testing Requirements**

- Verify safety and liveness under leader changes, equivocation, delayed or reordered messages, network partitions, and recovery.
- Verify persistence before voting, retrieval from alternative peers, corrupt or withheld payload handling, lane forks, and retention when a voter has not observed the resulting certificate.
- Verify deterministic results across input permutations, duplicates, nonce conflicts and gaps, fee ties, invalid transactions, and execution boundaries.
- Verify durable execution before voting, historical committee verification, delayed certificates, and detection of conflicting application commitments.
- Test committee joins, exits, rejoins, weight changes, cross-boundary lane certification, execution lag, empty workloads, and both transition leashes.
- Verify durable state recovery and signing safety across crashes at persistence and voting boundaries. Inconsistent state MUST prevent normal participation.
- Verify monotonic anchor advancement, retention obligations, admission window boundaries, bounded memory and storage usage, and resumption after downstream stalls.
- Sustained and burst tests MUST demonstrate stable operation under representative workloads and committee conditions. Results MUST distinguish ordering, execution, and application certification latency and include resource use and tail latency.

## **Questions and Work Deferred to Future SIPs**

Sedna [RMK+25] proposes distributing verifiable coded fragments across proposer lanes instead of replicating complete transactions. Its intended benefits include lower dissemination overhead and reduced exposure of transaction contents before reconstruction. A future SIP will specify its integration with Autobahn, including reconstruction thresholds, transaction eligibility, availability guarantees, and incentives against withholding fragments [MR26]. Multiple proposers and deterministic ordering do not prevent participants from observing public lane data and submitting competing transactions before a cut closes. Encrypted transaction submission could reduce this opportunity by withholding transaction contents until ordering commitments are irreversible, as such an encrypted mempool will be proposed in a later SIP. The economics covering the upgrade to the consensus protocol will be proposed in a later SIP covering several other economic changes too. Emergency removal rules for the committee will be introduced in a future SIP.

## **References**

- Neil Giridharan, Florian Suri-Payer, Ittai Abraham, Lorenzo Alvisi, and Natacha Crooks. [*Autobahn: Seamless high speed BFT*](https://doi.org/10.1145/3694715.3695942). Proceedings of the ACM SIGOPS 30th Symposium on Operating Systems Principles, SOSP ’24. ACM, 2024.

- Benjamin Marsh, Steven Landers, Jayendra Jog, and Alejandro Ranchal-Pedrosa. [*Sei Giga*](https://arxiv.org/pdf/2505.14914v3). Whitepaper, arXiv:2505.14914v3, 2026.

- Benjamin Marsh and Steven Landers. [*MEV in multiple concurrent proposer blockchains*](https://doi.org/10.3389/fbloc.2026.1782707). Frontiers in Blockchain, volume 9, article 1782707, 2026.

- Alejandro Ranchal-Pedrosa, Benjamin Marsh, Lefteris Kokoris-Kogias, and Alberto Sonnino. [*Sedna: Sharding transactions in multiple concurrent proposer blockchains*](https://arxiv.org/abs/2512.17045v2). arXiv:2512.17045v2, 2025.

- Benjamin Marsh and Alejandro Ranchal-Pedrosa. [*A mechanism design overview of Sedna*](https://arxiv.org/abs/2603.17614v1). arXiv:2603.17614v1, 2026.

- Benjamin Marsh. [*ordering*](https://github.com/muursh/ordering/tree/4cfa9a7721c4db2138286b46a0f4b661be83dbef). GitHub repository, 2026. Rust implementation, commit *`4cfa9a7.`*

- Philip Su. [*SIP-3: Deprecating CosmWasm & Cosmos*](https://github.com/sei-protocol/sips/blob/main/sips/sip-3.md). Sei Improvement Proposals, 2025.
