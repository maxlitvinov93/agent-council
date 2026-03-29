# Trading Bot Risk Reference — Nassim Taleb

Philosophy: Nassim Nicholas Taleb. Domain: fat-tailed risk, ruin avoidance, antifragility, position sizing.
Stack context: Rust trading bot on Polymarket (binary outcome prediction markets), BTC-linked contracts, maker/taker strategies, order book interaction via REST/WS APIs.

Every finding must describe the **concrete ruin path** — not just "this is risky."
Taleb: "The central problem is that if there is a possibility of ruin, cost-benefit analyses are no longer possible."

---

## Principle 1: Ruin is absorbing — never risk total capital loss

*Taleb: "The probability of ruin is the probability of ruin times infinity."*
*Taleb: "If you incur a tiny probability of ruin as a 'one-off' risk, survive it, then do it again (another 'one-off' deal), you will eventually go bust with probability 1."*

Ruin is not a bad day. Ruin is the state from which you cannot recover. A bot that can lose 100% of its capital in a single sequence of events has expected lifetime = 0, regardless of expected return.

### What to check

**Hard capital floor**
- Is there a hardcoded minimum balance below which the bot halts ALL activity? Not a soft warning — a `panic!` or `process::exit` that kills the process.
- Is this floor checked BEFORE placing any order, not just on a periodic timer?
- Can the floor be bypassed by configuration? If `min_balance` is a config parameter that defaults to 0, someone will deploy with 0.
- Severity: **P1 always.** No floor = the bot can drain the wallet to zero.

**Maximum position as fraction of capital**
- Is there a hard cap on total exposure (sum of all open positions) as a percentage of available capital?
- The cap must be enforced at the order-placement level, not just at the strategy level. If the strategy says "buy 10" but capital only supports 5, the order layer must reject.
- Severity: **P1** if no hard cap exists. **P2** if the cap exists but is only checked in strategy logic (can be bypassed by direct order calls).

**No "double or nothing" on drawdown**
- After a loss, does the bot increase position size to recover? This is the gambler's ruin path. Martingale-like logic — even disguised as "averaging down" or "DCA on loss" — guarantees eventual ruin in fat-tailed environments.
- Severity: **P1** for any position-size increase triggered by recent losses.

---

## Principle 2: Fat tails are the norm — crypto is not Gaussian

*Taleb: "In Extremistan, inequalities are such that one single observation can disproportionately impact the aggregate."*
*Taleb: "If you don't know the probability distribution of the variable you're working with, you have no business using mathematical formulas that assume normality."*

BTC moves of 10-20% in minutes have happened repeatedly (March 2020, May 2021, November 2022). Polymarket settlement depends on external price feeds that can gap, lag, or diverge. Any model that assumes bounded variance is lying.

### What to check

**Hardcoded volatility assumptions**
- Is there a constant like `MAX_EXPECTED_MOVE = 0.05` that gates risk calculations? In crypto, 5% moves are routine, not extreme. 20% moves happen multiple times per year.
- Are Gaussian confidence intervals (2-sigma, 3-sigma) used for risk limits? In a fat-tailed distribution, 6-sigma events occur orders of magnitude more often than the Gaussian predicts.
- Severity: **P1** if volatility assumptions gate position sizing or stop-loss logic. **P2** if used only for informational logging.

**Historical lookback windows**
- Is the bot calibrating risk parameters from a short history window (e.g., last 24h volatility)? Calm periods produce low volatility estimates, which produce large positions, which produce catastrophic losses when volatility mean-reverts.
- Taleb: "The turkey problem — the turkey is fed for 1000 days and each day confirms its model that the farmer is a friend."
- Severity: **P2** for risk parameters derived solely from recent calm-period data without a floor.

**Stress test for 10x moves**
- If BTC drops 20% in 5 minutes, what happens to every open position? Trace the code path. Does the bot try to close positions into a market with no bids? Does it place market orders at any price?
- Severity: **P1** if no explicit handling exists for extreme moves.

---

## Principle 3: Position sizing — fractional Kelly, never full Kelly

