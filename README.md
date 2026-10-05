# Star Raid

A project puts its token on a locked wall at a fixed price, and verified holders buy it out together
in about a minute on Kuru's on-chain order book. Only what the wall sells counts toward the prize, so
bots, wash trades and farms cannot collect.

Built for Monad Metropolis 2026, Track 01 (Onchain Finance & Trading).

## How a raid works

A sponsor posts a wall: an amount of its token, a price cap, a prize, and a window. Verified holders
buy into that wall. A raider never pays more than the cap, because the router refuses to fill above
it. The window ends on a block drawn by Pyth Entropy after the fact, so nobody can time the close.
The prize is split by how much each seat bought **from the sponsor's wall**, not from any other ask,
and it pays out after a hold.

One Lil Stars token id is one seat per raid. That is the whole sybil defence: seats are scarce and
named, so a farm cannot multiply itself into the payout.

## Status

Nothing is deployed. This repo is the skeleton: plan, tickets, and the rules an agent works under.
Addresses and verified contracts will be listed here when they exist, and not before.

## Plan

- `docs/plan/DESIGN.md` — the shape of the system
- `docs/plan/TICKETS.md` — the epics and their tickets
- `AGENTS.md` — the rules every coding agent reads first
- `AI_USAGE.md` — where AI was used and where it was not

The spec itself is kept outside this repo, because it carries estimates that are nobody's business
but the builder's.

## Safety

This is hackathon code and none of it has been audited. Do not put money into it expecting the
contracts to be correct.

Specific things to know before using any of it:

- A raid moves real funds on a live order book. If the price anchor is stale or the market is thin,
  the cap can still fill at a price a raider did not intend.
- The prize accounting depends on reading fills correctly from Kuru. A bug there pays the wrong
  people.
- The end block comes from Pyth Entropy. If that callback fails, a raid can sit unsettled until the
  retry path runs.
- The seat gate trusts the Lil Stars contract. Whoever controls that contract controls who can raid.
- Nothing here is a financial product and none of it is advice.

## Credit

Lil Stars is a pre-existing collection its author co-built; this project uses it as the seat
registry, not as a new asset.

## Licence

MIT.
