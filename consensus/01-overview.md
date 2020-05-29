# Consensus in Spacemesh

## Overview

Consensus is an essential part of every blockchain platform. In many ways, a decentralized consensus engine is what makes a blockchain a blockchain (rather than, say, a centralized database). The choice of consensus mechanism has ramifications that echo up and down the blockchain stack, from the basic architecture of the full node to aspects of user experience (UX) such as wallet design and time to finality. For this reason, the choice of consensus engine is one of the most important design decisions we can make when building a blockchain platform.

With respect to the rest of the blockchain client architecture, consensus can be thought of as a background process: the consensus engine waits for new information to arrive, in the form of new blocks, then kicks into action every so often and identifies a set of blocks that are deemed canonical given all of the information available to the node at that point in time.

The consensus mechanism is one of the primary differences between Spacemesh and other contemporary blockchain platforms. Unlike most other blockchains, Spacemesh employs two distinct consensus mechanisms, which work in concert to achieve final consensus on the canonical structure of the mesh. It is the way these mechanisms work that give Spacemesh its eponymous mesh-like data structure.

Note that it’s important not to conflate consensus with mining. Indeed, there is some conceptual overlap between the two so the distinction can be a bit blurry. Mining is _the process by which a node establishes eligibility to produce blocks, and subsequently produces blocks deemed valid by the network._ (Secondarily, it is also the mechanism by which the network’s native token is distributed.) Consensus, by contrast, is _the process by which_ all nodes _come to consensus on the canonical set of blocks and transactions that form the ledger._

There is a circular dependency between consensus and mining. Through mining, nodes create and submit proofs that they are eligible to propose blocks and participate in mining (in Spacemesh terminology, this is called an [Activation Transaction (ATX)](../mining/05-atx.md)). These are an input to the consensus engine: only blocks produced by an eligible miner, in an eligible slot, are syntactically valid and may be deemed canonical.

The output of the consensus engine is also an input to the mining process. The consensus engine tells a node which blocks it should vote for in a newly proposed block: i.e., only those blocks that are visible and valid as of the time when it creates the block.

You can read much more about the mining process [here](../mining/01-overview.md).

<a name="validity"></a>
### Block Validity in Spacemesh

There are two sets of validity rules for a proposed block: _syntactic validity_ and _contextual validity_. A block is syntactically valid if it follows the rules of the protocol, is correctly constructed, and contains no invalid transactions. A syntactically valid block is also said to be contextually valid once the Tortoise declares it as such. This happens while the block receives enough votes in favor (in fact, when the difference between votes for and against passes the _irreversibility threshold)._

