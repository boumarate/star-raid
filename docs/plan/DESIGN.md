# Star Raid: design

Companion to `SPEC.md`. **SPEC §16 (review round 1) overrides this file where they differ:** wall-fill-only counting, one seat per owner, a clone maker per raid, anchor modes, `expire`/`recoverWall`, the `Bind` signature, and LST markets instead of MON/USDC. Everything here is built against Kuru's mainnet MON/USDC market and Kuru
testnet, with the source at `Kuru-Labs/Kuru-contracts-dex-public@2060bb27`. File:line references are
to that commit.

## 1. Repo layout

```
contracts/            Foundry
  src/RaidVault.sol
  src/RaidRouter.sol
  src/SeatGate.sol
  src/lib/KuruBook.sol        wall status, tick rounding, interfaces
  src/lib/PriceAnchor.sol     Chainlink + Pyth + Kuru mid checks
  test/                       unit, fork (143), invariant
  script/                     deploy, post, open, close, settle
keeper/               TypeScript: schedule, start guard, close, settle retries
live/                 TypeScript: one WebSocket in, SSE out
indexer/              Envio HyperIndex (config.yaml, schema.graphql, handlers)
app/                  Next.js, Mera accounts
docs/plan/            SPEC, DESIGN, TICKETS without odds
```

## 2. Kuru facts the code relies on

| Fact | Where |
|---|---|
| `addSellOrder(uint32,uint96,bool)` takes base from the caller's margin balance; postOnly reverts on any match | `OrderBook.sol:254-267` |
| New order id = `s_orderIdCounter()` after the call | `:272-273` |
| Owner = `_msgSender()`; cancel requires owner | `:376`, `:531` |
| `batchCancelOrders` reverts on a filled or cancelled order | `:534-535` |
| Buy loop stops at the limit price, vault levels included | `:871` |
| Base credited to taker's margin balance | `:923` |
| Fully filled order keeps `size`; status from level head | `:604-622`, `:1155-1160` |
| Maker proceeds credited to maker margin, rounded down | `:1266` |
| Flip orders delete on fill; never use one for the wall | `:1180-1224`, `:1197` |
| `MarginAccount.deposit(user, token, amount)` (native: `token=0`, `msg.value`), `withdraw(amount, token)` to caller | `MarginAccount.sol:217-246` |

Events (one topic each, decode data):

| Event | topic0 |
|---|---|
| `Trade(uint40 orderId, address makerAddress, bool isBuy, uint256 price, uint96 updatedSize, address takerAddress, address txOrigin, uint96 filledSize)` | `0xf16924fb…1581` |
| `OrderCreated(uint40,address,uint96,uint32,bool)` | `0xb81bbaf1…f94c` |
| `OrderCanceled(uint40,address,uint32,uint96,bool)` | `0xd9c08981…1079` |

## 3. Contracts

### 3.1 `RaidVault`

One vault contract, many raids keyed by `raidId`. Holds the sponsor's base and bounty.

```solidity
struct Terms {
    address sponsor;
    address market;        // Kuru OrderBook
    address base;          // address(0) for native MON
    address quote;         // USDC
    uint96  wallSize;      // base, sizePrecision units
    uint128 bounty;        // USDC
    uint16  capBps;        // default 50
    uint128 targetQuote;   // USDC through the router, counted
    uint64  w0; uint64 w1; // block window
    uint32  minLen;        // E is drawn in [w0 + minLen, w1]
    uint32  hold;          // seconds after settle before claim
    uint128 seatCap;       // max counted USDC per seat
}
enum Status { Posted, Open, Closing, Closed, Settled, Aborted }
struct Raid {
    Terms   terms;
    Status  status;
    uint32  capPrice;      // Kuru price units
    uint40  wallId;
    uint64  endBlock;      // E, 0 until drawn
    uint64  settledAt;
    bool    won;
    uint128 countedTotal;  // Σ min(counted, seatCap)
}
```

Functions:
- `post(Terms)` payable: pulls base and bounty (plus any rollover credit of the sponsor), checks
  `w0 > block.number`, `w1 - w0 ≤ 400`, `capBps ∈ [20, 200]`, `bounty ≤ 10%` of the wall's value at
  post time. Emits `Posted`.
