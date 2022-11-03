---
layout: post
title:  "Bitcoin Bits and Bobs 3: Base58"
date: 2022-11-02 21:09:21 -0400
categories: bitcoin rust
---

Another subject I found interesting is [Base58](https://en.bitcoin.it/wiki/Base58Check_encoding), which is a special byte encoding scheme, similar to Base62, with a few changes. Just a quick summary, a character encoding like Base62 allows you represent number using more than just decimal (i.e. using more digits than just 0-9).

You may have heard of hexadecimal, which uses Base16 encoding (0-9 plus A-F representing 10-15). Base62 uses 0-9 + a-z + A-Z. In this case the letters represent different numbers enirely ranging all the way up from 10 - 61. Use a number base higher than 10 allows you to represent a number using fewer text characters, which helps with storing and typing Bitcoin addresses.

Bitcoin addresses use Base58, which is Base62 with a slight modification: characters that look similar (O and 0, l and I) are not included in the character alphabet. This makes it harder to misread and mistype Bitcoin addreses.

So, I want to write some code to encode normal, positive decimal integers into Base58, and to be able to decode again. I will be using the very helpful reference on [Learn Me A Bitcoin](https://learnmeabitcoin.com/technical/base58) to go along with this.

Let's begin by defining the alphabet:

```rust
static B58_ALPHABET: &[u8] = b"123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
```

Okay, so the easy place to start is encoding a normal integer as a Base58 string. Why is this easy? Because it can't fail. Decoding a string has a potential for failure: you can pass invalid string data. We'll handle that last.

Here's how you encode:

```rust
pub fn encode(num: u64) -> String {
    // Reserve a byte vector of some minimal but usually sufficient amount
    let mut output: Vec<u8> = Vec::with_capacity(10);

    let mut num = num;
    while num > 0 {
        let idx = num % 58;
        output.push(B58_ALPHABET[idx as usize]);
        num = num / 58;
    }

    // Handle special case
    if output.is_empty() {
        output.push(b'1'); // 1 in Base58 is actually 0
    }

    // Reverse to correct byte order
    output.reverse();

    // Convert to string
    String::from_utf8(output).expect("This should never fail")
}
```

So what's happening here? The meat of the method is the loop. It takes the input number modulus 58, which gives us the index of the character in alphabet to include, which it pushes to the vector. Following that, it does integer division of the input number by 58, and that becomes the next number in the iteration. Repeat until there are no more numbers to factor.

You're left with a byte array that represents the string (but in reverse). So we reverse the vector and return it as a string.

Now let's write some tests to verify:

```rust
#[test]
fn test_encode() {
    let test_cases = [(0, "1"), (20, "M"), (9999, "3yQ")];

    for (inp, out) in test_cases {
        assert_eq!(encode(inp), out);
    }
}
```

Alright, so now onto the slightly trickier side of the operation: decoding a Base58 string into an integer. As mentioned above, this is a fallible operation, since not every string will be a valid Base58 number. So to begin, let's define an error type that we can use for the various ways decoding can fail:

```rust
#[derive(Debug, PartialEq, Eq)]
pub struct Base58Error;

impl std::fmt::Display for Base58Error {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("Unparseable base58 value")
    }
}

impl std::error::Error for Base58Error {}
```

Now that we have our error type, we can define some helper functions.

```rust
fn b58pos(byte: u8) -> Result<usize, Base58Error> {
    B58_ALPHABET
        .iter()
        .position(|d| *d == byte)
        .ok_or(Base58Error)
}

fn decode_factor((idx, byte): (usize, u8)) -> Result<u64, Base58Error> {
    let pos = b58pos(byte)?;
    let exp: u32 = idx.try_into().or(Err(Base58Error))?;
    let factor = pos * 58usize.pow(exp);
    Ok(factor as u64)
}
```

`b58pos` take a give byte (e.g. `'Q'`) and returns it's position in the Base58 alphabet. In the event it doesn't find it (an invalid byte was passed), it returns `Base58Error` we created above.

`decode_factor` takes an index and a byte as a tuple (you will see why soon). It uses the byte to locate the position in the Base58 alphabet. It multiples that position number by 58 raised to the power of the index from the given string. This function calculates one digit of the Base58 string.

We can finally compose the true `decode` function:

```rust
pub fn decode(inp: &str) -> Result<u64, Base58Error> {
    inp.bytes().rev().enumerate().map(decode_factor).sum()
}
```

Given an input string, it converts it into bytes, then reverses it, then iterates over the bytes values with an index (this is where `(idx, byte): (usize, u8)` come in from above), then maps the `decode_factor` across the values. This effectively maps `decode_factor` across every byte, returning the expected `u64`, finally summing them up for the final answer.

Let's write some test!

```rust
#[test]
fn test_decode() {
    let test_cases = [(0, "1"), (20, "M"), (9999, "3yQ")];
    for (out, inp) in test_cases {
        assert_eq!(decode(inp).unwrap(), out);
    }
}

#[test]
fn test_invalid_decode() {
    assert_eq!(decode("0OIl"), Err(Base58Error));
}
```

Of course, we make sure to valid and invalid strings while parsing.

And that's it! That's how encode and decode a basic Base58 number. Note: This is the not the same as encoding a bitcoin address, which is a bit more involved than just converting to Base58.