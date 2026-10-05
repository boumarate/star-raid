# AI use

Star Raid is built with AI assistance and this file says where, because the Metropolis rules ask and
because a reader should not have to guess.

- **Planning.** The spec, the design and the ticket breakdown were written with Claude in a planning
  session before any code, then reviewed and cut by hand. The measured sections are not AI output:
  the Kuru market figures come from 616k fills read off chain, the contract behaviour from reading
  `Kuru-Labs/Kuru-contracts-dex-public` at `2060bb27`, and the backtest from 20,194 simulated raids
  against 5,881 real book snapshot pairs.
- **Code.** Contracts, tests, keeper and app are written with Claude Code. Every diff is reviewed
  before it lands. Tests are written to fail first against the behaviour they name.
- **Not AI.** The decision of what to build, the price-cap mechanism, the seat gate, the choice to
  count only what the sponsor's wall sells, and every mainnet transaction.

Faisal Firdani is responsible for all of it, including the parts a model wrote.
