# Data structures

This document lays out the various top-level data structures that are used throughout the Spacemesh protocol, with details on their contents and how they're linked.

## Mesh

The mesh is the set of all layers.

## Layer

A layer is defined as a set of blocks.

## Block

A block contains a set of transactions. It may be syntactically or contextually valid or invalid. It also contains a pointer to an ATX that establishes the eligibility of the miner to produce the block in the layer.

## Transaction

## ATX

## P2P Messages

Each message is signed using the sender's public key. In addition to the message data itself (the payload), it includes a protocol, a client version, a timestamp, the public key of the message originator, a network ID, whether the message is a request or a response, the request ID, and the type of message in the specified protocol. All messages are serialized on the wire using [the XDR standard](https://en.wikipedia.org/wiki/External_Data_Representation).
