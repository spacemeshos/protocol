# The Hare Protocol

## Description
The hare protocol is used to achieve consensus on a layer i.e. a set of blocks. The general problem is the [Byzantine Agreement Problem](https://en.wikipedia.org/wiki/Quantum_Byzantine_agreement). Our protocol is based on the [ADDNR18 paper](https://eprint.iacr.org/2018/1028.pdf) and differs mainly on the fact that we want to achieve consensus on a set of values rather than a single value.

It is known in advance when an agreement process should start for each layer. On the other hand, the time it takes to achieve agreement can vary, depending on the number of faulty/malicious participants in the consensus. Hence,  multiple consensus instances can be running at the same time.


## Definitions
`N` - the number of active participants. <br />
`f` - the number of active dishonest participants. <br />
`Pi` - A participant in the consensus. <br />
`Si` - The current set blocks observed by Pi. <br />
**Byzantine Agreement on Sets** - parties {Pi} are said to achieve byzantine agreement on sets {Si} if three conditions are satisfied:
1. `Consistency:` Every honest party outputs the same set S’.
2. `Validity 1` (“all honest witnessed”): If for every honest party Pi v is in Si then v is in S'
3. `Validity 2` (“no honest witness”): If for no honest party Pi v in Si then v not in S'

**Roles** - During the BA, at each round, a party can be assigned with one of the following roles:
* `Passive` - Listens and follows the protocol. This is the default role of a participant meaning every participant is at least passive.
* `Active` - Also acts as a speaker and share his knowledge.
* `Leader` - a participant who can give a proposition to the network. There is only one leader.

**Virtual Participation** - virtual participation is an optimization made to ease a participant’s role. After a participant terminates, he sends a notify message declaring his commitment to his corresponding set. From here on out, all other participants can refer to his notify message as if it was sent in the current round.

**Certificate** - a proof to witnessing f+1 commits on a set S.

**Safe Value Proof** - a proof made by a participant to ensure that a set S satisfies validity 1 and validity 2 in respect to a specific round k. An SVP also includes a certificate with which we can ensure consistency.

## The Protocol
The protocol begins with a `pre-round`. This round is executed only once and its goal is to remove values which shouldn't be considered at all (according to Validity 2).
- At the begining of the round each active party sends his set of values
- At the end of the round, each value that hasn't received f+1 witnesses is removed

The protocol repeatedly iterates through 4 rounds until a consensus is reached. 
Each round longs a constant time. A `roles oracle` is used to generate roles in each round (more on that later on).

**Round 0**
- Active participants broadcast their current status to the network (S, k , ki)
- At the end of this round, a participant can form an SVP based on the statuses he collected during the round

**Round 1**
- The leader can now give his proposition set T with the corresponding SVP P to the network
- Each participant can validate the proposition sent by the leader and consider that set T on the next round

**Round 2**
- Active participants announce their will to commit to the proposed set T
- If a participant observes f+1 participants willing to commit to T, he commits to T and constructs the matching certificate (which is the proof that he witnessed f+1 commits to T) and sends a notify message stating that he commited to T
- If equivocation is detected by a party P, he doesn't commit to T

**Round 3**
- Upon receiving a valid notify message the party updates his internal state according to the attached set

Termination: if at any point of the protocol, a party P receives f+1 notify messages on the same set S, it commits to S and terminates.


![Message Validation](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/msg_validation.svg?sanitize=true)
![Round 1](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round1.svg?sanitize=true)
![Round 2](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round2.svg?sanitize=true)
![Round 3](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round3.svg?sanitize=true)
![Round 4](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round4.svg?sanitize=true)
