# The Hare Protocol

## Description
The Hare protocol is a consensus protocol that achieves consensus on a set of values. In our case, we want to achieve consensus on a set of block ids (the set of values) in a specific layer (the instance/layer id).

The general problem is the [Byzantine Agreement Problem](https://en.wikipedia.org/wiki/Quantum_Byzantine_agreement). Our protocol is based on the [ADDNR18 paper](https://eprint.iacr.org/2018/1028.pdf) and differs mainly on the fact that we want to achieve consensus on a set of values rather than a single value.

It is known in advance when an agreement process should start for each layer. On the other hand, the time it takes to achieve agreement can vary, depending on the number of faulty/malicious participants in the consensus. Hence, multiple consensus instances may be run concurrently.  

On this page we will discuss the core protocol concepts and clarify it. Please note we do not intend to talk about the implementation itself, we might mention practicale ways that overlap with it.

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
3. `Validity 2` (“no honest witness”): If for no honest party Pi value v in ISi then v is not in S'. In other words, all values in S' should have had at least one honest witness.
4. `Termination` - All honest participants terminate with overwhelming probablity.

#### Active Participants
In each round, a new committee is selected randomly by the oracle (which we assume its existance).
Picking a random committe over having all the nodes participate in the protocol is good for two main reasons:
1. Less participants means lower communication requirments.
2. Random election of participants means the participants are not predicted hence less likely to be exposed to DDOS attacks (for example).
In the proposal round, the expected number of participants is low (1 for example). In all other rounds we expect N (active) participants.

#### Commit Certificate - C(S,K)
A proof that consists of f+1 commit messages on the same set S for the same iteration K.
f+1 commit messages implies at least one of them is honest and hence the set S is ensured to be valid.

#### Safe Value Proof (SVP)
A proof made by a participant to ensure that a set S satisfies `validity 1` in respect to a specific round K. An SVP also includes a certificate with which we can ensure consistency.

---

## The Protocol

**Pre-Round**
The protocol begins with one `pre-round`. This round is executed only once and its goal is to remove values which shouldn't be considered at all according to `Validity 2`
- At the beginning of the pre-round each active party sends his set of values ISi
- At the end of the pre-round, each value that hasn't received f+1 witnesses is removed. This ensures that we are left only with values that has at least one honest witness and therefore satisfying `validity 2`

The protocol repeatedly iterates through up to 4 rounds until a consensus is reached.

Each round longs a constant time. A `Roles Oracle` is used to generate roles in each round.

**Status Round**
- Active participants broadcast their current status to the network (S, k , ki)
- At the end of this round, a participant can form an SVP based on the statuses he collected during the round

**Proposal Round**
- The leader can now give his proposition set T with the corresponding SVP P to the network
- Each participant can validate the proposition sent by the leader and consider that set T on the next round

**Commit Round**
- Active participants announce their will to commit to the proposed set T
- If a participant observes f+1 participants willing to commit to T, he commits to T and constructs the matching certificate (which is the proof that he witnessed f+1 commits to T) and sends a notify message stating that he committed to T
- If equivocation is detected by a party P, it doesn't commit to T. Equivocation means that the leader sent two or more contradicting proposals to the network

**Notify Round**
- Upon receiving a valid notify message the party updates his internal state according to the attached set

**Termination**
If at any point of the protocol, a party P receives f+1 notify messages on the same set S, it commits to S and terminates. Note that after an honest party has terminated, the other honest parties are ensured to terminate up to the following round, thanks to the gossip properties.

## Eligibility Oracle

Eligibility to participate in a round (being active) is determined by the oracle mechanism.
The oracle is required to provide three main properties:
A. Eligibility is under consensus - meaning all honest participant who receive a message will either classify it as active or as passive.
B. The eligibility can be calculated in advance only up to some configurable limit of time before it can be used to participate.
C. Eligibility is detrmined for each round in a layer separately and at random.

### Proof & Validation
The Threshold
Thresholding a random input can be used to determine eligibility. I.e. we can set the threshold a participant should pass in order to participate. For example, we can set a threshold of X participants over the space of a 32 bit uint which is simply X/2^32. Randomly picking numbers in [0,2^32-1] will result in an expected number of X participants passing the threshold and hence X actives.

**Proof**
A paritipant who wants to prove eligibility attaches the signature of the VRF message.
The VRF message is the tuple {agreed value, Layer, Round}. Signing the layer and the round is what ensures property C.
The value is provided by the Hare Beacon. It aims to ensure that the value is under consensus. I.e. all partipants will use the same value (hence complying with property A). The value, for example, can be taken from some point in the past of the mesh which complies with property B.
The reason we use a VRF signature is that we want to make sure the signing process is not grindable. This is ensured by the property of VRF messages where each output has a single source.

**Validation**
In order to validate the vrf message the receiver should:
1. Reconstruct the matching VRF message and check the signature against that message.
2. Validate the the hash of signature of the VRF message passes the treshold.

**Leadership**
The leader eligibility is derived by the same process described above. The only difference is that the expected number of actives is set to 1. Since eligibility is random, it is possible that more than one leader will exist in the same (proposal) round. To handle this, we agree on a way to order the signatures in order to decide who is the real accepted leader. For example, this can be done by comparing bytes and accepting lowest ranked leader. 

## Message Validation & Processing Flow

### Contextual Validity
A message M(TYPE, K) where K is the round counter (K%4 is the round K/4 is the iteration) is said to be contextually valid iff the reciever received the message on round K%4 of iteration K/4.
For example, a message of type status round should arrive during the status round and in addition if K=8 then the receiver should be in the second iteration.
A message that arrives at the correct iteration but one round before the correct round is considered early.
A late message is a contextually invalid message.


### Syntactic Validity
Syntax validity assures the stracture of the message is correct. For example if it is a proposal message then it should include an SVP.

![Message Validation](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/msg_validation.svg?sanitize=true)
![Round 1](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round1.svg?sanitize=true)
![Round 2](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round2.svg?sanitize=true)
![Round 3](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round3.svg?sanitize=true)
![Round 4](https://raw.githubusercontent.com/spacemeshos/protocol/hare/hare/svg/round4.svg?sanitize=true)
