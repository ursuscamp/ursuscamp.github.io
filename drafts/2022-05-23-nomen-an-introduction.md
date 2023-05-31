---
layout: post
title:  "An Introduction to Nomen"
date:   2023-05-23 22:01:47 -0400
categories: bitcoin nostr
---

- [Digital Identity: A valuable resource](#digital-identity-a-valuable-resource)
- [Defeating ICANN](#defeating-icann)
- [Nomen](#nomen)
- [Indexers](#indexers)
- [Conclusion](#conclusion)

This is the first of a series of posts I hope to write about various details around Nomen.

## Digital Identity: A valuable resource

Public key cryptography is a revelation that made privacy possible in the digital era. The power of public key cryptography means that everyone can have their own keypair, and there's no need to have anyone arbitrate who gets which key. Because the keyspace is so massive, it is effectively infinite. It's more or less impossible for anyone to randomly pick the same key as anyone else.

But the human brain isn't wired for public key cryptography. Our minds can't remember dozens of bytes of random data for everyone in our life. It's just not possible for almost anyone at all.

This highlights the importance of **human-readable** digital identity. While cryptographic keys are essentially an infinite resource, unique names are not. Within a certain context, there are only so many possible useful names that can exist, especially if we stipulate that they must be unique. How can we administer such a scarce resource?

## Defeating ICANN

Domain names are one type of digital identity. Domain names are globally unique identifiers that point to a series of records stored on servers and propagated through the DNS protocol. While the DNS protocol is somewhat decentralized, the actual administration of domain names is decidedly not. One organization, called ICANN, controls the leasing of top-level domains (think `.com`, `.net`, etc.). They are like the central bank of domain name registrars. Could we somehow have globally unique names, without a central arbitrator?

This is a problem many in the crypto community have attempted to solve. Many blockchains have their own namespace. You may see them all the time in Twitter handles (`.eth`, `.sol`, `.btc`, etc).

I find this problem to be particularly interesting, and decided to take a swing at it.

## Nomen

Nomen is an attempt to design a protocol that will allow everyone to generally agree which names belongs to which person. All names are globally unique. Bitcoin decides order, Nostr carries data. The beauty of this is that there is no need to bootstrap any new P2P protocols. Names are first come, first serve. The first to claim a name on-chain are the recognized owners of that name. Any further attempts to claim it are ignored. Record updates are published through Nostr.

Getting a name involves publishing a claim in a bitcoin `OP_RETURN` output. It's a small output, just a hash and a few extra bytes of metadata. The hash represents the unique identifier. It is simply a hash of `NAME + OWNER PUBLIC KEY`. Then a Nostr event, called a `record event`, is signed and broadcast, which includes the name as metadata.

The Nostr event must be signed by the **same key** that is the owner of the name. This is critical. An indexer takes the name and pubkey from the record event, and recreates the hash to verify it on-chain. If they match, and **if it is the first such claim made on Bitcoin**, the indexer will acknowledge the name as valid.

## Indexers

Indexers are the name servers of the Nomen protocol. They scan the Bitcoin blockchain for Nomen claims, and watch public Nostr relays for corresponding events. The [Nomen Explorer](https://nomenexplorer.com) is the first such indexer. It's running publicly now, but you can run it yourself if you wish to.

The Explorer has tools available to claim your own name, as well. You must provide an unsigned PSBT, and it will add the necessary output and return it you to sign and broadcast with your own Bitcoin wallet. It's completely trustless, the Explorer never has your Bitcoin or your private keys. Make sure to slightly over-estimate your fees to account for the bit of extra data from the new output.

The Explorer also has a CLI you can use to claim a name, if you prefer to use it that way.

## Conclusion

It's actually very simple. A bitcoin transaction and a Nostr event, two pieces that fit together that form a **provably unique** name. This represents the core of the Nomen protocol.

In future posts, I may dig into the advantages and weaknesses of the Nomen protocol, scaling improvements that may come, and other protocol enhancements.

For more details, please check out the first indexer, the [Nomen Explorer](https://nomenexplorer.com). For the more technically minded, read our short [spec](https://github.com/ursuscamp/nomen/blob/master/docs/SPEC.md) over on GitHub.