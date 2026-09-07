# PowerPerp

**A pool with a weight, so a liquidity provider chooses how much of the move they want instead of always getting the square root of it.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://power-perp.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/PowerPerpHook.sol`](src/hooks/PowerPerpHook.sol)
- **Licence:** Apache-2.0

## How it works

Providing to a constant-product pool is a position, and almost nobody picks it deliberately. Holding `x*y = k` against a price move gives a payoff proportional to the square root of the price, always, with no way to ask for more or less. A provider who is bullish and wants most of the upside, and a provider who wants to sit closer to stable, are handed the identical exposure and told it is a fee opportunity.

The generalisation has been known since Balancer: hold the weighted geometric mean constant, x^w * y^(1-w) = k and the position's value moves with `price^(1-w)`. 5` that is the square root every constant-product pool gives. 8` the pool holds mostly currency0 and the provider keeps most of its move.

2` the pool is mostly currency1 and the provider is nearly flat to currency0. One parameter, and the provider's exposure is something they chose. The swap maths follows from the invariant.

18 fixed point. The exponent is a real number, not an integer, so this needs genuine `pow`; it uses PRB Math rather than an approximation, because a curve that is subtly wrong in the tail is a curve that pays somebody to find the tail. The reason this is worth having on v4 specifically: v4 pools are concentrated-liquidity pools, and concentration and weighting are different tools.

Concentration says where you are willing to quote. Weighting says what exposure you want while you quote. Nothing in v4 offers the second, and a hook that replaces the curve is the only place it can live.

## Prior art

Weighted geometric-mean pools are Balancer's, and have been since 2020. The Squared and power-perpetual submissions for v4 build leveraged products on top of a pool rather than changing the pool's own curve. Bringing the weighted invariant to v4 as a custom curve, so a provider can pick their exposure to the pair they are quoting, is the contribution here; the maths is deliberately the well-understood one rather than something new.

## Where it does not help

A weighted pool is more exposed to one side by construction, so a provider who picks a high weight and is wrong about direction loses more than a constant-product provider would. That is the point of the parameter, not a defect, but it does mean the weight is a directional view and should be chosen as one. Gas is also higher than a constant-product pool: every quote evaluates a real-exponent `pow`, which costs a few thousand gas more than a multiply.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyInitialized()` | Hook was already initialized. |
| `AmountTooSmall()` | A deposit was too small to mint any shares, or a withdrawal too small to return anything. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExpiredPastDeadline()` | A liquidity modification order was attempted to be executed after the deadline. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InsufficientReserves()` | The pool cannot fill this swap without emptying the side being bought. |
| `InvalidFee()` | The fee must be below 100%. |
| `InvalidNativePayer(address)` | The native currency was settled on behalf of a `payer` other than the contract paying it. |
| `InvalidNativeValue()` | Native currency was not sent with the correct amount. |
| `InvalidWeight()` | The weight must leave both sides a real share; a pool weighted entirely to one side is not a pool. |
| `LiquidityOnlyViaHook()` | Liquidity was attempted to be added or removed via the `PoolManager` instead of the hook. |
| `NoLiquidity()` | A quote was requested against an empty pool. |
| `PoolNotInitialized()` | Pool was not initialized. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `TooMuchSlippage()` | Principal delta of liquidity modification resulted in too much slippage. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # PowerPerp
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, custom-curve, weighted, lp-economics, exposure
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/power-perp
cd power-perp
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
