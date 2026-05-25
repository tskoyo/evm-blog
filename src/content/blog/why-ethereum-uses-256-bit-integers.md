---
title: 'Why Ethereum uses 256-bit integers (and how I built one in Rust)'
description: 'Building U256 from scratch taught me more about the EVM than any Solidity tutorial ever did.'
pubDate: 'May 25 2026'
heroImage: '/blog-placeholder-1.jpg'
---

A few weeks ago I decided to learn how the Ethereum Virtual Machine actually works. Not the "call this function with ethers.js" level — the actual byte-level execution. I'm going to EthGlobal Lisbon in July and I want to build an MEV bot in Rust, which means I need to understand the EVM at a level most blockchain developers never reach.

The first thing I had to build was a 256-bit integer type. I assumed it would be a five-minute thing. It took me a full day.

Here's what I learned, and why it matters.

## Why blockchains can't use floating-point numbers

When I first started, I asked the obvious question: why does Ethereum need a 256-bit integer at all? Rust already has `u64` and `u128`. Why reinvent the wheel?

The answer starts with a more fundamental question: **why can't blockchains use floats?**

Try this in Rust:

​```rust
fn main() {
    let a: f64 = 0.1;
    let b: f64 = 0.2;
    println!("{}", a + b);          // 0.30000000000000004
    println!("{}", a + b == 0.3);   // false
}
​```

This isn't a Rust bug. It's how IEEE 754 floating-point works on every CPU in the world. Most decimal numbers can't be represented exactly in binary, so you get rounding errors.

For a banking app, this is annoying. For a blockchain, it's catastrophic. Imagine a smart contract that pays out when `balance == expected_amount`. If floating-point math gives you `0.30000000000000004` instead of `0.3`, the condition silently fails. Even worse, two nodes running the same transaction on different CPU architectures might get different rounding results — and now your blockchain nodes disagree on the outcome. Consensus breaks.

The fix is simple: **never use floats. Only integers.**

## How Ethereum represents ETH amounts

If you can only use integers, how do you represent 2.5 ETH?

You don't. You define the smallest unit and only ever count that.

In Ethereum, the smallest unit is `1 wei`. There are `10^18` wei in 1 ETH. So:

- `1 ETH    = 1_000_000_000_000_000_000 wei`
- `2.5 ETH  = 2_500_000_000_000_000_000 wei`
- `0.001 ETH = 1_000_000_000_000_000 wei`

Every calculation on-chain happens in wei, as a plain integer. When MetaMask shows you "2.5 ETH", it's just dividing the raw wei value by `10^18` for display. The chain itself never sees the decimal.

Same idea for ERC-20 tokens. USDC has 6 decimal places, so `1 USDC` is stored on-chain as `1_000_000`. The `decimals()` function on the contract just tells wallets how many places to shift for display.

## Why 256 bits specifically

Now I had my answer for "why integers." But why such big integers?

A `u64` maxes out at about `1.8 × 10^19`. That's barely enough to hold a single ETH whale's balance in wei. Some token amounts can be even larger. We need more bits.

But the deeper reason is this: **Ethereum's word size matches its hash size.**

Keccak256, the hash function the EVM uses everywhere, produces 256-bit outputs. By making the native integer type also 256 bits, you can store a hash in one stack slot, one storage slot, one memory word. This makes the EVM's internal representation symmetric and clean.

Addresses being 160-bit and balances being arbitrarily large are nice bonuses, but the hash-size match is the main driver.

## Building U256 from scratch

There's no `u256` type in Rust. So I had to build one.

The natural representation is 32 bytes:

​```rust
pub struct U256(pub [u8; 32]);
​```

Each byte holds a value from 0 to 255. Together, 32 bytes represent a single 256-bit number. The leftmost byte holds the most significant bits, the rightmost byte holds the least significant.

This is just base 256. Same as decimal, but each "digit" can be 0–255 instead of 0–9. The number 665537 in base 256 is `[..., 10, 39, 193]`:

​```
10  × 256² = 655360
39  × 256¹ =   9984
193 × 256⁰ =    193
              ──────
              665537 ✓
​```

## The hard part: addition

Adding two U256s is the same as long addition by hand, but in base 256. You start from the rightmost byte and carry any overflow leftward:

​```rust
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
​```

The trick is using `u16` to catch overflow. The maximum value of `sum` is `255 + 255 + 1 = 511`, which fits in a `u16` but not a `u8`. We keep the low 8 bits (`sum as u8`) as the result for this byte, and shift the rest right by 8 (`sum >> 8`) to get the carry — which is at most 1.

Why right to left? Same reason elementary school addition starts from the ones column: carries flow leftward, and you need to know what's coming from the right before you can compute the current column.

## What this taught me

When I started, I thought I'd spend an afternoon on this. I spent a full day.

Most of that day wasn't writing code. It was understanding **why**:

- Why integers, not floats
- Why 256 bits specifically
- Why we store ETH as wei
- Why we work in base 256 with 32 bytes
- Why we propagate carries right-to-left
- Why `wrapping_add` exists at all instead of just `add`

That last one is interesting. Solidity used to silently wrap on overflow — meaning `0 - 1` would give you a number close to `2^256`, which led to a long line of exploits. That's why every old Solidity codebase imports `SafeMath`, and why Solidity 0.8+ panics on overflow by default.

Understanding `wrapping_add` and `wrapping_sub` from scratch made me understand exactly why those exploits worked. The U256 doesn't know what "negative" means — it just wraps around like a clock. Subtracting 1 from 0 gives you `U256::MAX`. If you don't explicitly check for underflow before subtracting, that's a bug waiting to be exploited.

## What's next

This is the first post in a series I'm writing as I build a toy EVM interpreter in Rust, with the goal of building an MEV bot at EthGlobal Lisbon in July.

Next up: implementing the actual EVM opcodes — ADD, MUL, SUB, and the stack machine that ties them together. From there: memory, storage, control flow, and eventually using `revm` to simulate real Uniswap swaps locally.

The code so far is on GitHub: [tskoyo/toy-evm](https://github.com/tskoyo/toy-evm)