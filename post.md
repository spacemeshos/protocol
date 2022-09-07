# Mining - Proof of Space-time

Proof of Space-Time (PoST) is a new cryptographic primitive, designed to replace Proof of Work (PoW) as a proof-of-resource-consumption scheme, and so as a Sybil-resistance mechanism for permissionless cryptocurrencies. It was introduced in 2016 by [Tal Moran](https://talmoran.net/) and [Ilan Orlov](https://www.cs.bgu.ac.il/~ilanorv/) in a paper titled [_Simple Proofs of Space-Time and Rational Proofs of Storage_](https://eprint.iacr.org/2016/035.pdf).

A PoST allows a prover to convince a verifier that they expended a certain amount of a “space-time” resource: a specific amount of space allocated over a specific time period, during which the space cannot be used for anything else. PoST requires significantly less energy than PoW, since the “difficulty” can be increased by extending the time period over which the space is allocated, without introducing additional ongoing computational work.

PoST is a two-phase protocol. In the first phase, the _initialization phase_ (executed once), the prover commits to generate and store some data, derived from a specific seed. In the second phase, the _execution phase_ (executed periodically), the prover proves that they still have access to the data at some particular point in time.

Unfortunately, the protocol does not allow a prover to prove they _stored_ the data, since an adversary can instead store only the initial seed (which can be used to regenerate the data). To overcome this problem, generating the data involves computational work, which can be parameterized such that the cost of regenerating the data is much higher than the cost of simply storing it.

This leads to the definition of the PoST resource as a trade-off between space-time and computational work, where under reasonable cost assumptions, a rational user will always prefer to use the lower-cost space-time resource over computational work (while at the same time a non-rational user has greater cost and does not gain any advantage). It means that there may be a strategy which does not require a prover to expend space over time, but if the parameters are set correctly, this strategy will be significantly more expensive.

For this reason, using PoST in the context of a cryptocurrency, where profit is the main motive for participation, we can guarantee that rational users would prefer to use the space-time resource, so it can be claimed that the cryptocurrency is energy efficient. However, if the _initialization_ cost became lower than the cost of storing the data due to fluctuation in the market price of compute power relative to storage, the protocol essentially devolves to PoW. To avoid that, a _market-based mechanism_ can be used to determine the difficulty level, similar in spirit to the difficulty adjustment in PoW-based cryptocurrencies such as Bitcoin.

## Proof of Space-time vs. Proof of Space

Two very similar proposals for Proof of Space, with similar intentions, have been previously published ([1](http://dx.doi.org/10.1007/978-3-662-48000-7_29), [2](http://dx.doi.org/10.1007/978-3-319-10879-7_31)). Unlike Proof of Space, however, the PoST definition takes into account amortization attacks (using the same space resource for different proofs), rationality of using the space-time resource over CPU work, and an explicit time element (ensuring that the space is committed for a fixed length of time). Moreover, PoST uses a different and much simpler technique, which explicitly allows a space-time tradeoff, and only relies upon the security of cryptographic hash functions.

Unlike existing Proof of Space constructions, the computational difficulty of the PoST initialization phase can be adjusted without linearly increasing either the required amount of space or the cost of verifying the proof. Hence it is possible to use it to prove that a reasonable amount of storage has been committed over a long period of time. In addition, in the case of an adjustment in difficulty (due to CPU and storage market relative price fluctuation), PoST supports simple, incremental difficulty adjustment—that is, already-initialized users only have to pay the marginal work cost between difficulty levels, instead of rerunning the entire initialization phase from scratch. A downside of PoST is that the prover must read its entire storage for every proof, so the algorithm requires linear time, while in Proof of Space this is not required, so the algorithm runs in polylog time. This makes Proof of Space suitable for producing proofs in short intervals (e.g., every 10 seconds), while PoST is suitable for longer intervals (e.g., weeks or even months).

## Construction

To allow public verifiability, and to ensure that the data generated in the initialization phase is cheaper to store than to regenerate, generating this data must entail Proof of Work. To avoid the prover having to send all of the data to the verifier, the prover doesn’t construct one large proof, but rather a table containing many entries, where each entry is a PoW that can be independently verified.

### Phases

* **Initialization**: the prover performs computational work that results in a dataset, generated with respect to a specific seed, organized as table entries. The seed may be a public key, so that an adversary which does not hold the corresponding private key cannot claim custody over the same data.
* **Execution**: Given a challenge, the prover generates a proof that they have access to the data generated in the initialization phase. The challenge (i.e., the subset of data that it refers to) should be unpredictable in order to make sure that the prover stores the entire data, and not only those parts needed for the proof.

**The execution phase can be repeated multiple times** without rerunning the initialization phase. This point is critical, since the initialization phase requires work, while the execution phase is energy efficient (i.e., it requires relatively few computational steps). Thus, although using PoST to generate a single proof does not give any advantage over PoW, using it to generate many such proofs allows the work required during the initialization phase to be amortized over all of the proofs, which makes the construction energy efficient.

### Proving

Proving occurs at the end of the initialization phase (using an empty challenge), and on every subsequent iteration of the execution phase (as a response to a received challenge). This makes proving interactive between the prover and a verifier, except during the initialization phase, where the challenge is empty and, thus, non-interactive.

The _time_ element of the space-time resource is the elapsed time between successive phases of generating the PoST proof. Each PoST proof refers to the previous proof, i.e., the proofs are chained (not unlike the way in which each Bitcoin block refers to the previous block). The first proof points to the initialization phase proof (akin to a genesis block). In this fashion, two sequential proofs which each independently attest to the data having been available for, say, one week, can be chained together to prove that the prover had the data for two full weeks. For more information on how the _elapsed time_ component can be proven in a similar fashion, see the next section on [Proof of Elapsed Time](poet.md).

To generate a proof, the prover commits to the data by constructing a Merkle tree whose leaves contain the data generated during the initialization phase (i.e., one or more of the generated table entries). Each internal node in the tree contains the output of a hash function on the concatenation of its children, salted with the challenge. This forces the prover to commit to their knowledge of the leaves (i.e., all of the data) at the time they generate the proof.

Once constructed, the prover needs to provide Merkle proofs (paths) to the root of the tree for a random set of, e.g., 100 entries, which serve as a _challenge._ The indices for these entries can be selected by the verifier as a response to the prover commitment (Merkle root). However, such an algorithm requires back-and-forth, interactive communication between the prover and the verifier, and as such is not practicable for use in a production blockchain protocol, which requires that _anyone_ can publicly, independently verify a proof in a _non-interactive fashion._ This is accomplished by using the [Fiat-Shamir heuristic](https://en.wikipedia.org/wiki/Fiat%E2%80%93Shamir_heuristic): the Merkle root hash is used to deterministically derive the random set of indices, i.e., the challenge. (Note that, as presented here, the _execution phase_ still requires interaction since the PoST proof must be generated in response to a received, unpredictable challenge; see [NIPoST](nipost.md) for an explanation of how this is also made non-interactive.)

The prover cannot merely generate the data for the selected entries and skip all others, because they are forced to commit to the entire table and to a specific Merkle root before the challenge indices are known. If they happened to “skip” an entry which will end up being selected, the proof would be invalid. This means that the prover is forced to either generate a large fraction of the table, or spend a lot of computational work trying to find a Merkle root that will produce a “good” set of indices. By setting the parameters correctly, we ensure that in the second case the amount of work the prover must do is more than the initialization cost.

The publicly-verifiable proof contains the Merkle root, the data for the selected entries from the PoW initialization phase, and their Merkle proofs. Common labels from the random set of distinct Merkle paths are eliminated to compress the proof.

### Verifying

The verifier first derives the same set of indices (the challenge), similarly to the prover, using the Fiat-Shamir heuristic. Then she computes the Merkle root from the commitment Merkle proofs to verify that it matches the expected root value. Then she extracts the chosen entries from the proven leaves (each containing a group of entries), and verifies that the PoW is valid for their respective indices and seed.

## Why generic PoW isn’t sufficient

Generating the data in the initialization phase involves computational PoW, but the standard definitions of PoW do not rule out an adversary that can store a small amount of data and use it to regenerate the entire data set with very low computational overhead (i.e., generate a large dataset, then compress it, and decompress it later). Thus, to ensure the prover must indeed store the entire data set, the protocol is constructed using _incompressible proof-of-work_ (IPoW), a more restrictive variant of PoW that ensures that the prover cannot store compressed data, nor can they “compress” the required work to generate those data (i.e., perform less work).

The most popular form of PoW, the Hash-Preimage PoW, was introduced in [Hashcash](http://www.hashcash.org/papers/hashcash.pdf) and is currently used in Bitcoin and Ethereum. It involves finding a certain value (called a nonce) that, when added to some fixed data (e.g., the contents of a block, known here as the _preimage_) and hashed, satisfies a certain difficulty criteria. This is not yet an incompressible PoW: although compressing the hash output is impossible, compressing the preimage isn’t.

However, the Hash-Preimage IPoW sets an upper limit on the amount of storage we can fill for a given initialization cost (number of hashes): with a predetermined sequence of values, one just needs to store a single bit for each item, indicating whether the item is a “good” preimage or not. Thus, due to the compression scheme, each hash invocation can “contribute” at most a single bit to the total storage.

### The Partial-Hash IPoW

In order to be able to fill more space without increasing the initialization cost, a different IPoW is needed. _Partial-Hash IPoW_ is a simple solution where the amount of work for every IPoW is always a single hash invocation, but the amount of storage the hash output would fill is adjustable (from one bit up to the entire output length). It is considered incompressible because, although the hash input might be smaller than the storage the output fills, the required CPU work for regenerating the output cannot be made cheaper, since it is already just a single hash invocation.

### PoST deterministic nature

The entire PoST construction, from the Partial-Hash IPoW for generating the data to the Merkle tree commitment and proofs, is completely deterministic: e.g., the number of required hash invocations for any given space-time resource is known in advance. This makes PoST different in nature from PoW, where successfully generating a proof is probabilistic. In contrast, PoST allows us to guarantee that anyone spending the required resources is eligible to produce a valid proof. Thus PoST can be used to replace the lottery-based mechanism for block production (and so for rewards) in PoW with a deterministic one, meaning that leader-election is not a required part of the consensus protocol, enabling a [race-free cryptocurrency protocol](https://spacemesh.io/race-freeness/).

## What about time?

As presented above, the PoST construction allows a prover to assert, deterministically and verifiably in a non-interactive fashion, that they have generated and have access to a particular dataset derived from a known seed based on their unique ID. Where does the “time” component derive from? In other words, how does the prover assert, in the same fashion, that they not only performed the initial generation of this dataset, but that they still have access to this data over time?

The next section, [Proof of Elapsed Time](poet.md), expands upon the explanation above and explains in detail how this part works.
