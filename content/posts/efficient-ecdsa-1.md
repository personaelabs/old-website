---
title: "Efficient ZK ECDSA (part 1)"
date: 2022-11-10T22:12:03.284Z
authors: ["Personae"]
type: posts
draft: false
slug: "efficient-ecdsa-1"
category: "5 mins read"
tags: ["cryptography"]
description: "Reviewing a construction to reduce constraints for private ECDSA signature verification"
math: true
---

This is the first post of a 2 part series on our research improving private ECDSA signature verification, stemming from this [ETHResearch post](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629) and implemented in this [repository](https://github.com/personaelabs/efficient-zk-ecdsa). In this post, we motivate our research and introduce our key insights. In the follow-up, we will discuss on-chain extensions, provide a full security proof, and introduce implementations in newer proving systems like PLONK.

{{< toc >}}

# Motivation

## ECDSA & ring signatures

The **Elliptic Curve Digital Signature Algorithm**, or [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Signature_verification_algorithm) for short, is a key tool in the identity layer of blockchains like Ethereum and Bitcoin. In particular, each address on these chains is the hash of an ECDSA public key, which itself is a secp256k1 elliptic curve point. We sign all transactions with an ECDSA public key, which verifies we know the corresponding private key without revealing it.

Proving you know one of the private keys in a _group of public keys_ without revealing it is a necessary primitive for many of the anonymous speech applications Personae is interested in ([cabal](https://cabal.xyz), [heyanon!](https://heyanon.xyz), [storyform](https://storyform.xyz), [heyanoun](https://heyanoun.xyz)). Surprisingly, this simple extension requires much heavier cryptographic machinery than a normal signature; there's an entire field of research dedicated to this problem called [**ring signatures**](https://en.wikipedia.org/wiki/Ring_signature). Unfortunately these methods aren't compatible with Ethereum/Bitcoin's ECDSA keys out of the box, so we opt to use an even more overpowered tool in zkSNARKs.

The zkSNARK method privately inputs a group member's public key $pk$ and a signature $s$, and publicly inputs the entire group $G$ (usually succinctly as a [Merkle root](https://decentralizedthoughts.github.io/2020-12-22-what-is-a-merkle-tree/)). The circuit verifies $s$ was generated from $pk$ AND that $pk$ is in the group $G$. Sounds simple enough! Unfortunately, the math for the signature verification is very SNARK-unfriendly. In particular, there's a lot of "wrong field arithmetic" --- the necessary operations happen over the [_base field_](https://zcash.github.io/halo2/background/curves.html#cycles-of-curves) of the secp256k1 curve which is different from the [_scalar field_](https://zcash.github.io/halo2/background/curves.html#cycles-of-curves) of the curves used in snarkjs. If that means nothing to you, don't worry -- it just means we have a huge blow-up in constraints to make sure all of our elliptic curve math is done correctly. And if that also means nothing to you, don't worry -- it just means computing a proof is very computation and memory-intensive, requiring ~1GB of proof metadata and ~5 minutes of browser computation on a MacBook. The [0xPARC blog post](https://0xparc.org/blog/zk-ecdsa-2) that first introduced this construction dives into this in more detail.

## Client-side proving

Okay, so if our gigachad friends at 0xPARC already implemented this method in [circom-ecdsa](https://github.com/0xPARC/circom-ecdsa), why don't we just use that for our applications? Well, a 5 minute proving time that only works on high-end laptops is from from a viable UX. And generating proofs can't just be offloaded to the cloud -- for full privacy it's necessary these **proofs are computed on the user's device**. Otherwise you'll need to send your plaintext public key to a server to generate your proof, which grants it the power to breaking your anonymity if it wishes (short of _even_ fancier cryptography like [FHE](https://sprl.it/) or [MPC](https://eprint.iacr.org/2021/1530)). There's some half-server half-client models that have been deployed; we analyze those further in the [appendix](#appendix-half-client--half-server-models) of this post.

But we claim any solution with a server obscures the true unlock of zero-knowledge cryptography. Another framing of the importance of client-side proving is that it allows people to **generate authoritative claims of identity without any trusted party**, a power us measly humans didn't have until this technology started to become practical. This means the user can have full custody over their personal data instead of trusting other (potentially misaligned) institutions to manage and verify it.

But the straightforward ZK ECDSA construction is just too expensive for everyone to run on their devices, especially mobile phones. And as more of identity moves on-chain with innovations in [DeSoc](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4105763) and [SBTs](https://www.radicalxchange.org/concepts/social-identity/), **private ECDSA group membership will be an increasingly important tool** in maintaining anonymity and owning more of our identity. And so starting with the ideas in this [post](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629), we've been on a journey to keep bringing the proving cost down until all devices can easily generate a proof.

# Key insights

## Take computation out of the SNARK

Our first insight was that parts of the ECDSA signature verification equation could be taken out of the SNARK without sacrificing privacy. To see this, we'll review how ECDSA signatures work. Using similar notation to the [ECDSA wikipedia article](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Signature_verification_algorithm), we have the following terms:

{{<table "table table-striped table-bordered">}}
| Term | Definition |
| :----: | :----: |
| $G$ | generator |
| $n$ | order of group |
| $k$ | random number $\lt n$ |
| $R$ | $k * G$ |
| $r$ | x-coordinate of $R$ |
| $a$ | private key |
| $Qa$ | public key defined as $a * G$ |
| $m$ | hash of message |
| $s$ | $k^{-1} (m + a) \mod n$ |
{{</table>}}

An ECDSA signature from public key $Qa$ on the hashed message $m$ is the pair $(r, s)$. The verification equation[^1] checks if:

$$
R = ms^{-1} * G + rs^{-1} * Qa
$$

which you can see in 0xPARC's [original implementation](https://github.com/0xPARC/circom-ecdsa/blob/d87eb7068cb35c951187093abe966275c1839ead/circuits/ecdsa.circom#L129). It's a good exercise to check [correctness](https://en.wikipedia.org/wiki/Digital_signature#Definition) of this signature, we recommend you do that for better understanding! For simplicity, we rewrite this as

$$
s * R = m * G + r * Qa
$$

If we look carefully at the definition of $R$, we see it is just a random element on the curve [^2]. And so moving $R$ and thus $r$ outside of the SNARK shouldn't reveal anything about our user's public key. We further rewrite the equation as

$$
\begin{aligned}
s * R - m * G &=  r * Qa \\\
r^{-1}s * R - r^{-1}m * G &= Qa \\\
s(r^{-1} * R) - (r^{-1}m * G) &= Qa
\end{aligned}
$$

From this rewrite, we introduce two new terms:

{{<table "table table-striped table-bordered">}}
| Term | Definition |
| :----: | :----: |
| $T$ | $r^{-1} * R$ |
| $U$ | $-r^{-1}m * G$ |
{{</table>}}

which we substitute above to get

$$ s \* T + U = Qa $$

In the original method, we would've had to do a total of 2 field multiplies, 1 `PrivToPub`, 1 `SecpMultiply`, and 1 `SecpAdd` inside of the circuit. But because $T$ and $U$ are defined using $R$ and $m$, we can compute them outside of the SNARK. This means we only have to do 1 `SecpMultiply` and 1 `SecpAdd` inside of the circuit. Okay, 3 operations cut! Is this progress?

Unfortunately, this only cuts 100k constraints from the original 1.5mil constraint circuit, because `SecpMultiply` is by far the most expensive operation. This is implemented as `ecdsa_verify_no_precompute` in this [file](https://github.com/personaelabs/efficient-zk-ecdsa/blob/8477a39b5a3735724981cd99d19cf36ddb9e8c51/circuits/ecdsa_verify_no_precompute.circom) for reference. But hope is not lost yet! This rearranged equation is key for our next insight.

## Precomputing point multiples

The original 0xPARC code uses a clever optimization to speed up their `PrivToPub` subroutine, which multiples the generator point $G$ by a scalar $a$. Because $G$ is known in advance, we can store _precomputed multiples_ of $G$ directly in the circuit to reduce the number of operations in `PrivToPub`.

With the above rearrangement of the verification equation, we can do the same thing for $T$ to speed up the `SecpMultiply` between $s$ and $T$! However, because $T$ changes for every proof, we need to compute these inputs on the fly and pass them in as public inputs. Using the notation of the [0xPARC code](https://0xparc.org/blog/zk-ecdsa-2#:~:text=In%20our%20implementation), we determined a `stride` of 8 was most efficient, meaning we compute $(2^{8i} * j) * T$ for all $i \in [0, 31], j \in [0,255]$. Precomputing these multiples directly in JS is very slow, so we rewrote the cache computation in Rust and [compiled to WASM](https://github.com/personaelabs/efficient-zk-ecdsa-wasm) to majorly reduce the overhead when proving in browser.

This cache of points means we can skip a number of costly operations in normal `SecpMultiply`, bringing our total constraints to 163,239. This is a more than **9x** drop from the original circuit! The full logic is

- Public inputs: $U$ and precomputed $T$ multiples
- Private inputs: $s$, $Qa$
- Checks
  - Using zkSNARK
    $$
    s * T + U = Qa
    $$
  - Outside of the zkSNARK
    - Check that $U$ and and precomputed multiples of $T$ are correctly computed

which is implemented as `ecdsa_verify` [here](https://github.com/personaelabs/efficient-zk-ecdsa/blob/8477a39b5a3735724981cd99d19cf36ddb9e8c51/circuits/ecdsa_verify.circom) in circom.

## To keccak or not to keccak

We also include a version of the circuit called `ecdsa_verify_pubkey_to_eth_addr` that converts the public key to an address using Keccak, which is a total of 315,175 constraints. This is a more than **5x** drop from the old ECDSAVerify + PubkeyToAddress circuit. This circuit is important because many on-chain groups are of addresses not public keys, like airdrop lists.

However, if _all_ of the Ethereum addresses in a particular group have sent a transaction (or interacted with a smart contract) at least once, then we can use `ecdsa_verify` instead (which requires meaningfully fewer constraints than `ecdsa_verify_pubkey_to_eth_addr`). This is because we can extract the address's public key from the ECDSA signature of the transaction.

## Off-chain verification

Although verifying and storing the proof on Ethereum is good practice in terms of security and convenience, our method makes on-chain verification costly due to number of precomputed multiples we include. Therefore, the proofs from this model must be stored on “cheaper” storage, such as decentralized storage networks (e.g. IPFS, Arweave). From there, clients and servers can verify the proofs on their own, which is an acceptable solution for many non-financial privacy applications.

---

# Appendix: half client & half server models

There's an alternate construction pursued by other folks in the space that leads to more efficient proofs, but sacrifices full anonymity and requires a trusted server with an baby jub jub EdDSA public key $pk_s$. What in the world is a "baby jub jub EdDSA public key" and why do these constructions use it? We will break down each of these mystery words one by one.

**Baby jub jub** is an elliptic curve defined in this [EIP](https://eips.ethereum.org/EIPS/eip-2494) by Barry, Marta, and Jordi. It is a twisted Edwards curve, which means we can use the Edwards-curve Digital Signature Algorithm (**EdDSA**) instead of ECDSA. But the reason we use this random ass curve is because it has the _same base field as the scalar field of BN254_, which is the curve used in snarkjs Groth16. This means that all the circuit math happens in the "right field" for baby jub jub's elliptic curve operations, so verifying a baby jub jub EdDSA signature in a (snarkjs Groth16) circuit doesn't require any bigint math or range checks.

Users send their public key $pk_u$ and a signature $s_u$ to a trusted server, which verifies $s_u$ manually and then gives you $C$, which is a signature of $pk_u$ (and a nullifier $n$ usually, but we'll skip that in this description) using their EdDSA public key $pk_s$. This $C$ can be thought of a **certification** from the trusted server, and can be used to efficiently "prove" you know the private key for $pk_u$ (prove is in quotation marks because it isn't really a _proof of knowledge_ in the cryptographic sense, you're just trusting a third party).

On the client-side, we use a circuit that privately inputs your $pk_u$ and the server's signature $C$, and then publicly inputs the server's public key $pk_s$ and the members of the group $G$. The circuit logic checks $C$ is indeed a signature of $pk_u$ by $pk_s$ (which is efficient af in a circuit, due to the above logic!) AND that $pk_u$ is in $G$.

This is really clever for a few reasons! After the user signs up, each subsequent proof of membership is very efficient due to baby jub jub EdDSA being SNARK-friendly. And adding a nullifier $n$ to $C$ allows for each member to have a unique private ID, which isn't possible with ECDSA out of the box (but we have [research](https://eprint.iacr.org/2022/1255.pdf) that fixes this and is being deployed soon!).

But the efficiency also comes with drawbacks, which may or may not be dealbreakers for certain use-cases. First, the trusted server can falsely sign public keys and give the corresponding $C$ to anyone it wants to "prove" ownership. This can be avoided if the server does the verification of the signature and computation of $C$ inside a circuit to prove it was given a valid signature for the public key signed in $C$.

Second, the trusted server knows the set of addresses it has given a badge to. This means if people use their badge to anonymously speak or vote, they don't speak with the full anonymity of their group -- they speak under the anonymity set of addresses that have received a badge from the server. The server can keep this sign-up set private (meaning they can doxx) or public (meaning the first few sign-ups have a small anonymity set).

Third, this method requires users to get a certification for every group they are in before doing any anonymous signalling. From a UX standpoint, this is an costly extra step that can be removed with faster ECDSA signature verification proofs.

As a result, we don't see this being the correct long-term solution, and are committed to the hard work of making these proofs purely client-side. But we appreciate this work and its impact on the space!

<!-- Footnotes themselves at the bottom. -->

[^1]: The actual verification only checks that the x coordinate of the RHS is equal to $r$. However, if the verifier is given $R$ then checking the full equation is equivalent.
[^2]: In [deterministic ECDSA](https://datatracker.ietf.org/doc/html/rfc6979#section-3.2), $k$ isn't fully random and is roughly derived from a hash of the user's private key. Revealing $R$ generated deterministically is still secure in our method, which we analyze [here](https://hackmd.io/HQZxucnhSGKT_VfNwB6wOw#Does-R-leak-anything-about-s-and-Q_a).
