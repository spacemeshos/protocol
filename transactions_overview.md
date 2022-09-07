# Transactions

## Overview

The transaction is one of the most basic data structures used in Spacemesh and in many other blockchain protocols. In Spacemesh, there are three types of transactions: a simple coin transfer, an activation transaction (ATX), and a smart contract call. Activation transactions are a special type of transaction used as part of the mining process. They are [described in the mining docs](atx.md). Smart contract transactions are part of the experimental [Spacemesh Virtual Machine (SVM)](https://github.com/spacemeshos/svm) project, which is not in production yet. Throughout the rest of this document, the term "transaction" refers to a coin transfer.

In Spacemesh, a transaction is an order to transfer funds from one account to another. The transaction specifies the amount to be transferred, the sender's wallet address, the recipient's wallet address, and the fee to be paid to the miner that mines the transaction. The transaction is signed using the private key of the sender, which is how the protocol knows that the transaction is legitimate (i.e., that it originated with the holder of the sender's private key).

## Account model

Spacemesh uses the account model for storing value. This means that users are encouraged to reuse accounts for multiple transactions, similar to a bank account, rather than generate a new address for every transaction (as in the UTXO model). When multiple transactions increase the balance of an account, the funds from different transactions become fungible and indistinguishable from one another.

An account in Spacemesh is a mapping from a wallet address to a balance of Smesh tokens. Note that there is not a one-to-one mapping between _users_ and _addresses,_ as one user may control many addresses (or, less often but also possibly, one address could be controlled by many users).

Even though a transaction is signed, so that the protocol can verify that it came from the sender, without additional protections, transactions are still susceptible to a [replay attack](https://en.wikipedia.org/wiki/Replay_attack): if Alice sends funds to Bob, Bob could resend the same transaction (that Alice already signed) to the network to steal additional funds from Alice. In order to prevent replay attacks, each account also stores a transaction counter (some other platforms such as Ethereum call this a "nonce"). Each transaction must declare a counter, and a transaction is only valid if it has a counter that matches the current value of the account counter.

### Address format and signature scheme

Spacemesh uses the standard `curve25519` to generate keypairs. A private key is a random 32 byte number, and the public key is generated from the private key. The Spacemesh address is the 20-byte suffix of the public key. In other words, the wallet address may be expressed as:

`address = Bytes[12..31](public_key)`

## Transaction structure

The actual transaction data structure in Spacemesh contains the following:

- Recipient's address (20 bytes)
- Amount to transfer (8 bytes)
- Amount of the fee to be paid (to the miner) (8 bytes)
- Transaction counter (8 bytes)
- Signature (64 bytes)

The transaction is signed using the private key corresponding to the sender's account. Note that the sender's address is not _explicitly_ included in the transaction. This is because it can be _derived_ from this signature (using [public key extraction](https://crypto.stackexchange.com/questions/18105/how-does-recovering-the-public-key-from-an-ecdsa-signature-work/18106) applied to the EdDSA signature scheme).

The total size of a transaction is 108 bytes.

### Syntactic validity

Technically, any message that is 108 bytes long can be interpreted as a syntactically valid transaction. However, the transaction won't necessarily, ultimately be applied to the global state unless [certain conditions](#contextual-validity) are met later.

## Mining transactions

### Mempool and gossip

Miners receive incoming, unprocessed transactions via the [gossip network](p2p_overview.md) and locally over GRPC. When a miner receives a transaction, it checks that the transaction is valid and that it hasn't already seen the transaction before (i.e., it's not already in the mempool, nor in the mesh as part of at least one block).

The node next performs the following basic checks on the transaction:

- Is it syntactically valid? (i.e., the right length)
- Does the sender account exist, and contain a sufficient balance (>= the amount + fee)?
- Is the transaction counter correct?

If these are all true, then the node saves the transaction into its mempool (each miner maintains its own mempool) and gossips it to the network. If not, then the transaction is rejected, is not added to the mempool, and is not gossiped.

Note that the node considers other _pending outgoing transactions_ when performing this check. In other words, if Alice sends three transactions in a row, and none of them have been confirmed yet, they are only considered valid and added to the mempool if they have sequential transaction counters that start at the current counter value, and if none of the transactions "overdrafts" the account (including both the transaction amount and the fee). In particular, note that this calculation _does not consider pending incoming transactions._

### Assembling blocks

Each mining node is incentivized to include as many fee-paying transactions as possible into blocks that it assembles. Note that it is not trivial to determine which transactions will pay fees. This is because only transactions that are _[contextually valid](#contextual-validity) in the instant when they're [applied to the global state](#global-state)_ pay fees (invalid transactions are discarded). Moreover, unlike in a blockchain, in Spacemesh a miner actually has no control over the final order of transactions in a layer. This is because the miner submits only a single block to the layer, and does not know the ultimate [order of blocks and transactions](#transaction-ordering) in that layer. If another conflicting transaction appears before the transaction in question—e.g., one that spends all of the funds in the sending account, or has the same transaction counter—then the transaction in question may ultimately be deemed invalid, and discarded, without paying a fee.

For the same reason, in Spacemesh, _one invalid transaction does not invalidate a block._

## Applying transactions

The process of determining the current canonical state is simply the process of ordering and applying all of the transactions in the canonical mesh, in order. This process is explained in greater detail here.

Transactions are applied one by one, in the order in which they appear in blocks, to the global state. Valid transactions cause the state to be updated. Of course, since Spacemesh defines a canonical ledger, the order in which transactions are applied is very important.

### Transaction ordering

Transaction order in Spacemesh is defined in the following way:

1. Blocks in layer _n_ come before those in layer _n+1_
1. The IDs of all [contextually valid](consensus_overview.md#validity-rules) blocks in layer _n_ are listed in ascending order
1. These block IDs are concatenated and hashed
1. The hash sum is used as the seed for a [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) (a type of pseudorandom number generator)
1. A [Fisher-Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle) is performed on the list of block IDs using the output of the Mersenne Twister
1. Transactions from the ordered blocks are then applied, in the order they appear in each block, ignoring duplicates (only the first appearance of a transaction determines its ordering)

### Contextual validity

A transaction is deemed contextually valid if, _at the moment when a transaction is applied to the global state,_ the following conditions apply:

1. The transaction appears in a [contextually valid block](consensus_overview.md#block-validity-in-spacemesh)
1. The origin account (derived from the signature) exists
1. The counter on the account matches the transaction counter
1. The account balance is greater than or equal to the transaction amount + fee

### Global state

The transactions in a given layer are applied to global state, [in order](#transaction-ordering), when that layer is finalized.

#### Finality

There are two stages of finalization in Spacemesh: the Hare and the Tortoise. A layer (and all of the transactions it includes) is applied to global state when the Hare protocol reaches consensus about it, usually within four minutes of when the blocks are produced. When the Tortoise reaches this layer later (depending on network parameters and conditions, this will usually be two layers later) it can reaffirm the output of the Hare or reverse its conclusion. In the latter case, both transactions and rewards are rolled back and reapplied based on the new set of valid blocks.

In the future, self healing will also possibly cause a rollback, but this is not implemented yet.

For more information on the consensus process, see [Consensus](consensus_overview.md).

#### State transition function

The entire process of applying the transactions for a given layer is performed as a single, atomic database transaction. Transactions are applied by being passed through the following state transition function:

1. The origin account balance is decremented by the transaction amount + fee
1. The recipient account balance is incremented by the transaction amount
1. The origin account transaction counter is incremented by one

After each pass over the list of transactions, another pass is performed on the remaining (unapplied) transactions, in the same order, until no transaction from the list can be applied. The worst case performance is `O(N^2)`, and the expected (i.e., non-malevolent) case is `O(N)`, where `N` is the number of transactions in the layer. (In fact, it's `O(N*M)` where `M` is the length of the longest chain of intra-layer dependencies. However, `M` is expected to be very small since a wallet should not create transactions that depend upon other transactions in the same layer.)

Fees are distributed to the miner at the same time, but using an independent state transition. This is because transaction fees are pooled per layer.

### Fees and rewards

As in Bitcoin and other blockchain platforms, a Spacemesh transaction pays a fee to incentivize a miner to include the transaction in a block. However, fees and mining rewards work a bit differently in Spacemesh.

#### Mining rewards

Time in Spacemesh is divided into fixed-length units of time called [layers and epochs](README.md#spacemesh-basics). An epoch consists of a fixed number of layers. Each layer is five minutes long and contains a set of blocks.

The Spacemesh protocol distributes 50 Smesh (SMH) (subject to the Smesh minting schedule) to the miners who contributed blocks to each layer. The amount of the reward paid to each miner depends on the number of blocks contributed by that miner, and on the total number of blocks contributed in that layer. A miner that contributes more blocks receives more reward.

#### Transaction fees

Like the wait staff in a restaurant pooling tips, transaction fees in Spacemesh are also pooled per layer and evenly distributed to all miners who contributed blocks to the layer, proportional to how many (contextually valid) blocks they contributed.

#### Timing

Both mining rewards and fees are credited to miners as part of the same atomic database transaction where all transactions in a layer are applied.

#### Block weights

At present, both block rewards and fees are divided equally among all the published, [contextually valid](consensus_overview.md) blocks in a layer: in other words, a miner that contributed four blocks (and was eligible to contribute at least four blocks) would receive precisely twice the reward and twice the fees for that layer as a miner who contributed (and was eligible to contribute) two.

However, this is subject to change as Spacemesh adds support for _block weights._ Under the system of block weights, each miner will instead receive a share (of rewards and fees) based on the product of storage x ticks they declared in their [activation transaction (ATX)](atx.md). This share is divided by the number of blocks they are _expected_ (i.e., eligible) to produce during the entire epoch. Each contextually valid block they ultimately produce will grant them a portion of this share. (E.g., if a miner is eligible to produce 50 blocks during a given epoch, and only produces 25, it will receive only half of the share. The rest will be distributed among all published blocks along with the rest of the pool.)
