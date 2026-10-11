# Kolan

**A token on an ordinary Uniswap v4 pool, where a purchase can be undone until
the window the buyer paid for ends.**

[Website](https://kolan.c0llback.com) · [Docs](https://kolan.c0llback.com/docs) ·
[X](https://x.com/kolan_protocol)

A Uniswap purchase is final the moment it lands: if the price falls afterwards,
the buyer takes the whole fall. Kolan lets a buyer pay for a window instead.
Until it ends, the purchase can be undone, and every wei of the ETH spent on the
tokens comes back. The buyer's worst case is known before they buy: the premium
and the tax.

---

## How it works

### The pool

The token's whole supply sits in a Uniswap v4 pool, native ETH against KLN, and
the hook is the pool's only liquidity provider. Nobody else can add liquidity,
donate or open another pool on the hook, and no function hands the liquidity to
anyone.

Every swap is a plain v4 swap, priced by v4's own math, so the Uniswap
interface, aggregators and DexScreener see real liquidity and a real price.

### The tax

1% of the ETH side of every swap, buy or sell, exact input or exact output, and
of what a window spends. It is booked as ERC-6909 claims during the swap and paid
to the dev, in ETH, by `claimDev`. It is the only thing the dev ever receives.
The hook's own buybacks are not taxed.

### Opening a window

`buy` takes out of the pool the tokens a swap of the same ETH would buy now — the
next steps of liquidity past the price — and holds them with the ETH.

The pool is left with a hole there. A buyer crosses it for free and pays the
price after it, as if the purchase had happened. A seller cannot reach the
window's ETH, because it is not in the pool.

### Closing it

- `cancel(id)`: the owner, before expiry. Every wei of the ETH goes back to them,
  and the tokens go back into the pool.
- `accept(id)`: the owner at any time, or anyone once the window has expired.
  The tokens go to the owner, and the ETH goes into the pool.

Where things go back:

- tokens, on `cancel`: into the hole if the price is still at or above it,
  otherwise over the same width just past the current price, where the next
  buyer takes them first;
- ETH, on `accept`: into the hole if the price has gone past it, which leaves the
  pool exactly as after a plain buy; otherwise over the same width just below the
  current price, as support for sellers.

Nothing is ever burned in place of going back: if a cancelled window's tokens
were burned instead, anyone could remove supply from the pool for the price of a
premium.

### The premium

Paid on top of the purchase, never refunded. The window lasts
`6 h × premium / 0.05 ETH`, from 30 minutes to 7 days; past 7 days the excess
premium is refunded. The premium may be at most 30% of the ETH the purchase
spends.

When the window closes, after the tokens or the ETH are back, the premium buys
tokens from the pool and those tokens are burned.

### The liquidity book

The hook keeps its own record of its positions: non-overlapping ranges, each one
v4 position, chained in tick order, with a hint pointing near the last change so
a lookup only walks the few ranges around the price. Every change goes through
two operations: carve (open a window) and add (put tokens or ETH back).

## Parameters

| | |
|---|---|
| Supply | 1,000,000 KLN, fixed. Nothing can mint more; anyone can burn their own. |
| Pool | native ETH / KLN, LP fee 0, tick spacing 10 |
| Tax | 1% of the ETH side |
| Window | 6 hours per 0.05 ETH of premium |
| Window bounds | 30 minutes to 7 days |
| Premium cap | 30% of the ETH the purchase spends |

All are immutables of the hook: no function can change them.

## Contracts

```
src/
  KolanHook.sol       the Uniswap v4 hook: the tax, the windows
  LiquidityBook.sol   the hook's record of its positions, and the two operations
                      that change them: carve and add
  KolanToken.sol      a plain ERC-20, fixed supply, burnable
```

Where to look first: [`LiquidityBook._plan`](src/LiquidityBook.sol) is where a
window decides what to take out of the pool, and
[`KolanHook._closeWindow`](src/KolanHook.sol) where it decides where things go
back.

## Interface

### Writing

| Function | Who | What it does |
|---|---|---|
| `buy(recipient, premium, minTokens, deadline)` | anyone | Opens a window for `recipient`. Returns `(id, tokens)`. |
| `cancel(id)` | the owner, before expiry | Every wei of the window's ETH back to the owner, the tokens back into the pool. |
| `accept(id)` | the owner any time, anyone after expiry | The tokens to the owner, the ETH into the pool. |
| `claimDev()` | anyone | Pays the tax collected so far to `DEV`. |

`buy` takes as value the ETH to spend, plus the tax on it, the premium and a
1,000 wei rounding allowance (`ROUNDING_ALLOWANCE`):

```
msg.value = eth + eth × TAX_BPS / 10_000 + premium + 1_000
```

Whatever the window does not use is refunded in the same transaction. `minTokens`
and `deadline` protect against the price moving before the transaction lands.

### Reading

| Function | Returns |
|---|---|
| `quote(eth)` | the tokens a window spending `eth` would hold, and the ETH it would actually spend |
| `windowFor(premium)` | the window a premium buys, in seconds, before the bounds apply |
| `windows(id)` | owner, expiry, number of cuts, ETH, tokens, premium |
| `cutsOf(id)` | the ranges the window took out of the pool |
| `held()` | the ETH and tokens held for all open windows |
| `devOwed()` | the tax collected and not yet claimed |
| `poolKey()`, `poolId()` | the pool |

### Events

`Taxed`, `DevPaid`, `WindowOpened`, `WindowClosed`.

## What the code guarantees

- **No owner.** Nothing is upgradeable, pausable or ownable, and no function
  changes a parameter. `DEV` only receives the tax.
- **Locked liquidity.** `beforeAddLiquidity`, `beforeDonate` and
  `beforeInitialize` revert for anyone but the hook. The only liquidity the hook
  ever takes out is a window's tokens, which go either to the buyer who paid for
  them or back into the pool.
- **Solvency.** The ETH and tokens of open windows are tracked in `held`, and
  every `buy`, `cancel` and `accept` ends by checking the hook still holds at
  least that much.
- **Exact refunds.** `cancel` returns exactly the ETH the window spent on tokens.
  The tax and the premium are not refunded.
- **Bounded work.** A window touches at most 16 ranges of the book; past that it
  is filled in part and the rest of the ETH refunded.

## License

Business Source License 1.1 (`BUSL-1.1`), with no additional use grant. It
converts to MIT on 2030-09-23.
