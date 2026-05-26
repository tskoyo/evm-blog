---
title: 'Why Ethereum uses 256-bit integers (and how I built one in Rust)'
description: 'Building U256 from scratch taught me more about the EVM than any Solidity tutorial ever did.'
pubDate: 'May 25 2026'
---

A few weeks ago, I decided to learn how the Ethereum Virtual Machine really works under the hood. My main goal is to understand MEV attacks on a deeper level and figure out what mechanisms can be used to prevent them — so I started by exploring the EVM itself.

The first thing I learned is that we never store floats on-chain. My question was: why?

The reason is how `IEEE 754` floating-point works on every CPU. Here's an example in Rust:

```rust
fn main() {
    let a: f64 = 0.1;
    let b: f64 = 0.2;
    println!("{}", a + b);          // 0.30000000000000004
    println!("{}", a + b == 0.3);   // false
}
```

`IEEE 754` can't represent the majority of decimal numbers exactly in binary. This is a huge problem for blockchains — if two nodes are validating the same transaction, they might compute slightly different values, and consensus breaks. That would be catastrophic.

That's why the decision was made to use only integers. But this raises another question: how do you represent a decimal amount like `2.5 ETH` on-chain if you can only use integers?

The answer is wei — Ethereum's smallest unit. 1 ETH equals `10^18` wei. So `2.5 ETH` is calculated as `2.5 × 10^18 = 2_500_000_000_000_000_000`, and that integer is what gets stored on-chain.

- `1 ETH = 1_000_000_000_000_000_000 wei`
- `2.5 ETH = 2_500_000_000_000_000_000 wei`
- `0.001 ETH = 1_000_000_000_000_000 wei`

## Why 256 bits specifically

Now I had my answer for "why integers." But why such big integers?
A u64 maxes out at about `1.8 × 10^19`. That's barely enough to hold a single ETH whale's balance in wei. Some token amounts can be even larger. We need more bits.

But the deeper reason is this: Ethereum's word size matches its hash size.
The EVM uses a hash function called Keccak256. You feed it any input — a transaction, a contract's bytecode, a storage key — and it always spits out exactly 256 bits. Change one byte of the input and the output looks completely different. You can't reverse it, and it's practically impossible to find two inputs that produce the same output.

The EVM uses it everywhere. Contract addresses, storage slot locations, transaction IDs, Merkle trees in block headers. If something needs to be identified or looked up, there's a keccak256 call behind it.
By making the native integer type also 256 bits, a hash fits in one stack slot, one storage slot, one memory word. No splitting, no padding, no special cases.

Here's why that matters in practice. Every time someone swaps tokens on Uniswap, the contract needs to look up the pair's reserves. Those reserves live in a mapping. The EVM finds their storage location by hashing the token addresses with keccak256 — one hash, one 256-bit result, one slot read.
Now imagine the EVM used 128-bit words instead. That same hash would have to be split across two slots. Every reserve lookup would need two reads instead of one. Every write would take two writes. Extra logic to stitch the halves together.

On a busy day, Uniswap alone processes tens of thousands of swaps. Multiply that overhead across every mapping access in every contract, and the cost adds up fast.
That's the real reason U256 exists. Not just because balances can be large, but because the EVM's most fundamental operation produces 256-bit outputs. Everything is simpler when one hash equals one slot.



## Building U256 from scratch

There's no `u256` type in Rust, so I had to build one. The natural representation is 32 bytes:

```rust
pub struct U256(pub [u8; 32]);
```

Each byte holds a value from 0 to 255. Together, 32 bytes represent a single 256-bit number. The leftmost byte holds the most significant bits, the rightmost byte holds the least significant.

This is just base 256 — same as decimal, but each "digit" can be 0–255 instead of 0–9. For example, the number 665537 in base 256 is `[..., 10, 39, 193]`:

```
10 × 256² =  655360
39 × 256¹ =    9984
193 × 256⁰ =    193
            ───────
             665537 ✓
```

## The hard part: addition

Adding two U256s works the same way as long addition by hand — but in base 256. You start from the rightmost byte and carry any overflow leftward:

```rust
pub fn wrapping_add(self, other: Self) -> Self {
    let mut result = [0u8; 32];
    let mut carry: u16 = 0;

    for i in (0..32).rev() {
        let sum = self.0[i] as u16 + other.0[i] as u16 + carry;
        result[i] = sum as u8;
        carry = sum >> 8;
    }

    U256(result)
}
```

The trick is using `u16` to catch overflow. The maximum value of `sum` is `255 + 255 + 1 = 511`, which fits in a `u16` but not a `u8`. We keep the low 8 bits (`sum as u8`) as the result for this byte, and shift the rest right by 8 (`sum >> 8`) to get the carry — which is at most 1.

Why right to left? Same reason elementary school addition starts from the ones column: carries flow leftward, and you need to know what's coming from the right before you can compute the current column.

## What I learned

When I started this, I thought I'd be writing code all day. Instead, most of the day was spent asking *why*. Here's what I walked away with:

- why we only use integers on-chain
- what wei is
- what U256 is and how it's built
- what keccak256 is

Honestly, this exercise made me realize how much of "blockchain development" is just calling APIs without understanding what they do underneath. Building U256 myself was the first time I felt like I actually understood what an Ethereum transaction really *is*.

This is just the foundation. Next, I'll be implementing the actual EVM opcodes — ADD, MUL, SUB, the stack machine, memory, storage.