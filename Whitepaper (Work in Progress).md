# DataChain Whitepaper

(Work in progress)


## Problem Statement

A group of data producers want to publish data to a censorship-resistant system that will order them, retain them for a predefined period, and make them available to data consumers.

A typical group of data producers may be users participating in a social network or sensors participating in a project that periodically send measurements.

Today, the typical solution to this problem is a streaming service like Apache Kafka. However, such systems are centrally operated, which means the operator can censor messages.

## Scope

This document specifies **DataChain**, a decentralized data-streaming engine that provides **globally ordered message delivery**.

A DataChain Network **requires at least one “good” participant** that has an inherent interest in making the chain work.

However, the DataChain design ensures that **censoring the network for long comes with an exponential cost** (exponentially high staked amount to be precise).


## Terminology

The **DataChain Engine** consists of the software components that allow the deployment of a **DataChain Network**. Different projects may deploy their own DataChain Networks, customized and configured to their needs.

The **DataChain Blockchain** is the data structure that holds the data of a DataChain Network.

Throughout this document we will use the term **DataChain** to refer to any of these components, when it is clear what it refers to.

## High Level Design
Each DataChain Network depends on a parent EVM-compatible blockchain, which we call the **parent chain**.

A DataChain Network’s users are defined by a smart contract. Each user gets a unique **UID**, and one or more **public keys**. It is up to the identity smart contract to define the signup flow (permissionless, permissioned, free, paid, etc) and the public keys associated with each user.

DataChain’s minimum data unit is the **message**. Each message contains a data blob (application- and network-specific) and metadata (blob hash, signature, etc).

Messages are grouped in **blocks**, and each block is linked to the previous one.

There are two types of nodes in a DataChain network:
- **Miners** assemble messages in blocks and broadcast the next valid block.
- **Hubs** listen for blocks, and when they receive a valid block they add it to their instance/version of the blockchain.

In this architecture, miners serve new blocks to the network and hubs receive these blocks. Hubs are observers, in the sense that they are free to accept or reject blocks, or pick the one they prefer if they have to choose between two different blocks (fork).

Applications built on top of a DataChain network connect to one or more hubs.

> [!NOTE]
> Readers familiar with blockchain design, can think of messages as transactions, miners as sequencers and hubs as full nodes.

## Message Validity

While there are many resemblances between a DataChain blockchain and a typical blockchain, there is an important difference: **The validity of a DataChain message is atomic and depends on the validity of the message headers.**

This means that a message with a valid signature is valid.

> [!NOTE]
TBD: custom DataChain deployments may add additional validity rules that depend on the data inside a transaction. We have to define how/when this is permitted.

This is in contrast to a typical blockchain transaction which may have a valid signature, but may be invalid for many reasons, for example, if it tries to spend funds that are not available.

## Messages

Messages are encoded in protobuffer format:

```
message Message {
	bytes data = 1; // Contents of the message
  bytes hash = 2; // Hash digest of data
  HashScheme hash_scheme = 3; // Hash scheme that produced the hash digest
  bytes signature = 4; // Signature of the hash digest
  SignatureScheme signature_scheme = 5; // Signature scheme that produced the signature
  bytes signer = 6; // Public key or address of the key pair that produced the signature
}

enum HashScheme {
  HASH_SCHEME_NONE = 0;
  HASH_SCHEME_BLAKE3 = 1; // Default scheme for hashing MessageData
}

enum SignatureScheme {
  SIGNATURE_SCHEME_NONE = 0;
  SIGNATURE_SCHEME_ED25519 = 1; // Ed25519 signature (default)
  SIGNATURE_SCHEME_EIP712 = 2; // ECDSA signature using EIP-712 scheme
}
```


# Architecture

## Staking Contract
A ETH staking contract is deployed on the parent chain. The contract allows any address to stake ETH. Staked ETH can be reclaimed after 100 days (TBD).

## Miners
Miners have two sources of data:
- They listen for events on the parent chain. (They may also proactively query the parent chain.)
- They receive messages from clients

Each miner has a public key. When a miner stakes ETH on the staking contract, it also states the public key it will be using.

## Onchain Events

Miners listen for specific onchain events such as events emitted by the Identity smart contract (user registration, user public key addition/removal), or the Staking smart contract (new stake, stake removed).

Miners encapsulate these events as special messages of type `OnchainEvent`. A detailed description and message specification of `OnchainEvent` messages will be added to this section later.

A special `OnchainEvent` is `ONCHAIN_EVENT_BLOCK` that contains a parent chain  block hash and its height.

## Block structure and construction
Every block consists of a block header and the block body (list of messages).

### OCEC (OnChain Events Count)
Every DataChain block an **OnChainEventsCount (OCEC)** which is the number of OnChainEvents the block and all the previous blocks contain.

```
ocec = sum( count(e) for e in OnChainEvents in all blocks )
```

### Block Score

Every DataChain block has a **BlockScore**.

