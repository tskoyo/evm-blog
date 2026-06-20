---
title: "Use alloy to interact with the blockchain"
description: I came from JavaScript expecting to wrestle with ABI files. alloy's sol! macro does something I didn't see coming — it turns a smart contract into compile-time Rust types.
pubDate: 'Jun 13 2026'
heroImage: '../../assets/cargo-alloy.png'
---
 
In JavaScript, when you want to talk to a contract, you bring along its ABI — usually a chunk of JSON you import as a runtime value. The library reads that JSON and figures out how to encode your calls. It works, but the ABI is just *data*: if you typo a function name or pass the wrong argument type, nothing stops you until it blows up at runtime.
 
`alloy` does something very powerful. It gives you a macro called `sol!` where you paste the actual Solidity interface — the real function signatures, straight from the contract — and it generates **Rust types** from them at compile time. The contract's interface becomes part of the type system. Typo a function name and it won't compile. Pass the wrong type and it won't compile. The ABI still exists — alloy generates it for you from the Solidity — but you never touch it directly, and you get compile-time safety on top.
 
Here's what that looks like. I wanted to interact with a Uniswap V2 pool, so I pasted the parts of the [`IUniswapV2Pair` interface](https://github.com/Uniswap/v2-core/blob/master/contracts/interfaces/IUniswapV2Pair.sol) I cared about:
 
```rust
sol! {
    #[sol(rpc)]
    interface IUniswapV2Pair {
        function getReserves() public view returns (uint112 reserve0, uint112 reserve1, uint32 _blockTimestampLast);
        function token0() external view returns (address);
        function token1() external view returns (address);
    }
}
```
 
This doesn't run any Solidity. It mimics the interface — it generates the Rust types and the encoding logic needed to *call* those functions on a real deployed contract. The `#[sol(rpc)]` attribute is what tells alloy to generate the callable contract bindings (not just the type definitions). From here on, `getReserves()` and `token0()` are real methods I can call from Rust, type-checked.
 
## Connecting to a chain
 
To read anything, you need a connection to a node. alloy calls this a *provider*, and you build one with `ProviderBuilder`:
 
```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenv().ok();
    let rpc_url = std::env::var("ALCHEMY_RPC_URL")?;
    let provider = ProviderBuilder::new().connect(&rpc_url).await?;
}
```
 
The `rpc_url` is an endpoint from a node provider (I'm using Alchemy's free tier).`ProviderBuilder::new().connect(...)` opens the connection, and `.await?` because it's a network call.
 
## Reading from the pool
 
Now the payoff. I point the generated interface at a specific pool address and call its methods:
 
```rust
let pair_addr = address!("B4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc");
let pair = IUniswapV2Pair::new(pair_addr, &provider);
 
let token0 = pair.token0().call().await?;
let token1 = pair.token1().call().await?;
 
println!("Token0: {token0}");
println!("Token1: {token1}");
```
 
That address is the USDC/WETH pool on mainnet. `IUniswapV2Pair::new(addr, &provider)` ties the interface to that deployed contract through my connection, and then `pair.token0().call().await?` actually reaches out to the chain and reads the value.
  
```rust
let reserves = pair.getReserves().call().await?;
println!("reserve0: {}", reserves.reserve0);
println!("reserve1: {}", reserves.reserve1);
```
 
`getReserves` returns all three values from the Solidity signature — `reserve0`, `reserve1`, and the timestamp — as fields on a struct, named exactly as they were in the interface I pasted. That naming carrying through is the `sol!` macro paying off: I never decoded a return value by hand.
 
## What I learned
 
- alloy's `sol!` macro lets you paste real Solidity and get compile-time-checked Rust types — the ABI is generated for you, not pasted as runtime JSON like in JS
- `#[sol(rpc)]` is what turns those types into callable contract bindings
- you talk to a chain through a *provider*, built with `ProviderBuilder` and an RPC endpoint
- a deployed contract is `Interface::new(address, &provider)`, and from there calls are just `.method().call().await?`
  
 