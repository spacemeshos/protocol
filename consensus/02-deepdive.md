# Protocol deep dive

As described in [Overview](01-overview.md), consensus in Spacemesh consists of two independent but complementary protocols, called the Tortoise and the Hare. This document provides more details and context on these protocols, explaining how they fit into the broader landscape of BFT consensus protocols, and why they're well-suited to the needs of Spacemesh.

## Tortoise

The Tortoise consensus mechanism was custom-designed to suit the needs of Spacemesh. It's a traditional Byzantine agreement protocol that adds a self-healing feature. It's based on previous work done on [Meshcash](https://eprint.iacr.org/2017/300) by Bentov, Hubáček, Moran and Nadler.

The Tortoise protocol has three particularly important properties:

1. **Consensus:** it allows all nodes to achieve consensus (on the canonical set of blocks in each layer)
1. **Irreversibility:** older history is harder to change than more recent history
1. **Self-healing:** all nodes will eventually converge and achieve consensus even if they temporarily disagree

### How it works

The Tortoise counts votes, cast in more recent blocks, for and against previous blocks. These votes indicate whether those previous blocks should be considered [contextually valid](01-overview.md#block-validity-in-spacemesh), i.e., whether, from the perspective of the voter, the block was received on time. In most cases, the vast majority of nodes will agree on a given block's validity, and there will be an overwhelming number of votes cast for or against the block. In rare cases, such as in the case of a [balancing attack](https://arxiv.org/abs/1612.09426), a block may not clearly cross either threshold.

Votes are processed in "clusters" of 800—i.e., 800 blocks that vote on previous blocks (the number 800 is a product of the statistical security parameter of `2^-40`). If these votes are overwhelmingly positive or negative, e.g., net +600 in favor of a block ("This block is valid") or net -700 against a block ("This block did not arrive on time"), no additional work is needed to achieve consensus. If the net vote for a block is closer to zero, e.g., +100, then the protocol enters "self-healing" and all honest parties factor in the result of a "weak coin" toss into their subsequent votes.

### Self-healing

Self-healing is the ability to converge, and to achieve consensus, even if nodes are in initial disagreement, such as in case of a network partition or an attack. Self-healing takes time, and the exact amount of time it takes depends upon both protocol parameters (such as the properties of the beacon, as well as upon the situation when it runs (such as how many honest nodes remain). If all honest nodes come back online, then convergence should happen more or less instantly. Convergence is still possible even in the case of an ongoing partition or attack, but in this case it will take hours.

Note that, due to self-healing, the Tortoise requires at least 2/3 honest majority. This is because the protocol works using votes, and there must be a large enough margin between honest and dishonest votes.

Self-healing works via a _separate byzantine agreement mechanism_ that does not depend on shared view of history, but depends, rather, on a beacon.

### Tortoise beacon

The Tortoise beacon uses a VRF-based protocol to generate a "weak coin" toss. In each round, each party calculates the output of a VRF based on their identity and the round number. Parties with a sufficiently low result publish their results and the lowest result wins. The value of the "coin toss" is one bit, e.g., the highest-order bit, of this lowest result.

If the lowest result comes from an honest participant, it will be published on time and everyone will receive this result, and all honest parties will agree on it. The "weak" nature of the coin comes from the fact that, in some cases, a dishonest adversary may "win" and be able to bias the results by, e.g., choosing not to publish their result, publishing it late, etc.

In the case where all honest parties _do_ agree on the result of the "coin toss," they will all adjust their next round of votes on the blocks in question _in the same direction_ (e.g., "heads" means "vote yes"), and convergence will be achieved in the following round. In the case where an adversary successfully attacks the result, convergence will be delayed one round. Note that the honest parties only need to "win" this coin toss game once in order to achieve convergence, and over time the likelihood of this occurring only grows.

### Compared to other protocols

Blockchains based on Nakamoto-consensus, such as Bitcoin, also have this property, but it's much more difficult to achieve _without_ Proof of Work, as in the case of Spacemesh. This is because, in most BFT-style protocols, if the network loses consensus for even a moment, it's impossible to re-establish consensus. In protocols that have committees that nominate a committee for the following round, for instance, without agreement on the current committee, there's no way for the network to reach agreement on the next valid committee.

In a PBFT-style system, such as those based on HotStuff BFT, losing consensus in this manner might mean that different participants have a different view on which transactions are confirmed. Self-healing ensures that this cannot happen in Spacemesh.

Another important difference between the Tortoise and a BFT-style protocol is that the Tortoise is capable of a more robust form of consensus: even if assumptions fail for a while, things that have been in consensus for a while remain in consensus. Contrast this wih a BFT-style protocol, where you cannot rely on history in such a scenario, since it can be reversed.

## Hare

The Hare is a BFT-compatible algorithm and is based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf), with the difference that we want to achieve consensus on a set of values rather than a single bit value.

### What does it give us?

Because of the self-healing mechanism (described above), Spacemesh would function without the Hare. This is because self-healing guarantees that Spacemesh nodes will eventually converge and reach consensus from any starting condition, even without the initial, bootstrapped agreement that the Hare gives us. However, we would lose three things without the Hare:

1. **Time:** self-healing takes time.
1. **Efficiency:** the self-healing protocol is less efficient (measured in terms of both computational and communication complexity) than the Hare. We have put considerable work into optimizing the Hare.
1. **Robustness:** while self-healing requires > 2/3 honest majority, the Hare requires only a simple honest majority, and depending on the size of the committee it can theoretically tolerate up to 2/3 malicious participants. This is because it takes advantage of strong synchrony assumptions.

The Hare is parameterized to support up to approx. 800 participants. This is based on a statistical security parameter of `2^-40`.

### Compared to HotStuff and Tendermint

- [HotStuff](https://arxiv.org/pdf/1803.05069.pdf) requires a known set of participants (i.e., it does not support player replaceability), cannot be made permissionless, doesn't scale to thousands of participants, and does not easily convert to set agreement (i.e., agreement on a set of valus rather than on a single bit value). The requirement to have a known set of participants opens a DoS attack vector on known future participants.
- HotStuff and Tendermint both require a 2/3 honest majority, whereas the Hare requires only a simple honest majority
- HotStuff and Tendermint make different assumptions about network synchrony: they both assume a _partially-synchronous network_ (i.e., there exists a "global stabilization time" after which all messages may be assumed to have been delivered, but you don't know exactly when this time is). Hare, on the other hand, assumes a _fully-synchronous network,_ i.e., that all messages are delivered within a known period of time (which is specified as a protocol parameter, and affects layer time). As a result, HotStuff and Tendermint can make progress faster than Hare, but at the cost of having a lower corruption threshold.
- HotStuff and Tendermint achieve a form of consensus known as _state machine replication_ (SMR): this is a classical form of PBFT consensus where participants in the protocol agree upon an ordered sequence of transactions. The Hare protocol plays a different role in the Spacemesh protocol, and thus its goal is not to achieve SMR-style consensus. Rather, every time the Hare runs, its goal is to run for a finite number of rounds and achieve a "one-time agreement" (on the set of valid blocks in the current layer), then terminate. Terminating an SMR-style consensus protocol is more difficult since it's unclear how to ensure that all participants have achieved precisely the same state--indeed, these protocols are designed to run through an indefinite number of rounds.