Why is there a need for two types of validity? Why not, for instance, allow all syntactically valid blocks to be contextually valid? The answer is that, due to the [race free nature](../README.md#why-race-free) of the Spacemesh protocol, a miner may choose to publish a block later than they should (e.g., they were eligible to produce a block in layer 200, but they decide to actually publish the block in, say, layer 205). Since the published block is _syntactically valid,_ if the block were valid in the eyes of the protocol and of other nodes, it would cause a change to history and to the global network state, as new miners that join the network, for example, don't know when the block was actually published. We need to make sure that such blocks do not become part of the canonical ledger history: hence, even a _syntactically valid_ block may be deemed _contextually invalid_ if it wasn't published at the right time. Syntactically valid blocks form the mesh while contextually valid blocks form the ledger--or, put another way, the contextual validity of the blocks in the mesh (or lack thereof) is used to construct the ledger.

The purpose of the two consensus protocols, described below, is to determine which syntactically valid blocks are _also contextually valid._ For this reason, the rest of this document is unconcerned with syntactic validity. For the purposes of the rest of this document, "valid" should be taken to mean "contextually valid." By the same token, "blocks" refers to syntactically valid blocks.


### Phases

Blockchain consensus and block production, in general (not in Spacemesh specifically), work in the following way:

1. Eligibility: There is a mechanism by which candidate block producers, i.e., miners, assert verifiably to the network that they are eligible to produce blocks. This serves as the network’s Sybil resistance mechanism, which prevents the network from being overrun by a large number of candidate producers controlled by a single entity. To achieve strong Sybil resistance, there must be a cost associated both with establishing eligibility, and with maintaining ongoing eligibility.
2. Leader election: There is a mechanism by which the protocol selects one or more eligible candidate block producers to produce one or more blocks at a given block height.
3. Block production: Block producers elected by the protocol produce blocks at a given block height, according to the rules of the protocol, and distribute those blocks to the network.
4. Fork choice rule: There is a mechanism by which any node in the network, based only upon their view of the network at any given point in time, and all block data they’ve received up to that point, can construct for themselves a canonical view of the blockchain data structure that is shared by other nodes with the same or similar views of the network at the same time. In concrete terms, a node must be able to decide on its own which set of blocks represent the canonical “longest chain,” and which transactions are included in that chain, in which order.


### Proof-of-Work-based protocols

For point of comparison, and since proof of work is relatively well understood and relatively easy to reason about, following is an abridged explanation of the mechanisms that Proof-of-Work-based chains, such as Bitcoin and Ethereum, use to address each of these phases.

1. Eligibility: All nodes are eligible to produce blocks at any block height, subject to the rules of the protocol, including the current PoW difficulty level and their ability to produce valid blocks accepted by the rest of the network. The cost of establishing and maintaining eligibility is the cost of the electricity that must be spent to participate in PoW mining. (Miners that don’t meet this threshold and thus fail to ever produce a valid block are effectively “invisible” from the perspective of mining.)
2. Leader election: Leadership is implicitly associated with being the first node to successfully produce, and distribute, a valid block at a given block height. (Note that, in proof-of-work-based protocols, technically speaking block production happens _before_ leader election, since we only know if a block producer is eligible after they've successfully produced a block.)
3. Block production: Each leader candidate may produce any block they wish as long as that block follows the rules of the protocol. For instance, it must refer to the previous block and must not contain any invalid transactions. In most cases, block producers will choose to produce a block that contains as many high-fee transactions as possible to maximize revenue, but note that even empty blocks are valid according to protocol rules.
4. Fork choice rule: The chain containing the most accumulated proof of work, measured as cumulative block difficulties up to the chain tip, is considered the current canonical chain. Note that this means finality in PoW is always probabilistic as there may always be another chain with greater total work than the current chain.


### Proof of Space-time

In contrast to PoW-based chains, Spacemesh employs a set of consensus mechanisms collectively referred to as [Proof of Space-time (PoST)](../mining/02-post.md). Here’s how each mechanism functions to achieve consensus, at a high level.

1. Eligibility: Just as in PoW, all nodes are eligible to produce blocks in a permissionless fashion. In Spacemesh, however, instead of allocating CPU power, and by extension electricity, miners allocate hard disk space for a specified period of time. This allocation prevents Sybil attacks as explained above. To gain eligibility to produce blocks, a miner needs to send a _proof of space-time_ that asserts that the space-time resource was indeed allocated to the network. Then, once a new [epoch](../README.md#spacemesh-basics) starts they become active and are eligible to produce several blocks in that epoch. See [Mining](../mining/01-overview.md) for details on how this process works.
2. Leader election: The protocol uses a [Verifiable Random Function (VRF)](https://en.wikipedia.org/wiki/Verifiable_random_function) to sample a set of miners eligible to produce blocks at each block height (the set of blocks at a given height is called a [layer](../README.md#spacemesh-basics) in Spacemesh terminology). Note that, unlike in PoW, _multiple_ miners are eligible to produce blocks at each layer.
3. Block production: Each miner eligible to produce a block in a given layer may optionally do so (but is not punished if it chooses not to). As in Bitcoin, the miner may include any set of valid transactions they like in that block, and an empty block is syntactically valid according to the protocol.
4. Fork choice rule: There are no forks in Spacemesh by design. The following section explains how and why this is the case.


### The Tortoise and the Hare

Spacemesh employs two consensus mechanisms in parallel which together allow each node to determine the canonical set of blocks at each layer. The Tortoise and the Hare refer to the two independent, complementary consensus mechanisms that take as input a given node’s view of all of the blocks it has seen as of a given layer and output a set of blocks deemed canonical by the protocol at that layer, i.e., a canonical ledger. The Tortoise is a slow, vote-based mechanism that requires each node to count the votes for and against each previous block, tallying these votes to eventually achieve consensus. However, it cannot run on the most recent layer since there are not yet any newer blocks voting on these blocks. This is where the Hare comes in.

Why do we need two consensus protocols? At the most basic level, we just want to make sure that each miner votes for blocks that arrived on time, so late blocks cannot change history. The solution is that each block votes for all the blocks that its producer saw on time.

The problem with this approach is that votes on adversarial blocks can become "balanced," if an adversary makes sure that half the network receives a block on time and half doesn't. For this reason, it's important that all honest miners vote the same way: with a big enough honest majority, this balancing attack won't work.

In order to facilitate this, we run a permissionless Byzantine agreement protocol, the Hare, which outputs the set of blocks to be voted on. It bootstraps the Tortoise mechanism by allowing each node to quickly determine which recent blocks it should vote for. This helps the Tortoise quickly converge on the final, canonical ledger. The Hare runs on a random subset of nodes at every layer, producing a tentative, fast consensus on the set of canonical blocks that form that layer.


### Mesh Structure

There are many blocks in every layer, and many transactions in every block. Each block includes a view that lists other blocks that are visible and valid according to the node that produced the block at the time the block was produced. As such, the overall structure of the mesh is a [directed acyclic graph (DAG)](https://en.wikipedia.org/wiki/Directed_acyclic_graph). There is furthermore a strict topological ordering to the blocks in a given layer, and to the transactions in each block. The valid blocks in the mesh (which are used to form the ledger) thus could form a chain. This strict ordering is necessary to determine transaction ordering, i.e., to turn the _mesh_ into a ledger. Blocks are ordered first by layer, then by block ID within a given layer. Transactions are ordered by block (i.e., the block in which they first appear), then by index within a block. If the same transaction appears in multiple blocks, it is counted only the first time it appears.

See [Transaction ordering](../transactions/01-overview.md#ordering) for more information.

## Tortoise


### Overview

The Tortoise protocol is the mechanism by which the Spacemesh network achieves final, eventual consensus on the set of blocks and transactions that form the canonical ledger. It’s a relatively slow, vote-based mechanism that tallies votes for and against each block in majority-rule fashion. For this reason, it cannot be run on a recently-produced layer of blocks, since not enough votes have yet been collected that refer to the new layer. Therefore the output of the Hare consensus mechanism is used to bootstrap the Tortoise. See the Hare section, below, for information on how tentative consensus is achieved on recent layers.


### Voting

Each time a miner produces a new block, they include in that block a “votes” field that lists one or more previous blocks that the miner also considers valid at the time the block is produced. Each time a new block links to a given older block, the vote tally for that older block increases by one. Any block generated _after_ a given, older block that _doesn’t_ vote for that block is _counted as a vote against the block._ A vote is weighted proportional to the amount of space-time resources it represents: e.g., votes cast by a miner that has committed 200gb will count twice as much as votes cast by a miner that has committed 100gb to the protocol.


### Tallying votes

The Tortoise protocol consists of a recursive algorithm that tallies all votes for and against each block, in majority-rule fashion: any block with a net tally greater than the irreversibility threshold becomes part of the canonical ledger. This tally algorithm may be run multiple times for a given layer until there is a clear majority voting in favor of a particular set of blocks in this layer, i.e., a majority so overwhelmingly large that it cannot be reversed because honest miners keep increasing it. At this point, the layer is finalized and set as the “best canonical layer.” This process repeats itself every time new votes are received.


### Self-healing

While the Tortoise protocol is very secure, it is fragile in the sense that its security depends upon assumptions about the protocol holding all the way from genesis. Over a long enough period of time, the likelihood of some event occurring that causes the protocol to fail approaches one. In order to recover from such a failure, Spacemesh includes a feature known as self-healing.

Self-healing works by having all honest nodes agree, at every layer, on a random coin toss. When the margin of votes cast for and against a given block is very narrow, honest nodes will rely on the outcome of the coin toss to decide whether or not to vote for the block in question. This allows all nodes to reconverge on the same consensus over time.


### Details

The Tortoise protocol is complex and the full details of the protocol are beyond the scope of this document. For more details on the protocol, including formal security proofs, see [the full Spacemesh protocol paper](https://spacemesh.io/spacemesh-protocol-v1-0/).


## Hare


### Overview

The Hare protocol is run once per layer. Its purpose is to allow each node to quickly determine the set of blocks in the layer to be voted for by _all honest miners._

In contrast to the Tortoise protocol, the Hare protocol can achieve consensus rapidly on the network’s shared view of the current data set, i.e., on which recently-proposed blocks should be considered valid and become candidates for finalization by the Tortoise. It does not rely on votes included in blocks, as the Tortoise does. In this way, the Hare protocol is used to bootstrap the consensus of the slower, eventual Tortoise mechanism by allowing nodes to rapidly decide which existing blocks they should vote for as they produce new blocks. It’s a BFT-compatible algorithm that involves participation by a randomly-selected subset of the current set of eligible block producers, and achieves consensus in four rounds.

Each node internally runs the Hare protocol at the end of each layer. It passes in all of the blocks it received in time, randomly draws a role, and then participates by following the protocol. At the end of four rounds, the Hare outputs a set of blocks that should tentatively (i.e., until the Tortoise confirms them) be considered canonical for the layer in question.

When the node proposes a block in the future, it uses the output of the Hare, and includes votes (in the newly-proposed block) for blocks that the Hare confirmed, thus bootstrapping the Tortoise with the output of the Hare.

### Roles

At the beginning of each round (rounds 0-4, as described in the following section), a miner draws a role that tells it whether it's _active_ (i.e., participating) or _passive_ (i.e., just observing) in this round. In order to draw a role, each miner uses a [Verifiable Random Function (VRF)](https://en.wikipedia.org/wiki/Verifiable_random_function) and if the output passes some threshold (which depends on the number of total active miners) then it is active; otherwise the miner is passive for that round.

Miners change roles from round to round in order to prevent denial-of-service attacks against the Hare. If a miner kept the same role thoughout all of the rounds, then, after revealing their role in an earlier round, they could be targeted by an attack in subsequent rounds.

In round two (proposal round), a leader (with the ability to broadcast a proposal) is chosen. This also happens using a VRF, but as opposed to the assignment of active/passive roles as described above, even if a miner passes the required threshold before this round and subsequently sends a proposal, that miner is still not certain that it's the actual leader (since another miner may have drawn a lower number). There is, actually, just one leader, the one with the smallest VRF output, but this isn't known with certainty until the _end_ of the round.

### Rounds

The goal of the Hare protocol is to achieve consensus among all honest miners about which blocks to vote for (i.e., the set of valid blocks) in each layer. The protocol is based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf) with the difference that we want to achieve consensus on a set of values rather than a single bit value. The protocol takes place over four rounds, which are preceded by a pre-round:

0. Pre-round: each active participant shares their current view of blocks that are valid. At the end of the round, each then factors in the views shared by other participants and updates their view by removing blocks that didn't receive enough support from other participants.
1. Status round: each active participant broadcasts a status message reporting its updated view.
2. Proposal round: each active participant broadcasts a proposal to the group, based on the results of the previous round. One of these participants will be randomly chosen the leader.
3. Commit round: each active participant independently determines who was elected leader, reviews the proposal from this leader, and signals its willingness to commit to it to the group. By the end of this round, each participant that received a valid proposal from the leader (with no conflicting proposal broadcast by the leader) _and_ a sufficient number of commit messages from other participants creates a “commit certificate” including all of this information.
4. Notify round: each active participant holding a commit certificate broadcasts it to the group. If a sufficient number of commit certificates are received from other participants, the protocol terminates and each participant knows the canonical set. (If not, the protocol returns to the Status round and iterates.)


### Validity rules

Consensus can be said to be achieved when the following three conditions are satisfied:

1. When the consensus mechanism terminates, every honest participant outputs the same set of canonical blocks.
2. If every honest party observed a given, valid block, then that block is included in the output set. Put another way, the output set contains the intersection of the valid blocks reported by all honest parties.
3. If no honest party observed a given block (or every honest party deemed it invalid), then that block is not included in the output set. Put another way, every block that makes it into the output set must have had at least one honest witness.

Hopefully the intuition here is clear: Blocks submitted by _all_ honest miners definitely make it into the output set. Blocks submitted by _no_ honest miners do not.

What about blocks submitted by _some_ but not _all_ honest miners? The validity rules say nothing about this case, which is to say that they may or may not be included in the output set (depending how many honest miners saw it on time, and thus submitted it, and also upon the elected leader). This is by design, since the goal of the protocol is precisely to agree on the validity of such blocks. Since, according to validity rule #1, all honest participants agree upon the output set, based on the output set of the Hare they will _all_ vote for or against such a block in subsequent iterations of the Tortoise (regardless of whether or not they submitted the block as part of the Hare).


### Details

The Hare protocol is somewhat more complicated than can be explained here. For more details, see [The Hare Protocol](https://github.com/spacemeshos/protocol/wiki/The-Hare-Protocol). See also the [Hare Protocol Implementation Notes](https://github.com/spacemeshos/go-spacemesh/wiki/Hare-Protocol-Implementation) for information on how the protocol is implemented in go-spacemesh. Finally, for more formal proofs of these assertions, see [the full Spacemesh protocol paper](https://spacemesh.io/spacemesh-protocol-v1-0/).

Read on for more information on [Spacemesh consensus in the context of other consensus mechanisms](02-deepdive.md).
