---
title: "Efficient ECDSA (part 2)"
date: 2022-11-10T22:12:03.284Z
authors: ["Personae"]
type: posts
draft: true
slug: "efficient-ecdsa-2"
category: "5 mins read"
tags: ["cryptography"]
description: "Second part"
math: true
---

## Things to cover

1. Full benchmarks
   - browserstack on many devices to get better numbers
2. Security proof
   - Finish a full write-up
   - Get it reviewed by others
3. On-chain extension
   - Original method, but you also check Keccak inside
   - Fewer precomputes for smaller input size
   - Hashing the precomputes
4. Halo2 implementation
   - KZG work (first halo2wrong, hype up axiom)
   - IPA with secq curve + ECFFT
5. Other research
   - https://hackmd.io/@HBBHZjW4TU-_rDF7GQh3lg/ryYwRzEGi

---

# Various stuff that didn't make the first post

## Off-chain privacy applications

Points to cover (could also go in Nouns post)

- more identity is moving on-chain but proofs themselves don't need to be on-chain
- the proof itself can be in decentralized storage and verified by clients and servers
- a verification server is necessary for posting to web2 platforms

Even though more identity is moving on-chain

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

## Groth16 notes

This research may be some of our final work with pure Groth16, as our research shifts fully to newer proving stacks using PLONKish arithmetizations and alternate backends like Halo2, Nova, and Spartan. To be clear, Groth16 isn't going away anytime soon; its constant verification time is just too good for on-chain verification. However, instead of compiling entire circuits to Groth16, we will compile using a different backend, and then verify the outputted proof in a Groth16 circuit. In other words, we likely start to use Groth16 as a _proof compressor_ rather than a full proving system.
