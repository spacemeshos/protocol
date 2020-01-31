# Consensus in Spacemesh

## Overview

Consensus is an important part of every blockchain platform. In many ways, a decentralized consensus engine is what makes a blockchain a blockchain (rather than, say, a centralized database). The choice of consensus mechanism has ramifications that echo up and down the blockchain stack, from the basic architecture of the full node to aspects of user experience (UX) such as wallet design and time to finality. For this reason, the choice of consensus engine is one of the most important design decisions we can make when building a blockchain platform.

With respect to the rest of the blockchain client architecture, consensus can be thought of as a background process: the consensus engine waits for new information to arrive, in the form of new blocks, then kicks into action every so often and identifies a set of blocks that are deemed canonical given all of the information available to the node at that point in time.

The consensus mechanism is one of the primary differences between Spacemesh and other contemporary blockchain platforms. Unlike most other blockchains, Spacemesh employs two distinct consensus mechanisms, which work in concert to achieve final consensus on the canonical structure of the mesh. It is the way these mechanisms work that give Spacemesh its eponymous mesh-like data structure.

Note that it’s important not to conflate consensus with mining. Indeed, there is some conceptual overlap between the two so the distinction can be a bit blurry. Mining is _the process by which a node establishes eligibility to produce blocks, and subsequently produces blocks deemed valid by the network._ Secondarily, it is also the mechanism by which the network’s native token is distributed. Consensus, by contrast, is _the process by which _all nodes_ come to consensus on the canonical set of blocks and transactions that form the mesh._

There is a circular dependency between consensus and mining. Through mining, nodes create and submit proofs that they are eligible to propose blocks and participate in mining (in Spacemesh terminology, this is called an [ATX](../mining/05-atx.md)). These form one input to the consensus engine: only blocks produced by an eligible miner, in an eligible slot, are valid and should be deemed canonical.

The output of the consensus engine is also an input to the mining process. The consensus engine tells a node which blocks it should vote for in a newly proposed block: only blocks that are visible and valid as of the time when it creates the block.

You can read much more about the mining process [here](../mining/01-overview.md).


### Phases

Blockchain consensus and block production, in general (not in Spacemesh specifically), work in the following way:



1. Eligibility: There is a mechanism by which candidate block producers, i.e., miners, assert deterministically and verifiably to the network that they are eligible to produce blocks. This serves as the network’s Sybil resistance mechanism, which prevents the network from being overrun by a large number of candidate producers controlled by a single entity. To achieve strong Sybil resistance, there must be a cost associated both with establishing eligibility, and with maintaining ongoing eligibility.
2. Leader election: There is a mechanism by which the protocol selects one or more eligible candidate block producers to produce one or more blocks at a given block height.
3. Block production: Block producers elected by the protocol produce blocks at a given block height, according to the rules of the protocol, and distribute those blocks to the network.
4. Fork choice rule: There is a mechanism by which any node in the network, based only upon their view of the network at any given point in time, and all block data they’ve received up to that point, can construct for themselves a canonical view of the blockchain data structure that is shared by other nodes with the same or similar views of the network at the same time. In concrete terms, a node must be able to decide on its own which set of blocks represent the canonical “longest chain,” and which transactions are included in that chain, in which order.


### Proof of Work-based protocols

For point of comparison, and since proof of work is relatively well understood and relatively easy to reason about, following is an abridged explanation of the mechanisms that Proof of Work-based chains, such as Bitcoin and Ethereum, use to address each of these phases.



1. Eligibility: All nodes are eligible to produce blocks at any block height, subject to the rules of the protocol, including the current PoW difficulty level and their ability to produce valid blocks accepted by the rest of the network. The cost of establishing and maintaining eligibility is the cost of the electricity that must be spent to participate in PoW mining. (Miners that don’t meet this threshold and thus fail to ever produce a valid block are effectively “invisible” from the perspective of mining.)
2. Leader election: The first node to successfully produce an eligible block at a given block height is the implicit leader, but only gains the ability to produce that single block.
3. Block production: The leader may produce any block they wish as long as that block follows the rules of the protocol. For instance, it must refer to the previous block and must not contain any invalid transactions. In most cases, the leader will choose to produce a block that contains as many high-fee transactions as possible to maximize revenue, but note that even empty blocks are valid according to protocol rules.
4. Fork choice rule: The chain containing the most accumulated proof of work, measured as cumulative block difficulties up to the chain tip, is considered the current canonical chain. Note that this means finality in PoW is always probabilistic as there may always be another chain with greater total work than the current chain.


### Proof of Space-time

In contrast to PoW-based chains, Spacemesh employs a set of consensus mechanisms collectively referred to as [Proof of Space-time (PoST)](../mining/02-post.md). Here’s how each mechanism functions to achieve consensus, at a high level.



