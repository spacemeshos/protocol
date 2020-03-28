# Sync

## Overview

In order for a node to fully participate in the Spacemesh protocol, including the [Consensus protocol](../consensus/01-overview.md), it is essential that the node be aware of the current state of the network. This includes knowing the current [layer and epoch](../intro.md#spacemesh-basics), the latest blocks and [transactions](../transactions/01-overview.md), and the current [set of eligible miners](../mining/05-atx.md). (Other aspects of the protocol, such as the canonical ledger and the global state, are things the node can work out for itself based on these data.) The Sync protocol is the means by which a node achieves this: it sends and receives messages over the Spacemesh [P2P network](../p2p/01-overview.md), and listens for new blocks and transactions.

## Gossip vs. request

There are fundamentally only two ways that a node synchronizes data in Spacemesh: gossip and direct request.

### Gossip

The [gossip protocol](../p2p/01-overview.md#gossip) is the primary way that data are propagated throughout the network. Every time a block receives a new, syntactically valid transaction, it immediately shares it with all of its peers. Every time a miner produces a new candidate block, or receives one from a peer, it immediately shares it with all of its peers. In this way, these data propagate quickly throughout the entire network.

### Direct request

When the node learns of a new block via a gossip message from one of its peers, the first thing it does is to attempt to find all of the other pieces of data that the block points to. A block is deemed syntactically invalid if it points to something, such as another block or an ATX, that's unavailable. The node checks its local database, but if it cannot find the data, then it sends a request to its peers for the missing data. If a peer has the data in question, it responds with the data.

It continues to attempt to resolve dependencies as they arise: for example, if new block A points to a new block B, and new block B points to a new ATX Q, the node will attempt to fetch A, B, and Q, and it will not be able to determine the syntactic validity of A until it has fetched all of the other data that A depends on. This includes other blocks, transactions, and ATXs (as well as the data that they in turn rely on!). 

## Initial sync

When a node first joins the network, of course it knows nothing about the current state of the network. It completes a [bootstrap and peer discovery](../p2p/01-overview.md#bootstrap-and-peer-discovery) process to find peers to pair and exchange data with. It next checks whether it's in sync with the network: i.e., whether the latest layer that it knows about (the genesis layer, the only layer a new node knows about) is the latest known layer in existence. It will discover newer layers from its peers, thus beginning the sync process. There is no explicit "initial sync"; rather, the sync happens implicitly, via the direct request method described above, as the node receives new data—new blocks and layers—from its peers, and subsequently requests the data that they rely on.

Once this initial sync process completes, the node switches into a mode known as "weakly synced" and begins to listen to gossip messages to be notified about new data.

## Data availability

In Spacemesh, data structures contain pointers to other data structures. For example, blocks point to previous blocks, and to ATXs. This is to save space in the mesh: it doesn't make sense for every block to contain a copy of all of the other blocks it points to, or to contain the ATX it points to, since many blocks may point to the same one! However, this presents a challenge of data availability: given a piece of data, such as a block, how do we ensure that all of the data that it relies on is available?

In short, any data structure is considered invalid in Spacemesh if any of the data it relies on are unavailable (i.e., if none of a node's peers have the missing data). Miners should never mine, or gossip, a block with invalid pointers.

## Layers, the clock, and sync

The syncer polls against the clock and the database to figure out the most recent layer a node should know about, and the most recent layer it actually knows about. Based on this information, it can re-enter syncing mode if the node falls out of sync.

## Weakly vs. fully synced

There are two notions of sync in Spacemesh: weakly and fully synced.

**Weakly synced** means that a node knows about the most recent layer in the mesh (which it can calculate using the clock). However, a node that is weakly synced has no way of knowing whether it's seen all of the blocks for this layer.

**Fully synced** means that a node knows about all past blocks, up to and including the present layer. It achieves this by listening, via gossip, for all of the blocks in the current layer. Since blocks vote recursively on valid past blocks (see [Tortoise](../consensus/01-overview.md#tortoise)), it knows that once it's seen all of the blocks in the current layer and recursively fetched the blocks they rely on, it is fully synced.

## Synchronization status and participation in consensus

A node begins listening to gossip messages once it's weakly synced (in order to prevent a DoS attack). Other subprotocols, however, require the node to be fully synced: the Hare, block generation, and ATX generation are all paused until a node is fully synced.

A node can only participate in the Hare consensus mechanism once it is fully synced.
