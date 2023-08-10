# Sentinel Chains: A Novel Two-Way Peg

## Introduction

I have had some thoughts recently regarding sidechains with two-way pegs, which is inspired partly by the recent Twitter discourse (read: bickering) over Drivechains. Two-way pegs for sidechains are a kind of Holy Grail, and like all Holy Grails, difficult (or seemingly impossible) to achieve. One way pegs are already very possible by burning Bitcoin in an OP_RETURN output, and including in the OP_RETURN data some kind of anchor to the sidechain, establishing the coins being moved.

This is easy to conceive and easy to understand. The only issue is that since coins cannot be moved back, it's impossible to maintain a 1:1 price peg. The sidechain Bitcoin would have a price ceiling, but not a price floor. Also, it seems unlikely that Bitcoiners would ever get on board with one-way pegs, since burning Bitcoin seems to give many of us the heebie jeebies.

This brings us back to the problem of two-way pegs. Ideally, you could move the Bitcoin back and forth in some fashion between side chain and main chain, so that the peg is naturally maintained. To some degree, Bitcoin full nodes need to be aware of the sidechain's existence. They must track it to make sure that peg-outs happening on the Bitcoin main chain are valid peg-outs and not filthy lies. Some proposals for this have existed in the past, the most "production-ready" of these being Drivechains. They are generally softfork proposals directly in Bitcoin Core.

But what if we could do sidechains that have two-way pegs _and_ we could do it without any code changes to Bitcoin Core at all? Let me tell you all about a crazy idea I have: Sentinel chains.

## The Idea

A Sentinel chain is a sidechain that, to put it simply, relies a decentralized web of trusted watchtowers (the sentinels) to make sure that illegal peg-outs do not occur. So, how might this work?

It could likely work in many potential ways that I am not not even aware, but here is one way I could envision it occurring:

Pegging in could work by paying to an address that is known as that Sentinel chain's address. That address represent a script that anyone can pay from, according to Bitcon's consensus rules. That transaction might also include an OP_RETURN whose data represents an address on the Sentinel chain to pay to after peg-in.

Pegging out would work by making a peg-out transaction of some kind on the Sentinel chain, followed by a corresponding transaction on the main, paying out from a Sentinel main chain UTXO with the amount corresponding to the peg-out amount, and any change going back into the Sentinel chain address.

Obviously, this raises the question: How do we prevent stealing? This is where the sentinels come in: Sentinels are trusted entities that operate watchtowers that looks for peg-out thefts and broadcast Nostr events about which transactions to purge from mempools or which blocks to invalidate (in the event the theft gets mined).

## Sentinels and Trust

So who gets to be a Sentinel? That's the best part: anyone can be a Sentinel! Anyone operating a sidechain node alongside a Bitcoin node can choose to broadcast theft events to public Nostr nodes. Any Bitcoin node runners or miners can pick and choose which Sentinels they trust. For instance, as a Bitcoin node runner and user of Bob's Sidechain, you may run a daemon alongside Bitcoin Core. The daemon watches known public Nostr relays for theft events from 100 Bob's Chain Sentinels you trust. Some of them may be popular individuals in the Bob's Chain community or high profile companies in the space. You decided that if 75 out of 100 of the Sentinels broadcast a theft event, then you will trust that and purge it from your mempool, or invalidate any blocks it is mined in. Assuming we have bootstrapped a Sentinel network successfully, the game theory should allow us to quickly arrive at consensus, usually before a transaction would even be mined in a block.

## Bootstrapping a Chain

Getting a sidechain up and running might be a tricky process. Unless there was an already established and accepted network of Sentinels watching for thefts, you run into a chicken-and-egg problem of how to prevent thefts from occurring before the sidechain even really gets a chance to get popular enough to have reinforcing network effects.

It seems likely that the first Sentinel chains would be sponsored by companies who have pull with miners. For instance, if Blockstream and their partners decided to transition Liquid from a federated sidechain to a Sentinel chain, they might make sure they have a majority of miners watching for thefts before they even made their transition chain open to the public.

Even without corporate support, the community could still bootstrap a Sentinel chain with patience and some help from Bitcoin Script. Imagine the Sentinel chain address was also a script that said no peg-outs could occur until time after chain launch. If we launched the chain in 2024, the script could say that nothing from that address could be spent until 2029. That would give the community confidence that they have five years to bootstrap their Sentinel network before they have to worry about thefts.

## Covenants Could Help

Sentinel chains could work perfectly fine as is with no changes to Bitcoin Core. However, with some covenant proposals in the pipeline, those could certainly aid in Sentinel chain confidence and security. Imagine for a moment if there is a network partition, Sentinel chain bug or other unforseen circumstance, that caused a theft to go through temporarily. There might be some block where a peg-out theft was mined, and was allowed to exist for a time. This means that someone could double-spend by buying some good and service with stolen Sentinel Chain money before it gets re-orged.

What if a Sentinel address was a covenant, and anyone withdrawing from that covenant had to wait 10000 blocks before they could spend their Bitcoin after withdrawing? That could prevent chaos in the event that some theft continues for some short-lived existence.

## A New Future for Soft Forks?

Some of the more discerning readers may have noticed that this is actually a kind of soft fork, just with no changes to Bitcoin Code. This means that any Core-only main-chain maxis never even have to care. Without a code change in Core, there's no chance that a bug will rob them of their Bitcoin.

This begs the question, can the method of having Sentinels on Nostr be used to potentially enforce other opt-in soft fork activations without any Bitcoin Core changes? Quite possibly! A question for another day, perhaps.