1. Eligibility: Just as in PoW, all nodes are eligible to produce blocks in a permissionless fashion. Unlike PoW, however, instead of establishing eligibility by participating in PoW mining and burning electricity, PoST miners instead establish and maintain eligibility by allocating hard drive space and using that space to generate proofs of spacetime on an ongoing basis. A candidate miner allocates free hard drive space, produces an eligibility proof, submits it to the network, then waits until the following epoch to begin producing blocks (and collecting block rewards). See [Mining](../mining/01-overview.md) for details on how this process works.
2. Leader election: The protocol uses a random beacon to sample a set of miners eligible to produce blocks at each block height (called a “layer” in Spacemesh terminology, as each layer contains many blocks). Note that, unlike in PoW, _multiple_ miners are eligible to produce blocks at each layer.
3. Block production: Each miner eligible to produce a block in a given layer may optionally do so (but is not punished if it chooses not to). As in Bitcoin, the miner may include any set of valid transactions they like in that block, and an empty block is valid according to the protocol. Unlike Bitcoin, and in order to ensure the broadest possible set of transactions included at each layer, miners are incentivized to include transactions that no other miner has included in their block.
4. Fork choice rule: Because there are multiple blocks in each layer, and there is no difficulty-based PoW mining, one cannot simply select the chain with the greatest accumulated work. As a result, the fork choice rule is one of the most complex and unique parts of the Spacemesh protocol, and it’s explained in more detail in the following section.


### The Tortoise and the Hare

Spacemesh employs two consensus mechanisms in parallel which together allow each node to determine the canonical set of blocks at each layer. The Tortoise and the Hare refer to the two independent, complementary consensus mechanisms that take as input a given node’s view of all of the blocks it has seen as of a given layer and output a set of blocks deemed canonical by the protocol at that layer, i.e., a canonical mesh. The Tortoise is a slow, vote-based mechanism that requires each node to count the votes for and against each block in more recent, valid blocks, eventually arriving at a consistent view of consensus. However, it cannot run on the most recent layers since there are not yet any newer blocks voting on these blocks. This is where the Hare comes in. It is a permissionless Byzantine agreement protocol that runs on a random subset of nodes at every layer, producing a tentative, fast consensus on the set of canonical blocks that form that layer, thus bootstrapping the Tortoise mechanism by allowing each node to quickly determine which recent blocks it should vote for.


### Mesh Structure