*Taleb: "The Kelly criterion... is aggressive and will lead to ruin in the real world. Half-Kelly or less."*
*Taleb: "Ergodicity means that what you get is what the ensemble gets. If you can go bust, your time average differs from the ensemble average."*

Full Kelly maximises the geometric growth rate under perfect information. You never have perfect information. Parameter estimation error in edge and variance makes full Kelly a ruin machine.

### What to check

**Kelly fraction enforcement**
- If the bot uses Kelly or any formula-based sizing: is there a multiplier cap (0.25-0.5 of the Kelly-optimal size)?
- Is the edge estimate (`p - implied_p`) treated as uncertain? A 2% estimated edge with 50% estimation error means your real edge could be negative. Full Kelly on a negative edge is fast ruin.
- Severity: **P1** for full Kelly sizing. **P2** for Kelly > 0.5 without explicit justification.

**Maximum bet size per trade**
- Is there an absolute dollar cap per single order, independent of Kelly or any formula?
- The cap must be a constant, not a function of recent PnL. After a winning streak, capital grows, Kelly says bet more — but estimation error hasn't changed.
- Severity: **P1** if no absolute cap. **P2** if the cap scales linearly with balance without dampening.

**Bet size vs. liquidity**
- Does the bot check that its order size is a small fraction of available book depth? A $1000 order into a $500 book moves the price against you. A $5000 order into a $500 book IS the market.
- Severity: **P2** for orders placed without checking book depth.

---

## Principle 4: Payoff asymmetry — always know your maximum loss

*Taleb: "Make sure that the probability of the unacceptable is nil."*
*Taleb: "I want to live in a world where I make small errors, not large ones."*

Binary outcome markets (Polymarket) have naturally bounded payoffs: you can lose your position or gain (1 - price). But the bot can amplify losses through repeated entry, fees, slippage, and timing.

### What to check

**Worst-case per-market exposure**
- For a given market/condition, what is the maximum the bot can lose? Sum all positions in that market (including pending orders that could fill). Is this sum bounded?
- If the bot can open new positions while old ones are still active on the same market, exposure compounds without limit unless explicitly capped.
- Severity: **P1** if per-market exposure is unbounded.

**Fee-inclusive PnL**
- Are PnL calculations including fees? A strategy that is profitable before fees and unprofitable after fees will silently drain the wallet. Every backtest, every live metric, every threshold comparison must use fee-adjusted numbers.
- Taleb: "The difference between theory and practice is larger in practice than in theory."
- Severity: **P1** if fee adjustment is missing from the decision logic (position entry/exit). **P2** if missing only from reporting.

**Redemption/settlement risk**
- After a market resolves, the bot must redeem winning positions. What if redemption fails? What if the redemption transaction reverts? Is there retry logic? Is the unredeemed position tracked so it isn't counted as lost?
- Severity: **P2** for missing redemption retry. **P3** for missing tracking of pending redemptions.

---

## Principle 5: Daily and session loss limits — the stop-loss for the whole system

*Taleb: "People who are concerned with small losses and small gains will never blow up."*
*Taleb: "The key is not avoiding losses. The key is that the losses you take should be small relative to your wealth."*

Individual trade risk is necessary but not sufficient. A bot can lose 1% per trade twenty times in a row and be down 18%. Loss limits must exist at every level: per trade, per market, per session, per day, per lifetime.

### What to check

**Session/daily loss circuit breaker**
- Is there a maximum cumulative loss per time window (session, day) that triggers a full halt?
- "Halt" means: cancel all open orders, stop placing new orders, alert the operator. Not "reduce size" — full stop.
- Is the circuit breaker checked BEFORE each new order, not just on fills?
- Severity: **P1** for no daily loss limit. **P2** if the limit exists but only logs a warning.

**Loss limit accounting**
- Does the loss limit track realised losses only, or realised + unrealised (mark-to-market)? An open position at -50% that hasn't been closed is still a loss.
- Does it include fees and slippage, or just fill prices?
- Severity: **P2** for loss limits that ignore unrealised losses or fees.

**Cooldown after loss**
- After hitting a loss limit, is there a mandatory cooldown before resuming? The worst time to trade is immediately after a large loss — both psychologically (for manual overrides) and statistically (regime may have changed).
- Severity: **P3** for missing cooldown mechanism.