- `open(raidId)`, keeper only, `block.number ∈ [w0 - 10, w0]`:
  1. `PriceAnchor.mid()`: Chainlink ≤120 s stale, Pyth `getPriceNoOlderThan(id, 60)`, divergence ≤50
     bps, and Kuru `bestBidAsk` mid within 50 bps of the oracle mid. Otherwise `Aborted` and full
     refund.
  2. `capPrice = roundUpToTick(mid × (10000 + capBps) / 10000)`; while `s_sellPricePoints(capPrice).head
     != 0`, `capPrice += tickSize` (at most 5 steps, else abort). An empty level means nobody is ahead
     of us in FIFO at our price.
  3. `MarginAccount.deposit(address(this), base, wallSize)`; `addSellOrder(capPrice, wallSize, true)`;
     `wallId = s_orderIdCounter()`; assert `s_orders(wallId).owner == address(this)` and price.
  4. Emits `Opened(raidId, capPrice, wallId, mid)`.
- `close(raidId)` payable, anyone, `block.number > w1`: requests Pyth Entropy (fee 1.4 MON, paid from
  the fee reserve), status `Closing`. Callback `entropyCallback(seq, provider, rand)` sets
  `endBlock = w0 + minLen + rand % (w1 - w0 - minLen + 1)` and status `Closed`. If no callback in 200
  blocks, anyone can call `close` again (new request).
- `settle(raidId)`, anyone, status `Closed`:
  1. Wall status from the level head (DESIGN §2). Cancel only if active. If Kuru's admin cancelled it
     (`price == 0`) skip the cancel.
  2. Withdraw base and USDC proceeds using margin balance differences taken around the calls; send both
     to the sponsor.
  3. `won = router.countedTotal(raidId) ≥ targetQuote` (counted = buys with `block ≤ endBlock`, capped
     per seat). If lost, move the bounty to `rollover[sponsor]`.
  4. `settledAt = block.timestamp`. Emits `Settled(raidId, won, wallFilled, baseBack, quoteBack)`.
- `rollover[sponsor]` is only spendable by `post`. There is no withdraw for it.
- Admin: `setKeeper`, `pause` (blocks `post` and `open` only; never blocks `settle` or claims).

Invariants (tested):
- The sponsor can never receive the bounty of a raid that ran.
- Nothing can cancel the wall between `open` and `endBlock` except Kuru's admin.
- `settle` succeeds whether the wall is active, fully filled, partly filled or admin-cancelled.

### 3.2 `RaidRouter`

```solidity
function raid(uint256 raidId, uint128 quoteIn, Seat calldata seat) external;
```

1. Require status `Open`, `block.number ∈ [w0, w1]`, `seat` valid via `SeatGate.use(raidId, seat,
   msg.sender)` (first use binds the seat to the sender; later buys by the same sender reuse it).
2. `IERC20(quote).transferFrom(msg.sender, this, quoteIn)`; `MarginAccount.deposit(this, quote,
   quoteIn)`.
3. `size = quoteIn × sizePrecision × pricePrecision / (capPrice × quoteUnit)` rounded down; require
   `size ≥ minSize`.
4. Record `b0 = getBalance(this, base)`, `q0 = getBalance(this, quote)`, `c0 = s_orderIdCounter()`.
5. `addBuyOrder(capPrice, size, false)`.
6. If `s_orderIdCounter() > c0` and `s_orders(last).owner == this && isBuy && price == capPrice`,
   `batchCancelOrders([last])`.
7. `baseOut = getBalance(this, base) - b0`; `quoteSpent = quoteIn - (getBalance(this, quote) - q0 +
   …)`, computed from the balance after cancel.
8. Withdraw leftover quote and refund it to the sender. Keep `baseOut` escrowed in the router under
   `(raidId, seat)`.
9. Store `Buy{player, seat, block.number, baseOut, quoteSpent}`; update `seatSpent[raidId][seat]`.
   Emit `Raided(raidId, player, seatId, block.number, baseOut, quoteSpent)`.
10. Gas limit set by the app: 800k plus 40k per maker order it may cross.

Counting (view, used by `settle` and the UI): for each buy with `block ≤ endBlock`, add `quoteSpent`
to the seat; `counted(seat) = min(seatSum, seatCap)`; `countedTotal = Σ counted`.

Claims:
- `claim(raidId, seat)` after `settledAt + hold`: releases escrowed base; if won, pays
  `bounty × counted(seat) / countedTotal`.
