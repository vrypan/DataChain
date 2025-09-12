# DataChain Witepaper

(Work in progress)


## Problem Statement

A group of data producers want to publish data to a censorship-resistant system that will order them, retain them for a predefined period, and make them available to data consumers.

A typical group of data producers may be users participating in a social network or sensors participating in a project that periodically send measurements.

Today, the typical solution to this problem is a streaming service like Apache Kafka. However, such systems are centrally operated, which means the operator can censor messages.

## Scope

This document specifies **DataChain**, a decentralized data-streaming engine that provides **globally ordered message delivery**.


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

In this architecture, miners serve new blocks to the network and hubs consume new blocks. Applications built on top of a DataChain network connect to one or more hubs.

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

Miners listen for specific onchain events such as events emitted by the Identity smart contract (user registration, user public key addition/removal), or the Staking contract (new stake, stake removed).

Miners encapsulate these events as special messages of type `OnchainEvent`. A detailed description and message specification of `OnchainEvent` messages will be added to this section later.

A special `OnchainEvent` is `ONCHAIN_EVENT_BLOCK` that contains a parent chain  block hash and its height.

## Block construction

Miners construct the next blockchain block according to these rules.

1. Every block must contain exactly one `ONCHAIN_EVENT_BLOCK` message.  
2. Every block must contain the hash of the previous block.  
3. A block can only contain messages not present in earlier blocks.  
4. All messages included in a block are lexicographically ordered by hash, after prepending a byte that prioritizes system events (ONCHAIN_EVENT_BLOCK, then ID registrations, storage rents, public keys, etc.).  
5. All messages included in a block must be sequentially valid, i.e. if N and messages 1..N-1 are valid, then message N is valid.

> [!NOTE]
> A DataChain Network may be configured to have a slower block generation rate than the parent chain. In this case, and additional restriction that valid `ONCHAIN_EVENT_BLOCK` messages must contain a height divisible by a constant number can be added.

> [!NOTE]
> Re: rules #4 and #5. There are cases when a block contains onchain events that may define the validity of a message, for example the (onchain) registration of a new user, and a message signed by the user.
