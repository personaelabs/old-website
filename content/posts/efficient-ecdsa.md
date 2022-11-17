---
title: "Efficient private ECDSA signature verification"
date: 2022-11-10T22:12:03.284Z
authors: ["Dan Tehrani and vivboop"]
type: posts
draft: false
slug: "efficient-ecdsa"
category: "5 mins read"
tags: ["cryptography"]
description: "Reviewing a construction to reduce constraints for private ECDSA signature verification"
math: true
---

---

# Outline for Post

Link to original: https://www.notion.so/personae-labs/zkSNARK-efficient-ECDSA-blog-post-bb3957d7e5534155b73e5c07aae443ea

## Intro

1. Introduce ECDSA (identity layer of Ethereum/Bitcoin, secp256k1, verifying ECDSA signature -> verifies ownership of address)
2. If you want to act pseudonymously as a member of an on-chain group, need to wrap signature verify in a zkSNARK (along with a merkle tree check, explain that ECDSA is snark-unfriendly)
3. If user generates these proofs on their device, no one can doxx (no middleman in this process, user owns their data, briefly introduce alternate models from Sismo/BWL/Semaphore)
4. We also don't always need on-chain verification for interesting apps (introduce heyanon/storyform, verification cost can be larger)
5. Given this setup, can we move some computation outside of the SNARK to save proving time?

## Rearranging the equation

Go through version of circuit that doesn't involve any precomputed multiples and just uses SecpMultiply. We should also add this circuit to codebase to show how much precomputation helps. Also I'm not actually 100% sure what the savings are lol.

## Pre-computing multiples

Explain how precomputing multiples can speed up the multiplication. Different from circom-ecdsa where they just precomputed multiples of the generator, we precompute multiples of a non-fixed point using a seperate WASM binary

## Can we still verify on-chain?

same as below

## To keccak or not to keccak

Should note that on-chain groups can sometimes be all addresses (for example, an airdrop) but if everyone in the on-chain group has made a transaction (for example, a group of people that bought an SBT) means you can extract the public key and can make the group directly with public keys. Thus, you can skip the costly keccak operation.

## From here

same as below

---

<!-- ### On-chain groups

