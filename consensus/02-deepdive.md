# Protocol deep dive

As described in [Overview](01-overview.md), consensus in Spacemesh consists of two independent but complementary protocols, called the Tortoise and the Hare. This document provides more details and context on these protocols, explaining how they fit into the broader landscape of BFT consensus protocols, and why they're well-suited to the needs of Spacemesh.

## Tortoise

The Tortoise consensus mechanism was custom-designed to suit the needs of Spacemesh. It has three particular properties:

1. **Consensus:** it allows all nodes to achieve consensus (on the canonical set of blocks in each layer)
1. **Irreversibility:** older history is harder to change than more recent history
1. **Self-healing:** all nodes will eventually converge and achieve consensus even if they temporarily disagree

### Self-healing

One nice property the Tortoise gives us is self-healing: the ability to converge, and to achieve consensus, even if nodes are in initial disagreement, such as in case of an attack upon the network, or if many nodes suddenly go offline. Self-healing takes time, and the exact amount of time it takes depends upon both protocol parameters (such as the properties of the [beacon](#vrf-beacon)), as well as upon the situation when it runs (such as how many honest nodes remain). The best case scenario to achieve convergence is hours. If an attack ceases even temporarily, convergence will occur immediately. Even under constant attack, however, convergence will eventually occur.

Note that, due to self-healing, the Tortoise requires at least 2/3 honest majority. This is because the protocol works using votes, and there must be a large enough margin between honest and dishonest votes.

Self-healing works via a _separate byzantine agreement mechanism_ that does not depend on shared view of history, but depends, rather, on a [VRF-based beacon](#vrf-beacon).

### Other protocols

Blockchains based on Nakamoto-consensus, such as Bitcoin, also have this property, but it's much more difficult to achieve _without_ Proof of Work, as in the case of Spacemesh. This is because, in most BFT-style protocols, if the network loses consensus for even a moment, it's impossible to re-establish consensus. In protocols that have committees that nominate a committee for the following round, for instance, without agreement on the current committee, there's no way for the network to reach agreement on the next valid committee.

In a PBFT-style system, such as those based on HotStuff BFT, losing consensus in this manner might mean that different participants have a different view on which transactions are confirmed. Self-healing ensures that this cannot happen in Spacemesh.

Another important difference between the Tortoise and a BFT-style protocol is that the Tortoise is capable of a more robust form of consensus: even if assumptions fail for a while, things that have been in consensus for a while remain in consensus. Contrast this wih a BFT-style protocol, where you cannot rely on history in such a scenario, since it can be reversed.

## Hare

The Hare is a BFT-compatible algorithm and is based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf), with the difference that we want to achieve consensus on a set of values rather than a single bit value.

### What does it give us?

Because of the self-healing mechanism (described above), Spacemesh would function without the Hare. This is because self-healing guarantees that Spacemesh nodes will eventually converge and reach consensus from any starting condition, even without the initial, bootstrapped agreement that the Hare gives us. However, we would lose three things without the Hare:

1. **Time:** self-healing takes time.
1. **Efficiency:** the self-healing protocol is less efficient (measured in terms of communication complexity: the number of messages passed) than the Hare. We have tuned the Hare to have the lowest possible communication complexity.
1. **Robustness:** while self-healing requires > 2/3 honest majority, the Hare requires only a simple honest majority, and depending on the size of the committee it can theoretically tolerate up to 2/3 malicious participants. This is because it takes advantage of strong synchrony assumptions.

The Hare is designed to support up to approx. 800 participants.

### Compared to HotStuff and Tendermint

- [HotStuff](https://arxiv.org/pdf/1803.05069.pdf) requires a known set of participants (i.e., it does not support player replaceability), cannot be made permissionless, doesn't scale to thousands of participants, and does not easily convert to set agreement (i.e., agreement on a set of valus rather than on a single bit value). The requirement to have a known set of participants opens a DoS attack vector on known future participants.
- HotStuff and Tendermint both require a 2/3 honest majority, whereas the Hare requires only a simple honest majority
- HotStuff and Tendermint make different assumptions about network synchrony: they both assume a _partially-synchronous network_ (i.e., there exists a "global stabilization time" after which all messages may be assumed to have been delivered, but you don't know exactly when this time is). Hare, on the other hand, assumes a _fully-synchronous network,_ i.e., that all messages are delivered within a known period of time (which is specified as a protocol parameter, and affects layer time). As a result, HotStuff and Tendermint can make progress faster than Hare, but at the cost of having a lower corruption threshold.
- HotStuff and Tendermint achieve a form of consensus known as _state machine replication_ (SMR): this is a classical form of PBFT consensus where participants in the protocol agree upon an ordered sequence of transactions. The Hare protocol plays a different role in the Spacemesh protocol, and thus its goal is not to achieve SMR-style consensus. Rather, every time the Hare runs, its goal is to run for a finite number of rounds and achieve a "one-time agreement" (on the set of valid blocks in the current layer), then terminate. Terminating an SMR-style consensus protocol is more difficult since it's unclear how to ensure that all participants have achieved precisely the same state--indeed, these protocols are designed to run through an indefinite number of rounds.

## VRF beacon

(to fill in)
