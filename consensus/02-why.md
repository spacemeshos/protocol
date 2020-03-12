# Why we chose the Tortoise and the Hare

As described in [Overview](01-overview.md), consensus in Spacemesh consists of two independent but complementary protocols, called the Tortoise and the Hare. This document provides more context on these protocols, explaining how they fit into the broader landscape of BFT consensus protocols, and why they're so well-suited to the needs of Spacemesh.

## Tortoise

- The difference between the Tortoise and a BFT-style protocol (including the ones discussed below) is that the Tortoise is capable of a more robust form of consensus: even if assumptions fail for a while, things that have been in consensus for a while remain in consensus. Contrast this wih a BFT-style protocol, where you cannot rely on history in such a scenario, since it can be reversed.
- Furthermore, the Tortoise has self-healing capabilities: once conditions return to normal (i.e., assumptions are met once again), the system can fully recover. In a BFT-style protocol, if consensus fails once, it fails for good.

The Tortoise is quite a robust consensus mechanism in its own right, and to be clear, Spacemesh could function with just the Tortoise. However, to add efficiency, we need the Hare as well.

## Hare

The Hare is a BFT-compatible algorithm and is based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf), with the difference that we want to achieve consensus on a set of values rather than a single bit value.

- The difference between Hare and [HotStuff](https://arxiv.org/pdf/1803.05069.pdf) is that HotStuff requires a known set of participants (i.e., it does not support player replaceability), cannot be made permissionless, doesn't scale to thousands of participants, and does not easily convert to set agreement (i.e., agreement on a set of valus rather than on a single bit value). The requirement to have a known set of participants opens a DoS attack vector on known future participants.

