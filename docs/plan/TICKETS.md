# Star Raid: tickets

One GitHub issue per epic with its tickets as a checklist, one branch and one PR per epic
(`Closes #n`), many small commits with short messages, merged without squash. Order is the build
order; P0 is the demo, P1 raises the score, P2 is cut first (SPEC §13). SPEC §16 (review round 1)
overrides older wording below; the R-tickets carry its fixes.

## E0. Groundwork (P0, before the repo)
- [x] Name check "Star Raid": no crypto or Monad project; only a 2019 indie game (Binary Moon). X not checked.
- [ ] Confirm Ethereum Jakarta is an onboarded community (ask Monad in the portal Support page).
- [ ] Create the portal team and project, pick Track 01 and the four bounties.
- [ ] Buy the domain (he buys it; `starraid.xyz` and `starraid.gg` were free on 26 Sep).
- [ ] Repo `zexoverz/star-raid` public, MIT, `docs/plan/` without odds, `AI_USAGE.md`, `AGENTS.md`.

## E1. Kuru spine (P0)
- [ ] `KuruBook` interfaces for OrderBook and MarginAccount on the repo's ABIs.
- [ ] Wall status from the level head, with a fork test on a filled, partial and cancelled order.
- [ ] Tick rounding and empty-level search.
- [ ] Fork test: contract deposits native MON, places a postOnly ask, reads its id.
- [ ] Fork test: capped `addBuyOrder` never fills above the limit with cheaper asks present.
- [ ] Fork test: overshoot remainder found and cancelled in the same tx.
- [ ] Fork test: margin balance difference equals base received.

## E2. RaidVault (P0)
- [ ] Terms, storage, `post` with checks and events.
- [ ] `PriceAnchor`: Chainlink staleness, Pyth freshness, divergence, Kuru mid check.
- [ ] `open`: anchor, cap price, empty level, deposit, wall, id assert.
- [ ] `close` with Pyth Entropy request and callback; retry after 200 blocks.
- [ ] `settle`: active, filled, partial, admin-cancelled (Safe impersonation) walls.
- [ ] Rollover ledger with no withdraw path.
- [ ] `abort` path with full refund when the anchor fails.
- [ ] Invariant tests (DESIGN §9).

## E3. RaidRouter and SeatGate (P0)
- [ ] Seat kind 0: `ownerOf` plus nullifier.
- [ ] `raid`: pull USDC, deposit, capped buy, cancel remainder, attribution, refund leftover.
- [ ] Escrow of bought base per seat.
- [ ] Counting with `endBlock` and seat caps.
- [ ] `claim` after hold and `exitEarly` with forfeit.
- [ ] Sponsor and affiliate rejection.
- [ ] delegate.xyz v1 path (P2).
- [ ] Seat kind 1: EIP-712 attestation from a Self backend check (`SelfBackendVerifier`), World ID as fallback (P2).

## E4. Keeper and live service (P0)
- [ ] Trade feed from `monadLogs`; start guard (60 s ≤20 bps, 5 min ≤50 bps, last trade ≤300 blocks).
- [ ] Open, close and settle on Finalized; retries; Telegram alert.
- [ ] Reschedule inside the sponsor's slide range.
- [ ] Live service: one WebSocket, Multicall per head, SSE fan-out, proposed vs finalized frames.
- [ ] Railway project with a fixed domain.