In privacy applications like [Tornado Cash](https://github.com/tornadocash), [Heyanon](https://heyanon.xyz/), and [Storyform](http://storyform.xyz/), users are able to prove statements without revealing their identities. In private coins like Tornado Cash, you can prove that you own a private key that corresponds to a particular UTXO in the UTXO Merkle tree, without revealing which UTXO you are trying to spend. And in Heyanon and Storyform, you can broadcast a message alongside a zero-knowledge proof that you belong to a particular group of people (e.g. you own a Devcon Bogota POAP).

ECDSA is one of the most popular digital signature algorithms in the blockchain space, with both Ethereum and Bitcoin using ECDSA on the secp256k1 curve. These ECDSA public keys form part of "identity layer" of web3, and

For example, Ethereum and Bitcoin both use ECDSA on the secp256k1 curve, meaning ECDSA compatibility is crucial for integrating with the existing identity layer of major blockchains today.

Unfortunately, zkSNARKs and ECDSA do not pair well. More precisely, a zkSNARK that verifies ECDSA signatures requires an expensive implementation for full correctness, and as a result is not currently practical for privacy applications. We will not go in-depth about why that is the case in this blog post, but you can read more about it [here](https://0xparc.org/blog/zk-ecdsa-2).

### Privacy applications and ECDSA

In privacy applications like [Tornado Cash](https://github.com/tornadocash), [Heyanon](https://heyanon.xyz/), and [Storyform](http://storyform.xyz/), users are able to prove statements without revealing their identities. In private coins like Tornado Cash, you can prove that you own a private key that corresponds to a particular UTXO in the UTXO Merkle tree, without revealing which UTXO you are trying to spend. And in Heyanon and Storyform, you can broadcast a message alongside a zero-knowledge proof that you belong to a particular group of people (e.g. you own a Devcon Bogota POAP).

In a lot of cases, these privacy applications would want to leverage the existing ECDSA identity layer of blockchains like Ethereum. Although as said at the beginning of this blog post, zkSNARKs which is the foundation of these privacy applications, and ECDSA don’t work with each other well.

But as a result of [some iterations of ideas](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629), we have found that there are techniques that can be applied to make these two cryptographic primitives work together better. -->

## Efficient ECDSA signature verification inside a zkSNARK

A basic ECDSA signature verification circuit written in Circom results in an R1CS with approximately 1.5 million constraints, a proving key that is close to a GB large, a constant sized proof, and a proving time [that is non-trivial](https://github.com/0xPARC/circom-ecdsa#benchmarks). We propose a method that reduces the required R1CS constraints to 163,239 and a proving key size that is 119MB large, but at the cost of a larger proof with more public inputs.

This is accomplished by modifying the ECDSA verification to off-load computations from the zkSNARK.

The standard ECDSA verification equation of a signature $(r, s)$ is:

$$
s * R = m * G + r * Qa
$$

where

$$
R \rightarrow \text{point with $r$ as its x-coordinate} \\\
G \rightarrow \text{generator point of secp256k1} \\\
Qa \rightarrow \text{public key}
$$

We want to verify that the given $(r, s)$ is indeed a signature of $Qa$, but without knowing about $Qa$. At first, we attempted to

**Since $r$ doesn’t reveal anything about the public key, we can do the calculation pertaining to $r$ _outside_ of the zkSNARK.** We rewrite the equation as

$$
\begin{aligned}
r^{-1}s * R &= r^{-1}m * G + Qa \\\
r^{-1}s * R - r^{-1}m * G &= Qa \\\
s(r^{-1} * R) - (r^{-1}m * G) &= Qa
\end{aligned}
$$

The resulting zkSNARK will be

- Public inputs
  $$
  U = -(r^{-1} * m * G)
  $$
  - The powers of $r^{-1} * R$ (represented as $T$) to do scalar multiplication efficiently using the cached [windowed method](https://en.wikipedia.org/wiki/Elliptic_curve_point_multiplication#Windowed_method). (more on this below)
- Private inputs
  $$
  s
  $$
  $$
  Qa
  $$

* Checks
  - Using zkSNARK
    $$
    s * T + U = Qa
    $$
  * Outside of the zkSNARK
    - Check that U and T are correct

As a result, we are only required to do a single point scalar multiplication and a single point addition inside a zkSNARK.

_You can find the Circom implementation of this method [here](https://github.com/personaelabs/efficient-zk-sig)!_

**The cached windowed method**

Originally mentioned in [this blog post by 0xPARC](https://0xparc.org/blog/zk-ecdsa-2), the cached windowed method is a way to efficiently compute scalar multiplication inside a zkSNARK. In short, we precompute (i.e. cache) the multiples of the point that we want to operate on, and use the cache to aid the scalar multiplication inside the zkSNARK. In the method we propose, it is necessary to compute $(2^{i * stride} * j) * T$ for all $i \in 0,…,31, j \in 0,…,255$

where

- stride = 8
- $T = r^{-1} * R$)

which results in $2^8 * 32 = 8192$ elliptic curve points to pass as public inputs.

We made this cache computation [available in WASM](https://github.com/personaelabs/efficient-zk-sig-wasm-helper) to minimize the overhead when proving in a browser.

## Benchmarks

TBD

\*Paste the benchmarks in README

The reason for having two circuits `ecdsa_verify` and `ecdsa_verify_pubkey_to_eth_addr` is: in most cases ECDSA verification inside zkSNARK is combiend with a Merkle inclusion proof. Proving knowledge of a private key that corresponds to a particular public key without revealing the public key, is not really useful per see. There needs to be a Merkle tree that you want to prove that you are one of the leaves.

Since an Ethereum address is a keccak hash of an ECDSA public key, and it’s not possible to retrieve a public key only from an Ethereum address, we need to keccak hash the verified public key inside the zkSNARK. And check that the hash (i.e the Ethereum address) is part of a particular Merkle tree.

Although if _all_ Ethereum addresses in a particular group have sent a transaction (or interacted with a smart contract) at least once, then we can use `ecdsa_verify` (which requires meaningfully fewer constraints than `ecdsa_verify_pubkey_to_eth_addr`) to construct a zero-knowledge proof of membership scheme. This is because we can extract the public key which corresponds to an Ethereum address, from the ECDSA signature of the transaction.

## On-chain verification

Although verifying and storing the proof on Ethereum is a good practice in terms of security and convenience, because our method requires the _cache_ of $T$, which is a 4-dimensional vector with a total of 65536 elements, on-chain verification is eliminated as an option (with some nuance; more on this below).

Therefore it will be required to store the proofs on “cheaper” storage, such as decentralized storage networks (e.g. IPFS, Arweave), and lazily verify the proofs. Which would be an acceptable solution for non-financial privacy applications.

But there might be cases where you want to securely store and verify the proofs on Ethereum. We haven’t come up with a clear solution to achieve this, but there are ideas that we have considered.

### Check the hash of the cache

We can make the cache of $T$ a private input, and expose (as a public input) the hash of the cache. Then the verifier can check the hash against the _hash of the expected cache_.

However, hashing 65536 values inside a zkSNARK wouldn’t be cheap in terms of number of constraints. If there were a way to hash such a volume of values in a zkSNARK, we can downsize the public inputs, making it possible to store the entire proof on-chain.

### Prove the knowledge of $s$ by signing with it

This is a totally different method than the above method, and will likely take another blog post to explain it to the fullest. But this is another way that could realize on-chain verification. In short, there is a way to verify that $s$ (of the signature ($r$, $s$)) is correct, by **interpreting $s$ as a secret key**. Recall that the verification equation is $s * R = m * G + r * Qa$. Thus, we can interpret $s$ as the private key that corresponds to the public key $m * G + r * Qa$, with generator $R$! And proving the knowledge of $s$ is achievable by **signing with it**.

_More details about this method can be found in [this Ethresearch post](https://ethresear.ch/t/efficient-ecdsa-signature-verification-using-circom/13629)._

The properties of this method:

- Upside
  - Doesn't require the cache to be passed in as public input, making on-chain verification _somewhat_ feasible.
- Downsides
  - There will still be additional data that needs to be stored on-chain, which is used to verify the computation _outside_ of the zkSNARK. Storing those proofs will require non-trivial gas.
  - We need to check a keccak hash inside a zkSNARK, which requires approximately additional 150,000 constraints.

## Expanding to other instances of client-side proving

The technique of _off-loading computation from zkSNARK_ likely could be applied in other situations as well. But the extent of efficiency improvement, depends on the context. That is, as mentioned at the beginning, the reason why making ECDSA for zkSNARK faster is valuable, is that the current identity layers of major blockchains are built on ECDSA, **and most wallets only support signing with ECDSA.** If we are operating under different assumptions, (e.g. we can prove the knowledge of a private key by generating a proof directly using the private key i.e. prove that privKey \* G = pubKey), then we don’t need to use any methods in this blog post.

But if we are operating under a different restricted assumption (e.g. we can sign with _EdDSA_ but can’t directly use the private key when proving), then some of the heuristics in this blog post might come in handy.

## From here

We have implemented the method in Circom using Groth16 as the proving system, but with the advent of proving systems such as Halo2, we could make proving even more lightweight (we hope to release the Halo2 implementation soon!). The benchmarks above are measured on a Macbook Pro with an M1 Pro chip, and still require a proving time that is not pleasant in terms of UX. Making proving faster in high-end laptops is the first step, but realizing proving on mobile devices is another chasm.
