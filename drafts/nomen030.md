# Nomen Gets A Protocol Upgrade

With the recent release of v0.3.0 of [Nomen Explorer](https://nomenexplorer.com), the reference indexer for Nomen, comes the release of protocol v1, the planned upgrade of the current protocol version, v0. I wanted to discuss why this upgrade was necessary.

## Introduction to Nomen

Nomen is a protocol designed as kind of a Nostr-native universal identifier/DNS hybrid. It uses the Bitcoin blockchain to order claims to unique, global identifiers (like domain names) and Nostr to distribute records and data associated with those name.

## TLDR

- v0 had data availability limitations
- v1:
  - resolves those limitations
  - is strictly an upgrade (no need to issue new v0 names)
  - allows name transfers
  - is backwards-compatible with v0 (names share the same namespace)
- The on-chain protocol can be considered **LONG TERM STABLE** and unlikely to change significantly in the future, except for value-added upgrades
 
Keep reading if wish more a detailed explanation.

## Problems with v0

When publishing your name on the Bitcoin blockchain, v0 involved this `OP_RETURN` commitment: `"NOM" 0x00 0x00 <FINGERPRINT:5> <NSID:20>`. The first ten bytes are metadata and tangential to this. Focus on `NSID`. What is the NSID?

It is short for "namespace ID" and referred to a hash that uniquely identifies the owner's pubkey + the name. The indexer would record the NSID, then look for corresponding Nostr events on public relays that it could use to reconstruct the NSID and then the indexer would know the name and the owner with the associated NSID.

This presents a data availability problem, however. It is impossible for the indexer to get a complete picture of the name without associated Nostr events. But the availability guarantees of Nostr events are poor. They are ephemeral by nature. This makes it dicey to claim a name because if there is no published Nostr event available for the name on indexer's relays, then it can't know what name belongs to the claim.

It then becomes difficult to know if someone has already registered a name before you are trying. It could create this situation:

1. Alice claims `name`, but does not publish a Nostr event.
2. Bob claims `name`. Because they have different pubkeys, the NSID is different and Bob cannot know it was already claimed.
3. Bob publishes Nostr event, begins using their name.
4. Alice publishes a Nostr event for their claim, completely rugging Bob and anyone of Bob's social graph.

The `FINGERPRINT` metadata above was an initial attempt to fix this issue, but it had its own issues which are out of focus for this discussion. Suffice to say, it didn't resolve it well enough, and introduced new issues of its own.

Additionally, there were issues with the original design and enabling name transfers to new pubkeys. Because of the poor guarantees of data availability of Nostr events, anyone who received a name transfer would need to keep copies of signed Nostr events for previous ownership changes *forever* and make sure they are occasionally republished so that public Nostr relays have them available for indexers.

If the current owner ever lost even a single event in the ownership chain, and they can't find the original person to re-sign a new event, then the ownership chain is broken forever and the name is essentially untransferrable at that point. This is less than ideal, obviously!

That's just from the user's perspective. From an indexer perspective, linking on-chain NSID to Nostr event back to NSID then back to Nostr, for an entire ownership chain, would have been an annoying, complicated and brittle graph traversal.

As Nomen started to get more attention, some of these problems came into focus for me, and I decided this was a must-fix issue.

## Resolved by v1

So how does v1 resolve this problem? It does it by making sure all ownership data is on the Bitcoin timechain, just as God intended. The on-chain claim now looks like: `"NOM" 0x01 0x00 <PUBLIC KEY:32> <NAME:3-43>`.

Now, the owner and the name are both on chain, so the indexer can tell immediately who the owner is, and whether a name is already registered. This also allows the protocol to enable transfers by putting the transfer data (including signature authorizing transfer) on chain as well. If this sounds like a lot of data, don't worry, it's not! This is not like a JPEG on chain. All of these things fit in `OP_RETURN` outputs (80 bytes or less).

Bonus points for being backward compatible with v0. Because the `NSID` is just constructed from the pubkey and the name, v1 names can be deconflicted with v0 names, allowing them to occupy the same namespace. Additionally, this allows us to provide for v0 names to be **UPGRADED** to v1 by simply providing a special case where a v1 CREATE that matches a previous NSID is actually considered an upgrade. This will allow v0 to be transferrable by being upgraded first.

This protocol upgrade fulfills the original design vision of Nomen, which is that Bitcoin determines who the owner of the name is and Nostr is used to distribute records for the name.

## What's next for Nomen?

There are some potential value-added upgrades for the future:

1. [Name lists on Nostr](https://github.com/ursuscamp/nomen/issues/13): Currently, names can be searched on the indexer's webpage or via REST API calls. If indexers can publish their name lists to Nostr, then Nostr clients don't need to make external API to search for names.
2. [Name recovery path](https://github.com/ursuscamp/nomen/issues/11): Pre-specified recovery paths for names, so if your main key is stolen and name transferred, you can recover your name from a cold storage key.
3. [Handling squatters](https://github.com/ursuscamp/nomen/issues/8)
4. [Sidechain namespaces](https://github.com/ursuscamp/nomen/issues/14): Blockspace won't be cheap forever. We can open additional namespaces on sidechains. For instance, you could reserve names on the Liquid sidechain using `.liquid` suffix.
