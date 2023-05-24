---
layout: post
title:  "An Introduction to Nomen"
date:   2023-05-23 22:01:47 -0400
categories: bitcoin nostr
---

- [Blockchains: What are they good for?](#blockchains-what-are-they-good-for)
- [Digital Identity: A valuable resource](#digital-identity-a-valuable-resource)
- [Killing ICANN](#killing-icann)
- [Nomen: Origins](#nomen-origins)
- [Nomen: Reality](#nomen-reality)
- [Conclusion](#conclusion)

## Blockchains: What are they good for?

It's an adage in the Bitcoin community that the only thing that blockchains are useful for is money. As a meme, it has power. It works as a statement of principle. However, from a computer science perspective, it's not strictly true.

Blockchains are technically useful for anything that could benefit from decentralized (or pseudo-decentralized) ordering guarantees. One thing, maybe the most important thing, that can benefit from that is money. Another popular use case is digital collectibles (NFTs).

One that particularly fascinates me, and which I think is very important (if a step below money), is digital identity. Specifically, human-readable names.

## Digital Identity: A valuable resource

Public key cryptography is a revelation that made privacy possible in the digital era. The power of public key cryptography means that everyone can have their own keypair, and there's no need to have someone arbitrate who gets which key. Because the keyspace is so massive, it is effectively infinite. It's more or less impossible for anyone to randomly pick the same key as anyone else.

But the human brain isn't wired for public key cryptogphay. Our minds can't remember 32 bytes of random data for everyone in our life. It's just not possible for almost anyone at all.

This brings us to the next level: **human-readable** digital identity. While cryptographic keys are effectively an infinite resource, unique names are not. Within a certain context, there are only so many possible useful names that can exist, especially if we stipulate that they must be unique. How can we administer such a scarce resource?

## Killing ICANN

Domain names are one type of digital identity. Domain names are globally unique identifiers that point to a series of records stored on servers and propagated through the DNS protocol. While the DNS protocol is somewhat decentralized, the actual administration of domain names is decidedly not. One organization, called ICANN, controls the leasing of top-level domains (think `.com`, `.net`, etc.). They are like the central bank of domain name registrars. The head of the snake, so to speak. Could we somehow have globally unique names, without a central arbitrator?

This is a problem many in the crypto community have attempted to solve. Many blockchains have their own namespace. You may see them all the time in Twitter handles (`.eth`, `.sol`, `.btc`, etc).

I find this problem to be particularly fascinating, and decided to take a swing at it.

## Nomen: Origins

In 2022/2023, multiple new protocols exploded that helped some ideas congealing in my mind to form into Nomen.

The first new protocol were Ordinals. Ordinal Theory is just an accounting system layered on top of Bitcoin's UTXO model. Despite the fact that the protocol has absolutely zero consensus-level enforcement from miners and nodes, it exploded in popularity anyway. I think there is an important lesson in here that not every system of property, even in the digital space, requires the same level of guarantees that are provided by total blockchain consensus. In many cases, a few simple rules and a **clear order of events** is enough for social cooperation.

The second new protocol was Nostr, which defines a messaging format for clients to communicate through relays. Even thought Nostr has been around for a few years, certain factors combined to cause an explosion of Nostr development and tools to start forming in early 2023.

## Nomen: Reality

These concepts and protocols came together in my mind to form what became Nomen: a simple protocol for deciding to whom a globally unique name belongs. Bitcoin is used for ordering, Nostr is used for data (record propagation). The two must go hand in hand.

Registering a name involves publishing a claim in a bitcoin `OP_RETURN` output. It's a small output, just a hash and a few extra bytes of metadata. The hash represents the unique identifier. It is essentially a hash of `NAME + OWNER PUBLIC KEY`. Then a Nostr event, called a `record event`, is signed and broadcast, which includes the name as metadata.

The Nostr event must be signed by the **same key** that is the owner of the name. This is critical. An indexer (which could be a separate piece of software such as the [Nomen Explorer](https://nomenexplorer.com), or it could even be a Nostr relay itself) takes the name and pubkey from the record event, and recreates the hash to verify it on-chain. If they match, and **if it is the first such claim made on Bitcoin**, the indexer will acknowledge the name as valid.

## Conclusion

It's actually very simple. A bitcoin transaction and a Nostr event, two pieces that fit together that form a **provably unique** name. This represents the core of the Nomen protocol.

In future posts, I may dig into the advantages and weaknesses of the Nomen protocol, scaling improvements that may come, and other protocol enhancements.