---

## Principle 6: Correlation blindness — when diversification fails

*Taleb: "Correlations become 1 in a crisis. Diversification works in mild markets and fails when you need it most."*

If the bot trades multiple Polymarket conditions (e.g., "BTC > 60K", "BTC > 65K", "BTC > 70K"), these are NOT independent bets. A single BTC crash moves all of them simultaneously.

### What to check

**Correlated exposure aggregation**
- Does the bot track total exposure across correlated markets? Five positions on BTC price outcomes at different strikes are one bet on BTC direction, not five independent bets.
- Is there a "group exposure" concept that caps total notional across correlated conditions?
- Severity: **P1** if correlated markets are treated as independent for position sizing. **P2** if correlation is acknowledged but only loosely tracked.

**Same-direction accumulation**
- Can the bot be long on "BTC > 60K", "BTC > 65K", and "BTC > 70K" simultaneously with full size on each? A 15% BTC drop wipes all three. The combined loss is 3x what any single position suggests.
- Severity: **P1** for unbounded same-direction accumulation across correlated markets.

---

## Principle 7: Basis risk — the map is not the territory

*Taleb: "Models should be thought of as metrics, not as descriptions of reality."*

On Polymarket, a "BTC > 70K by March 31" contract settles based on a specific price feed at a specific time. The bot's reference price (Binance, Coinbase, Chainlink) may diverge from the settlement price feed.

### What to check

**Price feed divergence**
- Does the bot use the same price source as Polymarket settlement? If not, how large can the divergence be?
- During high volatility, exchange prices can diverge by 1-5%. A bot trading at Binance prices when Polymarket settles on Chainlink can be systematically wrong.
- Is there a divergence threshold that triggers a halt? If the bot detects its reference price is >X% from the expected settlement feed, it should stop trading.
- Severity: **P1** if no divergence monitoring exists. **P2** if monitoring exists but doesn't trigger a halt.

**Stale price detection**
- What happens if the price feed stops updating? Is there a staleness check (e.g., no update in >30 seconds → stop trading)?
- The failure mode: the bot sees BTC at $65,000 (last update 5 minutes ago), places orders based on that price, while BTC has moved to $62,000. Every order placed is systematically mispriced.
- Severity: **P1** for no staleness detection on the primary price feed.

**Settlement timing edge cases**
- For contracts expiring at a specific time: does the bot account for clock drift between its system clock and the settlement oracle?
- Does the bot stop trading sufficiently before expiration to avoid being caught in illiquid end-of-life markets?
- Severity: **P2** for missing pre-expiry wind-down logic.

---

## Principle 8: Silent failures — the deadliest bugs don't crash

*Taleb: "The worst risks are the ones we don't see."*
*Taleb: "Absence of evidence is not evidence of absence — particularly in the domain of risks."*

A bot that crashes is safe — it stops trading. A bot that silently fails is lethal — it continues trading with broken assumptions.

### What to check

**Wallet balance verification**
- Does the bot verify its wallet has sufficient USDC before placing orders? Not just at startup — before EVERY order.
- What if the balance check API returns an error? Does the bot assume the last-known balance is still valid, or does it halt?
- Severity: **P1** if balance is only checked at startup. **P1** if API errors default to "assume funds available."

**Order rejection handling**
- When an order is rejected by the exchange (insufficient funds, invalid price, market closed), does the bot detect and handle this?
- The failure mode: bot sends order, API returns error, bot doesn't parse the error, continues as if the order is open. Now its internal state diverges from reality.
- Is there reconciliation logic that periodically compares internal order state with exchange state?
- Severity: **P1** for unhandled order rejections. **P2** for missing periodic reconciliation.

**Position sync drift**
- Does the bot's internal position count match what the exchange reports? Over time, missed fills, partial fills, and network errors cause drift.
- Is there a scheduled full reconciliation (e.g., every N minutes, fetch all positions from the exchange and compare)?
- Severity: **P2** for no reconciliation mechanism.