## E5. App (P0; Mera P1)
- [ ] Lobby and terms pages.
- [ ] Join: seat pick with Lil Stars art, amount, confirm sheet with cap, hold, early exit, Kuru admin line.
- [ ] Raid screen: target counter, wall bar, raider feed, end window meter.
- [ ] Draw moment and results page (counted vs not, wall share, vault fills, refused attempts).
- [ ] Claim page.
- [ ] Share card design (storyboard frame 08): Star art, outcome, verified people, amount under the cap, bots turned away, your seat, next raid. No price, no PnL, no referral payout.
- [ ] `/r/{raid}/{seat}` page with an Open Graph image rendered server-side at 1200×630 (`app/r/[raid]/[seat]/opengraph-image.tsx` and `twitter-image.tsx`, `ImageResponse` from `next/og`, `params` is a Promise in Next 16; confirmed in Context7 26 Sep; containers need `display: flex`, fonts loaded from file), `twitter:card=summary_large_image`.
- [ ] "Post on X" via X's web intent with text and link prefilled; Save image; Copy link.
- [ ] Cards for lost raids ("prize rolls to the next raid").
- [ ] Measure card link opens to joins (UTM on the link), shown in the traction numbers.
- [ ] Mera passkey account, held session scoped to router and allowance (P1).
- [ ] Stateless test rehearsal: clear storage mid-demo, account reconstructs (P1).
- [ ] Sponsor console (P2).

## E6. Indexer (P1)
- [ ] Envio config for 143, schema, handlers for vault, router and Kuru `Trade` with maker = vault.
- [ ] Derived `Player` and `Leaderboard`.
- [ ] Hosted deploy; results and history pages read from it.

## E7. Traction (raiders and mainnet raids P0; outside sponsor P1, never blocks submission)
- [x] List Kuru mainnet markets: 597 registered, 5 traded in 7 days, 16 dead-market tokens still active.
- [ ] Pick 2-3 of shMON, gMON, JAMES, Kintsu sMON, 143, LeverUp and find their teams (CHOG later, as a partnership).
- [ ] Pitch one Monad project for a sponsored raid or a letter of intent (draft for his go).
- [ ] Ask Kuru about a listing path for a sponsor market (draft for his go).
- [ ] Testnet market via `deployProxy` for the thin-market demo.
- [ ] Announce raids through Lil Stars and ETHJKT; raid schedule in Asia slots before 14:00 UTC.
- [ ] Run 10-15 mainnet raids 6-12 Oct; publish the numbers after each.

## R. Review round 1 fixes (P0, folded into E1-E5)
- [ ] E1: fork tests move to the LST markets (shMON/MON, gMON/MON, sMON/MON); sMON rate function CONFIRM.
- [ ] E1: wall fill per buy from the wall's remaining size before and after, and a self-wash test (own ask under the wall earns zero).
- [ ] E2: one minimal clone maker per raid; settle sweeps the clone.
- [ ] E2: anchor modes (LST rate, oracle mid, sponsor-fixed) with max(anchor, Kuru mid) and ≤20 bps divergence.
- [ ] E2: `expire()`, Entropy timeout to `E = w1`, one pending request, `getFeeV2()`.
- [ ] E2: settle records the outcome; `recoverWall()` retries under try/catch through a Kuru hard pause.
- [ ] E2: abort returns rollover to the ledger; target ≥ k × bounty; bounty in bps of wall fill.
- [ ] E3: one seat per owner wallet per raid, owner cap across its Stars.
- [ ] E3: EIP-712 `Bind{holder, player, tokenIds, expiry}` so Mera accounts can use Stars held elsewhere.
- [ ] E3: per-block running totals; `E` drawn from the last 25% of the window.
- [ ] E3: non-seat buys accepted but uncounted; base withdrawn every buy; claims CEI and `nonReentrant`.
- [ ] E5: session copy fixed; raider accounts keep 10 MON; gas estimate × 1.5 with a ceiling.
- [ ] E7: publish team wallets, exclude them, report numbers net of the team.
- [ ] Schedule: scope frozen 4 Oct, testnet raids 4-5 Oct, first mainnet raid small.

## E8. Submission (P0)
- [ ] Sourcify `exact_match` for every contract.
- [ ] README: what it is, addresses, how to raid, the Kuru admin disclosure, AI tools used, Lil Stars
      named as a pre-existing foundation.
- [ ] Technical demo video (3 min) and pitch video (2 min).
- [ ] Portal: target users, evidence of demand, retention plan for Kuru; judge access instructions.
- [ ] Share the repo with `metropolis@hackathon.monad.xyz`.


## Example Usage

Resolved parameter handling for issue #9.
