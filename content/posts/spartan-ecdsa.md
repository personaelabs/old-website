---
title: "Introducing Spartan-ecdsa"
date: 2023-01-14T00:00:00Z
authors: ["Personae"]
type: posts
draft: false
slug: "spartan-ecdsa"
category: "5 mins read"
tags: ["cryptography"]
description: "Introducing Spartan-ecdsa"
math: true
---

We introduce [Spartan-ecdsa](https://github.com/personaelabs/spartan-ecdsa), which to our knowledge is the fastest open-source method to verify secp256k1 ECDSA signatures in zero-knowledge.

{{< toc >}}

Spartan-ecdsa is based on the [Spartan Non-Interactive Zero-knowledge Proof (SpartanNIZK)](https://eprint.iacr.org/2019/550), which is a proving system in the Spartan zkSNARKs family. Spartan is a family of prover and verifier efficient zkSNARKs for proving R1CS satisfiability, developed by [Srinath Setty](https://twitter.com/srinathtv). SpartanNIZK doesn’t require a trusted setup, and most importantly in the case of ECDSA verification, **can work with on elliptic curve where discrete log holds**.

This is uncommon in other popular elliptic curve based zkSNARK proof systems. For example, [Groth16](https://xn--2-umb.com/22/groth16/) and PSE's [fork of Halo2](https://github.com/privacy-scaling-explorations/halo2) require pairing-friendly curves like BN254. Zcash's original [Halo2](https://github.com/zcash/halo2) uses IPA and doesn't need pairing-friendly curves, but the scalar field of the curve must have [high 2-adicity](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/) for efficient and practical FFT.[^1]

The original implementation of SpartanNIZK uses the curve Curve25519 in its backend; in Spartan-ecdsa, we use the **_secq256k1 curve_**. This choice of curve allows us to perform **right-field arithmetic** for ECDSA verification, avoiding the costly range checks that have slowed down previous implementations.

# Right-field arithmetic

In zero-knowledge proving systems, the statements being proven are usually encoded in the [scalar field](https://zcash.github.io/halo2/background/curves.html#cycles-of-curves) of an elliptic curve. In ECDSA, these statements are mostly elements in $F_p$, the base field of the secp256k1 curve. In our previous attempt to implement zero-knowledge ECDSA verification using Groth16 with snarkjs, we needed to encode elements in $F_p$ to the scalar field of the BN254 curve, which is smaller than $F_p$. This is called wrong-field arithmetic and introduces an enormous number of constraints to ensure all math is being done correctly.

**However, in Spartan-ecdsa, we encode values in $F_p$ without doing any wrong-field arithmetic by using the secq256k1 curve** which is defined as follows.

$$
\begin{aligned}
y^2 = x^3 +  7 \mod (\mathrm{order\ of\ }F_q) \\\
F_q\mathrm{: scalar\ field\ of\ secp256k1}
\end{aligned}
$$

_The Sage code in [appendix 2](#appendix-2-secp--secq-cycle-in-sage) demonstrates how secp256k1 and secq256k1 relate._

Importantly, secq256k1 has a scalar field order that is exactly the same size as $F_p$, which allows us to do most of the ECDSA arithmetic in **_right-field_**. Furthermore, not only we can do arithmetic in right-field , but also since secq256k1 forms a cycle with secp256k1, we can construct recursive proofs using the two curves; recursion allows us to combine multiple proofs into a single proof, without incurring additional costs on verification. This property will be important for building applications like [ETHDos](https://ethdos.xyz/), and also for verifying proofs on blockchains like Ethereum.[^2]

## Tricks from efficient-zk-ecdsa

Although we are now able to do right-field ECDSA verification by using SpartanNIZK and secq256k1, ECDSA verification consists not only of arithmetic in $F_p$ but also in $F_q$, which is the scalar field of secp256k1 (i.e. the base field of secq256k1).

However, using the technique from [our previous post](https://personaelabs.org/posts/efficient-ecdsa-1/), we can perform the operations in $F_q$ outside of the circuit without sacrificing privacy. By doing so, we are only left with statements that can be proven in the right-field!

# Benchmarks

## Public key group membership proof

In the Spartan-ecdsa Typescript library, we provide an interface to prove membership in a group of ECDSA public keys, without revealing any information about the proven public key. The approximate benchmark of this proving method is as follows.

{{<table "table table-condensed">}}
| Benchmark | # |
| :----: | :----: |
| Constraints | 8,076 |
| Proving time | 4 seconds |
| Proof size | 16kb |
| Verification time | tbd |
{{</table>}}
_Measured on a M1 MacBook Pro with 80Mbps internet speed_

The proving time has improved **by 10x** from [our previous implementation](https://github.com/personaelabs/efficient-zk-ecdsa). We believe that this magnitude of improvement will open the doors for more applications using membership proofs to be developed.

However, there are yet some difficulties in practice. Let's say you want to create an anonymous forum where you can only post if you own an NFT of a particular collection. First, the Ethereum addresses of the NFT owners must to collected to create a group, which the users can prove membership to. However, even though ownership of an NFT is a public record, **the ECDSA public keys of those addresses are not always accessible.** Since an Ethereum address is a hash of a public key, the public key is only accessible only if at least one transaction has been sent from that Ethereum address; if there is a transaction from the address, the public key can be extracted from the ECDSA signature of the transaction. While if the address has no transaction record, then the public key of that address won't be publically accessible information.

To counter this issue, in the case of our anonymous form example, **we can require the users to submit their public key when they sign up to the forum, only if the address of the user has never sent a transaction before** (i.e. if the public key of the address is inaccessible) (the user should send their public key using Tor for complete anonymity). The forum adds the submitted public key to the group, so the user can prove membership anonymously. However, **if the user creates a post immediately after they submit their public key, the post and the submitted public key can be linked.** So the forum should only allow the user to make their **first post after some time has passed** (e.g. 24h~), to protect the anonymity of the user.

## Ethereum address group membership proof

As shown in the above example, the inaccessibility of public keys requires us to implement a design that might not lead to the best user experience. To counteract this, instead of proving membership to a group of ECDSA public keys, we could prove membership to a group of Ethereum addresses. Spartan-ecdsa provides an interface to do this, which has the following approximate benchmark.

{{<table "table table-condensed">}}
| Benchmark | # |
| :----: | :----: |
| Constraints | 159,775 |
| Proving time | 60 seconds |
| Proof size | 38kb |
| Verification time | tbd |
{{</table>}}
_Measured on a M1 MacBook Pro with 80Mbps internet speed_

As the benchmark shows, proving membership to an Ethereum address group takes significantly longer. This is because we need to do a Keccak hash to convert a public key into an Ethereum address during proving.

In some applications, proving can be done in the background while the user does another action (e.g. entering long-form text), which will be a way to manage the long proving time. However, the underlying high cost of proving isn't the best foundation to build a great user experience.

In summary, Spartan-ecdsa provides two different proving methods with different tradeoffs, for application developers to choose.

# Future work

## Speeding up Keccak

Managing a group of Ethereum addresses instead of the inner public keys is easier, and will result in a better user experience. To bring down the proving time in such a setup, we need to reduce the number of Keccak constraints. One possibility is to use the lookup argument. Keccak consists of numerous bitwise XORs, and lookups allow us to arithmetize them in an efficient way.

## On-chain verification

Although Spartan-ecdsa made a significant leap in prover efficiency for ECDSA verification, the proofs are too large to be verified on blockchains like Ethereum. To do on-chain verification, the proofs need to be further compressed. Fortunately, there is a significant amount of open-source effort being made toward on-chain verifiable proof generation to realize zkEVMs. We imagine Spartan-ecdsa leveraging these efforts to make on-chain proof verification a possibility. Furthermore, as mentioned above, we can also use recursion to combine proofs to make per-proof cost verification lower and to make on-chain verification more affordable.

**Credit roll**

Thanks for reading! Names within each group are sorted alphabetically by first name.

- Contributors: Dan Tehrani, Lakshman Sankar, Vivek Bhupatiraju
- Post writers: Dan Tehrani
- Post reviewers: Vivek Bhupatiraju, Lakshman Sankar
- Upstream work: Nalin Bhardwaj, Srinath Setty

[^1]: If your field does not have high 2-adicity, you can use [ECFFT](https://eprint.iacr.org/2022/1542) as introduced by StarkWare. It is slower in complexity than raw FFT but still far more efficient than naive $O(n^2)$ algorithms. A more gentle overview of the algorithm is provided by William Borgeaud [here](https://solvable.group/posts/ecfft/).
[^2]: It is important to note that unlike the [Pasta curves used in Halo2](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/),the secp/secq curve cycle is not FFT friendly. Therefore it is not compatible per se with most of the FFT-based recursive systems unless ECFFTs can be used.

---

# Appendix 1: Under the hood

_The following is an incomplete list of the components which comprise Spartan-ecdsa_.

## Upstream dependencies

**Spartan**

We use a fork of Spartan that operates over the secq256k1 curve.

**Circom**

Circom is a language for defining arithmetic circuits. We use a fork of Circom that arithmetize over the secp256k1 base field (i.e. secq256k1 scalar field).

**Nova-Scotia**

[Nova-Scotia](https://github.com/nalinbhardwaj/Nova-Scotia) is a Circom R1CS compiler developed by [Nalin Bhardwaj](https://twitter.com/nibnalin). We use a fork of Nova-Scotia to compile Circom circuits into a binary format that Spartan can process. We slightly modify Nova-Scotia to be compatible with secq256k1.

## Our implementations

**Secp256k1 elliptic curve operations in Circom**

The Secp256k1 elliptic curve circuits were inspired by the implementation of the [Halo2 ECC gadget](https://zcash.github.io/halo2/design/gadgets/ecc).

**Secq256k1 hash-to-curve**
We implement hash-to-curve for the secq256k1 curve, following the specification of [draft-irtf-cfrg-hash-to-curve-10](https://www.ietf.org/archive/id/draft-irtf-cfrg-hash-to-curve-10.html). hash-to-curve is used for generating secure Pedersen commitment generators, which are used in Spartan.

**Poseidon**
[Poseidon](_https://eprint.iacr.org/2019/458.pdf_) is a zk-friendly hash function. We implement and instantiate Poseidon which hashes Secp256k1 field elements. The implementation was inspired by [Neptune](https://github.com/filecoin-project/neptune), a Poseidon implementation from [Filecoin](filecoin.io).

# Appendix 2: Secp/Secq cycle in Sage

{{< highlight "background-color: rgb(5, 5, 5)" >}}
p = 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f
q = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141

# Secp256k1

P = GF(p)
aP = P(0x0000000000000000000000000000000000000000000000000000000000000000)
bP = P(0x0000000000000000000000000000000000000000000000000000000000000007)
Secp256k1 = EllipticCurve(P, (aP, bP))
Secp256k1.set_order(q)

# Secq256k1

Q = GF(q)
aQ = P(0x0000000000000000000000000000000000000000000000000000000000000000)
bQ = P(0x0000000000000000000000000000000000000000000000000000000000000007)
Secq256k1 = EllipticCurve(Q, (aQ, bQ))
Secq256k1.set_order(p)

print(
"Secp256k1 group order == Secq256k1 base field order:",
Secp256k1.order() == Secq256k1.base_field().cardinality()
)

print(
"Secp256k1 base field order == Secq256k1 group order:",
Secp256k1.base_field().cardinality() == Secq256k1.order()
)

{{< /highlight >}}