```
BlockScore = number_of_messages / number_of_miner_blocks
```

- `number_of_messages` is the number of messages in the block 
- `number_of_miner_blocks` is the number of blocks this miner signed in the last N (for example 100) previous blocks of the chain.

The score is higher if a block has more messages, and goes down as a miner adds more blocks.

### Block header data
The block header has the following fields:
- Block Hash
- Previous Block Hash
- Miner address
- Block signature
- Block OCEC
- Number of messages in the block

### Block construction
Miners construct the next blockchain block according to these rules.

1. Every block must contain exactly one `ONCHAIN_EVENT_BLOCK` message.  
2. Every block must contain the hash of the previous block.  
3. A block can only contain **new** messages (not present in previous blocks).  
4. All messages included in a block are lexicographically ordered by hash, after prepending a byte that prioritizes system events (ONCHAIN_EVENT_BLOCK, then ID registrations, storage rents, public keys, etc.).  
5. All messages included in a block must be sequentially valid, i.e. if N and messages 1..N-1 are valid, then message N is valid.

> [!NOTE]
> A DataChain Network may be configured to have a slower block generation rate than the parent chain. In this case, and additional restriction that valid `ONCHAIN_EVENT_BLOCK` messages must contain a height divisible by a constant number can be added.

> [!NOTE]
> Re: rules #4 and #5. There are cases when a block contains onchain events that may define the validity of a message, for example the (onchain) registration of a new user, and a message signed by the user.

## Hubs

Hubs watch the blocks published by miners and add them to the copy of the blockchain they keep locally:

- Hubs do not listen proactively for onchain events, but they validate them when they find them in a block.
- Hubs validate blocks and reject invalid ones.
- If they are missing blocks, they ask other hubs to provide them.

In addition to checking a block validity as defined in the previous section, hubs also check that the miner signing it has staked the required amount. (Described in detail in the next section.)

At any point in time, a hub follows a chain, with blocks B0, B1,..., Bn. A new block (NB) is added only if `OCEC(NB)>OCEC(Bn)`.

A hub may receive two new blocks, Ci and Dj, from two miners, that correspond to chains C0..Ci and D0..Dj.

Assuming that `OCEC(Ci)>OCEC(Bn)` and `OCEC(Dj)>OCEC(Bn)`, the hub will use the following rules to decide on the next chain tip:

1. If `OCEC(Ci)>OCEC(Dj)`, then the next tip is Ci. (Most onchain events)
2. If `OCEC(Ci)==OCEC(Dj)`, and `count(C ∩ B) > count(D ∩ B)`, then the next tip is Ci (minimum reorg)
3. If `OCEC(Ci) == OCEC(Dj)`, and `C ∩ B == D ∩ B`, and `BlockScore(Ci) > BlockScore(Dj)`, then the next tip is Ci (highest BlockScore).

## Staking

**Staking amount.** Every miner must stake a small amount of ETH in a smart contract that allows withdrawals after X days (for example, 0.1 ETH, that can be withdrawn after 90 days). The onchain event of the deposit must be present in the chain proposed by the miner.

Hubs accept a block only if the staked amount is above the current staking threshold.

**Staking Window** (or **window** for short) is the number of blocks BlockScore is calculated over.

All hubs are initialized with a predefined `window` and `staking amount`, which they adjust as they process new blocks.

If the last `window` blocks (ex. 100) were mined by more than `window/5` unique miners, then `window` is adjusted to `window*2` and `staking amount` is also doubled. (No onchain change is required, just what the hubs expect as valid value.)

This means that an attacker that intends to use multiple miners, will need an exponential amount of ETH staked to maintain the attack.

Correspondingly, if the last `window` blocks were mined by less than `window/5` unique miners, then `window` is halved and so is the `staking amount.

### Example:

Let’s say `window=100`, `stake=0.1` and the network has 15 miners. An attacker who wants to force their blocks will have to:
1. Create 5 miners and stake 0.5 for 60 days. The best they can do with this is mine the next 5 blocks.
2. If they increase the number of miners they control (for example add 20 more), then `window` will be adjusted to 200, and the stake will be 0.2. So now they need to stake additional 0.2*20 = 4 ETH.
3. For the next 80 blocks, the additional stake will be 8 ETH and
4. For the next 160 blocks, the additional stake will be 64 ETH.

So, in this example, an attacker who wants to mine all the next 265 blocks, will have to stake 76.5 ETH. Assuming the network generates one block every 5 seconds, will need 76 ETH to censor the network for 22 minutes, and then it will go back to normal. 

### Censorship by “big” miners
A miner with privileged access to new messages can try to censor specific messages, or may be obliged to do so. Since they have access to messages that other miners are not aware of, they can always create blocks with more messages, and win the next block. 

However, as they fill the chain with more blocks, their blocks get a lower BlockScore (percentage of contributed blocks goes up) which makes it easier for a competing miner to propose a new block that includes the censored messages and has a higher BlockScore, even with fewer messages.
