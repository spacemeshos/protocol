# Protocol deep dive

As described in [Overview](consensus_overview.md), consensus in Spacemesh consists of two independent but complementary protocols, called the Tortoise and the Hare. This document provides more details and context on these protocols, explaining how they fit into the broader landscape of BFT consensus protocols, and why they're well-suited to the needs of Spacemesh.

## Tortoise

The Tortoise consensus mechanism was custom-designed to suit the needs of Spacemesh. It's a traditional Byzantine agreement protocol that adds a self-healing feature. It's based on previous work done on [Meshcash](https://eprint.iacr.org/2017/300) by Bentov, Hubáček, Moran and Nadler.

The Tortoise protocol has three particularly important properties:

1. **Consensus:** it allows all nodes to achieve consensus (on the canonical set of blocks in each layer)
1. **Irreversibility:** older history is harder to change than more recent history
1. **Self-healing:** all nodes will eventually converge and achieve consensus even if they temporarily disagree

### How it works

The Tortoise counts votes, cast in more recent blocks, for and against previous blocks. These votes indicate whether those previous blocks should be considered [contextually valid](consensus_overview.md#block-validity-in-spacemesh), i.e., whether, from the perspective of the voter, the block was received on time. In most cases, the vast majority of honest nodes will agree on a given block's validity, and there will be an overwhelming number of votes cast for or against the block. In rare cases, such as in the case of a [balancing attack](https://arxiv.org/abs/1612.09426), a block may not clearly cross either threshold.

Votes are processed per layer, and target around 800 votes per layer (this is in order to meet the network's [statistical security parameter](https://en.wikipedia.org/wiki/Security_parameter) of `2^-40`; if layers are much smaller than this, they may be grouped together into larger _clusters_). If the votes are overwhelmingly positive or negative, e.g., net +600 in favor of a block ("This block is valid") or net -700 against a block ("This block did not arrive on time"), no additional work is needed to achieve consensus. If the net vote for a block is closer to zero, e.g., +100, then instead of voting based on what they observed, all honest nodes vote according to a coin flip. This is how the Tortoise achieves self-healing.

### Self-healing

Self-healing is the ability to converge, and to achieve consensus, even if nodes are in a state of initial disagreement, such as in case of a network partition or an attack. Self-healing takes time, and the exact amount of time it takes depends upon both protocol parameters (such as the properties of the beacon), as well as upon the proportion of honest nodes. If there are no attacking nodes for a long enough period of time (one cluster), then convergence should happen more or less instantly. Convergence is still possible even in the case of an ongoing partition or attack, but in this case it will take longer.

Note that, in order for self-healing to work, the Tortoise requires a greater than 2/3 honest majority. This is because the protocol works using votes, and there must be a large enough margin between honest and dishonest votes.

Self-healing works via a _separate byzantine agreement mechanism_ that does not depend on shared view of history.

### Weak coin beacon

The weak coin beacon uses a VRF-based protocol to generate a weak coin toss. In each round, each party calculates the output of a VRF based on their identity and the round number. Parties with a sufficiently low result publish their results and the lowest result wins. The value of the "coin toss" is one bit, e.g., the least significant bit, of this lowest result.

If the lowest result comes from an honest participant, it will be published on time, everyone will receive this result, and all honest parties will agree on it. The "weak" nature of the coin comes from the fact that, in some cases, a dishonest adversary may win and be able to bias the results by, e.g., choosing not to publish their result, publishing it late, etc.

In the case where all honest parties _do_ agree on the result of the coin toss, they will all adjust their next round of votes on the blocks in question _in the same direction_ (e.g., "heads" means "vote yes"), and convergence will be achieved in the following round. In the case where an adversary successfully attacks the result, convergence will be delayed one round. Note that the honest parties only need to win this coin toss game once in order to achieve convergence, and over time the likelihood of this occurring only grows.

Importantly, the beacon _does not rely upon any previous data_ and, thus, does not rely upon consensus having been previously achieved or maintained. This is what enables the self-healing property.

An astute reader will notice that, unlike, e.g., the Hare beacon, there is no random input to the beacon VRF, since the input is just a node's identity and the current round number. This means that an adversary can grind on this identity and could possibly generate many identities to give them an advantage in biasing the weak coin toss. This is an area of active research, but our current thinking is that we should allow grinding on VRF identities but make it expensive to do so.

### Compared to other protocols

Blockchains based on Nakamoto consensus, such as Bitcoin, also have this "self-healing" property: some node will always produce the next block, and the ability to produce a new block doesn't depend upon consensus having been achieved in a prior round. In this way, over time all nodes will always converge around the "longest chain." However, it's much more difficult to achieve this property _without_ Proof of Work, as in the case of Spacemesh. This is because, in most BFT-style protocols, if the network loses consensus even briefly, it's impossible to re-establish consensus. In protocols with committees that nominate and notarize a committee for the following round, for instance, without agreement on the current committee, there's no way for the network to reach agreement on the next valid committee.

In a PBFT-style system, such as those based on HotStuff BFT, losing consensus in this manner might mean that different participants have a different view on which transactions are confirmed. Self-healing ensures that this cannot happen in Spacemesh.

Another important difference between the Tortoise and a BFT-style protocol is that the Tortoise is capable of a more robust form of consensus: even if assumptions fail for a while, things that have been in consensus for a while remain in consensus. Contrast this with a BFT-style protocol, where you cannot rely on history in such a scenario, since it can be reversed.

## Hare

The Hare is a BFT algorithm based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf), with the important difference that we want to achieve consensus on a set of values rather than a single bit value.

### What does it give us?

Because of the self-healing mechanism (described above), Spacemesh would function without the Hare. This is because self-healing guarantees that Spacemesh nodes will eventually converge and reach consensus from any starting condition, even without the initial, bootstrapped agreement that the Hare gives us. However, we would lose two things without the Hare:

1. **Time:** self-healing takes time.
1. **Efficiency:** the self-healing protocol is less efficient (measured in terms of both computational and communication complexity) than the Hare. We have put considerable work into optimizing the Hare to make it as efficient as possible.

Depending on committee size the Hare can theoretically tolerate up to 1/2 malicious participants. This is because it takes advantage of strong synchrony assumptions. In practice, however, for efficiency purposes the Hare is parameterized to support up to approx. 800 participants with a 2/3 honest majority assumption. This is based on the aforementioned statistical security parameter.

### Hare beacon

Eligibility to participate in each [round of the Hare protocol](consensus_overview.md#rounds) is based on the output of a VRF, which includes randomness from the Hare beacon. This makes it impossible for an adversary to grind on an identity and produce identities that would bias the results of the Hare too much, since they could only forecast Hare eligibility for a small number of future rounds.

The Hare has the advantage that it can rely on the Tortoise for its beacon, which makes it much simpler than the Tortoise beacon. We can use any value that honest parties are guaranteed to have previously agreed upon _even if the Hare isn't working,_ such as the set of block IDs from a previous layer that's sufficiently far back.

In the case of a network partition or attack, the Hare protocol may cease to work. It may not achieve a sufficient vote quorum or, in the extreme case, nodes may not even agree on the beacon value. In this case, the Hare will keep trying to run at every new layer until the Tortoise's self-healing mechanism achieves convergence, at which point the Hare will begin to function again.

### Compared to HotStuff and Tendermint

- [HotStuff](https://arxiv.org/pdf/1803.05069.pdf) requires a known set of participants (i.e., it does not support player replaceability), cannot be made permissionless, doesn't scale to thousands of participants, and does not easily convert to set agreement (i.e., agreement on a set of values rather than on a single bit value). The requirement to have a known set of participants opens a DoS attack vector on known future participants.
- HotStuff and Tendermint make different assumptions about network synchrony: they both assume a _partially-synchronous network_ (i.e., there exists a "global stabilization time" after which all messages may be assumed to have been delivered, but you don't know exactly when this time is). The Hare, on the other hand, assumes a _fully-synchronous network,_ i.e., that all messages are delivered within a known period of time (which is specified as a protocol parameter, and affects layer time). As a result, HotStuff and Tendermint can make progress faster than Hare, but at the cost of having a lower corruption threshold: HotStuff and Tendermint both require a 2/3 honest majority, whereas the Hare requires only a simple honest majority.
- HotStuff and Tendermint achieve a form of consensus known as _state machine replication_ (SMR): this is a classical form of PBFT consensus where participants in the protocol agree upon an ordered sequence of transactions. The Hare protocol plays a different role in Spacemesh, and thus its goal is not to achieve SMR-style consensus. Rather, every time the Hare runs, its goal is to run for a finite number of rounds and achieve a "one-time agreement" on the set of valid blocks in the current layer, then terminate. Terminating an SMR-style consensus protocol is more difficult since it's unclear how to ensure that all participants have achieved precisely the same state—indeed, these protocols are designed to run through an indefinite number of rounds.

## Other Proof of Space-based protocols

### Chia

The Chia protocol aims to emulate Bitcoin and Nakamoto consensus more generally, but to replace Proof of Work with a novel mechanism called [Proof of Space](https://www.chia.net/assets/ChiaGreenPaper.pdf). In order to emulate the PoW lottery mechanism, Chia utilizes a VDF to create a "temporal lottery." Chia protocol and Spacemesh are only superficially related, in that both require commitment of space as a scarce resource. High-level differences include:

- In Spacemesh, many blocks are produced at each layer, giving rise to its signature mesh (i.e., DAG) structure. Chia is a more traditional blockchain.
- Chia requires stronger Proof of Space properties: a unique Proof of Space, and a VDF. By contrast, Spacemesh uses much weaker primitives: a [Proof of Spacetime](post.md) (that isn't required to have a unique proof) and a Proof of Sequential Work (a.k.a. [Proof of Elapsed Time](poet.md), also not necessarily unique).
- Unlike Chia, Spacemesh is race free. Crucially, the slots in which a miner is eligible to produce blocks in Spacemesh do not depend on the proof of resource use. They're based solely on a miner's ID (plus a random input from a beacon), as described above.
