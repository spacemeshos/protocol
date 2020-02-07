# Sync

## Overview

In order for a node to fully participate in the Spacemesh protocol, including the [Consensus protocol](../consensus/01-overview.md), it is essential that the node be aware of the current state of the network. This includes knowing the current [layer and epoch](../intro.md#spacemesh-basics), the latest blocks and [transactions](../transactions/01-overview.md), and the current [set of eligible miners](../mining/05-atx.md). (Other aspects of the protocol, such as the canonical ledger and the global state, are things the node can work out for itself based on these data.) The Sync protocol is the means by which a node achieves this: it sends and receives messages over the Spacemesh [P2P network](../p2p/01-overview.md), and listens for new blocks and transactions.

## Gossip vs. request

There are fundamentally only two ways that a node synchronizes data in Spacemesh: gossip and direct request.

### Gossip

The [gossip protocol](../p2p/01-overview.md#gossip) is the primary way that data are propagated throughout the network. Every time a block receives a new, syntactically valid transaction, it immediately shares it with all of its peers. Every time a miner produces a new candidate block, or receives one from a peer, it immediately shares it with all of its peers. In this way, these data propagate quickly throughout the entire network.

### Direct request

When the node learns of a new block via a gossip message from one of its peers, the first thing it does is to attempt to find all of the other blocks that that block refers to. It checks its local database, but if it cannot find a referenced block, then it sends a request to its peers for the missing data. If a peer has the data in question, it responds with the data.

## Initial sync

When a node first joins the network, of course it knows nothing about the current state of the network. It completes a [bootstrap and peer discovery](../p2p/01-overview.md#bootstrap-and-peer-discovery) process to find peers to pair and exchange data with, and those peers begin to send it new gossip messages.

## Data availability


## Layers and ticks

## Weakly vs. fully synced

## Synchronization status and participation in consensus

A node can only participate in the Hare consensus mechanism once it is fully synced.