**Log-only error handling**
- `log::error!("something went wrong"); continue;` — logging an error and continuing is the Rust equivalent of an empty catch block. If the error means state is corrupt, continuing corrupts further.
- Every `log::error!` in the main trading loop should be audited: does the bot's internal state remain consistent after this error? If not, halt.
- Severity: **P1** if logged errors involve order state, position state, or balance state.

---

## Principle 9: Flash crash scenarios — open positions during market dislocations

*Taleb: "History doesn't crawl. It jumps."*
*Taleb: "The problem with tail events is that they are tail events."*

When BTC drops 15% in 10 minutes, the Polymarket order book empties. The bot has open positions and pending orders.

### What to check

**Pending orders during extreme moves**
- When the price moves beyond a threshold, does the bot cancel all pending orders immediately?
- A limit buy at 0.70 for "BTC > 65K" is reasonable when BTC is at $67,000. When BTC is at $58,000, that order is a gift to whoever fills it.
- Is the cancel logic fast enough? If the cancel is an HTTP request that takes 2 seconds during high load, orders can fill in the meantime.
- Severity: **P1** for no emergency cancel logic. **P2** if cancel logic exists but relies on slow paths.

**Market order prohibition**
- Does the bot ever place market orders (or limit orders priced to guarantee immediate fill)? In a thin book during a crash, a market order fills at catastrophic prices.
- A market sell of 100 shares into a book with 10 shares at 0.50 and 90 shares at 0.01 results in an average fill of ~0.06.
- Severity: **P1** for market orders without price bounds. **P2** for aggressive limit orders without slippage protection.

**Position wind-down under stress**
- If the bot decides to exit a position during a volatile period, does it check that the exit price is within acceptable bounds? Exiting at any price to "stop the bleeding" can realize worse losses than holding.
- Severity: **P2** for exit logic without minimum acceptable price.

---

## Principle 10: Liquidity risk — the order book is a mirage

*Taleb: "Liquidity is there when you don't need it and gone when you do."*

Polymarket order books are thin. Displayed depth can vanish instantly as market makers cancel orders during volatility.

### What to check

**Book depth validation before order placement**
- Does the bot check available depth at the desired price level before placing a taker order?
- Measure depth as: sum of shares available within X cents of the desired price, not just the top-of-book.
- Severity: **P2** for taker orders placed without depth check. **P3** for maker orders (they provide liquidity, but still face adverse selection).

**Impact estimation**
- For larger orders, does the bot estimate its own price impact by walking the book?
- A $500 taker order into a book with $50 at each cent-level will move the price 10 cents against you. The bot should calculate the volume-weighted average fill price BEFORE submitting.
- Severity: **P2** if the bot places orders > $100 without impact estimation.

**Maker order vulnerability**
- For maker (limit) orders: is there a mechanism to detect when the bot is being adversely selected? If the bot's resting orders are filled only when the price moves against them (the "toxic flow" problem), the bot is systematically losing.
- Track fill-to-midpoint ratio: what fraction of fills occur when the mid price has already moved past the fill price?
- Severity: **P3** for missing adverse selection tracking. This is a profitability issue, not a ruin issue.

---

## Principle 11: Fee drag — death by a thousand cuts

*Taleb: "Small, nearly invisible costs compound into ruin just as effectively as a single catastrophe — just more slowly."*

On Polymarket: taker fees, maker fees (or rebates), gas costs for on-chain operations, redemption costs. A strategy that turns over its entire capital daily pays 365x the single-trade fee cost per year.

### What to check

**Fee accounting in edge calculation**
- Is the minimum required edge (the signal threshold to enter a trade) larger than the round-trip fee cost?
- Round-trip cost includes: entry fee + exit fee + potential slippage on both sides. If round-trip cost is 4% and the signal says "3% edge," the trade is negative EV.
- Severity: **P1** if edge calculations don't subtract fees. This is the most common cause of "profitable in backtest, unprofitable live."

**Cancel/replace cost tracking**
- How often does the bot cancel and replace orders? Each cancel-replace cycle on Polymarket has cost (gas, potential lost queue position).
- Is there a metric tracking total cancel cost per session? If cancel costs exceed trading revenue, the strategy is net negative.
- Severity: **P2** for no cancel cost tracking. **P3** if tracking exists but isn't used in strategy decisions.

