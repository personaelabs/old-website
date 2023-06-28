---
title: "Noun Nyms: Technical challenges"
date: 2023-06-28T00:00:00Z
authors: ["Personae"]
type: posts
draft: false
slug: "noun-nyms-challenges"
category: "10 mins read"
tags: ["identity", "nyms"]
description: "Technical challenges encountered while building Noun Nyms"
---
[Noun Nyms](https://nouns.nymz.xyz/) is where stakeholders of [NounsDAO](https://nouns.wtf/) discuss proposals, give hot takes, and post [epic self-introductions](https://nouns.nymz.xyz/posts/0x05817ead5dfd9118160a9e1fec72fc734cecde883769d5a501e8f3d5e0346a40), through [pseudonyms](https://nouns.nymz.xyz/users). The link between a pseudonym and the underlying ETH address is hidden by using zero-knowledge proofs, and users can create multiple pseudonyms. The link between two or more pseudonyms is concealed even when the pseudonyms come from the same ETH address.

There are unique challenges when building such an app that allows users to interact through their pseudonyms, without revealing the underlying identity, even to the server hosting the website. This post is intended to share the technical challenges we encountered while building Noun Nyms.

## Zero-Knowledge Proof of Membership

In Noun Nyms, we use a primitive called “Zero-Knowledge Proof of Membership” to let users prove they are indeed a Noun holder or a delegate, without revealing which Noun they own.

The primitive is implemented by a circuit, which encodes mainly two verification algorithms: ECDSA verification, and Merkle proof verification. The former verifies that the prover knows a private key that corresponds to an ETH address, and the latter verifies that the aforementioned ETH address is in a specified set. In Noun Nyms, the set consists of all Noun token holders and delegates.

The proving time for such a circuit used to take minutes, but with several revisions, we improved the proving time to less than 10 seconds[^1]. The techniques we employed are mainly the following two:

- Spartan-ECDSA
- Extraction of public keys from ETH addresses

**Spartan-ECDSA (**Speeds up the ECDSA verification**)**

Spartan-ECDSA is a tailor-built zero-knowledge proof system for verifying ECDSA signatures. We recommend our [previous post](https://personaelabs.org/posts/spartan-ecdsa/) for details of Spartan-ECDSA. 

**Extracting the public keys from ETH addresses (Speeds up Merkle proof verification)**

Addresses that appear in wallets are public key hashes.  An address is obtained by hashing a public key using an algorithm called Keccak256.  Naturally, when generating a proof that claims you own an ETH account in a given set,  you need to prove that you ran the Keccak256 algorithm correctly.  More precisely, you need to prove that the leaf of the Merkle proof corresponds to the Keccak256 hash of the public key verified in the ECDSA verification step. However, Keccak256 is a so-called “zk-unfriendly” algorithm, hence resulting in a long proving time. We overcome this by avoiding Keccak256 altogether by using a set of public keys, instead of a set of addresses. By using a set of public keys, you can directly link the public key checked in the ECDSA verification step to the leaf of the Merkle proof, and omit the expensive hashing.

So how do we extract a public key from an address?

A public key can be recovered from an ECDSA signature, and all Ethereum transactions are attached with an ECDSA signature. To obtain the public key of an address, we can get a single transaction that the address has made in the past, and recover the public key from its signature. Alternatively, we can fetch signatures from apps like [Snapshot](https://snapshot.org/) where votes are signed with Ethereum account.  However, if an address has never submitted a transaction before, and there are no publicly accessible signatures signed by that address, there’s no way to recover the public key. 

Currently, we don’t have access to the public keys of about x% of users. For those, we need to ask them to submit their public key through the app.

*Mention about faster proving to allow Keccak256 in browser*


## Avoiding associating multiple pseudonyms

In Noun Nymz, users can have multiple pseudonyms, and no one knows if two or more pseudonyms come from a single user, even the server hosting the website. Implementing features while preserving such privacy property comes with unique challenges. Notably, notifications for signed-in users, which is crucial to a forum application.

A regular non-pseudonymous app can implement notifications by simply allowing the frontend to fetch the notification of the signed-in user. However, in Nyms a signed-in user can have multiple pseudonyms associated with their identity. Asking the backend server to provide notifications for those pseudonyms at the same time reveals the association of their pseudonym. Therefore, to avoid revealing associations of pseudonyms to the backend server, the frontend fetches all activities and filters the relevant activities on the client side. Furthermore, the frontend stores the timestamp of the latest notification and uses that to reduce the bandwidth and processing required for the next fetch.

This appears redundant, but it’s necessary to avoid associating multiple pseudonyms. 

We will note that fully homomorphic encryption, when practical, could become an instrument for implementing such a feature optimally.

## Multisigs

*Will probably delete this since it’s not really a challenge and is an obvious mechanism*

Some Nouns are held in multi-signature walletz. Unlike a Noun held by an EOA, there are multiple public keys associated with the Noun for such wallets. We include those public keys in the set of users that can post, comment, and upvote in Noun Nyms. 

## Changing pseudonymity set
TBD

## What’s next

### Faster proving

TBD

<!-- Footnotes themselves at the bottom. -->

[^1]: The proving time varies between devices used and the condition (e.g. CPU temperature) of the device. It could take over 10s in some cases.