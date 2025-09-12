# DataChain Whitepaper

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
TBD: custom data chain implementations may add additional validity rules that depend on the data inside a transaction. We have to define how/when this is permitted.

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