**Turnover rate monitoring**
- What is the bot's daily turnover as a fraction of capital? High-frequency strategies with 10x daily turnover pay 10x the fee cost.
- Is there an alert when turnover exceeds a threshold? Runaway order placement (e.g., a bug causing rapid order-cancel-order cycles) can drain capital purely through fees.
- Severity: **P2** for no turnover monitoring. **P1** if no circuit breaker exists for abnormal order frequency.

---

## Principle 12: Overconfidence in models — all models are wrong, some are lethal

*Taleb: "The model is not reality. The model is a tool to help you think."*
*Taleb: "Don't tell me what you think, tell me what you have in your portfolio."*

Every parameter in a trading bot — edge estimate, volatility, mean-reversion speed, optimal spread — is estimated with error. The bot acts on these estimates as if they were truth.

### What to check

**Parameter sensitivity**
- What happens if the edge estimate is off by 50%? By 100% (i.e., the true edge is zero)? If the strategy is still viable at half the estimated edge, it's robust. If it blows up at 80% of estimated edge, the margin of safety is too thin.
- Are there unit tests or simulation sweeps that run the strategy with degraded parameters?
- Severity: **P2** for no parameter sensitivity analysis. **P3** if analysis exists but isn't automated.

**Regime change detection**
- Markets change character. A mean-reversion strategy in a trending market loses money on every trade. A trend strategy in a ranging market gets whipsawed.
- Does the bot detect when its assumptions are failing? Concrete check: track rolling hit rate (fraction of trades that are profitable). If hit rate drops below X% over N trades, halt and alert.
- Severity: **P2** for no regime change detection.

**Backtest vs. live performance monitoring**
- Is there a mechanism to compare live performance against backtest expectations? If the live Sharpe ratio is significantly worse than backtest, the model is wrong.
- Taleb's warning: backtests always overfit. A strategy that returned 50% in backtest and -5% live is not "underperforming" — the backtest was fiction.
- Severity: **P3** for missing live vs. expected performance comparison.

---

## Principle 13: Skin in the game — the operator must bear the consequences

*Taleb: "Never trust anyone who doesn't have skin in the game."*
*Taleb: "Systems don't learn when the person making the decision doesn't bear the consequences."*

A bot deployed by someone who doesn't check it is a bot destined for ruin. The code must force accountability.

### What to check

**Alerting on anomalies**
- Does the bot send alerts (Telegram, email, webhook) when: daily loss limit approached, unusual position size, price feed stale, order rejection rate high, balance below threshold?
- An alert that nobody reads is not an alert. Is there evidence the alerting channel is monitored?
- Severity: **P2** for no alerting mechanism. **P1** if the bot can lose >10% of capital without any notification.

**Manual kill switch**
- Is there a way to remotely halt the bot immediately? Not "send a config update and wait for it to poll" — a fast path that stops all trading within seconds.
- Severity: **P2** for no remote kill switch.

**Audit trail**
- Does the bot log every order placed, every fill received, every cancel, every balance check? If something goes wrong, can you reconstruct exactly what happened?
- Logs must include timestamps, prices, sizes, order IDs, and the strategy signal that triggered the action.
- Severity: **P2** for incomplete audit trail. **P3** for missing human-readable log format.

---

## Principle 14: Antifragility in bot design — benefit from disorder

*Taleb: "Antifragility is beyond resilience or robustness. The resilient resists shocks and stays the same; the antifragile gets better."*

A well-designed trading bot should perform BETTER during volatility events (if it's a volatility-buying strategy) or at least survive them unharmed (if it's not). The code should make surviving disorder the default.

### What to check

**Default-safe state**
- If the bot loses its network connection, what happens? It should default to: cancel all pending orders (if possible) and stop placing new ones. The safe state is "do nothing," not "continue with stale data."
- If the price feed websocket disconnects, does the bot continue trading on the last known price?
- Severity: **P1** if any disconnection leads to trading on stale data. **P2** if disconnection is handled but not tested.

