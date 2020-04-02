# Hare Protocol FAQ

Q: Do all online full nodes on the network participate in the consensus of every layer?

A: They all listen to protocol messages sent over the gossip network but they are not allowed to send messages.

---

Q: Does a different active set and leader being set each and every round?

A: In every round a committee of (about) N (probably 800) nodes is being elected randomly. This committee is the only one who can send messages during the relevant rounds (status, commit, notify or pre-round). In the case of a leader, it is the same idea, we set the params such that on average we will have 1 leader in the proposal round. So we do not set actual roles but instead for each round we make a query to the oracle with the expected members in the committee, this way we can check if a sender is eligible or not relatively to the current round.

In order to to terminate, an honest node must see f+1 notification messages on the same set (which will be the output set), once he sees them he can be sure that all other honest will see it in the following round (thanks to the gossip propagation property) and hence, can terminate.

---

Q: When saying the pre-round is only executed once. Do you mean once in a layer until consensus is reached? So we have this sequence: [pre-round]{[0,1,2,3]...} - meaning pre-round and then 1 or more iterations of the 4 rounds until consensus is reached?

A: Exactly. That is because the pre-round round is only used to take care of validity 2. Once we filtered our set from unprovable values we can be sure that the consensus will be only over values that are validity2-valid.

---

Q: So, the pre-round is used to filter blocks that we didn’t get from at least f+1 active participants?

A: Yes, because that will mean that every value (block id) that is not filtered after the pre-round had f+1 witnesses, meaning it was seen by at least one honest, meaning validity 2 is satisfied.

---

Q: What are `k` and `ki` discussed round 0 - they are not previously defined? Do you mean ROLES by ‘current status’? Not clear... So S is a set of blocks. Please define k, ki and status...

A: In the protocol, k is used to describe the current iteration. ki is used to describe the last iteration for which we have a certificate. The idea with ki is to always track the max iteration for which we accepted a proposal (see round 4) and to allow to accept only newer certificates as the protocol goes on.

Q:  Round 2: is the `notify message` sent to the whole network via gossip?

---

Q: Round 2: what is `equivocation` - it is not previously defined. Please define.. what does it mean to detect and equivocation?

A: Equivocation, generally means that someone tried to present more than one truth. Specifically itis used to describe a malicious leader that sends more than one proposal to the network - that is called leader equivocation or proposal equivocation. This behavior can be broaden to other message types, for example pre-round messages.

---

Q: Round 3: what internal state is updated? what notion, set, data, etc...

A: Ti is a temporal set. Si is the set in our state (at the beginning it is the initial set). When we do Si=Ti we simply update our set. In round 3 we also update the tracked certificate (since we have just created a new one).

---

Q: Termination: can it happen before round 3 or only after round 3? meaning, can the protocol ever terminate after round 2 or does each party always run 4 rounds?

A: It doesn't need 4 rounds but it can't terminate in the proposal round. But, after the proposal, at the end of round 3/sometime in round 4, we can terminate if we received enough notification messages (depends on the network propagation).
2. You are right, I will rephrase. This committee is the only one who can speak in this round. We pick a committee of size N for each round except for the proposal round for which we pick a committee of size 1.

Let's mark them Pre-Round and Round1 to Round4. In that case, the protocol can terminate at the end of round 3 (assuming zero propagation). If not we continue to round 4 as usual and after that back to round 1 (which is considered to be the next iteration)

---

Q: As long as less than 1/3 + are dishonest - does hare always eventually terminates with a consensus on a set even if more than 1 4-rounds iteration is needed?

A: Regarding the termination it is a bit delicate. When we talk about termination we actually relate to termination under the assumptions of our system which is 1. 2/3 honest majority. 2. Intersection of honest sets is not empty. The former implies that with extremely high probability we have a 1/2(+1) honest majority. The latter means the pre-round will never (actually, very high probability = 2^-40) end with empty set. Put these two together and it means that on expectation the consensus will terminate in two iterations (because the protocol ensures that once an honest is picked as leader - the consensus will terminate in that same iteration). Putting that aside, there is about 2^-40 probability that the protocol will never terminate.

---
Q: what happens when 1/3 +1 are dishonest? Does hare terminate w/o a set or does it never terminates? e.g. can the dishonest parties cause hare to never terminate?

A: Well the question is not well defined. Please check my answer to Q11. Generally, as long as dishonest leader is picked to lead - he can stall the consensus by one iteration. Since we assume half are honest we expect two iterations as I explained. On the other hand, if we tolerate f dishonest and there are f+1 or more (no matter if f is 1/3, 1/4 or 1/2 etc) then the assumptions are broken, nothing is guaranteed and you can end up with non consistent sets on termination or not terminate at all.
