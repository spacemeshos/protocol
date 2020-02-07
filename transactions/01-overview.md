# Transactions
## Overview

The transaction is one of the most basic data structures used in Spacemesh and in many other blockchain protocols. There are two types of transactions: a simple coin transaction, which works on the live Spacemesh network today, and a smart contract call, which will be added to the protocol in the future. (Throughout the rest of this document, the term "transaction" refers to a coin transaction.)

In Spacemesh, a transaction is an order to transfer funds from one account to another. The transaction specifies the amount to be transferred, the sender's wallet address, the recipient's wallet address, and the fee to be paid to the miner that mines the transaction. The transaction is signed using the private key of the sender, which is how the protocol knows that the transaction is legitimate (i.e., that it originated with the holder of the sender's private key).

## Account model

An account in Spacemesh is a mapping from a wallet address to a balance of Smesh tokens. Note that there is not a one-to-one mapping between _users_ and _addresses,_ as one user may control many addresses (or, less often but also possibly, one address could be controlled by many users).

Even though a transaction is signed, so that the protocol can verify that it came from the sender, without additional protections, transactions are still susceptible to a [replay attack](https://en.wikipedia.org/wiki/Replay_attack): if Alice sends funds to Bob, Bob could resend the same transaction (that Alice already signed) to the network to steal additional funds from Alice. In order to prevent replay attacks, each account also stores a transaction counter, also known as a "nonce." Each transaction must declare a nonce, and a transaction is only valid if it has a nonce that matches the current value of the account counter.

### Address format and signature scheme

Spacemesh uses the standard ED25519 curve to generate keypairs. A private key is 32 bytes of random data. The public key is derived from the private key using ECDSA derivation. The Spacemesh address is the 20-byte prefix of the hash of the public key. In other words, the wallet address may be expressed as:

`address = Bytes[0..159](HASH(ECDSAPUBKEY(privkey)))`

## Transaction structure

The actual transaction data structure in Spacemesh contains the following:

- Recipient's address (32 bytes)
- Amount to transfer (8 bytes)
- Amount of the fee to be paid (to the miner) (8 bytes)
- Nonce (8 bytes)
- Signature (32 bytes)

The transaction is signed using the private key corresponding to the sender's account. Note that the sender's address is not _explicitly_ included in the transaction. This is because it can be _implicitly derived_ from this signature using ECDSA derivation.

The total size of a transaction is 88 bytes.

### Syntactic validity

Technically, any message that is 88 bytes long can be interpreted as a syntactically valid transaction. However, the transaction isn't _contextually valid_, and thus won't be applied to the global state, unless certain conditions are met (see next section).

## Applying transactions

### Transaction ordering


### Contextual validity


### Global state

## Mempool

## Fees and mining rewards
