# AGENTS.md: Star Raid (Monad Metropolis 2026, Track 01)

Copy to the repo root at `git init`. Every coding agent reads this first.

## What this is
A sponsor sets a price cap and a USDC prize; verified humans (one Lil Stars token id = one seat per
raid) buy the sponsor's token together on Kuru's on-chain order book for about a minute; the end
block is drawn by Pyth Entropy after the window; the prize goes only to seats, after a hold. Plan:
`docs/plan/SPEC.md`, design: `docs/plan/DESIGN.md`, tickets: `docs/plan/TICKETS.md`. Work one epic
per branch and one ticket at a time.

## Hard rules
1. **Metropolis requires disclosing AI use.** After each ticket, append to `AI_USAGE.md`: ticket,
   files written or changed, what the human reviewed.
2. **Many small commits, short messages.** Low diff each, several per ticket, never squashed, no
   attribution trailers, SSH-signed.
3. **Kuru facts come from source, not docs.** The docs are wrong in places (event layout, argument
   order). Check `Kuru-Labs/Kuru-contracts-dex-public@2060bb27` and cite the line in the PR.
4. **Never read an order's `size` to decide if it is live.** Status comes from the price level head
   (`price == 0` cancelled; `head > id || head == 0` filled).
5. **Every raid buy is `addBuyOrder(capPrice, size, false)` then cancel of any resting remainder in
   the same tx.** Never `placeAndExecuteMarketBuy`: it has no per-level price cap.
6. **Attribution is by margin balance difference around our own calls**, never from `Trade` logs.
7. **`batchCancelOrders` reverts on a filled or cancelled order.** Check status before every cancel.
8. **Never use `block.prevrandao` or a future blockhash for anything that decides money.** The end
   block comes from Pyth Entropy only.
9. **Raiders pay in USDC.** Never design a flow that spends a raider's native MON beyond gas
   (Monad reserve balance).
10. **Set explicit gas limits.** Monad bills the limit. Player tx 800k plus 40k per extra maker.
11. **Tests must fail for the right reason.** Each rule above has a test that fails when the rule is
    removed and asserts the specific custom error. Fork tests run against chain 143.
12. **Mainnet reads for money are `finalized`.** The live UI may show `latest` as tentative.
13. **Never state a number the chain does not show.** The results page, the share card and the
    README report counted buys, wall share and refused attempts as measured; no price, no PnL.
14. **No secrets in the repo.** Env vars only; `.env` gitignored; never print keys, RPC tokens or the
    HyperSync token.
15. **Testnet keys only for development.** Mainnet deploys and raids happen only on his explicit go.
16. Library APIs are checked against current docs (Context7) before use. Anything marked
    **CONFIRM** in the plan is resolved and recorded in `docs/plan/decisions.md`.
17. Lil Stars art is pre-existing work of the Lil Stars team; credit it in the README and never
    alter the characters.

## Commands
- `cd contracts && forge build && forge test -vv`
- `forge test --fork-url $MONAD_RPC --match-path test/fork/*` (chain 143)
- `pnpm dev`, `pnpm build`, `pnpm test`
- `pnpm keeper`, `pnpm live`
- `cd indexer && pnpm envio dev`

## Layout
- `contracts/` Foundry: `RaidVault`, `RaidRouter`, `SeatGate`, `lib/KuruBook`, `lib/PriceAnchor`
- `keeper/` schedule, start guard, open, close, settle
- `live/` one WebSocket in, SSE out
- `indexer/` Envio HyperIndex
- `app/` Next.js: lobby, join, raid, draw, results, claim, terms, `r/[raid]/[seat]` share page with
  `opengraph-image.tsx`
- `docs/plan/` the plan; `docs/plan/decisions.md` for choices made during the build
