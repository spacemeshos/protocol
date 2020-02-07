# Transactions
## Overview

The transaction is one of the most basic data structures used in Spacemesh and in many other blockchain protocols. There are two types of transactions: a simple coin transaction, which works on the live Spacemesh network today, and a smart contract call, which will be added to the protocol in the future. (Throughout the rest of this document, the term "transaction" refers to a coin transaction.)

In Spacemesh, a transaction is an order to transfer funds from one account to another. The transaction specifies the amount to be transferred, the sender's wallet address, the recipient's wallet address, and the fee to be paid to the miner that mines the transaction. The transaction is signed using the private key of the sender, which is how the protocol knows that the transaction is legitimate (i.e., that it originated with the holder of the sender's private key).

## Account model

Spacemesh uses the account model for storing value. This means that users are encouraged to reuse accounts for multiple transactions, similar to a bank account, rather than generate a new address for every transaction (as in the UTXO model). When multiple transactions increase the balance of an account, the funds from different transactions become fungible and indistinguishable from one another.

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

Transactions are applied, one by one, to the global state. Valid transactions cause the state to be updated. Of course, since Spacemesh defines a canonical ledger, the order in which transactions are applied is very important.

<a name="ordering"></a>
### Transaction ordering

Transaction order in Spacemesh is defined in the following way:

1. Blocks in layer _n_ come before those in layer _n+1_
1. The IDs of all [contextually valid](../consensus/01-overview.md#validity) blocks in layer _n_ are listed in ascending order
1. These block IDs are concatenated and hashed
1. The hash sum is used as the seed for a [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) (a type of pseudorandom number generator)
1. A [Fisher-Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle) is performed on the list of block IDs using the output of the Mersenne Twister
1. Transactions from the ordered blocks are then applied, in the order they appear in each block, ignoring duplicates (only the first appearance of a transaction determines its ordering)

### Contextual validity

A transaction is deemed contextually valid, and is applied to the global state, if the following conditions apply:

1. The origin account (derived from the signature) exists
1. The nonce on the account matches the transaction nonce
1. The account balance is greater than or equal to the transaction amount fee

### Global state

When a transaction is applied to the global state, it is passed through the following state transition function:

1. The origin account balance is decremented by the transaction amount + fee
1. The recipient account balance is incremented by the transaction amount
1. The origin account nonce is incremented by one

After each pass over the list of transactions, another pass is performed on the remaining (unapplied) transactions, in the same order, until no transaction from the list can be applied.

## Mempool

Miners receive incoming, unprocessed transactions via the [gossip network](../p2p/01-overview.md) and locally over GRPC. When a miner receives a transaction, it checks that it's syntactically valid and that it hasn't already seen the transaction before (i.e., those that it's not already in the mempool, or in the mesh as part of at least one block). After that, it saves the transaction into its mempool. Each miner maintains its own mempool.

Miners are expected to include transactions in blocks that are predicted to be valid when they get applied to the global state, with a high likelihood.

Gossip network participants are expected to gossip _all syntactically valid_ transactions to the network, regardless of contextual validity.

## Fees and mining rewards

As in Bitcoin and other blockchain platforms, Spacemesh transactions pay fees to miners to incentivize them to include the transaction in a block. However, fees and mining rewards work a bit differently in Spacemesh.

### Mining rewards

Time in Spacemesh is divided into fixed-length units of time called [layers and epochs](../intro.md#spacemesh-basics). An epoch consists of a fixed number of layers.

Every five minutes, the Spacemesh protocol distributes 50 Smesh (SMH) to all miners who contributed at least one block to the mesh over the prior five minute period. The amount of the reward paid to each miner depends on the number of miners who contributed blocks, and the number of blocks contributed. A miner that contributes more blocks receives more reward.

### Rewards adjustment

As an incentive to miners to submit blocks with many transactions, the reward for a block with less than 15 transactions is reduced by 10%. This amount is distributed as a bonus evenly to all miners who submitted at least 15 transactions in a block.

### Transaction fees

Like the wait staff in a restaurant pooling tips, transaction fees in Spacemesh are also pooled, per layer, and evenly distributed to all miners who submitted at least one block in the layer.
