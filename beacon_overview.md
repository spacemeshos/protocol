# Beacon

## Overview

The spacemesh protocol requires a source of randomness to introduce unpredictability in the eligibility of making proposals and participating in the Hare protocol. This property is important for preventing a concentration attack where adversaries can influence the mesh content by creating a temporary majority at a given layer. 

Additionally in the Tortoise protocol, every smesher cast votes on all historical blocks when making proposals. When honest parties cannot decide how to vote on blocks, a weak coin is used to help them vote in the same direction on all undecided blocks in older layers. The generation of this weak coin also needs an element of randomness to prevent the adversary from unduly influencing the votes.

For the beacon to achieve its intended purpose, all honest parties must agree on the beacon value, and the value must be both unpredictable and hard to influence. As a result, the value cannot depend on any agreement about history, and needs to be generated recently enough to make it difficult for adversaries to prepare for an attack, yet allow ample time for the network to reach consensus on the value.

Spacemesh runs a beacon protocol to generate a random value as the beacon at the beginning of every epoch. A miner (a.k.a smesher in [Spacemesh parlance](https://github.com/spacemeshos/testnet-guide/blob/master/dict.md)) must satisfy all of the following conditions to participate in the beacon protocol in epoch _N_:


* It is online
* It is synced. i.e. It has all the data in the network up to the current layer and has listened to gossip for at least 2 layers.
* It has a valid ATX in epoch _N-1_

The beacon generated in epoch _N_ is declared by every smesher in its first ballot in epoch _N+1_. By design, the beacon is not recorded in every ballot in epoch _N+1_. The goal is to prevent smeshers from changing their beacon values mid-epoch. Having the correct beacon determines whether a smesher can generate proposals, participate in the Hare consensus protocol and eventually earn rewards.

The beacon protocol is essentially a simplified Tortoise consensus protocol: each sampled smesher first proposes its own seed of randomness, then all active smeshers cast votes on these proposals repeatedly until consensus is reached. The adversary learns the beacon value after the first round (if there's no attack, then the first round proposals will be exactly the final consensus). The purpose of the voting iterations is to ensure honest miners reach consensus even under adversarial attack, and the number is set such that the probability that they don't reach consensus is negligible. By the time the beacon value is revealed, there won’t be sufficient time for adversaries to take advantage of it in time for the next epoch via key grinding. Key grinding is an activity where malicious parties burn through a large number of keys to find one that gives them favorable future eligibility. Specifically in the Spacemesh protocol, the only way the adversary can "key grind" is by grinding **on their identity**. Once an identity is published, it commits the adversary to all future randomness.


## Uses

The beacon is used in the following components of Spacemesh:


### Hare Protocol

A smesher is eligible to participate in the hare protocol when its VRF output is valid and passes a certain threshold. The beacon is part of the input to the VRF. If a smesher uses a different beacon value than the rest of the network, its VRF output is deemed invalid by the rest of the network. Its hare messages will be rejected.


### Tortoise Protocol

Ballots are how smeshers vote for all historical blocks. The Tortoise protocol counts the votes in these ballots to reach consensus on historical blocks. A smesher must declare a beacon value in their first ballot (a.k.a the reference ballot) of an epoch and will not be able to change it for the rest of the epoch. All its remaining ballots in the epoch point to the reference ballot. The beacon values in these ballots are used to determine how smeshers' votes are counted.


#### Verifying Tortoise

In verifying tortoise, votes from ballots with beacon values different than what the verifying node believes is in consensus are not counted because these ballots are potentially malicious.


#### Self Healing

During self healing, votes from ballots with incorrect beacons are postponed. This is because in the case of self healing the network may be in the process of recovering from a network partitioning event where each partition has its own beacon value. The network needs to count all ballots, correct beacon or not, to eventually converge on their opinions about historical blocks. When a smesher’s node is in self-healing mode, ballots in layer _N_ with incorrect beacons will only be counted after _D_ layers have elapsed, where _D_ is the number of layers to delay the effects of ballots with the wrong beacon. In this case, their votes only start to matter when the network advances to layer _N+D_.
