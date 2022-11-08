---
layout: post
title:  "Bitcoin Bits and Bobs 4: VarInt"
date: 2022-11-07 21:14:49 -0500
categories: bitcoin rust
---

In the interest of continuing to work on parsing/encoding/decoding code, today we're looking into VarInt. If you read the [protocol documentation](https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer) on the wiki, you might be confused by the mention of multiple "variable length integer" encodings, one which used in the P2P protocol which is called CompactSize by Bitcoin Core, and another which is called VarInt by Bitcoin Core. Today we are working on CompactSize, which is frequently called VarInt everywhere else outside of Bitcoin Core. It is confusing, it's not just you. This [StackExchange question](https://bitcoin.stackexchange.com/questions/114584/what-is-the-different-between-compactsize-and-varint-encoding) offers some useful information to help you differentiate.

So, with that out of the way, on with the show. Typically, in memory or on disk, an integer is encoded as a number of bytes. For example, in Rust, a `u16` value is 16 bits, or two bytes. It's always stored as two bytes. A `u64` is always stored as 8 bytes. What if you want to have the option of storing numbers up to 8 bytes long, but most numbers will be much smaller than that. Can we save space, somehow? That's where variable length encodings come into play.

In VarInt, any number `<= 0xfc` (252 in decimal) is stored as a single byte. Any number `<= 0xffff` is stored as `0xfd0000`. Any number `<= 0xffffff` is stored as `0xfe00000000`. Any number higher than that is stored `0xff0000000000000000`. In all such cases the null bytes above are actually replaced by 16 bit, 32 bit or 64 bit integer representations in Little Endian byte order. For example `0x1234` would be encoded as `0xfd3412`. You can see, this gives the protocol the option to represent numbers up to 64 bits without having to send 8 bytes across the wire for every single number.

Here is the shape of our code:

```rust
pub fn encode(num: u64) -> Vec<u8> {
    todo!()
}

pub fn decode(bytes: &[u8]) -> IResult<&[u8], u64> {
    todo!()
}
```

Encoding is the most simple, we just take any number (in this case a u64) and output a byte array, or `Vec` in Rust parlance. Decoding has the option to fail (for instance, if the provided byte slice is too short), so it returns a `Result` value. In this case, an `IResult` from the `nom` crate, covered in a [previous entry](2022-10-25-bitcoin-bits-and-bobs-1.md) in this series.

It's also fairly easy to come up with some test cases for this. Here they are:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_encode() {
        assert_eq!(encode(0xfc), vec![0xfc]);
        assert_eq!(encode(0xfd), vec![0xfd, 0xfd, 0x00]);
        assert_eq!(encode(0x1234), vec![0xfd, 0x34, 0x12]);
        assert_eq!(encode(0x0226), vec![0xfd, 0x26, 0x02]);
        assert_eq!(encode(0x000f3a70), vec![0xfe, 0x70, 0x3a, 0x0f, 0x00]);
        assert_eq!(
            encode(0xfffffffffffffffe),
            vec![0xff, 0xfe, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff,]
        );
    }

    #[test]
    fn test_decode() {
        assert_eq!(decode(&[0xfc]).unwrap().1, 0xfc);
        assert_eq!(decode(&[0xfd, 0xfd, 0x00]).unwrap().1, 0xfd);
        assert_eq!(decode(&[0xfd, 0x34, 0x12]).unwrap().1, 0x1234);
        assert_eq!(decode(&[0xfd, 0x26, 0x02]).unwrap().1, 0x0226);
        assert_eq!(
            decode(&[0xfe, 0x70, 0x3a, 0x0f, 0x00]).unwrap().1,
            0x000f3a70
        );
        assert_eq!(
            decode(&[0xff, 0xfe, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0x01]).unwrap(),
            (vec![0x01_u8].as_ref(), 0xfffffffffffffffe_u64)
        );
        assert!(decode(&[0xff, 0xfe]).is_err());
    }
}
```

We are using the same examples for both encoding and decoding. We are testing `encode` and `decode` but the examples are reversed, of course. Additionally, when testing decoding, we also need to test the failure state if the input data is too short.

# Encoding

So how do we encode?

```rust
pub fn encode(num: u64) -> Vec<u8> {
    let mut output = Vec::with_capacity(9);
    if num <= 0xfc {
        let sl = (num as u8).to_le_bytes();
        output.extend_from_slice(&sl);
    } else if num <= 0xffff {
        output.push(0xfd);
        let sl = (num as u16).to_le_bytes();
        output.extend_from_slice(&sl);
    } else if num <= 0xffffffff {
        output.push(0xfe);
        let sl = (num as u32).to_le_bytes();
        output.extend_from_slice(&sl);
    } else {
        output.push(0xff);
        let sl = num.to_le_bytes();
        output.extend_from_slice(&sl);
    }
    output
}
```

We create a `Vec` with the necessary maximum capacity. We test each threshold of the variable length encoding, and use Rust's handy built-in `to_le_bytes()` integer method to convert each integer into it's Little Ending byte string. Really couldn't be much simpler. We push the tag byte onto the `Vec`, then extend it with the byte slice we created with `to_le_bytes`.

# Decoding

This is more complex due to parsing-related messiness. We'll define some helper methods to break this into easily digestible parts.

```rust
fn get_tag(bytes: &[u8]) -> IResult<&[u8], u8> {
    let (bytes, t) = take(1usize)(bytes)?;
    Ok((bytes, t[0]))
}
```

`get_tag` parses the first byte, returning the input minus one byte, and the parsed byte. This method, and all helper methods after this, will fail if the input data is too short.

Next, we create helper methods to parse 1 byte, 2 byte, 4 byte and 8 byte integers:

```rust
fn decode_1b(bytes: &[u8]) -> IResult<&[u8], u64> {
    let (bytes, t) = get_tag(bytes)?;
    if t <= 0xfc {
        return Ok((bytes, t as u64));
    }
    Err(nom::Err::Error(make_error(bytes, ErrorKind::Tag)))
}

