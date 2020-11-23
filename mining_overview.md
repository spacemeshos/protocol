# Mining

## Overview

Mining, which is known as Smeshing in [Spacemesh parlance](https://github.com/spacemeshos/testnet-guide/blob/master/dict.md), is the process by which nodes in the network create new blocks. In general, It has two distinct purposes:

1. To maintain the canonical list of transactions
2. To mint and distribute new tokens

From the canonical list of transactions one can compute the state of the system (now or at any point in the past). Due to the lack of a trusted central authority, one must rely on an honest majority to know the canonical list of transactions. But the notion of majority is problematic in a permissionless setting, due to the lack of identity verification: it may be relatively easy for an adversary to create many fake identities (known as “Sybils”).

Several strategies have been suggested for defending against such an attack (often referred to as “Sybil attack”), typically by making it costly to create and maintain an identity. For instance, one of the most popular strategies is to force users of the system to spend scarce resources in order to participate in mining. Creating multiple identities would require an attacker to spend a large amount of resources, approximately linear in the number of identities. If that resource is limited and costly, an attacker cannot spend it indefinitely. So by binding the notion of identity to a resource, we can create the notion of a Sybil-resistant majority, not strictly of actors but rather of resources.

Any bounded resource can be used as a “payment” for the eligibility to mine. This “payment” isn’t paid to anyone, it is simply wasted or “burnt.” Its purpose is to prevent Sybil attacks, and thus to make the honest majority security assumption viable. That majority can be used by nodes on the network to reach consensus regarding the canonical list of transactions. It thus[ solves](https://satoshi.nakamotoinstitute.org/emails/cryptography/11/#selection-47.45-47.72) a classic problem in distributed computing called the [Byzantine Generals Problem](https://people.eecs.berkeley.edu/~luca/cs174/byzantine.pdf), even among a permissionless, changing set of actors, and was first introduced by Satoshi Nakamoto with the inception of Bitcoin.

In order to incentivize miners to spend that resource, essential to the security of the network, miners are rewarded with newly minted tokens. This kills two birds with one stone: the initial distribution of tokens into circulation, and the incentive for nodes to support the security of the network.

## Proof of Work

The earliest, most popular, and most battle-tested form of resource payment is to commit computing power. It does not require any specialized infrastructure. In order to ensure that users actually do spend the appropriate resource payment, users employ a “Proof of Work” (PoW).

The most widely used implementation of PoW is the Hash-Preimage PoW (also known as [Hashcash](http://hashcash.org/papers/announce.txt)). It has useful properties:

* It’s relatively simple
* The work is easy to verify, hard to fake
* It is non-interactive (anyone can pick their own challenge and present the proof)

While effective, it also has some significant drawbacks:

* It incentivizes the use of dedicated hardware (e.g., an ASIC), making mining impossible using consumer-grade hardware such as a home PC
* It incentivizes individual miners to combine resources into a small number of centralized pools
* It leads to high cost of entry for mining, which sets a lower bound for the mining reward (since rational parties would only participate in mining if their reward exceeds their cost)
* It is environmentally wasteful and harmful (requiring dedicated hardware and high ongoing electricity consumption)
* It creates a lottery-based consensus protocol, and so a race among miners. This creates high competitive cost for large blocks, a harmful incentive to withhold blocks, unfair reward distribution due to head start, lower overall transaction throughput, etc.

## Proof of Stake

The most popular alternative to Proof of Work is called Proof of Stake, an approach to decentralized consensus that replaces computing power, the scarce resource committed by miners in Proof of Work as described above, with a bonded stake, typically denominated in the native token of the network in question. If an actor participating in securing the network, typically referred to in PoS systems as a validator, misbehaves, they may risk losing part or all of this stake (whereas in the case of PoW they would instead forfeit the cost of the electricity and computation expended to produce the invalid block). PoS has the benefit of being much more environmentally friendly than PoW since it totally obviates the need to perform wasteful computation, but it’s subject to its own set of deficiencies:

* A relatively high barrier to entry for participation in consensus, as a validator typically needs USD $1,000 or more worth of a token to begin validating.
* Strong assumptions are required to prevent long-range attacks. Whereas PoW only requires that the hash function used in the PoW algorithm be trusted, by contrast, PoS requires additional assumptions, such as checkpointing or secure erasures, to protect against long-range attacks where validators collude to cheaply produce many blocks until they overtake the current longest chain.
* A token distribution problem. In PoW (and [PoST](02-post.md)), tokens can be handed directly and exclusively to miners, such that PoW can be used to bootstrap a relatively fair initial token distribution. PoS, by contrast, requires that there be an initial set of validators who already hold, and stake, tokens. It therefore cannot be used to bootstrap initial token distribution.
* In the strictest sense, and in contrast to PoW and PoST, PoS is not permissionless since a new validator needs to convince an existing token holder to sell or lend them tokens for staking.
* While formal permissionlessness may be primarily of academic interest, for these reasons, PoS networks are in practice subject to capture by cartels of large token holders. The cost of such an attack, where a cartel may gain control of the majority of a network’s tokens and thus validation slots, is relatively low early in the life of a network, but it can lead to problems down the road. One example is cartel censorship, whereby a majority cartel prevents other parties from participating in consensus and may engage in subtle attacks such as prioritizing transactions in exchange for extra-protocol bribes.
* In practice many token holders opt to store their tokens on large, centralized exchanges, temporarily or over the long-term, giving such exchanges disproportional power to participate in consensus and collect rewards.

## Proof of Space-time

Spacemesh replaces Proof of Work (PoW) with a novel consensus mechanism called Proof of Space-time ([PoST](02-post.md)) that shares many of the benefits of PoW but is nearly as energy efficient as PoS. In a nutshell, PoST allows miners to use disk space rather than computation as the scarce resource for preventing Sybil attacks. After a one-time computationally-intensive initialization phase, the miner earns rewards by creating blocks with a computer that is almost completely at rest, for as long as they keep storing this data.

Storage equipment is a general-purpose item that can be readily obtained by individual consumers. It is also common that storage devices are underutilized, e.g., it is not unusual for a hard drive in a new computer to be half empty. Since many home users are able to participate and earn rewards without purchasing any mining resource in advance (in other words, at zero marginal cost), PoST could lead to much greater decentralization (i.e., more participants in consensus) than is possible under PoW or PoS.

PoST replaces the random sampling of PoW with deterministic eligibility criteria: every miner that expends a sufficient space-time resource will be eligible to generate a block. Since eligibility isn’t randomized, PoST is much less vulnerable to “grinding” attacks (where the adversary attempts to perform extra work—that is not specified by the honest protocol—in order to increase their probability of being selected in the lottery). This property, and the fact that multiple blocks are produced at each block height (known as a “layer” in Spacemesh), leading to a DAG data structure rather than a chain, mean that PoST enables a blockchain like Spacemesh to be [race free](../README.md#why-race-free).

PoST has many nice properties, but it also has some drawbacks that make it significantly harder to use for consensus. For instance, PoST allows a miner to prove that they generated the data required to submit an initialization proof, but not that they kept that data over a longer period of time (they may have regenerated it). As another example, PoST doesn’t in and of itself generate randomness the way PoW does, so for security we need another source of randomness to make it difficult to predict who will be eligible to generate a block, when. These and other challenges and workarounds will be discussed in later sections.

## Towards Proof of Space-time

The remainder of the articles in this section of the documentation will explain the various components of the mining protocol as implemented in Spacemesh: [Proof of Space-time (PoST)](02-post.md), used to attest that a miner generated and stored a specific amount of data for a specific length of time; [Proof of Elapsed Time (PoET)](03-poet.md), used to attest that a specific amount of objective, clock time has passed since a proof was generated; [Non-interactive Proof of Space-time (NIPoST)](04-nipost.md), which ties together the previous two types of proof; and, finally, [activation transactions (ATX)](05-atx.md), used by miners to attest to their eligibility in the Spacemesh protocol by submitting such a proof construct to the network.

[Read on](02-post.md) to learn more about the protocol and how its various pieces fit together.
