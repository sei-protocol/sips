| SIP-Number | X |
| ----- | ----- |
| Title | External transaction bundles support |
| Description | Offchain transaction bundle submission to enable MEV operation |
| Authors | Maks Pawlak \[seimev.simple103@passmail.net]<br>Bob DeFarmer \[bob@silostaking.io] |
| Type | Standard |
| Category | Framework |
| Created | 2025-04-11 |
| Status | Idea |

## **Background & Motivation**

Currently, Sei users have no ability to submit an order set of transactions to a validator, or choose where in a block their transaction is. This causes two key problems. 
1. Applications & users have no ability to protect themselves from malicious behavior (either on the part of the validator re-organizing transactions, or spam-bots spamming the network to sandwich transactions). 
2. Validators have no ability to run an organized auction for valuable transactions for a block (for instance in the case of a liquidation bot, it is a hybrid of geographical placement, getting transactions sent to the right validator, and uncertain fees for it). Traditionally, these problems have been solved by an introduction of a MEV workflow. Flashbots did this for Ethereum leading to the generation of millions in tips. Jito did this for Solana for a similar result. This proposal is to modify the Sei-client to allow Silo - or any user - to do this for Sei and unlock substantial economic value.
As the DeFi activity has risen with the launch of Sei V2, and Sei Giga around the corner, this is particularly a critical time to rollout these changes to the chain.

## **What is MEV?**

MEV or Maximal Extractable Value is a broad term describing a set of techniques for optimizing block production to maximize revenue. One common technique involves configuring transaction order by adding new transactions, reordering existing ones, or censoring specific transactions to generate a profit. A classic example is backrunning: when a trader detects a pending transaction that will affect the price of a single pool, they profit by submitting transactions that will realign the price of that pool with other pools that are correctly priced. This strategy only works when transactions execute in a precise order. Crucially, the transactions themselves are never modified, just the order. One of the fundamental principles of blockchains—that only the account owner can send transactions—remains true with MEV. From the blockchain's perspective, it's simply a series of ordinary transactions being executed.

## **Scope of this SIP**

The ability to submit an ordered set of transactions can be valuable enough that users will pay fees, which generate additional revenue for participating validators. We provide this context to clarify the motivation behind this SIP. Building a marketplace or managing bundle submission falls outside the core blockchain's scope, and this SIP focuses exclusively on technical ability to enable validators to receive and include transaction bundles in blocks.

## **Trust**

Bundles, as an entity, exist only outside the scope of the blockchain. Once processed, they are turned into a list of transactions. Bundles can be seen as another form of mempool, with direct injection into a validator and additional conditions on relations between transactions. No consensus rules are applied to mempool content, and similarly they cannot be applied to bundles. A validator must trust the source of the bundles as it cannot verify that it gets complete and valuable payloads, in a similar manner as it cannot verify it receives the complete content of a mempool.

## **Possible risks**

A pull-based approach where a validator queries a bundles server prevents the possibility to overwhelm validators. Possible malicious behavior we envision is a hostile actor providing large amounts of nonsensical transactions, as has been seen on other blockchains like Solana prominently. This would waste processing power and block space by including many failed transactions. While this won't harm the chain's integrity, if enough validators include too many garbage transactions, it may reduce the actual throughput for “real” user’s transactions. It is important to note that a similar attack can be performed currently - malicious actor can spam the mempool with transactions that won’t get processed successfully but will waste block space and validators' resources. 

Mitigation:

It remains unclear whether the ability to drop bundles or terminate connections should be part of this specification. Given the trust requirements, we believe such a system is unnecessary, though we acknowledge this position may change in the future.

## **Connection security**

We propose a standard gRPC connection from validators to a bundle servers. Details of how to authenticate and secure these connections are left to users, as setups and requirements will vary. The standard gRPC library supports TLS, and the implementation provided allows clients to optionally use x509 certificates to verify server identity. No on-chain identities are used for this process.

## **Architecture overview**

Bundles are prepared on a separate, centrally hosted bundle server (centrally hosted meaning having a well-known endpoint, not imposing on internal architecture). Validators connect to their chosen bundle server and request bundle data. The connection originates from the validator in an opt-in manner, allowing greater control.

The system operates independently from the existing nodes' P2P network.

While proposing a block, if any bundles are in-memory for given height, the validator includes them up to the block size limit. If the remaining space is not able to fit more bundles, or there are simply no more bundles to include, the rest of the block space is fitted with available transactions from the mempool. Transactions which were part of bundles are not included again if they were also a part of the mempool.

## **Protocol details**

The protocol is intentionally simple. In fact, it consists of one request and one response, as follows:
Validators, in a configurable interval, send:

`GetBundlesRequest` containing only:<br>
`uint64 min_block_height` - the minimum height bundles should be returned for

In response, they get

`GetBundlesResponse` which contains<br>
`map<uint64, Bundles> bundles`- a mapping of block height to an ordered list of bundles. This ordering should order bundles by value, from most to least valuable.

`Bundles` is a list of `Bundle`objects that contains:

`uint64 block_height` to signalize block height<br>
`repeated bytes transactions` which is list of ordered transactions in a form of raw bytes

## **Tendermint changes**

SEI maintains its own fork of Tendermint, the consensus engine, which has been renamed CometBFT in the upstream. To support bundles, SEI needs to remove a check from Tendermint that prevents transactions from being added or removed during the block building process. This check was also removed from the upstream codebase <https://github.com/cometbft/cometbft/blob/main/types/tx.go#L117>, and as such this is really a backport.

## **Feedback request**

Feedback on this proposal and document are welcome. However, we identified two areas where we would like to focus community to contribute ideas.

1. Block space management. Currently we include as many bundles as possible up to the block size limit. Other approaches can be based on fixed number of bundles, fixed amount of bytes allocated to bundles.

2. Handling misbehaving bundle servers. If required, what kind of mitigation should be used to prevent abusive providers to flood the network?

## **Implementation**

[An implementation](https://github.com/SiloMEV/sei-chain/pull/13) has been prepared by Silo team.
It is curretly being audited by Zenith. Finding will be publicly available when finished.

## **Advisors**

Sam Kleinman [https://github.com/tychoish]

## **Copyright**

CC0 1.0.