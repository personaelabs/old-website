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

This is uncommon in other popular elliptic curve based zkSNARK proof systems. For example, [Groth16](https://xn--2-umb.com/22/groth16/) and PSE's [fork of Halo2](https://github.com/privacy-scaling-explorations/halo2) use the KZG polynomial commitment which requires pairing-friendly curves like BN254. zcash's original [halo2](https://github.com/zcash/halo2) uses IPA and doesn't need pairing-friendly curves, but the scalar field of the curve must have [high 2-adicity](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/) for efficient and practical FFT.[^1]

The original implementation of SpartanNIZK uses the curve Curve25519 in its backend; in Spartan-ecdsa, we use the **_secq256k1 curve_**. This choice of curve allows us to perform **right-field arithmetic** for ECDSA verification, avoiding the costly range checks that have slowed down previous implementations.

## Right-field arithmetic

In zero-knowledge proving systems, the statements being proven are usually encoded in the [scalar field](https://zcash.github.io/halo2/background/curves.html#cycles-of-curves) of an elliptic curve. In ECDSA, these statements are mostly elements in $F_p$, the base field of the secp256k1 curve. In our previous attempt to implement zero-knowledge ECDSA verification using Groth16 with snarkjs, we needed to encode elements in $F_p$ to the scalar field of the BN254 curve, which is smaller than $F_p$. This is called wrong-field arithmetic and introduces an enormous number of constraints to ensure all math is being done correctly.

**However, in Spartan-ecdsa, we encode values in $F_p$ without doing any wrong-field arithmetic by using the secq256k1 curve** which is defined as follows.

$$
\begin{aligned}
y^2 = x^3 +  7 \mod (\mathrm{order\ of\ }F_q) \\\
F_q\mathrm{: scalar\ field\ of\ secp256k1}
\end{aligned}
$$

Importantly, secq256k1 has a scalar field order that is exactly the same size as $F_p$, which allows us to do most of the ECDSA arithmetic in **_right-field_**. Furthermore, not only we can do arithmetic in right-field , but also since secq256k1 forms a cycle with secp256k1, we can construct recursive proofs using the two curves; recursion allows us to combine multiple proofs into a single proof, without incurring additional costs on verification. This property will be important when we want to verify proofs on blockchains like Ethereum.[^2]

## Tricks from efficient-zk-ecdsa

Although we are now able to do right-field ECDSA verification by using SpartanNIZK and secq256k1, ECDSA verification consists not only of arithmetic in $F_p$ but also in $F_q$, which is the scalar field of secp256k1 (i.e. the base field of secq256k1).

However, using the technique from [our previous post](https://personaelabs.org/posts/efficient-ecdsa-1/), we can perform the operations in $F_q$ outside of the circuit without sacrificing privacy. By doing so, we are only left with statements that can be proven in the right-field!

## What we use

**Spartan**
We use a fork of Spartan that operates over the secq256k1 curve.

**Nova-Scotia**
[Nova-Scotia](https://github.com/nalinbhardwaj/Nova-Scotia) is a Circom R1CS compiler developed by [Nalin Bhardwaj](https://twitter.com/nibnalin). We use a fork of Nova-Scotia to compile Circom circuits into a binary format that Spartan can process. We slightly modify Nova-Scotia to be compatible with secq256k1.

## What we implement

We implement the optimized ECDSA verification circuit in Circom. The elliptic curve circuits were inspired by the implementation of the [Halo2 ECC gadget](https://zcash.github.io/halo2/design/gadgets/ecc).
We also provide a Typescript library that contains Spartan-ecdsa compiled to wasm, and an easy-to-use interface to prove ECDSA verification and membership in a Merkle tree of ECDSA public keys.

### Benchmarks

ECDSA verification with Spartan-ecdsa **only requires [num] constraints, and can be proven under Xs in-browser** on a Macbook pro.

TBD

## Future work

Although Spartan-ecdsa made a significant leap in prover efficiency for ECDSA verification, the proofs are too large to be verified on blockchains like Ethereum. In order to do on-chain verification, the proofs need to be further compressed. Fortunately, there is a significant amount of open-source effort being made toward on-chain verifiable proof generation for the purpose of realizing zkEVMs. We imagine Spartan-ecdsa taking advantage of these efforts to make on-chain proof verification a possibility. Furthermore, as mentioned above, we can also use recursion to combine proofs to make per-proof cost verification lower and to make on-chain verification more affordable.

**Credit roll**

Thanks for reading! Names within each group are sorted alphabetically by first name.

- Contributors: Dan Tehrani, Lakshman Sankar, Vivek Bhupatiraju
- Post writers: Dan Tehrani
- Post reviewers: Vivek Bhupatiraju
- Upstream work: Nalin Bhardwaj, Srinath Setty

[^1]: If your field does not have high 2-adicity, you can use [ECFFT](https://eprint.iacr.org/2022/1542) as introduced by StarkWare. It is slower in complexity than raw FFT but still far more efficient than naive $O(n^2)$ algorithms. A more gentle overview of the algorithm is provided by William Borgeaud [here](https://solvable.group/posts/ecfft/).
[^2]: It is important to note that unlike the [Pasta curves used in Halo2](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/),the secp/secq curve cycle is not FFT friendly. Therefore it is not compatible per se with most of the FFT-based recursive systems unless ECFFTs can be used.