**Graceful degradation hierarchy**
- When subsystems fail, does the bot degrade gracefully? Example hierarchy: (1) price feed fails → stop new entries, keep existing positions. (2) Order API fails → stop all order activity, monitor positions. (3) Balance API fails → full halt.
- Severity: **P2** for no degradation hierarchy (all-or-nothing only).

**Recovery from restart**
- If the bot process crashes and restarts, does it correctly load its state? Does it reconcile with the exchange before resuming trading?
- The failure mode: bot restarts, doesn't know about existing open positions, opens new ones, now has 2x intended exposure.
- Severity: **P1** for no state reconciliation on restart.

---

## Principle 15: Time-based risk — exposure grows with duration

*Taleb: "Time is the most underestimated risk factor."*

A position held for 1 hour has less time to encounter a black swan than a position held for 1 week. Long-duration exposure requires proportionally smaller size.

### What to check

**Position age tracking**
- Does the bot track how long each position has been open?
- Are there maximum position durations after which the bot forces an exit (or at minimum, alerts)?
- Stale positions tie up capital and accumulate tail risk without generating returns.
- Severity: **P2** for no position age tracking. **P3** if age is tracked but not acted upon.

**Weekend/holiday awareness**
- Crypto trades 24/7 but liquidity drops on weekends. Polymarket activity drops further. Holding positions through low-liquidity periods increases the risk that you can't exit when needed.
- Does the bot reduce position sizes or halt before known low-liquidity periods?
- Severity: **P3** for no liquidity-aware scheduling.

---

## Principle 16: The barbell strategy — conservative core, aggressive edge

*Taleb: "Put 85-90% of your portfolio in safe instruments. Use the remaining 10-15% for highly speculative, maximum-upside bets."*

Applied to bot architecture: the majority of capital should be protected by extremely conservative limits. Only a small fraction should be exposed to aggressive strategies.

### What to check

**Capital allocation tiers**
- Is capital divided into tranches with different risk profiles? Example: 80% in "safe" (tight limits, small size, high-confidence signals only), 20% in "aggressive" (wider limits, larger size, exploratory signals).
- If all capital is governed by the same parameters, a single bad parameter set risks everything.
- Severity: **P3** for single-tier capital allocation. This is a design recommendation, not a bug.

**New strategy quarantine**
- When a new strategy or parameter set is deployed, is it initially limited to a small fraction of capital?
- A new strategy should prove itself on 5-10% of capital before getting full allocation.
- Severity: **P2** for no quarantine mechanism for new strategies.

---

## Principle 17: The narrative fallacy — don't trust the story, trust the numbers

*Taleb: "We love stories. We love to assign cause and effect. We are narrative-creating machines."*

A strategy that "makes sense" narratively ("BTC is trending up, so buy YES on high-strike contracts") can still be negative EV after fees, slippage, and adverse selection.

### What to check

**Decision logging with quantitative basis**
- When the bot enters a trade, does the log include the quantitative signal (edge estimate, probability, confidence) and not just a qualitative label?
- `"signal=BUY, reason=trend_up"` tells you nothing. `"signal=BUY, edge=0.032, p_model=0.72, p_market=0.68, fee_cost=0.015, net_edge=0.017"` tells you everything.
- Severity: **P3** for missing quantitative decision logs.

**Performance attribution**
- Can profits and losses be attributed to specific signal components? If the bot has 3 signal inputs, which one is generating alpha and which is noise?
- Without attribution, you can't know if the strategy works for the right reasons.
- Severity: **P3** for missing performance attribution.

---

## Principle 18: Ergodicity — the ensemble average is not your average

*Taleb: "One hundred people going to a casino is not the same as one person going to a casino one hundred times."*

If 100 bots each run one session and average 5% return, that doesn't mean YOUR bot running 100 sessions will average 5%. If any session can produce ruin, the time-average diverges from the ensemble-average.

### What to check

**Multiplicative vs. additive risk**
- Does the bot's risk model treat returns as additive (each trade adds/subtracts from balance) or multiplicative (each trade multiplies the balance)?
- A 50% loss followed by a 50% gain leaves you at 75%, not 100%. Risk limits must account for the multiplicative nature of sequential returns.
- Severity: **P2** if risk calculations assume additive returns over multiple periods.

