---
title: "In-person heyanon at Devcon"
date: 2022-10-12T02:12:03.284Z
authors: ["vivboop"]
type: posts
draft: true
slug: "sbcheyanon"
category: "5 mins read"
tags: ["heyanon", "PSE"]
description: "Hyping up heyanon's Devcon experiments"
---

[heyanon](https://heyanon.xyz) is coming to Devcon! If you go to the Temporary Anonymous Zone (TAZ) booth on the first floor of the Agora center, you can join PSE's TAZ Semaphore group with an invite link. Then, conect to heyanon through the link on their experience page, where you can post to the @DevconAnon feed and reply to arbitrary tweet as @DevconAnon! Drop your spicy takes and burning questions as an anonymous Devcon attendee. And continue to use the bot after the event ends!

## Wait, what's heyanon?

heyanon is an [open-source app](https://github.com/personaelabs/heyanon) allowing users to anonymously tweet as a verified member of on and off-chain groups using ZKPs. In a world where users have to choose between the extremes of using their real-world identity or being fully pseudonymous, heyanon is working towards a future where you can attach a specific subset of your identity and reputation to your speech online. Feeds for different groups are curated by the [@heyanonxyz](https://twitter.com/heyanonxyz) Twitter account, which serves as the homepage of the project.

heyanon has primarily focused on Ethereum-based groups like NFT owners, Gitcoin Grantees, and victims of hacks. If a member of this group wants to post a message _m_, they create a ZK proof that verifies a member of the group signed the message _m_ that keeps the member and signature as private inputs. They send this proof to our server, and if it verifies correctly then we post a tweet with that message _m_!

However, the TAZ group gives a new _in-person_

The ZK proof for this circuit verifies that a member of each of these group signed a message, but keeps the member and signature private for anonymity.

Once you generate a semaphore ID at the [TAZ app](https://taz.appliedzkp.org), navigate to the "heyanon" option and start interacting with the Twitter feed! For these in-person experiments, we have a few goals:

## Goal 1: Highlighting the _magic_ of ZK

Any ZK developer will tell you that their work feels like a glitch in the universe. It just shouldn't be possible to get _privacy, succinctness, and verifiability_ all in one tool. But if you go through the math, step by step, it all somehow works. Unfortunately, ZK terms like "witnesses", "proving", "verify", "zero-knowledge", and "elliptic curve pairings" mask how utterly magical this technology is to the average person. So our goal was to use new terminology for every part of the zero knowledge stack. Your semaphore identity is your _wand_. Instead of creating proofs, we use this wand to _cast spells_. Instead of downloading a proving key, we're gathering _magical equipment_.

We hope this can inspire creative new ways to package zero knowledge technology, because it truly is unlike anything else in the Web2 and Web3 world.

So a separate goal of in-person heyanon was to use new, magical terminology for every part of the ZK stack. The Semaphore private key became your _artifact_. Instead of creating proofs, you used your artifact to _cast spells_. Each of the conference roles became magical too: attendees were _magicians_, presenters were _wizards_, previous attendees were _alchemists_, and organizers were _sorcerers_.

Isn't that just so much more fun?

## Goal 2:
