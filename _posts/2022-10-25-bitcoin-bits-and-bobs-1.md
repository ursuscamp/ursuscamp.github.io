---
layout: post
title:  "Bitcoin Bits and Bobs: Block Header Parsing"
date:   2022-10-25 23:30:25 -0400
categories: bitcoin rust
---

I have an interest in having a deeper understanding of the various parts of the Bitcoin protocol, so I'm going to write some pieces exploring various things that interest me as they come up. I'm calling it **Bitcoin Bits and Bobs**. Here is the [public repo](https://github.com/ursuscamp/bnb).

I thought I might begin by looking at the structure of a block, starting with the block header. I'm going to write a parser for a block header.

In order to parse it, I will use the Rust crate [nom](https://github.com/Geal/nom), a library of parser combinators. Parser combinators, if you're not aware, are small functions or objects that parse a very specific thing, and return success or failure, along with the parsed object and the remaining unparsed input.

In the case of `nom`, that return is an `IResult<I, O>`, where `I` is the remaining input, and `O` is the output. So when a parser returns `Ok(I, O)` it indicates a successful parse, and `Err` indicates a failure to parse. Since Bitcon is a largely Binary protocol, the input will be a slice of `u8`: `&[u8]`.

I chose the most recent block at the time of coding, block [76010](https://mempool.space/block/00000000000000000006ff82442b971512e1d7a5865c323a169ae8e238181430). Additionally, the [protocol documentation on the Bitcoin wiki](https://en.bitcoin.it/wiki/Protocol_documentation#Block_Headers) describes the structure of the block header.

Here is the basic structure described as a rust struct:

```rust
pub struct BlockHeader {
    version: i32,
    prev_block: [u8; 32],
    merkle_root: [u8; 32],
    timestamp: u32,
    bits: u32,
    nonce: u32,
}
```

So there are three datatypes we must be able to parse: `i32`, `u32` and a 32-byte bytestring. This will be fairly straightforward in `nom` as you will see. But first, let's get some test data.

Using blockchain explorers and the `bitcoin-cli`, I was able to figure out the necessary bits of data to test:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // https://mempool.space/block/00000000000000000006ff82442b971512e1d7a5865c323a169ae8e238181430
    // bitcoin-cli getblock "00000000000000000006ff82442b971512e1d7a5865c323a169ae8e238181430"
    static HEADER: &'static str = "04000020cc58a01f0c48627419b089b5020b658dcc1fd0e3c1a0000000000000000000009959dac9f575d922c975f275932f69342f921adc017159eef78fcd54184e1e7e9d8c566329a40717c5d73b67";
    static VERSION: i32 = 0x20000004;
    static PREV_BLOCK: &'static str =
        "00000000000000000000a0c1e3d01fcc8d650b02b589b0197462480c1fa058cc";
    static MERKLE_ROOT: &'static str =
        "7e1e4e1854cd8ff7ee597101dc1a922f34692f9375f275c922d975f5c9da5999";
    static TIMESTAMP: u32 = 0x63568C9D;
    static BITS: u32 = 0x1707a429;
    static NONCE: u32 = 0x673BD7C5;

    fn hb() -> Vec<u8> {
        hex::decode(HEADER).unwrap()
    }

    fn pbh() -> [u8; 32] {
        let mut h: [u8; 32] = hex::decode(PREV_BLOCK).unwrap().try_into().unwrap();
        h.reverse(); // Almost everything is in little endian so let's reverse the byte order of the hash
        h
    }

    fn mr() -> [u8; 32] {
        let mut m: [u8; 32] = hex::decode(MERKLE_ROOT).unwrap().try_into().unwrap();
        m.reverse();
        m
    }

    #[test]
    fn test_parse_block_header() {
        let hb = hb();
        let (input, bh) = parse_block_header(&hb).unwrap();
        assert!(input.is_empty());
        assert_eq!(
            bh,
            BlockHeader {
                version: VERSION,
                prev_block: pbh(),
                merkle_root: mr(),
                timestamp: TIMESTAMP,
                bits: BITS,
                nonce: NONCE
            }
        )
    }
}
```

`HEADER` is the hex value of the full binary block header. We can use the [hex](https://github.com/KokaKiwi/rust-hex) create to conveniently decode this. Also, notice in the convenience functions I wrote, I reversed the bytes to account for byte order in the block header vs. the hex representation from the blockchain explorer.

So, how can we parse the data? For parsing `i32` and `u32`, `nom` comes with handy built-in parsers for integers from binary data. In our case, we need `le_i32` and `le_u32` to parse integers from bytestrings in Little Endian byte order. Most things in Bitcoin are in Little Endian byte order.

If you are unaware of Endianness, you can read the [wiki page](https://en.wikipedia.org/wiki/Endianness) for more detailed information, but a quick summary is that little endian means least significant byte first, and big endian means most significant byte first. Our brains are used to thinking in big endian, so little endian can be tough to wrap your head around, but it's a very common byte order in computing.

Since `le_u32` and `le_i32` parsers are generic across inputs, we will wrap them in a convenience method just to make them easier to reason about without dealing with more type variables:

```rust
fn parse_i32(input: &[u8]) -> IResult<&[u8], i32> {
    le_i32(input)
}

fn parse_u32(input: &[u8]) -> IResult<&[u8], u32> {
    le_u32(input)
}
```

Parsing a bytestring into a fixed array for `[u8; 32]` is a just bit more tricky and we will combine multiple parsers for it:

```rust
fn parse_bytes(count: usize, input: &[u8]) -> IResult<&[u8], [u8; 32]> {
    map_res(take(count), |pb: &[u8]| pb.try_into())(input)
}
```

Breaking it down:

1. `take(count)` will take an exact number of bytes `count` from the input, and return a slice. If the exact number of bytes are not available, it will fail to parse.
2. `map_res` will take the input from the first parser (`take` in this case) and apply a fallible function to it. If the map function fails, it will wrap the inner `Err` into a parser error. In this case, that function attempts to fill a static array with the values from a slice. See this [stack overflow](https://stackoverflow.com/a/50080940) answer for some details on using the `TryInto` trait as seen here.

Finally, we can take these parser pieces and wrap them in a complete parser for a block header:

```rust
fn parse_block_header(input: &[u8]) -> IResult<&[u8], BlockHeader> {
    let (input, version) = parse_i32(input)?;
    let (input, prev_block) = parse_bytes(32, input)?;
    let (input, merkle_root) = parse_bytes(32, input)?;
    let (input, timestamp) = parse_u32(input)?;
    let (input, bits) = parse_u32(input)?;
    let (input, nonce) = parse_u32(input)?;
    Ok((
        input,
        BlockHeader {
            version,
            prev_block,
            merkle_root,
            timestamp,
            bits,
            nonce,
        },
    ))
}
```

You can see how incredibly convenient it is to make use of Rust's early failure operator `?` here. At every step, we parse the input, returning the remaining input, and the parsed value. This makes writing parsers exceedingly obvious and clear.

If you run the test above, it will pass!