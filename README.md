# Data Chain

A Decentralized Data Streaming Engine with Global Ordering

**TLDR;** DataChain is (will be) an open source engine that allows anyone to build their own decentralized data streaming infrastructure with global ordering.

## Inspiration: Farcaster

The idea originates from Farcaster, a decentralized social network. 

From the [Snapchain Whitepaper](https://snapchain.farcaster.xyz/whitepaper#problem)
> Farcaster has used a CRDT-based system called a deltagraph to decentralize social data. By defining every transaction as a CRDT operation, consensus is reached immediately without coordination at the local node. The changes are then gossiped out to peers which can lazily update their own state. The network served 100k users doing 500 TPS with 2GB/day state growth in early 2024.

> As the network grew to thousands of nodes, some of them get out of sync due to gossip failures. Since CRDTs are unordered, a node could only detect gossip failures by syncing manually with every other node and comparing all transactions. This becomes slow and eventually infeasible as the number of nodes and valid messages cross some threshold (see Appendix C). The lack of ordering also meant that the network cannot enforce global rate limits, and they must be localized to each node. The side effect of this is that a transaction that passes the limits on one node might be rejected by another. Without strict ordering, it's hard to guarantee both real-time delivery and strong consistency.

To deal with these problems, the Farcaster team designed Snapchain, a blockchain-like network:

> Snapchain is different from most blockchains because its transactions are not turing complete, are account independent and pruned often. A transaction is a "post", "like" or other social operation which only affects a single account. This is important for scaling since it prevents the network from being used for non-social purposes and makes sharding by account easy. Older transactions are pruned to clear data from inactive users or negating transactions, such as when a user likes and unlikes the something.

Snapchain is built on top of [Malachite](https://informal.systems/blog/malachite-decentralize-whatever), a Tendermint consensus engine.

### The problem with Tendermint

In Tendermint, validator counts are capped (~100–300) due to signature aggregation and communication overhead (all validators vote every block).

In Tendermint-style chains like Cosmos, a new node can signal its intent to  participate in block validation by staking tokens. When new tokens are staked or delegated, the node’s voting power increases, and periodically (every block or epoch), the protocol updates the active validator set to reflect the top-N stakers.

This model can not be applied to a Snapchain-like network that lacks the ability to transfer value (and to this extent, have a coin). As a result, a Snapchain-like network will have to depend on external governance.

For example, the Farcaster team has proposed a committee-based system, where a group of 15 voters, chosen through rough consensus from the Farcaster community, decides on 11 validators every 6 months.

## Not just Farcaster

Platforms like Apache Kafka used to publish and subscribe to streams of data are a common building block in IT. 

But there is no web3 Kafka-like component that can be used by decentralized protocols such as Decentralized Social (see Farcaster), or Decentralized Physical Infrastructure Networks (WeatherXM, Demo, etc.) which makes data aggregation and distribution a centralization chokepoint.

# What is DataChain

DataChain is a **Decentralized Data Streaming Engine with Global Ordering**.

DataChain is a blockchain-like network engine that groups new messages in blocks. It is operated by an open set of “miners” that can join and leave freely. Miners need to stake a token on an external EVM chain (other chain types can be easily supported in the future), but DataChain becomes aware of the stake without any intermediary.

This [Snapchain alternative proposal](https://gist.github.com/vrypan/1ae6a60ecb3741ab031b5b06c974acab) can be considered the first draft of the DataChain architecture.

The purpose of this project is to implement the DataChain engine. Any interested party should be able to run their own DataChain network by:

- Deploying up a few smart contracts on the EVM chain of their choice (identity and staking).
- Creating a configuration file that points to the corresponding smart contracts.
- Downloading and running the DataChain miner.

The DataChain miner is message agnostic, i.e. it will validate the message metadata (signature validation for example), but will treat every message as a blob.

Developers who want to customize the engine will be able to do so by adding message-specific validation rules (message structure, message size, or other constraints).