fn decode_2b(bytes: &[u8]) -> IResult<&[u8], u64> {
    let (bytes, t) = get_tag(bytes)?;
    if t == 0xfd {
        return le_u16(bytes).map(|(i, o)| (i, o as u64));
    }
    Err(nom::Err::Error(make_error(bytes, ErrorKind::LengthValue)))
}

fn decode_4b(bytes: &[u8]) -> IResult<&[u8], u64> {
    let (bytes, t) = get_tag(bytes)?;
    if t == 0xfe {
        return le_u32(bytes).map(|(i, o)| (i, o as u64));
    }
    Err(nom::Err::Error(make_error(bytes, ErrorKind::LengthValue)))
}

fn decode_8b(bytes: &[u8]) -> IResult<&[u8], u64> {
    let (bytes, t) = get_tag(bytes)?;
    if t == 0xff {
        return le_u64(bytes);
    }
    Err(nom::Err::Error(make_error(bytes, ErrorKind::LengthValue)))
}
```

These each more or less follow the same formula. Call `get_tag` and check if it's the expected value for that length. If it is, use a `nom` parser combinator (such as `le_u16`) to parse a correct sized integer from the input data, then upsize that integer to a `u64`. If the correct tag value is not found, return an error instead.

Finally, we can write a proper `decode` method, which is now actually very simple:

```rust
pub fn decode(bytes: &[u8]) -> IResult<&[u8], u64> {
    nom::branch::alt((decode_1b, decode_2b, decode_4b, decode_8b))(bytes)
}
```

We use nom's `alt` method to try each of our combinators in order, returning the first one that successfully parses.

Now we can run our tests and they all pass!

# Conclusion

These little encoding-type tasks are actually quite fun to code! I will certainly look for more options like this in the future while continuing to explore the Bitcoin protocol.