---
layout: post
title:  "Bitcoin Bits and Bobs: Merkle Trees"
date: 2022-10-27 22:36:09 -0400
categories: bitcoin rust
---

Another subject I find interesting in the world of Bitcoin is the [merkle tree](https://en.wikipedia.org/wiki/Merkle_tree). You see them mentioned all over the place, but it's hard to understand exactly what they are.

In short, it is a data structure that makes it relatively simple to verify that a piece of data is included in a hash, without revealing all of the other data in that hash. In the case of Bitcoin, the block header contains a "merkle root", which is the final hash output of a function that creates a merkle tree. The data in the merkle tree are the individual transaction hashes.

When implementing the full hash of transactions in the block header, there were really two options:

1. Concatenate all transaction hashes together, then hash that data.
2. Create a merkle structure to implement the hash.

Satoshi Nakamoto chose number 2, obviously. In [the white paper](https://bitcoin.org/bitcoin.pdf), Satoshi described the concept of an SPV node, or a simple payment verification node. It doesn't store any blocks, it just receives the block headers, and it can easily check if a transaction is in the block by requesting a subset of the other leaves from the merkle tree. No need to get every transaction from every block. It can reconstruct the merkle tree from a few leaves and know whether any of the user's transactions are in the block.

So how is the merkle tree constructed? In the case of Bitcoin, given a full list of transaction hashes, it hashes them in pairs, in the order they are included in the block. For example, given the transactions `A`, `B`, `C`, and `D`, the merkle tree is constructed by hashing `(A+B) => AB`, and `(C+D) => CD`. Now `AB` and `CD` can be concatenated and hashed, leaving one final hash value, and this is the merkle root.

If an SPV node received a new block header from a full node, and they wanted to know if the user's transaction `C` was included in the block without requesting the full transaction list, the node could request hashes `D` and `AB` and reconstruct the merkle root itself, verifying the transaction inclusion without any trust required.

Before we start, there are four important things to know about how merkle trees are constructed in Bitcion.

1. If a block only has one transaction in it, then the merkle root is just the transaction id (no further hashing is done).
2. All merkle hashses are actually double hashes, i.e. a SHA-256 hash of a SHA-256 hash.
3. The merkle root is stored in [Little Endian](https://en.wikipedia.org/wiki/Endianness) in the block header, as are transaction hashes, so we will need to reverse the data we get from block explorers, since those are shown in Big Endian.
4. If there aren't enough transactions to create perfectly even pairs, then the final odd transaction hash is doubled (concatenated with itself) before hashing.

So now let's build the thing! The final code can be found in the [public repo](https://github.com/ursuscamp/bnb).

First things first, we need to add the [`sha2`](https://github.com/RustCrypto/hashes) crate to `Cargo.toml`, so that we can calculate the `SHA-256` hashes required to reconstruct a merkle root. We also need to select a test case. I picked an [earlier block](https://mempool.space/block/0000000000013b8ab2cd513b0261a14096412195a72a0c4827d229dcc7e0f7af) because that had very low transaction counts, which makes it easy to work with in the unit tests. I also made sure to pick a block with an odd numbers of transactions.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // bitcoin-cli getblock 0000000000013b8ab2cd513b0261a14096412195a72a0c4827d229dcc7e0f7af
    // https://mempool.space/block/0000000000013b8ab2cd513b0261a14096412195a72a0c4827d229dcc7e0f7af
    static TRX: [&'static str; 9] = [
        "ef1d870d24c85b89d92ad50f4631026f585d6a34e972eaf427475e5d60acf3a3",
        "f9fc751cb7dc372406a9f8d738d5e6f8f63bab71986a39cf36ee70ee17036d07",
        "db60fb93d736894ed0b86cb92548920a3fe8310dd19b0da7ad97e48725e1e12e",
        "220ebc64e21abece964927322cba69180ed853bb187fbc6923bac7d010b9d87a",
        "71b3dbaca67e9f9189dad3617138c19725ab541ef0b49c05a94913e9f28e3f4e",
        "fe305e1ed08212d76161d853222048eea1f34af42ea0e197896a269fbf8dc2e0",
        "21d2eb195736af2a40d42107e6abd59c97eb6cffd4a5a7a7709e86590ae61987",
        "dd1fd2a6fc16404faf339881a90adbde7f4f728691ac62e8f168809cdfae1053",
        "74d681e0e03bafa802c8aa084379aa98d9fcd632ddc2ed9782b586ec87451f20",
    ];
    static ROOT: &'static str = "2fda58e5959b0ee53c5253da9b9f3c0c739422ae04946966991cf55895287552";

    fn transactions() -> Vec<Vec<u8>> {
        TRX.map(|t| hex::decode(t).unwrap().into_iter().rev().collect())
            .to_vec()
    }

    fn root() -> Vec<u8> {
        hex::decode(ROOT).unwrap().into_iter().rev().collect()
    }
}
```

Now that we have a test case, we can write some code.

First, we need to be able to concatenate our data correctly:

```rust
fn concat(data: &[Vec<u8>]) -> Vec<u8> {
    let mut init = data[0].to_vec();
    if data.len() == 1 {
        init.extend(data[0].iter());
    } else {
        init.extend(data[1].iter());
    }
    init
}
```

Straight forward enough? Given a list of binary values (expected to be either two or each, representing a merkle pair), we can concatenate them together. If the list length is 1, concatenate with itself. Otherwise, it should be two and we can concatenate both values together.

Next, we need to be able to construct a SHA-256 hash:

```rust
fn hash(data: &[u8]) -> Vec<u8> {
    let mut hasher = Sha256::new();
    hasher.update(data);
    hasher.finalize().to_vec()
}
```

But remember, we actually need to double hash the values in merkle trees!

```rust
fn double_hash(data: &[Vec<u8>]) -> Vec<u8> {
    hash(&hash(&concat(data)))
}
```

Finally, we need a function to construct a merkle tree:

```rust
pub fn calculate(hashes: Vec<Vec<u8>>) -> Vec<u8> {
    let mut hashes = hashes;
    while hashes.len() > 1 {
        hashes = hashes.chunks(2).map(|f| double_hash(f)).collect()
    }
    hashes.remove(0)
}
```

Given a list of bytestrings (transaction hashes in this case), we iterate them in pairs using `chunks(2)`, and run double hash on each pair. We save those values, and if there more than 1 hash remaining, we run another iteration. Eventually, there only be 1 hash remaining, and that is the merkle root.

Last but not least, we can run a test:

```rust
#[test]
fn test_calculate() {
    let mr = calculate(transactions());
    assert_eq!(mr, root());
}
```

Bonus points: Write a test case to make sure that a block with only a single transaction returns the expected value, as mentioned previously:

```rust
#[test]
fn test_calculate_with_one_transaction() {
    let trx = hex::decode("0e3e2357e806b6cdb1f70b54c3a3a17b6714ee1f0e68bebb44a74b1efd512098")
        .unwrap()
        .into_iter()
        .rev()
        .collect::<Vec<_>>();
    let mr = calculate(vec![trx.clone()]);
    assert_eq!(trx, mr);
}
```