There are many canonical blocks in every layer, and many canonical transactions in every block. Each block includes a view that lists other blocks considered canonical (i.e, visible and valid) by the node that produced the block at the time the block was produced. As such, the overall structure of the mesh is a [directed acyclic graph (DAG)](https://en.wikipedia.org/wiki/Directed_acyclic_graph). There is furthermore a strict topological ordering to the blocks in a given layer, and to the transactions in each block. Thus, conceptually, the entire mesh structure could be unwound into a chain. This strict ordering is necessary to determine transaction ordering, i.e., to turn the _mesh_ into a ledger. Blocks are ordered first by layer, then by block ID within a given layer. Transactions are ordered by block (i.e., the block in which they first appear), then by index within a block. If the same transaction appears in multiple blocks, it is counted only the first time it appears.


### Block Validity

There are two sets of validity rules for a proposed block: _syntactic validity_ and _contextual validity_. A block is syntactically valid if it follows the rules of the protocol, is correctly constructed, and contains no invalid transactions. A syntactically valid block is also contextually valid if all of the prior blocks that it refers to are valid, available blocks. See Blocks [LINK forthcoming] for more information.


## Tortoise


### Overview

The Tortoise protocol is the mechanism by which the Spacemesh network achieves final, eventual consensus on the set of blocks and transactions that form the canonical mesh. It’s a relatively slow, vote-based mechanism that tallies votes for and against each valid block in a majority-rules fashion. For this reason, it cannot be run on a recently-produced layer of blocks, since not enough votes have yet been collected that refer to the new layer. For this reason, the output of the Hare consensus mechanism is used to bootstrap the Tortoise. See the Hare section, below, for information on how tentative consensus is achieved on recent layers.


### Voting

Each time a miner produces a new block [LINK forthcoming], they include in that block a “view” field that lists one or more valid previous blocks. Each time a valid new block links to a given older block, the vote tally for that older block increases by one. Any block generated _after_ a given, older block that _doesn’t_ vote for that block is _counted as a vote against the block._ Votes are weighted proportional to the amount of spacetime resources it represents: e.g., votes cast by a miner that has committed 200gb will count twice as much as votes cast by a miner that has committed 100gb to the protocol.


### Tallying votes

The Tortoise protocol consists of a recursive algorithm that tallies all votes for and against each valid block, in a majority-wins fashion: any block with a net tally greater than zero becomes part of the canonical mesh. This tally algorithm may be run multiple times for a given layer, until there is a clear majority voting in favor of a particular view of the blocks in this layer, i.e., a majority so overwhelmingly large that it cannot be reversed. At this point, the layer is finalized and set as the “best canonical layer.” This process repeats itself every time new votes are received.


### Self-healing

The Tortoise protocol as described is a fast and secure consensus protocol. However, it is fragile in the sense that its security depends upon assumptions about the protocol holding all the way from genesis. Over a long enough period of time, the likelihood of some low probability event occurring at some point increases, and such an event might, in theory, cause the protocol to fail at a given layer. In order to recover from such a failure, Spacemesh includes a feature known as self-healing.

Self-healing works by having all honest nodes agree, at every layer, on a random coin toss. When the margin of votes cast for and against a given block is very narrow, honest nodes will rely on the outcome of the coin toss to decide whether or not to vote for the block in question. This allows all nodes to reconverge on the same consensus over time.


### Details

The Tortoise protocol is complex and the full details of the protocol are beyond the scope of this document. For more details on the protocol, including formal security proofs, see [the full Spacemesh protocol paper](https://spacemesh.io/spacemesh-protocol-v1-0/). 


## Hare


### Overview

In contrast to the Tortoise protocol, the Hare protocol can achieve consensus rapidly on the network’s shared view of the current data set, i.e., on which recently-proposed blocks should be considered valid and become candidates for finalization by the Tortoise. It does not rely on votes included in blocks, as the Tortoise does. In this way, the Hare protocol is used to bootstrap the consensus of the slower, eventual Tortoise mechanism by allowing nodes to rapidly decide which existing blocks they should vote for as they produce new blocks. It’s a BFT-compatible algorithm that involves participation by a randomly-selected subset of the current set of valid block producers [EXPAND: selected how? The role distribution oracle], and achieves consensus in four rounds.

Each node internally runs the Hare protocol every time the end of a round is reached. It passes in all of the blocks it received in time, draws a role from the role distribution oracle, and then participates in voting by gossiping [LINK forthcoming] votes. At the end of four rounds of voting, the protocol outputs a set of blocks that should tentatively (i.e., until the Tortoise confirms them) be considered canonical for the layer in question.

As the node proposes blocks in the future, it factors in this information by including votes for blocks confirmed by the Hare, thus bootstrapping Tortoise consensus with the output of the Hare.


### Rounds

The goal of the Hare protocol is to achieve consensus on a set of blocks to be included in a given layer. The protocol is based on [ADDNR18](https://eprint.iacr.org/2018/1028.pdf) with the difference that we want to achieve consensus on a set of values rather than a single bit value. The protocol takes place over four rounds, which are preceded by a pre-round, where each participant shares their current view of valid blocks. After this, a leader, with the power to make a proposal, is randomly elected, and the following four rounds occur:



1. Status round: each participant broadcasts a status message reporting its current accepted set.
2. Proposal round: the leader broadcasts a proposal to the group. Each participant that receives and accepts this proposal broadcasts a message to this effect.
3. Commit round: each participant reviews the proposal and signals its willingness to commit to it to the group. By the end of this round, each participant that received a valid proposal from the leader (with no conflicting proposal) _and_ a sufficient number of commit messages from other participants creates a “commit certificate” including all of this information.
4. Notify round: each participant holding a commit certificate broadcasts it to the group. If a sufficient number of commit certificates are received from other participants, the protocol terminates and each participant knows the canonical set. (If not, the protocol returns to round 1 and iterates.)


### Validity rules

Consensus can be said to be achieved when the following three conditions are satisfied:



1. When the consensus mechanism terminates, every honest participant outputs the same set of canonical blocks.
2. If every honest party observed a given, valid block, then that block is included in the output set. Put another way, the output set contains the intersection of the valid blocks reported by all honest parties.
3. If no honest party observed a given block (or every honest party deemed it invalid), then that block is not included in the output set. Put another way, every block that makes it into the output set must have had at least one honest witness.

The intuition here may not be immediately obvious, but if you think about it, you’ll see why these validity rules are sufficient for ensuring consensus. Rule #2 states that the output set contains the intersection of all of the valid blocks observed and submitted by all of the honest miners. This does not in and of itself rule out the possibility that the output set also contains some invalid blocks. However, rule #3 ensures that every block in the output set was observed by at least one honest miner. Together, these two validity rules ensure that _the set of blocks output by the Hare consensus mechanism is exactly equal to the set of valid blocks observed by all honest miners,_ no more and no less.


### Details

For more details on the Hare protocol, see [The Hare Protocol](https://github.com/spacemeshos/protocol/wiki/The-Hare-Protocol). See also the [Hare Protocol Implementation Notes](https://github.com/spacemeshos/go-spacemesh/wiki/Hare-Protocol-Implementation) for information on how the protocol is implemented in go-spacemesh. Finally, for more formal proofs of these assertions, see [the full Spacemesh protocol paper](https://spacemesh.io/spacemesh-protocol-v1-0/).