- `exitEarly(raidId, seat)` any time after settle: releases base now, forfeits the bounty share (it
  goes to the next raid's rollover).
- Buys after `endBlock` still receive their base (they paid for it) but earn no bounty.

### 3.3 `SeatGate`

```solidity
struct Seat { uint8 kind; uint256 id; bytes proof; }   // kind 0 = Lil Stars, 1 = attestation
```
- Kind 0: `LilStars.ownerOf(id) == player` or delegate.xyz v1 `checkDelegateForToken(player, owner,
  LilStars, id)`. Nullifier `used[raidId][id]` binds the token id to the first player.
- Kind 1: EIP-712 `Human{player, provider, expiry}` signed by our verifier key after an off-chain
  check. Nullifier on the provider's unique id hash.
- Sponsor and its listed affiliates are rejected.

## 4. Keeper

- Schedules raids from posted terms. At `w0 - 20` it computes the start guard from its own trade feed
  (60 s mid range ≤20 bps, 5 min trade span ≤50 bps, last trade ≤300 blocks ago). If the guard fails,
  it slides `w0`/`w1` forward by up to 30 min through `reschedule` (sponsor pre-authorises a slide
  range in the terms) and posts the reason in the lobby.
- Calls `open` at `w0`, `close` right after `w1`, `settle` after the callback. Acts only on Finalized
  blocks.
- Holds MON for Entropy fees and gas. Alerts to the THRONE Telegram if a step fails twice.

## 5. Live data path

- `live/` keeps one WebSocket to `wss://rpc.monad.xyz` (fallback `rpc1`, `rpc3`), subscribes to
  `monadNewHeads` and `monadLogs` for the market and our contracts.
- Each head: one Multicall at `latest` for `s_orders(wallId)`, the level head, `getL2Book(5,5)`,
  `countedTotal`, and the router's buy count. Sends a frame over SSE to all viewers.
- Frames carry `state: proposed | finalized`; the UI renders proposed values lighter and firms them on
  Finalized (425-600 ms later).
- No viewer talks to the RPC directly (rate limits 15-25 req/s).

## 6. App screens

1. **Lobby:** next raids with sponsor, cap, target, bounty, window, hold, and the terms link.
2. **Join:** Mera passkey (one ceremony), seat pick (Lil Stars shown by art), USDC balance, choose
   amount. The confirm sheet shows the cap price, that cheaper asks fill first, the hold, early exit,
   and Kuru's admin powers, in plain words.
3. **Raid:** target counter, wall bar (proposed/finalized), raider feed with Lil Stars avatars, end
   window meter ("the end will be drawn between block X and Y"), buy button repeating the session.
4. **Draw:** at `w1` the screen waits for the Entropy callback, then shows `E` and greys out buys after
   it.
5. **Results:** won or lost, counted vs not, wall share vs cheaper asks, vault fills, split per seat,
   non-seat attempts refused.
6. **Claim:** hold countdown, claim, exit early.
7. **Terms:** "distribution event" terms, sponsor exclusion, Kuru admin disclosure, Reg M line.
8. **Sponsor console:** post terms, fund, see past raids.

## 7. Mera

- `@category-labs/mera` 0.2.0; one passkey ceremony creates the account; the address re-derives
  after storage is cleared (the stateless test costs one extra passkey prompt, rehearse it).
- The session key lives in memory for the raid. Scope in our code: the session signer only builds
  `approve(router, amount)` and `router.raid(...)` calls; the approved amount equals what the raider
  chose, so a leaked session cannot spend more.
- Accounts are funded with USDC plus a little MON for gas (at least 1.2 s before the raid). Gas
  sponsorship through 7702 + paymaster is optional (E5 stretch).

## 8. Envio

- HyperIndex, chain 143, hosted. `max_reorg_depth` low; realtime via `wss://rpc.monad.xyz`.
- Entities: `Raid`, `Seat`, `Buy`, `Player` (lifetime counted USDC, raids joined), `WallFill` (Kuru
  `Trade` with maker = vault), `Leaderboard` (derived per raid and all time).
- The UI uses Envio for history, leaderboards and the results page; the live bar never waits for it.

## 9. Tests

- **Fork tests on 143** against the real MON/USDC market: open places a wall at an empty level;
  router buy with cheaper asks present never fills above `capPrice`; overshoot leaves a resting bid
  that is cancelled in the same tx; attribution equals the margin difference; settle on active,
  filled, partial and admin-cancelled walls (impersonate the Safe for the last case).
- **Testnet market** created with `deployProxy` (not `MonadDeployer`, which seeds the AMM vault):
  full raid where the wall is the only ask.
- **Unit:** seat nullifier, delegate path, attestation expiry, counting with `E`, seat caps,
  rollover, exit early, claim after hold.
- **Invariant:** sponsor never receives a bounty back; Σ claims ≤ bounty; escrowed base equals Σ
  unclaimed `baseOut`.
- Each test must fail when the rule it names is removed (checked once per rule).

## 10. Deploy

- Testnet first, then mainnet. Verify every contract on Sourcify (`exact_match`) before any raid.
- Keeper and live service on Railway (new project), fixed domain.
- Budget: Entropy 1.4 MON per raid, keeper gas, bounties from our pocket for our own raids.
