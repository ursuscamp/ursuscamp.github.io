---
layout: post
title:  "Credit Ecash and CheckTemplateVerify"
date: 2024-01-24 23:26:38 -0500
categories: bitcoin
---

_I want to think moonsettler for not only writing the ecash proposal that inspired this but also providing much feedback on the final article and talking with me about this subject at length._

## Check Your Templates

For the better part of the last year, after the publication of the [Enigma paper](https://app.sigle.io/polydeuces.id.stx/bo-iHio5_4iTlvWwXwZ9l) by [polyd](https://twitter.com/Polyd_), I have been interested in and researching [CheckTemplateVerify](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki) (`OP_CTV`).

`OP_CTV` is a Bitcoin Script enhancement that allows a Bitcoin transaction to lock itself to only be spent to the next specific output. This is a very restricted form of a [__covenant__](https://bitcoinops.org/en/topics/covenants/) smart contract. `OP_CTV` in particular commits to all of the outputs of the next transaction that spends that Bitcoin UTXO. Those outputs can include another CTV. Thus, a tree of transaction can be formed, where a Bitcoin UTXO slowly gets split over time as the whole tree unrolls on chain. There are many potential applications here for off-chain scaling with Lightning. For example, [timeout trees](https://bitcoinmagazine.com/technical/timeout-trees-a-solution-to-scaling-lightning-network-lsps) are a promising proposal based on `OP_CTV`.

`OP_CTV` works by commiting to a hash of the next transaction that spends it, and if the next transaction doesn't match the committed hash, the transaction is considered invalid. I experimented with `OP_CTV` in my [CTV playground](https://github.com/ursuscamp/ctv), which can be played with in signet or regtest on [Bitcoin Inquisition](https://github.com/bitcoin-inquisition/bitcoin).

## Credit Ecash

However, another engaging topic that caught my attention last year was Ecash. Specifically, the [Cashu](https://cashu.space) variety of ecash, which a blinded custodial ecash protocol. I read the [NUTs](https://github.com/cashubtc/nuts) and implemented the [BDHKE cryptographic primitives](https://github.com/ursuscamp/cash-test) in Rust as a learning exercise. Cashu is a fun protocol, it's a lot like developing on a permissionless Wallet of Satoshi protocol.

While I remain hopeful and enthusiastic about Cashu, it is custodial and as such has that trust trade-off from there is no escape. That is why I find [moonsettler's](https://twitter.com/4moonsettler) curious [proposal](https://gist.github.com/moonsettler/42b588fa97a1da3ac0adea0dd16dadf2) for a non-custodial **credit** ecash so interesting.

Credit ecash, as laid out by moonsettler, is an ecash in the sense of Cashu: satoshi-denominated blinded tokens issued by a mint. In this case, the tokens are non-custodial in nature. I suggest reading the paper a few times to grok the concept, but the basics work like this:

Two parties, myself and an [LSP](https://guide.bolt.fun/guide/lsp), create a 2-of-2 multisig output with collateral provided by myself. The LSP issues ecash in the amount equivalent to the collateral. I can use the ecash to make Lightning Network payments, just as Cashu tokens. Every so often, the tokens expire and I must turn in the tokens to the mint for a fresh set, and if I have a balance less than the locked capital, I can make a Lightning payment to make up the difference.

At no point does the LSP have a custodial hold on my Bitcoin. The ecash tokens act as credit extended to me by the LSP. They cannot rug me and take my Bitcoin because it is locked in a 2-of-2 multisig. In the event of a disagreement, we can send the UTXO to arbitration to decide how much Bitcoin belongs to which party. The arbitration transaction is signed by both parties before the transaction is even broadcast, which means that neither side can hold the other out of arbitration.

## Q&A

I had some questions about this proposal and took them to moonsettler for answers, which he graciously provided. I will paraphrase them below:

__Q: How do the mints make money?__<br>
_A: They can make money on transaction fees, but also on interest charged to users who do not wish to immediately pay the difference at the beginning of a new epoch._

__Q: Since my Bitcoin is locked into a multisig, how much extra liquidity is required for the LSP to hold for making payments?__<br>
_A: Probably a significant percentage of locked Bitcoin. You can probably expect people to use about 50% of their balance on average. Some will spend rarely, others will spend the whole balance._

__Q: What about cases where you receive more than you spend?__<br>
_A: Because of the private nature of ecash, the mint does not know which accounts are in a state of having a balance greather than their locked collateral, which means there is no incentive to rug the users in that regard. (Author: nevertheless, some more paranoid users will no doubt wish to move excessive funds to a different account or different LSP via the Lightning Network.)_

## And what about CTV?

So what does this have to do with CTV? Well, I think this system could play very nicely with CTV. As I mentioned previously, CTV (especially with [LNHANCE](https://github.com/bitcoin/bitcoin/pull/29198)) offers amazing potential scaling via Lightning channels in off-chain protocols like the timeout trees mentioned previously. But in fact, this proposal has the potential to play very nicely with just basic CTV and no additional off-chain protocols.

CTV offers a simple scaling solution called [Congestion Control](https://utxos.org/uses/scaling/). The typical example use case for these are when exchanges create batch withdrawal transactions in a CTV tree rather than a normal large batch transaction. This allow the exchange the commit a UTXO to a output which is small in vbytes but which also guarantees that it can be unrolled over time as the fees ebb and flow. The nice thing about this is that your receipt of Bitcoin is confirmed as soon as the CTV output confirms, even if the UTXO isn't spendable yet in the UTXO set. Some refer to this as as a vUTXO (virtual UTXO).

The issue with such a virtual UTXO is that, while it is secure once confirmed, it remains unspendable until it is unrolled. What if in the future due to high fees, it might take a very long time for the trees to unroll. Maybe even... years? That's a problem, for sure. But, also, maybe it's not.

One option is for the user to specify the output address as a [cold storage Lightning channel](https://utxos.org/uses/batch-channels/) (helpful X thread with adorable visuals can be found [here](https://twitter.com/OwenKemeys/status/1741575353716326835)), which can be revealed to a channel partner at a later time when convenient for the opener. This allows a one-way spend of these funds before the UTXO is on-chain.

But perhaps another option would be to open them to an LSP as credit ecash collateral. Here are benefits:

* Immediate spends.
* No channel management.
* Flexible liquidity, including __easy__ inbound liquidity.
* Send and receive privacy are superior to normal Lightning channel payments.

Even if the channel takes years to unroll, you get all of the benefits of batched channels with far less liquidity headaches. If cold channels are one way as some have said, then once a channel in a CTV leaf is spent, it is done and cannot be used anymore without cut-through. An ecash account has no such limitations! As long as the liquidity is confirmed, it can be used forever without channel updates.

While complicated and featureful off-chain Lightning channel protocols may offer _slightly_ better trust assumptions, the flexibility of the concept is exciting enough to keep me interested in further research.

All of this pretty much assumes that we get CTV-only activated. However it is highly likely that CTV will be activated with other op codes such as [OP_CSFS](https://github.com/reardencode/bips/blob/csfs/bip-csfs.mediawiki). These extra features will likely enable better NICs such as LN-Symmetry channels. See [here](https://delvingbitcoin.org/t/lnhance-bips-and-implementation/376/6?u=moonsettler) for more info on that.

However, if those LN-Symmetry channels are __still__ unidirectional, there is still a place in there for credit ecash for that reason alone! Of course, this post doesn't even begin to explore the benefits you might get from credit ecash mixed with covenant pools. Perhaps another day.

## Sources

* https://app.sigle.io/polydeuces.id.stx/bo-iHio5_4iTlvWwXwZ9l
* https://bitcoinmagazine.com/technical/how-ctv-can-help-scale-bitcoin
* https://bitcoinmagazine.com/technical/timeout-trees-a-solution-to-scaling-lightning-network-lsps
* https://utxos.org/uses/scaling/
* https://twitter.com/OwenKemeys/status/1741575353716326835
* https://gist.github.com/moonsettler/42b588fa97a1da3ac0adea0dd16dadf2
* https://cashu.space
* https://github.com/cashubtc/nuts
* https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki
* https://github.com/ursuscamp/ctv
* https://github.com/ursuscamp/cash-test
* https://bitcoinops.org/en/topics/covenants/
* https://github.com/bitcoin/bitcoin/pull/29198
* https://delvingbitcoin.org/t/lnhance-bips-and-implementation/376/6?u=moonsettler
* https://delvingbitcoin.org/t/lnhance-bips-and-implementation/376/6?u=moonsettler
