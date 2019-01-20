# The Hare Protocol

## Description
The hare protocol is used to achieve consensus on a layer i.e. a set of blocks. The general problem is the [Byzantine agreement problem](https://en.wikipedia.org/wiki/Quantum_Byzantine_agreement). Our problem differs mainly on the fact that we want to achieve consensus on a set of values rather than a single value.

It is known in advance when an agreement process should start for each layer. On the other hand, the time it takes to achieve agreement can vary, depending on the number of faulty/malicious participants in the consensus. Hence,  multiple consensus instances can be run at the same time.


## Definitions
**N** - the number of active participants. <br />
**f** - the number of active dishonest participants. <br />
**Pi** - A participant in the consensus. <br />
**Si** - The current set blocks observed by Pi. <br />
**Byzantine Agreement on Sets** - parties {Pi} are said to achieve byzantine agreement on sets {Si} if three conditions are satisfied:
1. **Consistency:** Every honest party outputs the same set S’.
2. **Validity 1** (“all honest witnessed”): If for every honest party Pi v is in Si then v is in S'
3. **Validity 2** (“no honest witness”): If for no honest party Pi v in Si then v not in S'

**Roles** - During the BA, at each round, a party can be assigned with one of the following roles:
* Passive - Listens and follows the protocol. This is the default role of a participant meaning every participant is at least passive.
* Active - Also acts as a speaker and share his knowledge.
* Leader - a participant who can give a proposition to the network. There is only one leader.

**Virtual Participation** - virtual participation is an optimization made to ease a participant’s role. After a participant terminates, he sends a notify message declaring his commitment to his corresponding set. From here on out, all other participants can refer to his notify message as if it was sent in the current round.

**Certificate** - a proof to witnessing f+1 commits on a set S.

**Safe Value Proof** - a proof made by a participant to ensure that a set S satisfies validity 1 and validity 2 in respect to a specific round k. An SVP also includes a certificate with which we can ensure consistency.

## The Protocol
The protocol repeatedly iterates through four rounds until a consensus is reached. Each round longs a constant time. A roles oracle is used to generate roles in each round (more on that later on).

**Round 0**
* Active participants broadcast their current status to the network.
* At the end of this round, a participant can form an SVP based on the statuses it collected during the round.

**Round 1**
* The leader can now give his proposition set S with the corresponding SVP P to the network.
* Each participant can validate the proposition sent by the leader and set his observed set accordingly.

**Round 2**
* Active participants announce their will to commit to their observed set S.
* If a participant observes f+1 participants willing to commit to S, he commits to S and constructs the certificate (which is the proof that he actually witnessed f+1 commits to S).

**Round 3**
* Active participants who have committed sends a notify message stating their choice is final (hence becoming a virtual participant).
* Each participant that committed in the previous round can now terminate.



![Message Validation](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/msg_validation.svg?sanitize=true)
![Round 1](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round1.svg?sanitize=true)
![Round 2](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round2.svg?sanitize=true)
![Round 3](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round3.svg?sanitize=true)
![Round 4](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round4.svg?sanitize=true)