**Path-dependent survival**
- Run the worst realistic sequence: 10 consecutive losing trades at maximum allowed size. Does the bot survive? Does it still have enough capital to recover?
- If maximum allowed loss per trade is 5% and the bot can lose 10 in a row, that's a 40% drawdown (multiplicative). Is this survivable?
- Severity: **P2** for no worst-case sequence analysis.

---

## Principle 19: Via negativa — what you don't do matters more than what you do

*Taleb: "The learning of life is about what to avoid."*
*Taleb: "Charlatans are recognisable in that they will give you positive advice."*

The most important lines of code in a trading bot are the ones that say NO.

### What to check

**Explicit rejection conditions**
- Is every order placement guarded by a chain of rejection conditions that must ALL pass?
- The conditions should be listed explicitly and checked sequentially: balance sufficient? Within position limit? Within daily loss limit? Price feed fresh? Edge above minimum? Book depth adequate?
- A missing check in this chain is a hole in the fence.
- Severity: **P2** if rejection conditions exist but are scattered across multiple functions (easy to miss one). **P1** if any critical rejection condition is missing.

**Whitelist vs. blacklist thinking**
- Does the bot define the conditions under which it SHOULD trade (whitelist), or the conditions under which it SHOULDN'T (blacklist)?
- Blacklists always have gaps. A whitelist defaults to "don't trade" and requires explicit conditions to enable trading.
- Severity: **P3** for blacklist-based trading conditions. This is a design principle.

---

## Principle 20: Redundancy is not waste — critical checks must be layered

*Taleb: "Redundancy is ambiguous because it seems like a waste if nothing unusual happens. Except that something unusual happens — usually."*

A single balance check before placing an order is one layer. Checking again after the order is acknowledged is two layers. Having the exchange reject insufficient-funds orders is three layers. All three should exist.

### What to check

**Defence in depth for capital protection**
- Count the layers: (1) Strategy-level position limit. (2) Order-level balance check. (3) System-level daily loss limit. (4) Exchange-level rejection. (5) Monitoring alert.
- If any single layer fails, do the remaining layers still prevent ruin?
- Severity: **P2** for fewer than 3 independent layers of capital protection.

**Independent subsystem monitoring**
- Is there a monitoring process separate from the bot that watches its positions and balance? If the bot's internal state is corrupted, it can't monitor itself.
- This can be as simple as a cron job that checks wallet balance and alerts if it drops below a threshold.
- Severity: **P2** for no external monitoring.

---

## Principle 21: The Lindy effect — distrust the new, respect the survived

*Taleb: "If a book has been in print for forty years, I can expect it to be in print for another forty years."*

Applied to trading bots: a strategy that has survived 6 months of live trading is more trustworthy than one that backtests well but has never run live. New code is the most dangerous code.

### What to check

**Deployment risk management**
- When new strategy code is deployed, is the budget reduced? New code should run with minimum capital until it proves itself.
- Is there a rollback mechanism? If the new version starts losing, can the operator quickly revert to the last known-good version?
- Severity: **P2** for full-capital deployment of untested code. **P3** for missing rollback mechanism.

**Change impact assessment**
- For any code change to the trading logic: what is the worst-case impact if the change has a bug? If a sign error in the edge calculation causes the bot to sell when it should buy, how much can it lose before detection?
- Every change to order placement, position sizing, or signal calculation must answer: "If this is wrong, what's the blast radius?"
- Severity: **P2** for changes to trading logic without explicit blast-radius analysis.

---

## Principle 22: Iatrogenics — intervention causes more harm than the disease

*Taleb: "In complex systems, intervention is more likely to cause harm than benefit."*

A bot that reacts to every tick is a bot that overtrades. A human who overrides the bot on every drawdown introduces more randomness, not less.

### What to check

**Minimum time between actions**
- Is there a cooldown between consecutive trades? A bot that can enter and exit the same position 10 times per minute is not trading — it's generating fees.
- Is there a cooldown between parameter changes? Adjusting the strategy every hour based on the last hour's performance is curve-fitting in real time.
- Severity: **P2** for no cooldown between trades. **P3** for no cooldown between parameter adjustments.

