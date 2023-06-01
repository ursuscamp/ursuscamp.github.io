---
layout: post
title:  "A Little Bit About Nomen"
date:   2023-05-31 23:04:33 -0400
categories: bitcoin nostr
---

_This has also been cross-posted to Nostr: https://snort.social/e/note15xq9u92kr525rygqzyglm99qs7cejy7udruhmzw22jflh0sy0dzqs3m2rk_

I wanted to share a little bit about Nomen, a new protocol that establishes rules for creating globally unique, human readable identifiers, similar to domain names but without a central planner like ICANN.

## How does it work?

Nomen uses the ordering guarantees of the Bitcoin blockchain as the arbitration method for opposing claims on names. In other words: first come, first serve. Nostr acts as the data layer of the protocol. Records and other data-heavy operations happen as Nostr events on public relays. Indexers, which are the name servers of the protocol, link on-chain claims to related Nostr events in order to piece together a full picture of the Nomen name set.

For a more complete view of the protocol, check out the specification on GitHub. It’s very simple: https://github.com/ursuscamp/nomen/blob/master/docs/SPEC.md

## What’s in a name?

A name is “complete”, so to speak, when there is a claim published on chain with a corresponding Nostr event that contains the records for that name. The on-chain claim is merely an OP_RETURN output, which contains a few bytes of metadata and a 20-byte hash of the ownership information (Hash-160 of NAME + PUBLIC KEY). A corresponding Nostr record event must be published that is signed by the owner of the name, containing a tag with the name that was claimed and a record set. Using the name and the public key, the indexer can recompute the hash and validate that this event is published by the true owner of the name. The record event is a replaceable event, so every time you want to update your record set, you just publish a new record event to replace the old.

The record set is a just a set of key/values that represent the pieces of identity you want published. Some examples are “WEB” for your website, “TWITTER” for your twitter handle, “NPUB” for your Nostr pub, etc.

## The Explorer

The Nomen Explorer (https://nomenexplorer.com) is the first public indexer. Not only does it allow you to explore the existing names out there already, but it has tools to claim a name for yourself and to update your records.

Claiming a name through the Explorer is a trustless process. All you need to do is create an unsigned Bitcoin transaction paid back to yourself (or whoever you want) and copy it to your clipboard. Then click “New Name” in the Explorer, and paste your PSBT into the box. Enter your name and pub key, then submit. The Explorer will add an extra output to the PSBT which is a 0 sat OP_RETURN, and spit the new PSBT back out to you. Load that PSBT into your Bitcoin wallet. You can examine it to make sure the Explorer isn’t doing any funny business (it’s not, I promise!), then sign it and broadcast. The indexer will pick it up after 3 confirmations.

Then you click “Update Records” and publish a new record set. Sign the event with your Nostr extension (Alby, nos2x, Nostore, etc), and you’re good to go. Within a few minutes, the explorer should pick it up and display it in the list. 

## Consensus

You might have noticed that, since the claim is published in an OP_RETURN, that means that the miners and validators don’t enforce consensus like they do with Bitcoin script. In Nomen, the indexers are the arbitrators of consensus. This is a looser social consensus mechanism, similar to something like you might find with Ordinal Theory. This has advantages, as you will see.

## Squashing Squatters

Any name system needs to have a way of dealing with squatters. That way may be absolute central control, or letting them run rampant, or somewhere in between. The softer consensus of Nomen provides a potential solution to this problem.

One possible protocol upgrade is a standard for something like a decentralized spam/blacklist for squatters. Indexers could choose to subscribe to streams of Nostr events from trusted parties that handle spammers and squatters. Any name claim on such a spam list would be ignored by an indexer, and the next claim taken as the correct one. 

In reality, just the existence of such lists should deter most squatters, since they would need to waste real Bitcoin in mining fees just to try something easily thwarted by indexers.

## Scaling and the Future

The ability to scale these names is important. Every name requires an output on-chain. This isn’t a massive concern (yet) because most people won’t ever need but one or a few of these names. But, still, how far can we scale this?

One example of low hanging fruit is to allow name owners to create sub-names by publishing sub-name Nostr events. These could be useful for families, or for businesses to offer names to their customers. For instance, Bob Smith might register the name “smith” and create a sub-name for each of his children: alice.smith, andy.smith, etc.

Another exciting possibility is to expand the protocol to sidechains, using custom naming schemes. If “smith” is registered on the Bitcoin blockchain, someone else may claim “smith.lqd” on the Liquid sidechain. 

The combination of non-sovereign sub-names and sovereign top-level names on sidechains means that this one protocol could potentially scale to the entire world with ease.
