---
layout: post
title:  "Sentinel Chains: A Novel Two-Way Peg"
date:   2023-08-10 10:57:00 -0400
categories: bitcoin
---

_Since original publication this has been edited some for style. There have been additional discussion of this proposal on [Stacker News](https://stacker.news/items/222480) and on [bitcoin-dev](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-August/021894.html)._

## Introduction

Lately, I've been contemplating the concepts of sidechains with two-way pegs, spurred in part by the ongoing Twitter discourse (or what could be less diplomatically referred to as "bickering") surrounding Drivechains. Two-way pegs within sidechains are akin to the proverbial Holy Grail—desirable but challenging, if not seemingly impossible, to attain. On the other hand, one-way pegs are already achievable by burning Bitcoin in an OP_RETURN output and anchoring the coins' movement to the sidechain through the OP_RETURN data.

While this concept is easy to grasp, its major drawback is the inability to facilitate a consistent 1:1 price peg, as coins can't be reversed. The sidechain's Bitcoin could establish an upper price ceiling but not a lower price floor. Moreover, it's unlikely that the Bitcoin community would readily embrace one-way pegs, given the discomfort many feel about burning Bitcoin.

This brings us back to the quandary of two-way pegs. Ideally, Bitcoin should be able to fluidly move back and forth between the main chain and the sidechain to maintain a stable peg. To achieve this, Bitcoin full nodes must acknowledge the sidechain's existence, monitoring it to ensure that Bitcoin peg-outs adhere to validity standards of the sidechain. Previous proposals for this purpose, notably Drivechains, have been presented, with some regarded as "production-ready" softfork solutions integrated into Bitcoin Core.

However, what if we could establish two-way pegged sidechains without requiring any modifications to Bitcoin Core's validation logic? Allow me to introduce an unconventional concept: Sentinel chains.

## The Idea

A Sentinel chain represents a sidechain that hinges on a decentralized network of trusted watchtowers—known as sentinels—to thwart unauthorized peg-outs. But how might this be realized?

While multiple approaches could work, one potential strategy involves pegging in through payments to a designated address on the Sentinel chain. This address corresponds to a script adhering to Bitcoin's consensus rules. The transaction could also include an OP_RETURN containing data that indicates an address on the Sentinel chain for the peg-in transfer.

For pegging out, a peg-out transaction is conducted on the Sentinel chain, followed by a corresponding transaction on the main chain. The latter transaction pays from a Sentinel main chain UTXO, the amount mirroring the peg-out value, while any change returns to the Sentinel chain address.

An obvious question arises: How do we thwart theft if anyone can withdraw from these UTXOs? Enter the sentinels. These trusted entities operate watchtowers, identifying peg-out thefts and broadcasting Nostr events to either clear the transactions from mempools or invalidate mined blocks should the theft proceed.

## Sentinels and Trust

Who qualifies as a sentinel? The intriguing part is that anyone can be a sentinel. Individuals running a sidechain node alongside a Bitcoin node can opt to broadcast theft events to public Nostr nodes. Bitcoin node operators and miners can then select which sentinels to trust. Suppose you're a Bitcoin node runner using Bob's Sidechain; your daemon monitors public Nostr relays for theft events issued by 100 trusted Sentinels affiliated with Bob's Chain. These sentinels might comprise prominent figures within the Bob's Chain community or reputable entities in that space. If 75 out of 100 sentinels broadcast a theft event, you'd accept its invalidity and eliminate it from your mempool, invalidating any corresponding blocks. In a well-established sentinel network, consensus should emerge swiftly through game theory, often before a transaction is even mined in a single block.

## Bootstrapping a Chain

Launching a sidechain can prove challenging. Without an established network of sentinels guarding against thefts, a chicken-and-egg dilemma arises. How do we prevent thefts before the sidechain gains traction and reinforces its network effects?

Initial sentinel chains might be sponsored by influential companies with miner ties. For example, if Blockstream and its associates decide to transition Liquid from a federated sidechain to a sentinel chain, they could ensure that a majority of miners monitor for thefts before the transition becomes active.

Even without corporate support, the community could bootstrap a sentinel chain patiently, aided by Bitcoin Script. Consider if the sentinel chain address were a script that prohibited peg-outs until a predetermined time post-chain launch. If the sentinel chain debuts in 2024, the script could stipulate no spending from that address until 2029. This timeframe would provide the community five years to establish a sentinel network before theft concerns arise.

## Covenants Could Help

While sentinel chains function well without altering Bitcoin Core, forthcoming covenant proposals could enhance their confidence and security. Picture a scenario involving network partitions, sentinel chain bugs, or unforeseen circumstances that briefly enable a theft. A block might contain a mined peg-out theft that persists temporarily, permitting someone to double-spend stolen sentinel chain funds before it's reorganized.

What if a sentinel address was a covenant requiring 10,000 blocks before post-withdrawal Bitcoin spending? This precaution could prevent chaos in cases of short-lived theft events.

## A New Future for Soft Forks?

Keen readers might discern that this approach resembles a soft fork, albeit without adjustments to Bitcoin Core. This ensures that Core-exclusive adherents needn't be concerned. Absent Core changes, the risk of a bug jeopardizing their Bitcoin holdings is eliminated. The only individuals exposed to risk are sidechain users.

This prompts the query: Could a Sentinel network on Nostr potentially enforce other opt-in soft fork activations without Bitcoin Core modifications? Quite possibly! A question for another day, perhaps.

## Inspirations

* [Drivechains](https://www.drivechain.info)
* [Softchains](https://gist.github.com/RubenSomsen/7ecf7f13dc2496aa7eed8815a02f13d1)
* [Some Day Peg](https://gist.github.com/RobinLinus/1102fce176f3b5466180addac5d26313)
