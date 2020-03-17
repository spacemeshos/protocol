# Protocol deep dive

As described in [Overview](01-overview.md), consensus in Spacemesh consists of two independent but complementary protocols, called the Tortoise and the Hare. This document provides more details and context on these protocols, explaining how they fit into the broader landscape of BFT consensus protocols, and why they're well-suited to the needs of Spacemesh.

## Tortoise

The Tortoise consensus mechanism was custom-designed to suit the needs of Spacemesh. It has three particular properties:

1. Consensus: all nodes to achieve consensus (on the canonical set of blocks in each layer)
1. Irreversibility: older history is harder to change than more recent history
1. Self-healing: all nodes will eventually converge and achieve consensus even if they temporarily disagree

### Self-healing

One nice property the Tortoise gives us is self-healing: the ability to converge, and to achieve consensus, even if nodes are in initial disagreement, such as in case of an attack upon the network, or if many nodes suddenly go offline. Self-healing takes time, and the exact amount of time it takes depends both upon protocol parameters (such as the properties of the beacon), as well as upon the scenario. Factors include how many honest nodes remain, and the best case scenario to achieve convergence is hours. If an attack ceases even temporarily, convergence will occur immediately. Even under constant attack, however, convergence will eventually occur.

Note that, due to self-healing, the Tortoise requires at least 2/3 honest majority. This is because the protocol works using votes, and there must be a large enough margin between honest and dishonest votes.

- The difference between the Tortoise and a BFT-style protocol (including the ones discussed below) is that the Tortoise is capable of a more robust form of consensus: even if assumptions fail for a while, things that have been in consensus for a while remain in consensus. Contrast this wih a BFT-style protocol, where you cannot rely on history in such a scenario, since it can be reversed.
- Furthermore, the Tortoise has self-healing capabilities: once conditions return to normal (i.e., assumptions are met once again), the system can fully recover. In a BFT-style protocol, if consensus fails once, it fails for good.

The Tortoise is quite a robust consensus mechanism in its own right, and to be clear, Spacemesh could function with just the Tortoise. However, to add efficiency, we need the Hare as well.

## Hare

The Hare is a BFT-compatible algorithm and is based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf), with the difference that we want to achieve consensus on a set of values rather than a single bit value.

### What does it give us?

Because of the self-healing mechanism (described above), Spacemesh would function without the Hare. This is because self-healing guarantees that Spacemesh nodes will eventually converge and reach consensus from any starting condition, even without the initial, bootstrapped agreement that the Hare gives us. However, we would lose three things without the Hare

1. Time: self-healing takes time.
1. Efficiency: the self-healing protocol is less efficient (measured in the number of messages passed) than the Hare.
1. Robustness: while self-healing requires > 2/3 honest majority, the Hare requires only a simple honest majority, and depending on the size of the committee it can theoretically tolerate up to 2/3 malicious participants.

### Compared to other BFT protocols

- The difference between Hare and [HotStuff](https://arxiv.org/pdf/1803.05069.pdf) is that HotStuff requires a known set of participants (i.e., it does not support player replaceability), cannot be made permissionless, doesn't scale to thousands of participants, and does not easily convert to set agreement (i.e., agreement on a set of valus rather than on a single bit value). The requirement to have a known set of participants opens a DoS attack vector on known future participants.

