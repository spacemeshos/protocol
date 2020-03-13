# Peer-to-Peer Networking
## Overview

The Spacemesh protocol leads to the construction of a global, distributed ledger. As such, each Spacemesh node must receive, validate, and store all transactions created by all other Spacemesh nodes. Also, nodes [participate in consensus](../consensus/01-overview.md) and listen to [PoET server](../mining/03-poet.md) results. This requires the nodes to have a communication infrastructure to transport messages across the network. For this, Spacemesh nodes use a proprietary peer-to-peer (P2P) protocol. Using P2P technology allows nodes to efficiently transmit messages without the need for a centralized message server.

## What is P2P?

[Peer-to-peer](https://en.wikipedia.org/wiki/Peer-to-peer) is a distributed application architecture that allows sharing of data and resources among nodes, called peers, that are similarly privileged (i.e., no single server or master node has any special capabilities). The P2P architecture also distributes workloads by partitioning the network into smaller subnetworks where each node can has access to and coommunicates with a small number of nodes in the same subnetwork.

Historically, peer to peer technology has mainly been used for file sharing, as popularized by [Napster](https://en.wikipedia.org/wiki/Napster) and, more recently, [BitTorrent](https://en.wikipedia.org/wiki/BitTorrent). Other usage cases have emerged, including distributed file systems and of course blockchain technology.

P2P is the perfect architecture for blockchain since it provides a means to connect to a network without relying upon a centralized server, enabling some of the core properties of blockchain including permissionlessness and censorship resistance.

## Building Blocks

### Topology

Each peer must discover and connect to one or more peers in order to pass messages to and from the network. Peer to peer architecture usually connects nodes in a mesh topology. Each node knows and communicates with some number of peers, but does not need to connect directly to all other nodes in the network in order to communicate with them (indeed, in any network of even modest size, this would quickly become impossible as the number of required connections grows exponentially). Specifically in the context of blockchain, it is in fact desirable that a single node _not_ know too many other nodes (e.g., their IP address and port) in order that a malicious node could not discover and attack too many other nodes.

<a name="discovery"></a>
### Bootstrap and Peer Discovery

Nodes need a mechanism, called discovery, that allows them to find and connect to peers. In order for a new node to discover peers and join the network, it needs to know the address of at least one other node (called a bootstrap node) at system startup. The new node queries the bootstrap node for a list of other nodes to connect to. After connecting to a peer as a bootstrap node, additional discovery takes place, where the newly added peers are queried for additional nodes as well. Each node maintains a local list of peers that it has learned of, known as an "address book": after the node is restarted, or if it loses its network connection, it can refer to its address book to find peers to connect to, thus bypassing the bootstrap phase.

### Message Broadcast

Nodes also need a way to send messages to other nodes—sometimes to a specific peer, sometimes to broadcast a message to the entire network—and a way to receive incoming messages from the network. A messaging protocol must be established that allows messages to traverse the network and reach all nodes, or one specific node.

<a name="authentication"></a>
### Peer Authentication

To prevent fraud and to prevent one node from performing malicious activities while masquerading as another node, a peer authentication system must also be established. Similar to the mechanism that's used in cryptocurrency wallets, these systems are usually based on public-private keypairs. Each peer generates a keypair and advertises its public key (which may or may not be distinct from the keypair used in the wallet), which can be used to encrypt messages intended for it. The keys can also be used to sign messages to prove that they originated from the holder of the private key.


## P2P in Spacemesh

The Spacemesh network is an unstructured peer-to-peer network. Each peer both connects to other peers and also accepts connections from peers. Each peer is identified by the peer's unique public key. This public key is used both to encrypt data in transit between peers, and for [peer authentication](#authentication). Note that this public key is not used outside of the P2P stack.

### Developing a custom P2P stack

Early on in the architecture process, we evaluated existing P2P stacks including the Ethereum stack and [libp2p](https://github.com/libp2p). We decided that these existing stacks were too tightly coupled to the protocols: for instance, the Ethereum P2P stack requires that all data be sent using a custom encoding called RDP, and libp2p necessitates use of Kademlia DHT, which is useful for file sharing but is not required by Spacemesh. We therefore decided to implement our own lightweight P2P stack. For more information on this decision process, see [The search for the perfect p2p library](https://medium.com/spacemesh/perfect-p2p-library-c559d1ca57dc) (but note that some of the information, such as the discussion of the DHT, is outdated).

### Protocols, multiplex, and gossip

Spacemesh [multiplexes](https://en.wikipedia.org/wiki/Multiplexing) messages from many sub-protocols (e.g., `discovery`, `hare`, `sync`) over each peer connection. Each individual protocol must register an incoming message handler with the P2P stack. When a protocol receives and validates a message, it may report back to the P2P stack that the message is valid, which allows the message to continue to propagate throughout the P2P network to the node's peers. In this way, the P2P stack is agnostic to the format and contents of messages belonging to individual sub-protocols.

### Messages

Each message is signed using the sender's public key. In addition to the message data itself (the payload), it includes a protocol, a client version, a timestamp, the public key of the message originator, a network ID, whether the message is a request or a response, the request ID, and the type of message in the specified protocol. All messages are serialized using [the XDR standard](https://en.wikipedia.org/wiki/External_Data_Representation).

### Peer Discovery

There are two methods by which peers may discover other peers, bootstrap and additional discovery (as [described above](#discovery)).

The following flow describes how a node requests other peers from another node:
1. Initiator must first send ping message, to verify that node is alive. The ping contains the node ID (public key), the sender's IP and port.
1. The recipient responds to the ping with a pong message, of the same format as the ping.
1. The initiator then sends a `getAddresses` message, requesting information for additional peers.
1. The recipient responds with a list of additional peers: ther node IDs, IP addresses, and ports.

### Session Initiation

At the beginning of each session initiation between two nodes, a secure P2P session needs to be established (using a TCP socket). To establish the session the initiator node first sends a handshake message containing its client version and the network ID it's trying to connect to. It then encrypts the message using a [Diffie-Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) shared secret key constructed using the initiator's private key and the recipient's public key. It sends the message, and its public key, to the recipient.

The recipient reconstucts the shared secret key, using its private key and the initator's public key, uses it to decrypt the handshake message, and checks the initiator's client version and network ID. If everything checks out, it responds to the handshake message and a session is established; if not, the TCP connection is closed and no further information is exchanged.

### Gossip

In a blockchain protocol like Spacemesh, many types of data, including transactions and blocks, must traverse the network and reach every node. The broadcast protocol that's used to achieve this is called the gossip protocol. Gossip allows each node to broadcast a message to the entire network, and also to receive, and forward on other gossip messages it receives.
