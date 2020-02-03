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

The Spacemesh network is an unstructured peer-to-peer network. Each peer both connects to other peers and also accepts connections from peers. Peers are identified by the node’s unique public key. This public key is also used to set up secure sessions.


### Messages

Each message is signed using the sender’s public key. 
All messages are wrapped with ProtocolMessage struct that contains
Metadata struct
String Next protocol: protocol of payload
String client version
Int64 Timestamp 
AuthPublicKey - the public key of the origin
Int32 Network Id
Payload - a struct that varies according to the protocol stated in the Next protocol field
[] bytePayload- the actual serialised protocol message
DataMsgWrapper wrapper - a struct of metadata to allow sessions for protocols:
Bool Req - whether this message is a request or response
Uint32 MsgType type of message in sub protocol
Uint64 req id - running id of messages
[] byte payload

All messages are serialized using [the XDR standard](https://en.wikipedia.org/wiki/External_Data_Representation).


### Peer Discovery

There are two methods by which peers may discover other peers.

#### Bootstrap

The first of these methods is called bootstrap (since it bootstraps the discovery mechanism). In order for a new node to discover peers and join the network, it needs to know the address of at least one other node at system startup. This way the new node can query the bootstrap node for a list of other nodes to connect to.

Second, after connecting to a peer as a bootstrap node, additional discovery takes place, where the newly added neighbors are queried for additional nodes as well.

All non gossip messages (protocol messages) are wrapped with a DataMsgWrapper message to create easier sessions of different kind for different protocols (discovery, sync, etc.)

The DataMsgWrapper consists of:
Boolean Req - is this message request or response
Uint64 request id
Uint32 msgType - type of message in a specific protocol

The following flow describes how a node requests other peers from another node:
Initiator must first send ping message, to verify that node is alive. 
Ping message consists of:
32 byte node id (ed25519)
Sender IP
Protocol port - on which the p2p protocol is to be send 
Discovery port
Pings are responded from the destination with pong messages, which are the same form as the ping messages
After ping response the initiator will send a getAddresses message that consists of:
msgType = 1
The returned message is an array of NodeInfo type
32 byte node id (ed25519)
Sender IP
Protocol port - on which the p2p protocol is to be send 
Discovery port 


### Session Initiation

At the beginning of each session initiation between two nodes, a secure P2P session needs to be established. Session is established using a tcp socket on port 7153
To establish the session the initiator node must create a handshake packet containing:
Client version - version of the P2P node
NetworkId - the Id of the network this node connects to (can be modified via parameter)
Port- the local incoming port on which this node awaits connections
The handshake packet then must be signed using a Diffie Helmann shared secret key that includes:
The receiver's public key.
The initiators private key
The initiators public key is then appended to the message

On the acceptor peer the following validations are performed:
A Diffie Helmann shared secret key is created using:
The initiators public key  - which is appended to the message
The receivers private key
The handshake packet is then decrypted using created shared secret
Client version and network id is validated
A session is initiated once a handshake packet is sent. If the handshake packet is invalid the session will be rejected by the acceptor and the underlying tcp connection will be closed by it. 


### Gossip

Gossip is how messages traverse the spacemesh network. It is important for each node to support and forward gossip messages. 