**Manual override audit**
- When the operator manually overrides the bot (force exit, change parameters, pause/resume), is this logged with a timestamp and the override values?
- After the incident: can you distinguish bot-initiated actions from human-initiated ones?
- Severity: **P3** for no override logging.

---

## Principle 23: Domain-specific ignorance — know what you don't know about Polymarket

*Taleb: "It is much more straightforward to figure out if something is fragile than to compute the exact probability of a specific adverse event."*

Polymarket has specific mechanics that create risks invisible to someone who only knows traditional exchange trading.

### What to check

**CTF token mechanics**
- Polymarket uses Conditional Token Framework. Does the bot correctly handle: token merging, splitting, redemption? A bug in token handling can lock funds permanently.
- Does the bot verify transaction success on-chain, or assume HTTP 200 = success?
- Severity: **P1** for no on-chain confirmation of token operations.

**Market resolution disputes**
- Polymarket markets can be disputed. A position the bot considers "won" may be reversed. Does the bot handle resolution disputes gracefully?
- Severity: **P3** for no dispute handling (rare but nonzero risk).

**API rate limits and bans**
- Does the bot respect Polymarket API rate limits? Getting rate-limited during a volatile period means you can't cancel orders while they're being adversely filled.
- Severity: **P2** for no rate limit awareness. **P1** if aggressive API usage could lead to a ban during critical moments.

---

## Principle 24: Concavity of harm — small is safe, large is lethal

*Taleb: "For the fragile, the cumulative effect of small shocks is smaller than the single effect of an equivalent single large shock."*

10 positions of $100 are safer than 1 position of $1000, even with the same total exposure. The $1000 position concentrates risk; the $100 positions allow individual exits.

### What to check

**Order fragmentation**
- Does the bot break large orders into smaller pieces? A single large order is more visible, harder to exit, and moves the market more.
- Is there a maximum single-order size, independent of the position-level limit?
- Severity: **P3** for no order fragmentation. This is an optimisation, not a safety issue, unless order size exceeds book depth (**P2**).

**Concentration limits**
- Is there a maximum fraction of capital that can be in a single market? A single condition? A single expiry date?
- The bot may have per-position limits but no aggregate limit, allowing 20 small positions in perfectly correlated markets.
- Cross-reference with Principle 6 (Correlation blindness).
- Severity: **P2** for no concentration limit across correlated exposures.

---

## Quick Reference: Severity Guide

| Severity | Pattern | Examples |
|----------|---------|----------|
| **P1 — Fix Now** | Path to ruin, capital loss, or silent corruption | No capital floor, unbounded position size, full Kelly, missing fee adjustment in edge, no emergency cancel, no balance check, log-only error handling on order state, trading on stale prices, no state reconciliation on restart, no on-chain confirmation |
| **P2 — Fix Soon** | Degraded safety margins or missing detection | Gaussian assumptions in risk, no daily loss limit enforcement, no book depth check, no reconciliation, no alerting, no kill switch, no regime detection, no parameter sensitivity, single-layer capital protection, no external monitoring |
| **P3 — Consider** | Robustness and operational maturity | Missing adverse selection tracking, no performance attribution, no override logging, no liquidity-aware scheduling, blacklist-based conditions |

### The Overriding Filter

Before writing any finding, apply the Taleb synthesis:

1. **Can this path lead to ruin?** If a sequence of events (however unlikely) can drain the account to zero, it is P1 regardless of probability. Taleb: "Ruin is forever."
2. **Does this assume normality?** If risk calculations use standard deviation, Gaussian confidence intervals, or "worst case = 3 sigma," flag it. Crypto tails are fat.
3. **Is the bot safe when wrong?** Assume the model is wrong, the edge estimate is off, and the price feed is stale. Does the bot still survive? If survival depends on the model being right, the architecture is fragile.
4. **Does the bot know what it doesn't know?** Every estimate should have a margin of error. Every parameter should have a conservative default. Every assumption should have a tripwire that fires when the assumption breaks.
5. **Would Taleb put his own money in this bot?** If the answer requires caveats ("yes, but only if volatility stays low"), the bot is fragile. Taleb would walk away.
