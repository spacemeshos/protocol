# Peer-to-Peer Networking
## Overview

The Spacemesh protocol leads to the construction of a global, distributed ledger. As such, each Spacemesh node must receive, validate, and store all transactions created by all other Spacemesh nodes. Also, nodes [participate in consensus](../consensus/01-overview.md) and listen to [PoET server](../mining/03-poet.md) results. This requires the nodes to have a communication infrastructure to transport messages across the network. For this, Spacemesh nodes use a custom-built peer-to-peer (P2P) protocol. Using P2P technology allows nodes to efficiently transmit messages without the need for a centralized message server.


## What is P2P?

[Peer-to-peer](https://en.wikipedia.org/wiki/Peer-to-peer) is a distributed application architecture that allows sharing of data and resources among nodes, called peers, that are similarly privileged. This means that no single server or master node has any special capabilities, unlike in a client-server architecture.

Historically, peer to peer technology has mainly been used for file sharing, as popularized by [Napster](https://en.wikipedia.org/wiki/Napster), VoIP services like Skype and, more recently, [BitTorrent](https://en.wikipedia.org/wiki/BitTorrent). Other usage cases have emerged, including distributed file systems and of course blockchain technology.

P2P is the perfect architecture for blockchain since it provides a means to connect to a network without relying upon a centralized server, enabling some of the core properties of blockchain including permissionlessness and censorship resistance.


## Building Blocks

### Topology

Each peer must discover and connect to one or more peers in order to pass messages to and from the network. Peer to peer architecture usually connects nodes in a mesh topology. Each node knows and communicates with some number of peers, but does not need to connect directly to all other nodes in the network in order to communicate with them (indeed, in any network of even modest size, this would quickly become impossible as the number of required connections grows exponentially). Specifically in the context of blockchain, it is in fact desirable that a single node _not_ know too many other nodes (e.g., their IP address and port) in order that a malicious node could not discover and attack too many other nodes.

<a name="discovery"></a>
### Bootstrap and Peer Discovery

Nodes need a mechanism, called discovery, that allows them to find and connect to peers. In order for a new node to discover peers and join the network, it needs to know the address of at least one other node (called a bootstrap node) at system startup. The new node queries the bootstrap node for a list of other nodes to connect to. After connecting to a bootstrap node, additional discovery takes place, where the newly added peers are queried for additional nodes as well. Each node maintains a local list of peers that it has learned of, known as an "address book": after the node is restarted, or if it loses a peer connection, it can refer to its address book to find peers to connect to, thus bypassing the bootstrap phase. Note that, in order to prevent an [eclipse attack](https://www.radixdlt.com/post/what-is-an-eclipse-attack/), it's important that a node regularly perform peer discovery with nodes that it's not currently connected to.

### Message Broadcast

Nodes also need a way to exchange messages with other nodes. This usually involves broadcasting a message to every node in the network, and, in turn, receiving such broadcast messages, a subprotocol called [gossip](#gossip). However, it is also useful to be able to exchange messages with a specific peer (usually, only one that the node is directly connected to, i.e., a "neighbor"). Therefore we need a messaging protocol that allows messages to traverse the network and reach all nodes, or one specific "neighbor" node.

<a name="authentication"></a>
### Peer Authentication

To prevent fraud and to prevent one node from performing malicious activities while masquerading as another node, we also require a peer authentication system. Similar to the mechanism that's used in cryptocurrency wallets, these systems are usually based on public-private keypairs. Each peer generates a keypair and advertises its public key (which may or may not be distinct from the keypair used in the wallet), which can be used to encrypt messages intended for it. The keys can also be used to sign messages to prove that they originated from the holder of the private key.


## P2P in Spacemesh

The Spacemesh network is an unstructured peer-to-peer network. Each peer both connects to other peers and also accepts incoming connections from peers.

### Custom P2P stack

Early on in the architecture process, we evaluated existing P2P stacks including the Ethereum stack and [libp2p](https://github.com/libp2p). We decided that these existing stacks were too tightly coupled to the respective protocols: for instance, the Ethereum P2P stack requires that all data be sent using a custom encoding called [RLP](https://github.com/ethereum/wiki/wiki/RLP), and libp2p necessitates use of [Kademlia DHT](https://en.wikipedia.org/wiki/Kademlia), which is useful for file sharing but is not required by Spacemesh. We therefore decided to implement our own lightweight P2P stack. For more information on this decision process, see [The search for the perfect p2p library](https://medium.com/spacemesh/perfect-p2p-library-c559d1ca57dc) (but note that some of the information, such as the discussion of the DHT, is outdated).

### Identity

Each peer is identified by the peer's unique P2P public key. This public key is used both to encrypt data in transit between peers, and for [peer authentication](#authentication). Note that this public key is not used outside of the P2P stack. The keypair that Spacemesh nodes use to identify themselves in the P2P network is distinct from both the keypair used in mining (signing blocks, ATXs, Hare messages, etc.) and the wallet keypair used to sign transactions.

This is a privacy feature: P2P IDs are not considered private since anyone on the network can determine the IP address of any P2P ID. In this way, a miner can change their P2P ID as often as they like (and, e.g., reconnect to the P2P network using a new ID). While traffic analysis can to some extent be used to associate different IDs belonging to the same node, there are additional steps one can take to protect their privacy. We are exploring the use of features such as [Dandelion](https://arxiv.org/pdf/1701.04439.pdf) in Spacemesh for this purpose.

### Protocols, multiplex, and gossip

Spacemesh [multiplexes](https://en.wikipedia.org/wiki/Multiplexing) messages from many subprotocols (`gossip`, `discovery`, `hare`, `sync`) over each peer connection. The P2P stack is agnostic to the format and contents of messages belonging to individual subprotocols.

#### Discovery

The discovery subprotocol is used to [discover new peers](#discovery) to connect to. It's implemented directly on top of the P2P layer.

#### Gossip

In a blockchain protocol like Spacemesh, many types of data, including transactions and blocks, must traverse the network and reach every node. This is achieved using the Gossip subprotocol. Gossip allows each node to broadcast a message to the entire network, and also to forward on other gossip messages it receives. Gossip is implemented on top of the P2P layer.

Each individual subprotocol must register an incoming message handler with the P2P stack. When a subprotocol receives and validates a gossip message, it may report back to the P2P stack that the message is valid, which allows the message to continue to propagate throughout the P2P network to the node's peers. This helps prevent DoS attacks by ensuring that bad data (blocks, transactions, etc.) don't traverse the network since honest nodes will never gossip them.

#### Hare

The Hare is one of the two main consensus engines in Spacemesh. As it involves message passing among nodes, and as all nodes need to receive these messages, it's implemented on top of the Gossip subprotocol. For more information, see [Hare](../consensus/01-overview.md#hare).

#### Sync

Unlike Gossip, the Sync subprotocol is used to exchange specific pieces of data with specific peers. For instance, when a new node comes online, it needs to fetch every layer, block, transaction, ATX, etc. since genesis. It does this by requesting all of the blocks for a given layer, then all of the transactions for a given block, etc., from its peers.

The Sync subprotocol is also implemented directly on top of the P2P layer. For more information, see [Sync](../sync/01-overview.md).

### Messages

Each message is signed using the sender's public key. In addition to the message data itself (the payload), it includes a subprotocol, a client version, a timestamp, the public key of the message originator, a network ID, whether the message is a request or a response, the request ID, and the type of message in the specified subprotocol. All messages are serialized on the wire using [the XDR standard](https://en.wikipedia.org/wiki/External_Data_Representation).

The subprotocol that generated the message then sends it to the P2P stack. It can either broadcast the message to all peers over the gossip network, or it can send the message to one directly-connected "neighbor" peer. Most subprotocols only broadcast messages, but the `sync` subprotocol sends messages to specific peers: for example, to respond to an incoming request for a specific block from a specific peer.

### Peer Discovery

As [described above](#discovery), a peer performs an initial discovery process, called bootstrap, when it first comes online, then subsequently performs additional peer discovery to ensure it has a sufficiently large list of other nodes, and to prevent eclipse attacks.

The following flow describes how a node requests a list of peers from another node:

1. The initiator must first send ping message, to verify that node is alive. The ping contains the node ID (public key), the sender's IP and port.
1. The recipient responds to the ping with a pong message, of the same format as the ping, to validate this information. (In other words, the recipient ensures the intiator is indeed listening at the IP and port they advertised. This prevents reflective DoS attacks.)
1. The initiator then sends a `getAddresses` message, requesting a list of peers.
1. The recipient responds with a list of additional peers: their node IDs, IP addresses, and ports.

Each node maintains an "address book" of known nodes. When it first starts up, or after a peer connection has been lost, it randomly chooses a peer from this address book to dial and attempt to initiate a new session with. This randomness is important to help prevent an eclipse attack. Actually, it doesn't choose completely randomly: a peer is less likely to attempt to reconnect to a node that it failed to connect to before.

### Session Initiation

When two nodes connect for the first time, a secure P2P session needs to be established (using a TCP socket). To establish the session the initiator node first generates a handshake message containing its client version and the network ID it's trying to connect to. It then encrypts the message using a [Diffie-Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) shared secret key constructed using its own private key and the recipient's public key. It sends the message, and its public key, to the recipient.

The recipient reconstucts the shared secret key, using its private key and the initator's public key, uses it to decrypt the handshake message, and checks the initiator's client version and network ID. If everything checks out, it responds to the handshake message and a session is established; if not, the TCP connection is closed and no further information is exchanged.
