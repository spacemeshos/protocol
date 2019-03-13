# The Hare Protocol

## Description
The Hare protocol is a consensus protocol that achieves consensus on a set of values for a specific instance id. In our case, we want to achieve consensus on a set of block ids (the set of values) in a specific layer (the instance id).

The general problem is the [Byzantine Agreement Problem](https://en.wikipedia.org/wiki/Quantum_Byzantine_agreement). Our protocol is based on the [ADDNR18 paper](https://eprint.iacr.org/2018/1028.pdf) and differs mainly on the fact that we want to achieve consensus on a set of values rather than a single value.

It is known in advance when an agreement process should start for each layer. On the other hand, the time it takes to achieve agreement can vary, depending on the number of faulty/malicious participants in the consensus. Hence, multiple consensus instances may be running concurrently.  

## Definitions
`N` - The number of active participants

`f` - The number of active dishonest participants

`Pi` - A participant in the consensus

`Si` - The current set of values observed by Pi

`ISi` - The initial set of Pi

#### Byzantine Agreement on Sets

Parties {Pi} are said to achieve byzantine agreement on sets {Si} if three conditions are satisfied:
1. `Consistency`: Every honest party outputs the same set S'
2. `Validity 1` (“all honest witnessed”): If for every honest party Pi value v is in ISi then v is in S'
3. `Validity 2` (“no honest witness”): If for no honest party Pi value v in ISi then v is not in S'. In other words, only values that are in ISi for f+1 parties should be in S'.

#### Roles
At each round while the protocol is running, a party can be assigned with one of the following 3 roles:
* `Passive` - Listens and follows the protocol. This is the default role of a participant so every protocol participant is at least passive
* `Active` - Also acts as a speaker and shares his knowledge
* `Leader` - A participant who can give a proposition to the network. There is only one leader

#### Certificate
A proof to witnessing f+1 commits on a set S.

#### Safe Value Proof (SVP)
A proof made by a participant to ensure that a set S satisfies `validity 1` in respect to a specific round k. An SVP also includes a certificate with which we can ensure consistency.

---

## The Protocol

**Pre-Round**
The protocol begins with one `pre-round`. This round is executed only once and its goal is to remove values which shouldn't be considered at all according to `Validity 2`
- At the beginning of the pre-round each active party sends his set of values
- At the end of the pre-round, each value that hasn't received f+1 witnesses is removed

The protocol repeatedly iterates through up to 4 rounds until a consensus is reached.

Each round longs a constant time. A `Roles Oracle` is used to generate roles in each round.

**Round 0**
- Active participants broadcast their current status to the network (S, k , ki)
- At the end of this round, a participant can form an SVP based on the statuses he collected during the round

**Round 1**
- The leader can now give his proposition set T with the corresponding SVP P to the network
- Each participant can validate the proposition sent by the leader and consider that set T on the next round

**Round 2**
- Active participants announce their will to commit to the proposed set T
- If a participant observes f+1 participants willing to commit to T, he commits to T and constructs the matching certificate (which is the proof that he witnessed f+1 commits to T) and sends a notify message stating that he committed to T
- If equivocation is detected by a party P, it doesn't commit to T. Equivocation means that the leader sent two or more contradicting proposals to the network

**Round 3**
- Upon receiving a valid notify message the party updates his internal state according to the attached set

**Termination**
If at any point of the protocol, a party P receives f+1 notify messages on the same set S, it commits to S and terminates. Note that after an honest party has terminated, the other honest parties are ensured to terminate up to the following round, thanks to the gossip properties.

![Message Validation](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/msg_validation.svg?sanitize=true)
![Round 1](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round1.svg?sanitize=true)
![Round 2](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round2.svg?sanitize=true)
![Round 3](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round3.svg?sanitize=true)
![Round 4](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round4.svg?sanitize=true)
