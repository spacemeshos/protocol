# Peer-to-Peer Networking
## Overview

The Spacemesh protocol creates a global, distributed, shared ledger. This means that each Spacemesh node must hold all transactions created by all other Spacemesh nodes. Also, nodes participate in [consensus algorithms](../consensus/01-overview.md) and listen to [PoET server](../mining/03-poet.md) results. All of this requires the nodes to have a communication infrastructure to transport and propagate messages across the network. For this, Spacemesh nodes use a proprietary peer-to-peer (P2P) protocol. Using P2P technology allows  nodes to efficiently transmit messages without the need for a centralized message server.

## What is P2P?

[Peer-to-peer](https://en.wikipedia.org/wiki/Peer-to-peer) is a distributed application architecture that allows sharing of data and resources among nodes, called peers, that are equally privileged. The architecture also distributes workloads by partitioning the network into smaller subnetworks where each node can access several other nodes in the same subnetwork.

Historically, peer to peer technology has mainly been used for file sharing, as popularized by [Napster](https://en.wikipedia.org/wiki/Napster). In the past few years, many other file sharing platforms, most notably [BitTorrent](https://en.wikipedia.org/wiki/BitTorrent), were created based on this architecture, and other usage cases have emerged, including distributed file systems and of course blockchain technology.

P2P is the perfect architecture for blockchain since it provides a means to connect to a network without relying upon a centralized server, enabling some of the core properties of blockchain including permissionlessness and censorship resistance.

## Building Blocks

### Topology

Each peer must discover and connect to one or more peers in order to allow messages to be passed to and from the network. Peer to peer architecture usually connects nodes in a mesh topology. Each node knows and communicates with some number of peers, but does not need to connect directly to all other nodes in the network in order to communicate with them (indeed, in any network of even modest size, this would quickly become impossible as the number of required connections grows exponentially). Specifically in the context of blockchain, it is in fact desirable that a single node _not_ know too many other nodes (e.g., their IP address and port) in order that a malicious node could not discover and attack too many other nodes.

### Peer Discovery

Nodes need a mechanism, called discovery, that allows them to find and connect to peers.

### Message Broadcast

Nodes also need a way to send messages to other nodes--sometimes to a specific peer, sometimes to broadcast a message to the entire network--and a way to receive incoming messages from the network. A messaging protocol must be established that allows messages to traverse the network and reach all nodes, or one specific node.

### Peer Authentication

To prevent fraud and to prevent one node from performing malicious activities while masquerading as another node, a peer authentication system must also be established.

## P2P in Spacemesh

### Peer Discovery

### Session Initiation

### Gossip
