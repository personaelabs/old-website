---
title: "Efficient ZK ECDSA (part 1)"
date: 2022-11-30T22:12:03.284Z
authors: ["Personae"]
type: posts
draft: false
slug: "efficient-ecdsa-1"
category: "5 mins read"
tags: ["cryptography"]
description: "Reviewing a construction to reduce constraints for private ECDSA signature verification"
math: true
---

<div id="spacing">

This is the first post of a two (maybe three?) part series on our research improving private ECDSA signature verification, stemming from this [ETHResearch post](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629) and implemented in this [repository](https://github.com/personaelabs/efficient-zk-ecdsa). In this post, we motivate our research and review our key insights. In the follow-up, we will provide a full security proof, discuss on-chain extensions, and introduce even faster implementations in proving systems like Halo2 and Nova.

There will be some math in this post! It might look scary! But the key insights of the method are simple and should teach you some fun cryptography.

{{< toc >}}

# Motivation

## ECDSA & ring signatures

The **Elliptic Curve Digital Signature Algorithm**, or [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Signature_verification_algorithm) for short, is a key tool in the identity layer of blockchains like Ethereum and Bitcoin. In particular, each address on these chains is the hash of an ECDSA public key, which itself is a secp256k1 elliptic curve point. We sign all transactions with an ECDSA public key, which verifies we know the corresponding private key without revealing it.

Proving you know one of the private keys in a _group of public keys_ without revealing it is a necessary primitive for many of the anonymous speech applications Personae is interested in ([heyanon!](https://heyanon.xyz), [storyform](https://storyform.xyz), [HeyAnoun](https://heyanoun.xyz)). Surprisingly, this simple extension requires much heavier cryptographic machinery than a normal signature; there's an entire field of research dedicated to this problem called [**ring signatures**](https://en.wikipedia.org/wiki/Ring_signature). Unfortunately these methods aren't compatible with Ethereum/Bitcoin's ECDSA keys out of the box, so we opt to use an even more overpowered tool in zkSNARKs.

The zkSNARK method, first [implemented by 0xPARC](https://github.com/0xPARC/circom-ecdsa), privately inputs a group member's public key $pk$ and a signature $s$, and publicly inputs the entire group $G$ (usually succinctly as a [Merkle root](https://decentralizedthoughts.github.io/2020-12-22-what-is-a-merkle-tree/)). The circuit verifies $s$ was generated from $pk$ AND that $pk$ is in the group $G$. Sounds simple enough!

Unfortunately, the math for the signature verification is very SNARK-unfriendly. In particular, there's a lot of "wrong field arithmetic" --- the necessary operations happen over the [_base field_](https://zcash.github.io/halo2/background/curves.html#cycles-of-curves) of the secp256k1 curve which is different from the [_scalar field_](https://zcash.github.io/halo2/background/curves.html#cycles-of-curves) of the curves used in snarkjs. If that means nothing to you, don't worry -- it just means we have a huge blow-up in constraints to make sure all of our elliptic curve math is done correctly. And if that also means nothing to you, don't worry -- it just means computing a proof is very computation and memory-intensive, requiring ~1GB of proof metadata and ~5 minutes of browser computation on a MacBook.

## Client-side proving

Okay, so if our gigachad friends at 0xPARC already implemented this method, why don't we just use that for our applications? Well, a 5 minute proving time that only works on high-end laptops is far from a viable UX. And generating proofs can't just be offloaded to the cloud -- for full privacy it's necessary these **proofs are computed on the user's device**. Otherwise you'll need to send your plaintext public key to a server to generate your proof, which grants it the power to break your anonymity if it wishes[^1]. There's some half-server half-client models that have been deployed; we analyze those further in the [appendix](#appendix-half-client--half-server-models) of this post.

But we claim any solution with a server obscures the true unlock of zero-knowledge cryptography. Another framing of the importance of client-side proving is that it allows people to **generate authoritative claims of identity without any trusted party**, a power us measly humans didn't have until this technology started to become practical. This means the user can have full custody over their personal data instead of trusting other (potentially misaligned) institutions to manage and verify it.

But the straightforward ZK ECDSA construction is just too expensive for everyone to run on their devices, especially mobile phones. And as more of identity moves on-chain with innovations in [DeSoc](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4105763) and [SBTs](https://www.radicalxchange.org/concepts/social-identity/), **private ECDSA group membership will be an increasingly important tool** in maintaining anonymity and owning more of our identity. And so starting with the ideas in this [post](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629), we've been on a journey to keep bringing the proving cost down until all devices can easily generate a proof.

# Notation

ECDSA signature verification has a bunch of moving parts. All operations are on elliptic curve points and happen modulo $n$, but you can mostly ignore those details for the sake of this post. Here are each of the variables we'll need:

{{<table "table table-striped table-bordered">}}
| Term | Definition |
| :----: | :----: |
| $G$ | secp256k1 generator |
| $n$ | order of group |
| $k$ | random number $\lt n$ |
| $R$ | $k * G$ |
| $r$ | x-coordinate of $R$ |
| $a$ | private key |
| $Q_a$ | public key defined as $a * G$ |
| $m$ | hash of message |
| $s$ | $k^{-1} (m + ar) \mod n$ |
| $s^{-1}$ | $k (m + ar)^{-1} \mod n$ |
{{</table>}}

Here are specific `circom` functions from 0xPARC's `circom-ecdsa` that we reference:

{{<table "table table-striped table-bordered">}}
| Function | Inputs | Description |
| :----: | :----: | :----: |
| `BigMultModP` | `a, b, p` | compute $a * b \mod p$
| `ECDSAPrivToPub` | `privkey` | multiply $G$ by `privkey` |
| `Secp256k1AddUnequal` | `a, b` | add secp points `a` and `b` |
| `Secp256k1ScalarMult` | `scalar, point` | multiply `point` by `scalar` |
{{</table>}}

An ECDSA signature from public key $Q_a$ on the hashed message $m$ is the pair $(r, s)$. The verification equation[^2] checks if:

$$
R = ms^{-1} * G + rs^{-1} * Q_a
$$

which you can follow in 0xPARC's [original implementation](https://github.com/0xPARC/circom-ecdsa/blob/d87eb7068cb35c951187093abe966275c1839ead/circuits/ecdsa.circom#L129). To get better intuition for this signature algorithm, we recommend you check [correctness](https://en.wikipedia.org/wiki/Digital_signature#Definition) -- that is, verify this equation will pass for a valid signature $(r, s)$!

If you follow the original code and/or work through the above equation, you'll see that we need a total of 2 `BigMultModP`, 1 `ECDSAPrivToPub`, 1 `Secp256k1ScalarMult`, and 1 `Secp256k1AddUnequal` inside of the circuit, for a total of 1,508,136 constraints. We want to remove/optimize as many of these functions as possible!

# Key insights

## Take computation out of the SNARK

Our first insight was that parts of the ECDSA signature verification equation could be taken out of the SNARK without sacrificing privacy. For simplicity we first rewrite the signature verification equation as

$$
s * R = m * G + r * Q_a
$$

If we look carefully at the definition of $R$, we see it is just a random element on the curve. And so moving $R$ and thus $r$ outside of the SNARK shouldn't reveal anything about our user's public key [^3]. We further rewrite the equation as

$$
\begin{aligned}
s * R - m * G &=  r * Q_a \\\
r^{-1}s * R - r^{-1}m * G &= Q_a \\\
s(r^{-1} * R) - (r^{-1}m * G) &= Q_a
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

$$ s \* T + U = Q_a $$

Because $T$ and $U$ are defined using $R$ and $m$, we can compute them outside of the SNARK and pass them in as public inputs. And so with this rewrite we only need to do 1 `Secp256k1ScalarMult` and 1 `Secp256k1AddUnequal` inside of the circuit. Okay, 3 operations cut! Is this progress?

Unfortunately, this only cuts 100k constraints from the original 1.5mil constraint circuit, because `Secp256k1ScalarMult` is by far the most expensive operation. This is implemented as `ecdsa_verify_no_precompute` in this [file](https://github.com/personaelabs/efficient-zk-ecdsa/blob/8477a39b5a3735724981cd99d19cf36ddb9e8c51/circuits/ecdsa_verify_no_precompute.circom) for reference. But hope is not lost yet! This rearranged equation is key for our next insight.

## Precomputing point multiples

The original 0xPARC code uses a clever optimization to speed up the `ECDSAPrivToPub` subroutine, which multiples the generator point $G$ by a scalar $a$. Because $G$ is known in advance, the code stores _precomputed multiples_ of $G$ directly in the circuit to reduce the number of operations in `ECDSAPrivToPub`. The [0xPARC blog post](https://0xparc.org/blog/zk-ecdsa-2) dives into this in more detail.

As $T$ is revealed publicly in our rearranged equation, we should be able to do the same technique on $T$ to speed up the `Secp256k1ScalarMult` between $s$ and $T$! However, because $T$ changes for every proof, we need to compute these multiples on the fly and pass them in as public inputs. Using the notation of the [0xPARC code](https://0xparc.org/blog/zk-ecdsa-2#:~:text=In%20our%20implementation), we determined a `stride` of 8 was most efficient, meaning we compute $(2^{8i} * j) * T$ for all $i \in [0, 31], j \in [0,255]$. Precomputing these multiples directly in JS is very slow, so we rewrote the cache computation in Rust and [compiled to WASM](https://github.com/personaelabs/efficient-zk-ecdsa-wasm) to majorly reduce the overhead when proving in browser.

This cache of points means we can skip a number of costly operations in normal `Secp256k1ScalarMult`, bringing our total constraints to 163,239. This is a more than **9x** drop from the original circuit! The full method is:

- Public inputs: $U$ and precomputed $T$ multiples
- Private inputs: $s$, $Q_a$
- Logic
  - Inside zkSNARK circuit
    $$
    s * T + U = Q_a
    $$
  - Outside of the zkSNARK
    - Compute $U$ and precomputed multiples of $T$ in WASM

which is implemented as `ecdsa_verify` [here](https://github.com/personaelabs/efficient-zk-ecdsa/blob/8477a39b5a3735724981cd99d19cf36ddb9e8c51/circuits/ecdsa_verify.circom) in circom.

## Off-chain verification

Although verifying and storing the proof on Ethereum is good practice in terms of security and convenience, our method makes on-chain verification costly due to number of precomputed multiples we include. Therefore, the proofs from this model must be stored on “cheaper” storage, such as decentralized storage networks (e.g. IPFS, Arweave). From there, clients and servers can verify the proofs on their own, which is an acceptable solution for many non-financial privacy applications. We implement this solution in [heyanoun](https://heyanoun.xyz).

The easiest way to make the method on-chain friendly is to reduce the number of precomputed multiples, which decreases the input size but increases the proving time. Another method is to SNARK-ify the "outside of zkSNARK" checks and then link those checks to the original circuit. This requires the original circuit to privately input the precomputed multiples of $T$, but publicly output a hash $h_1$ of all of them. You then have another circuit (which can be computed server-side!) that internally computes the correct multiples of $T$ one by one and also publicly outputs a hash $h2$ of all of them. Then, your verifier or smart contract must input both of these proofs and verify that $h1 == h2$!

## To keccak or not to keccak

We also include a version of the circuit called `ecdsa_verify_pubkey_to_eth_addr` that converts the public key to an address using Keccak, which is a total of 315,175 constraints. This is a more than **5x** drop from the old ECDSAVerify + PubkeyToAddress circuit. This circuit is important because many on-chain groups are of addresses not public keys, like airdrop lists.

However, if _all_ of the Ethereum addresses in a particular group have sent a transaction (or interacted with a smart contract) at least once, then we can use `ecdsa_verify` instead (which requires meaningfully fewer constraints than `ecdsa_verify_pubkey_to_eth_addr`). This is because we can extract the address's public key from the ECDSA signature of the transaction.

# Next up

In this post, we reviewed the math behind [`efficient-zk-ecdsa`](https://github.com/personaelabs/efficient-zk-ecdsa), which improves client-side proving of private ECDSA signature verification and thus private ECDSA group membership. The key insights require no math past high school Algebra I, yet are meaningful enough to make private group membership proving significantly smoother for our users. If you've made it to the end, hopefully you understand ECDSA better and are motivated to dig deeper into the beautiful world of _cryptography_ (we mean cryptography specifically, not its bastardization in "crypto"! two very different things! sorry to be that person!).

Next post is gonna be a banger fr. In collaboration with [Geometry](https://www.geometryresearch.xyz/), we'll first present a full security proof of the `efficient-zk-ecdsa method` (HackMD sketch [here](https://hackmd.io/HQZxucnhSGKT_VfNwB6wOw), from Nico of Geometry). We'll then present an alternate construction for on-chain verification using the ideas in this [post](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629/5).

But the meat of the post will be comparing implementations of private group membership in new proving [backends](https://a16zcrypto.com/measuring-snark-performance-frontends-backends-and-the-future/#:~:text=separating%20frontends%20and%20backends) like Halo2, plonky2, Nova, and Spartan. Compared to our current backend Groth16, these new backends can more easily tradeoff proving and verification time, meaning we can have both prover-friendly and verifier-friendly versions of our circuits. The former is useful for client-side proving, and the latter for on-chain verification. And some backends use commitment schemes like IPA and FRI that avoid [EC pairings](https://vitalik.ca/general/2017/01/14/exploring_ecp.html) entirely, meaning we have more flexibility in choosing the finite fields of our circuits.

If you want to do cutting-edge ZK research focused on bringing client-side proving to millions or billions of consumer devices, please reach out to vivek@personaelabs.org and dan@personaelabs.org. We would love to support your work via research grants!

---

# Appendix: half client & half server models

There's an alternate construction pursued by other folks in the space that leads to more efficient proofs, but sacrifices full anonymity and requires a trusted server with an baby jub jub EdDSA public key $pk_s$. What in the world is a "baby jub jub EdDSA public key" and why do these constructions use it? We will break down each of these mystery words one by one. Disclaimer: we are not professionals at abstract algebra, just some people who like math and want to share what we've learned recently! Please reach out if there's any mistakes below!

## Elliptic curve interlude

**Baby jub jub** is an elliptic curve defined in this [EIP](https://eips.ethereum.org/EIPS/eip-2494) by Barry WhiteHat, Marta Bellés, and Jordi Baylina. It is a twisted Edwards curve, which means we can use the Edwards-curve Digital Signature Algorithm (**EdDSA**) instead of ECDSA. But the reason we use this random ass curve is because it has the _same base field as the scalar field of BN254_, which is the curve used in snarkjs Groth16.

The base field can be thought of "where the elliptic curve math happens", and the scalar field "where the circuit math happens". More precisely, let's define an elliptic curve $E_p$ where addition and multiplication is all done modulo a prime $p$, or in the finite field $\mathbb{F}_p$. The number of points on the curve in $\mathbb{F}_p$ will be a different value $q$, which is related to $p$ by [Hasse's wonderful theorem on elliptic curves](https://en.wikipedia.org/wiki/Hasse%27s_theorem_on_elliptic_curves) (wonderful is not in the original name, we added that). Roughly, the witness variables are encoded into these elliptic curve points, and because these points form a [group](https://en.wikipedia.org/wiki/Elliptic_curve#Elliptic_curves_over_finite_fields) all of the circuit math is done modulo $q$ or the finite field $\mathbb{F}_q$.

Going back to baby jub jub, as its base field (where its elliptic curve math happens, like signature verification) is the same as BN254's scalar field (where the snarkjs circuit math happens), then we can just add and multiply curve points in the circuit without bigint math or range checks! Don't worry if this doesn't fully make sense, we're just trying to give you intuition for why EdDSA on this specific curve is fast in snarkjs. It'll be more important to understand this for future ECDSA research posts.

## Proving setup

Users send their public key $pk_u$ and a signature $s_u$ to a trusted server, which verifies $s_u$ manually and then gives you $C_u$. This is a signature from the server's EdDSA public key $pk_s$ of $pk_u$ (and usually a _nullifier_ $n$, but we'll ignore that for now). This $C_u$ can be thought of a **certification** from the trusted server, and can be used to efficiently "prove" you know the private key for $pk_u$. Prove is in quotation marks because this isn't a real _proof of knowledge_ in the cryptographic sense, you're just trusting a third party.

On the client-side, we use a circuit that privately inputs your $pk_u$ and the server's certification $C_u$, and then publicly inputs the server's public key $pk_s$ and the members of the group $G$. The circuit logic checks $C_u$ is indeed a signature of $pk_u$ by $pk_s$ AND that $pk_u$ is in $G$.

## Praise & criticism

This is really clever for a few reasons! First, the nullifier $n$: we mostly ignored it above (because it's a rich topic on its own!) but including $n$ essentially gives each user an unlinkable private ID. This is _necessary_ for any 1-signal-per member application like voting or airdrops, but unfortunately cannot be easily generated for ECDSA keys. Second, after the user signs up, each subsequent proof of private group membership is quick due to baby jub jub EdDSA verification being SNARK-friendly (which the above explanation hopefully gave you intuition for!). And finally, this "certification" technique isn't just restricted to making groups of Ethereum public keys. If you're okay trusting third parties, you can also create groups by logging in with web2 platforms like Twitter and GitHub and having the server verify your login and giving you the relevant certification.

What does this construction give up in exchange for these benefits? As covered in the main post, our view is that the biggest concession _is requiring a server at all_. Only when proving is done client-side can we start to use ZK to move identity and computation away from other institutions and into our own hands. But does that meaningfully change a user's privacy? After all, doesn't each post-signup proof keep your identity private? No, because the trusted server knows the **set of addresses and accounts it has given a certification to**. This means if people use their certification to anonymously speak or vote, they don't speak with the full anonymity of their group -- they speak under the anonymity set of addresses that have signed up on the server. The server can keep this sign-up set private (meaning they can doxx when the group is small) or public (meaning the first sign-ups have a small anonymity set). This works for lower-stakes applications, but a dealbreaker for any high-profile or sensitive groups.

In addition, we're trusting the server to correctly assign certifications to users, but these can be maliciously forged in the current model. In the case of Ethereum groups, the trusted server can maliciously sign a public key $pk_m$ and give the corresponding $C_m$ to anyone who wants to "prove" ownership of $pk_m$. This can be avoided if the server does the verification of the signature and computation of $C_m$ inside a zkSNARK to prove it was _actually_ given a valid signature from $pk_m$. But for non-Ethereum groups there isn't an easy way to SNARKify the validity, meaning it's fair game for the server or hacker to forge.

## Looking to the future

How necessary are the other benefits in the long-term? For the nullifier: although there is _currently_ no way to generate a nullifier from our ECDSA keys, a collaboration between [Personae and Geometry](https://eprint.iacr.org/2022/1255) has solved this problem and has implementations ready to be [deployed in wallets soon!](https://ethbogota-2022.netlify.app/) For the faster "subsequent" proofs, this is no longer an issue if ECDSA signature verification is fast enough to be done on client-side devices. These proofs also add a UX hurdle of getting and storing a certification for every address you want to prove properties about.

Due to these concerns, we don't see this approach being the long-term solution. Pure client-side proving is technically the most secure and aesthetically the cleanest, so we will continue to pursue solutions in that realm. But we deeply appreciate this creativity behind this method and its impact on the space!

<!-- Footnotes themselves at the bottom. -->

[^1]: You can technically use _even_ fancier cryptography like [MPC](https://eprint.iacr.org/2021/1530) or [FHE](https://sprl.it/) to avoid sending a plaintext public-key, but these methods are still in development.
[^2]: The actual verification only checks that $r$ is equal to the x coordinate of the RHS. This leads to $(r, -s)$ also being a valid signature. However, if the verifier is given $R$ then checking the full equation is equivalent.
[^3]: In [deterministic ECDSA](https://datatracker.ietf.org/doc/html/rfc6979#section-3.2), $k$ isn't fully random and is roughly derived from a hash of the user's private key. Revealing $R$ generated deterministically is still secure in our method, which we analyze [here](https://hackmd.io/HQZxucnhSGKT_VfNwB6wOw#Does-R-leak-anything-about-s-and-Q_a).

</div>
