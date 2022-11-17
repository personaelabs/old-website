---
title: "ECDSA nullifiers and their applications"
date: 2022-10-12T02:12:03.284Z
authors: ["yush_g"]
type: posts
draft: false
slug: "nullifier"
category: "20 mins read"
tags: ["nullifier", "zk"]
description: "Unique pseudonymity??"
math: true
---

**ZK ID systems**

Spurred by new advances like circom, groth16, and tornado.cash, [zkSNARKs](https://ethereum.org/en/zero-knowledge-proofs/) are already deployed in many real world settings. Zero knowledge identity systems based on anonymous proof-of-ownership are poised to be an interesting social experimentation tool in the coming years -- in order to unleash the full power of these systems, one critical primitive is deterministic nullifiers, an idea we will explain in this post. Specifically, the ability to reveal specific parts of your identity in lieu of the whole is an interesting new primitive that allows provable on chain identities like “BAYC holder” or “LP provider on Uniswap in last 24 hours” or maybe even one day “people with a mutation in the XYZ gene”, and early versions of such pseudonymous identity already exist, like [cabal](https://www.cabal.xyz/).

This is done via a zero knowledge proof of knowledge that their account satisfies that property without revealing who they are. Set membership proofs are a typical [usecase of zk-snarks for privacy](https://vitalik.ca/general/2022/06/15/using_snarks.html), and we can imagine a zk proof of the form "I can prove that I own the private key [via generating a valid signature], for some public key that is a leaf of the Merkle tree comprised of all eligible set members, with this public Merkle root." Signatures via ECDSA are nondeterministic, meaning a signer can produce an arbitrary number of signatures for a message. However, they do provide non-repudiation of signed data (i.e. it proves they signed it). We ultimately enable unique pseudonymity by deploying verifiably unique signatures on Ethereum.

This enables applications such as [semi-anonymous message boards](https://twitter.com/heyanonxyz), since a user merely needs to prove existence of at least 1 valid signature per message in order to be sure that such a message is legitimate. However, such applications have the advantage that there is no uniqueness constraint on the provers: that is, the same wallet proving themselves as a member multiple times is intended behavior. However, there are many applications that require a maximum of one action per user, like claiming an airdrop.

**One address <-> one nullifier**

For a concrete example, [a zero knowledge airdrop](https://github.com/nalinbhardwaj/stealthdrop) requires that an anonymous claimer can produce some unique public identifier of a claim to a single address on an airdrop, that does not reveal the value of that address. One can imagine a "claimer" can send a zk-proof of knowledge of set membership and private key ownership (whether through signature verification or something else). A public nullifier signal ensures they cannot claim again in the future. This unique public identifier is coined a "nullifier" because the address nullifies its ability to perform the action again.

Let’s consider a few different nullifier ideas here, and by understanding why they don’t work, we can define properties of a nullifier that would work. As a first example, something like a hash(public key, public set identifier) seems promising -- it’s unique for each address. However, you can easily reverse the hash by brute forcing the finite number of on-chain addresses, so alternate solutions are needed.

Maybe something like hash(secret key) could work then? No one can brute force them without all the secret key then. However, your nullifier would be the same across all the applications you use, which might be undesirable.

One can then imagine that hash(message, secret key) is a decent nullifier, where each app has (usually) one canonical message -- there's no way to reverse engineer the secret key, it’s lightweight and computable in a hardware wallet, and it is unique for each account and application. However, we can’t verify it without access to the secret key itself. For security reasons, we want a way to be able to do all of these computations without a user having to insert their private key anywhere, especially not copy paste it as plaintext. For anything that does need a private key, we want computation to be very lightweight so we can run it on a hardware wallet as well if needed: the complex elliptic curve pairing functions required to prove zk-snarks are not feasible to compute in current hardware wallets.

One can then imagine that a signature would be an ideal nullifier, and that was the initial idea for [stealthdrop nullifiers](https://github.com/stealthdrop/stealthdrop). You may have heard of a “deterministic signature scheme” -- if you deterministically sign some app-specific message, could hash[sign(message, pk)] be an effective nullifier? Ethereum has deterministic signatures, such that the randomness is deterministically derived from the private key (like hash[private key]). However, this randomness is unverifiable unless the zk-snark has access to the private key, so a malicious adversary could generate ~2^256 ECDSA signatures (and thus nullifiers) per message. Thus, usual Ethereum signatures don’t work as a nullifier.

It turns out there’s a little bit of literature on verifiable unpredictable functions that have essentially the same properties that we want, but the constructions we found don’t work out of the box with ECDSA (specifically, ECDSA uses the secp256k1 elliptic curve, but most of these functions use pairing-friendly curves). If we could use a pairing-friendly curve, we could have just used the deterministic BLS signature scheme and skipped this whole exercise!

Back to the drawing board -- let’s try to combine the intuitions about a simple hash based function with our desire for unique signatures. What if there was a function like hash[m, pk]^sk -- easy enough to calculate in a hardware wallet’s secure enclave, and possible to verify with only the public key? This is the key insight that we use to generate our nullifier scheme, by weaving this nullifier into a signature scheme.

But, 0xPARC, don’t [zcash](z.cash) and [tornado.cash](tornado.cash) get along just fine without more complex nullifiers? That’s only because they allow interactivity, so a user could simply hash a random string to start with, and use another hash of that preimage as their nullifier from then on: tornado.cash uses exactly this for claims. Other than the pesky pre-interaction requirement, the second issue is the size of the anonymity set, limited to everyone who signs such a message. The first tornado.cash user could only withdraw once several people had signed the message, and anonymity is limited by shaky assumptions like requiring other people to interact with the platform i.e. between deposits and withdraws, which has [already been used to break anonymity](https://www.tutela.xyz/). Finally, tornado.cash/[semaphore](https://medium.com/privacy-scaling-explorations/semaphore-v2-is-live-f263e9372579) reveals your nullifier in plaintext in-browser, allowing Chrome and other malicious extensions to access it. It would be better if only the owner of the private key secure enclave could access that information.

Beyond improvements to existing apps, noninteractivity enables new use cases for ZK ID systems, because it allows a large anonymity set to begin with. ZK airdrops also can’t have an interactive phase, or else they open the protocol to sybil attacks and spam. If your zk proof can verify set membership in the Merkle tree, the message via the signature, and the unique nullifier, then the anonymity set is always the full set of all eligible users, and no interaction is required. With an interactive nullifier such as tornado.cash’s, you have to update the anonymity set merkle tree with each new person.

Let’s review the properties we want in this nullifier. If we want to forbid actions like double spending or double claiming, we need them to be unique per account. Because ECDSA signatures are nondeterministic, signatures don’t suffice; we need a new deterministic function, verifiable with only the public key. We want the nullifier to be non-interactive, to uniquely identify the keypair yet keep the account identity secret. The key insight is that such nullifiers can be used as a public commitment to a specific anonymous account.

**A promising new standard**

Consider a signature that has two parts: a deterministic part based on a message [like hash(message)<sup>sk</sup> (mod p)] and secret key, and a non-deterministic part that allows it to be a signature algorithm. Then, we could use the deterministic part as a nullifier, and use both the deterministic and non-deterministic part to verify that it is a valid signature using only the public key.

Here’s an example of such an algorithm. It’s traditionally called a DDH-VRF -- a combination of the Goh-Jarecki EDL signature scheme and Chaum-Pederson proofs of discrete log equality, and inspired by BLS signatures, proposed by Kobi and 0xPARC. We’ve color coded the equivalent numbers so you can easily check the math.

Note that hash has to be a function that hashes directly to the curve, meaning the output is an (x, y) pair instead of a scalar. hash2 is a traditional hash function like sha256 or posiedon. This construction assumes the discrete log problem is hard. We use exponential notation here so you can apply your usual intuitions about discrete log, but these exponentiations are actually implemented as elliptic curve multiplications.

```
private to everyone except the secure enclave chip:
	sk
	r
public to world, calculated inside secure enclave:
hash[m, pk]sk 					<-- public nullifier
private input in zk-snark, private to world, public to wallet/user:
    c = hash2(g, pk, hash[m, pk], hash[m, pk]sk, gr, hash[m, pk]r)
    pk = gsk
	r + sk * c						<-- can be public
    gr 			[optional output]
    hash[m, pk]r 	[optional output]
verifier checks in SNARK (calculated by wallet, private to world):
    g[r + sk * c] / (gsk)c = gr
    hash[m, gsk][r + sk * c] / (hash[m, pk]sk)c = hash[m, pk]r
    c = hash2(g, gsk, hash[m, gsk], hash[m, pk]sk, gr, hash[m, pk]r)
    the set inclusion check of the public key
```

Note that in practice, we omit the last 2 optional private signals and skip the first two verifier checks, because we can recalculate the two optional outputs from the rest, and the hash output check will still pass and verify them all. Also, for wallets that separate the secure element with the secret key from the general computation chip, we usually do the hashing and calculations outside the secure element itself, and only need the secure element to call the function that multiplies our provided generator points by the secret key (represented by exponentiation here).

Although you don't have to understand all of the math, I’ll attempt to explain the intuition. c functions kind of like a [Fiat-Shamir hash](https://en.wikipedia.org/wiki/Fiat%E2%80%93Shamir_heuristic) for proof of knowledge of the secret key. Since it is almost impossible to reverse-engineer a desired hash function output, we can “order” some of the calculations. If we are feeding numbers into the hash function and getting c out, it’s highly likely that the inputs to the hash function were calculated before c was, which was then likely calculated before all the functions with c were. Thus, c was likely calculated prior to r + sk _ c. If the c was calculated first, we must have pre-committed to a specific g^r and thus r, so we must have known the secret key to be able to calculate r + sk _ c.

Conveniently, the verifier check doesn't require the secret key anywhere, just the public key (g^sk)! This makes it possible to check in a wallet or an extension -- the enclave generating proof knows the public key, but doesn’t know the secret key.

Importantly, the public nullifier is effectively random noise to anyone, and even knowing the full set of possible public keys leaves you with no idea if the nullifier is theirs or not.

Finally, all 6 of the signals emitted by the secure enclave, leaving no way to derive someone’s secret key. Even if these signals are all leaked accidentally, it only deanonymizes the user but doesn’t allow anyone to steal their funds. We will have a formal proof of this in a paper very soon!

This is promising since hardware wallets only need to be able to calculate hash functions, not entire ZK systems, whose security has yet to be formally verified and whose implementations we can likely see evolving over the next few years.

**The interactivity-quantum secrecy tradeoff**

Note that in the far future, once quantum computers can break ECDSA keypair security, most Ethereum keypairs will be broken, but migration to a quantum resistant keypair in advance will keep active funds safe. Specifically, people can merely sign messages committing to new quantum-resistant keypairs (or just higher bit keypairs on similar algorithms), and the canonical chain can fork to make such keypairs valid. ZK-SNARKs become forgeable, but yet secret data in past proofs still cannot ever be revealed. In the best case, the chain should be able to continue without a hitch.

However, if people rely on any type of deterministic nullifier like our construction, their anonymity is immediately broken: someone can merely derive the secret keys for the whole anonymity set, calculate all the nullifiers, and see which ones match. This problem will exist for any deterministic nullifier algorithm on ECDSA, since revealing the secret key reveals the only source of “randomness” that guarantees anonymity in a deterministic protocol.

If people want to keep post-quantum secrecy of data, you have to give up at least one of our properties: the easiest one is probably non-interactivity. For example, for the zero knowledge airdrop, each account in the anonymity set publicly signs a commitment to a new semaphore id commitment (effectively address pk publishes hash[randomness | external nullifier | pk]). Then to claim, they reveal their external nullifier and ZK prove it came from one of the semaphore ids in the anonymity set. This considerably shrinks the anonymity set to everyone who has opted in to a semaphore commitment prior to that account claiming. As a result, there probably needs to be a separate signup phase where people commit to nullifiers in order to seed the anonymity set. This interactivity requirement makes applications such as the zk airdrop or nicer tornado cash construction (in the usecases section) much harder. However, since hashes (as far as we currently know) are still hard with quantum computers, it’s unlikely that people will be able to ever de-anonymize you.

A recent approximation of 2n<sup>2</sup> qubits needed to solve discrete log via quantum annealing[^1] that failed to work on larger than n = 6 bit primes shows that we are likely several decades from this becoming a reality, and the n<sup>2</sup> qubits needed to solve RSA having predictions 30-40 years out[^2] suggest that it will likely take longer than that to solve discrete log. For a complete mental model of quantum x blockchains, we wrote an [overview post here](https://hackmd.io/vXWmu5QsSOGVSz9N03LXuQ).

We hope that people will choose the appropriate algorithm for their chosen point on the interactivity-quantum secrecy tradeoff for their application, and hope that including this information helps folks make the right choice for themselves. Folks prioritizing shorter term secrecy, like DAO voting or confessions of the young who will likely no longer care when they’re old, might prioritize this document’s nullifier construction, but whistleblowers or journalists might want to consider the semaphore construction instead.

**New usecases enabled**

Ok, so let’s say we go through all of the work to create such a canonical system, and manage to get all hardware wallets and browser wallets to implement it (or even just one). What can we do now, with anonymous uniqueness?

ZK airdrops become easy: we merely make sure you are in the Merkle tree of allowed users in zk, and use this nullifier as a check for double-claiming.

Message boards can have persistent anonymous identity, meaning someone can post under the same nullifier 3 times, and everyone knows that they are the same person, but not who that person is. You can also allow banning of anonymous accounts, and give accounts karma (reputation).

You can build anonymous sybil resistant apps like voting, where you need to ensure people haven’t double voted (ideally, you also use MACI to avoid receipts).

We can also build more user-friendly dark pools, where instead of leaking the nullifier string in plaintext on the frontend from a server via the website, the nullifier can simply be this ECDSA nullifier corresponding to some message in a standard format, like “Aztec/tornado note 5 for 10 eth”. Systems like Aztec currently get this via encrypting the random nullifier string with someone’s encryption key, but these nullifiers let you do this on the Ethereum base curve itself, which doesn't support encryption.

We think that wallets that adopt this standard first will hold the key for their users being able to interact with the next generation of ZK applications first. We’re bullish on a future where this is a standard as commonplace as ECDSA signing, within every secure enclave.

**Next steps**

So far, we have the [Gupta-Gurkan nullifier paper proving all the security proofs](https://aayushg.com/thesis.pdf), repository with the [nullifier calculation in Rust](https://github.com/zk-nullifier-sig/zk-nullifier-sig/), with [working circom circuits](https://github.com/geometryresearch/secp256k1_hash_to_curve/), and a [Metamask snap](https://ethglobal.com/showcase/zk-nullifier-snap-6a9sq) to calculate it.

In the future, we are hoping to formalize our spec into an ERC, and use that to integrate the nullifier calculation into [burner wallet](https://github.com/austintgriffith/burner-wallet), [Ledger core](https://github.com/LedgerHQ/app-ethereum), and [Metamask core](https://github.com/MetaMask/metamask-extension). We also want to try benchmarking the proof in different languages like Halo2 or Noir that might be faster i.e. if they use lookups for SHA256. If you’re interested in helping or using the scheme, reach out to us on Twitter ([@yush_g](https://twitter.com/yush_g/), [@personae_labs](https://twitter.com/personae_labs)) for a grant!

<!-- Footnotes themselves at the bottom. -->

## Notes

[^1]: ​​https://link.springer.com/chapter/10.1007/978-3-031-08760-8_8
[^2]: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9797855
