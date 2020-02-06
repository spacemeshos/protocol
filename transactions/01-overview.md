# Transactions
## Overview

The transaction is one of the most basic data structures used in Spacemesh and in many other blockchain protocols. There are two types of transactions: a simple coin transaction, which works on the live Spacemesh network today, and a smart contract call, which will be added to the protocol in the future. (Throughout the rest of this document, the term "transaction" refers to a coin transaction.)

In Spacemesh, a transaction is an order to transfer funds from one account to another. The transaction specifies the amount to be transferred, the sender's wallet address, the recipient's wallet address, and the fee to be paid to the miner that mines the transaction. The transaction is signed using the private key of the sender, which is how the protocol knows that the transaction is legitimate.

## Account model

An account in Spacemesh is a mapping from a wallet address to a balance of Smesh tokens. Note that there is not a one-to-one mapping between _users_ and _addresses,_ as one user may control many addresses.

### Address format and signature scheme

Spacemesh uses the standard ED25519 curve to generate keypairs. A private key is 32 bytes of random data. The public key is derived from the private key using ECDSA derivation. The Spacemesh address is the 20-byte prefix of the hash of the public key. In other words, the wallet address may be expressed as:

`address = Bytes[0..159](HASH(ECDSAPUBKEY(privkey)))`
