# Trading Strategy Quality Reference — Ernest Chan

Philosophy: Ernest P. Chan ("Quantitative Trading", "Algorithmic Trading", "Machine Trading"). Focus: momentum/trend strategies, backtesting methodology, transaction cost analysis, strategy evaluation, regime detection.
Stack context: Rust trading bot / prediction markets / BTC trend-following / OHLCV candle data / EMA/RSI/MACD signals / binary outcome markets.

Every finding must describe the **concrete P&L failure mode** — not just "this is bad practice."
This doc covers: signal quality, overfitting detection, edge calculation, backtesting methodology, transaction costs, parameter sensitivity, regime detection, and strategy decay. Market microstructure (order book, maker/taker) is out of scope — see the Avellaneda-Stoikov reference if applicable.

---

## Principle 1: A strategy without a stated edge is not a strategy — it is a bet

*Chan: "Before I even look at the data, I want to know what the economic rationale is for why this strategy should work."*

Every strategy must articulate its edge in one sentence before any code is written. "BTC trends after breakouts" is a hypothesis. "EMA crossovers on 5m candles capture momentum continuation with a 55% win rate and 1.3:1 reward-to-risk after fees" is an edge statement. Without the latter, the strategy is unfalsifiable.

### What to check

**Missing edge statement**
- Is there a comment or doc explaining *why* this strategy should make money? Not *how* it works mechanically, but *what market inefficiency* it exploits.
- If the only rationale is "this indicator worked in backtest," the strategy has no edge — it has a curve fit.
- Severity: **P1** — no edge statement means no basis to evaluate anything else.

**Vague edge claims**
- "Momentum works" is not an edge. What timeframe? What asset? What regime? Against what benchmark?
- Chan: the edge must survive the question "why hasn't this been arbitraged away?"
- Severity: **P2**

---

## Principle 2: The Sharpe ratio is the only honest metric — everything else lies

*Chan: "I will not even bother to look at a strategy that has a Sharpe ratio of less than 1 on a backtest, because after transaction costs and slippage, the live Sharpe will be much lower."*

CAGR is meaningless without risk context. A 200% return with 80% max drawdown is a strategy that will blow up — it just hasn't yet. The Sharpe ratio (annualised) normalises returns by volatility and is the single most important metric for strategy comparison.

### What to check

**Missing Sharpe calculation**
- Is the strategy evaluated only by total P&L or win rate? Both are misleading. A 90% win rate with a 10:1 loss ratio is a losing strategy.
- Is Sharpe annualised correctly? Daily Sharpe * sqrt(252) for equities, sqrt(365) for crypto (24/7 markets).
- Severity: **P1** if no risk-adjusted metric exists anywhere.

**Inflated Sharpe from overfitting**
- Backtest Sharpe above 3.0 is almost certainly overfit. Chan: live Sharpe is typically half of backtest Sharpe.
- If backtest Sharpe is 2.0, expect live Sharpe around 1.0. If backtest Sharpe is 1.2, the strategy is marginal and probably unprofitable live.
- Severity: **P2** for strategies deployed with backtest Sharpe below 2.0 and no out-of-sample validation.

**Win rate without payoff ratio**
- Win rate alone is meaningless. A trend-following strategy with 35% win rate and 3:1 payoff ratio is highly profitable. A mean-reversion strategy with 70% win rate and 0.3:1 payoff ratio is a disaster.
- Check: is the code logging both win rate AND average win / average loss?
- Severity: **P2**

---

## Principle 3: If you haven't tested out-of-sample, you haven't tested

*Chan: "The most common mistake in backtesting is using the entire data set for both parameter optimization and performance evaluation."*

In-sample performance proves nothing. The strategy was tuned to fit that data. Out-of-sample testing is the minimum bar. Walk-forward analysis (rolling train/test windows) is the gold standard.

### What to check

**No train/test split**
- Are the same data used to choose parameters AND to evaluate performance? This is the single most common quant mistake.
- Correct: train on 2023, test on 2024. Wrong: optimise on 2023-2024, report 2023-2024 returns.
- Severity: **P1** — results are meaningless without this split.

**Leaky test set**
- Was the test set "peeked at" during development? If a developer tweaked parameters after seeing test-set results, the test set is contaminated and is now in-sample.
- Check: how many times were parameters changed after running the strategy on "test" data? Each change consumes degrees of freedom.
- Severity: **P1** if parameters were tuned post-hoc on test data.

**No walk-forward validation**
- A single train/test split is fragile — the split point matters. Walk-forward (retrain every N periods, test on the next N) is robust.
- Chan: "I use walk-forward optimization as a matter of routine."
- Severity: **P2** for production strategies without walk-forward.

---

## Principle 4: Every threshold is guilty until proven innocent

*Chan: "The fewer parameters a strategy has, the less likely it is to be overfit."*

Every hardcoded number in a trading strategy is a parameter, whether or not it appears in a config file. `if rsi > 70` has a parameter: 70. `if ema_fast.period == 12` has a parameter: 12. Each parameter multiplies the overfitting risk.

### What to check

**Magic numbers in signal logic**
- Count every hardcoded numeric threshold in entry/exit logic. Each one is a degree of freedom that was (implicitly or explicitly) optimised.
- Chan's rule of thumb: you need at least 252 / N independent observations to justify N parameters, and even that is generous.
- Are these numbers justified by market structure (e.g., "70 RSI is a standard overbought level") or by data mining ("70 worked best in our backtest")?
- Severity: **P2** per unjustified threshold. **P1** if there are more than 5 free parameters in the signal logic.

**Parameter sensitivity**
- Change each threshold by +/-10%. Does the strategy still work? If a 0.005 epsilon becomes unprofitable at 0.004 or 0.006, the parameter is overfit to noise.
- Chan: "A robust strategy should show a plateau of profitability across a range of parameter values."
- If the code has no sensitivity analysis and the parameters were chosen from a single optimisation run, assume overfitting.
- Severity: **P1** for strategies with no sensitivity analysis deployed to production.

**Indicator period selection**
- Why EMA(12) and not EMA(10) or EMA(14)? If the period was grid-searched, how many periods were tested? The more you test, the more likely the best one is noise.
- Bonferroni correction: if you tested 20 EMA periods, your significance threshold should be 0.05/20 = 0.0025, not 0.05.
- Severity: **P2**

---

## Principle 5: Transaction costs are not a footnote — they are the strategy

*Chan: "Transaction costs are the #1 reason strategies that work in backtest fail in live trading."*

A strategy that generates 10bps of alpha per trade but incurs 15bps of transaction costs is not a strategy — it is a fee-transfer mechanism from you to the exchange. Fees must be modeled in the backtest, not applied as an afterthought.

### What to check

**Fees missing from backtest**
- Is the backtest calculating P&L before or after fees? Search for where fees are deducted. If the answer is "nowhere," multiply the number of trades by the fee rate and subtract from total P&L. Many "profitable" strategies become unprofitable.
- Severity: **P1** — a backtest without fees is fiction.

**Incorrect fee model**
- Maker vs taker fees are different (often 0% vs 0.01-0.04% on prediction markets, or 0.01% vs 0.05% on crypto exchanges). Is the code using the correct fee for the order type?
- Does the fee model account for the fact that limit orders are not guaranteed to fill? A backtest that assumes 100% fill rate on limit orders overstates returns.
- Severity: **P2** for wrong fee tier, **P1** for missing maker/taker distinction when the spread is thin.

**Slippage not modeled**
- Market orders move the price. The larger the order relative to the book depth, the worse the fill. Backtesting at the mid-price or at the close is optimistic.
- Chan: "I typically assume 1 tick of slippage for liquid markets." For illiquid markets (prediction market books with thin depth), slippage can be 1-5% per trade.
- Check: does the code model slippage or assume fills at the quoted price?
- Severity: **P2** for liquid markets, **P1** for thin order books.

**Cancel/replace costs**
- On markets with cancel fees (e.g., Polymarket), every cancelled order has a cost. If the strategy cancels and replaces frequently, the cancel cost can exceed trading profit.
- Check: is the backtest tracking cancel frequency and associated costs?
- Severity: **P1** if the strategy involves frequent order replacement on a market with cancel fees.

---

## Principle 6: The backtest must model what the live system actually does

*Chan: "Your backtest should simulate your live execution as closely as possible, including order types, position sizing, and rebalancing frequency."*

A backtest that enters at the close when the live system enters on a signal mid-candle is testing a different strategy. A backtest that assumes instant fills when the live system queues limit orders is testing a different strategy. Counterfactual accuracy matters.

### What to check

**Entry price mismatch**
- Does the backtest enter at open, close, or signal price? Does the live system enter at the same price?
- Common failure: backtest uses close price, live system fires signal mid-candle and enters at a different price. The difference compounds over hundreds of trades.
- Severity: **P2**

**Look-ahead bias**
- Does the signal calculation use data that wasn't available at decision time? Example: using the current candle's close to compute an indicator that triggers an entry on the same candle.
- Check: at the moment the code decides to enter, has the candle closed? If the signal uses `candle[i].close` and the entry is also on candle `i`, this is look-ahead bias.
- Severity: **P1** — this invalidates the entire backtest.

**Survival bias**
- Is the backtest only run on assets that exist today? Assets that delisted, went to zero, or were removed from the market are excluded, biasing results upward.
- For prediction markets: are resolved markets included in the backtest, or only active ones?
- Severity: **P2**

**Fill assumption**
- Does the backtest assume every limit order fills? In reality, limit orders at the best bid/ask may not fill if the market moves away.
- Chan: "Never assume you can buy at the bid or sell at the ask."
- Severity: **P2** for limit order strategies, **P3** for market order strategies.

---

## Principle 7: Measure the half-life of your edge

*Chan: "Every edge decays. The question is not whether but when."*

Markets adapt. Other participants discover the same signal. Infrastructure changes. A strategy that worked in 2024 may not work in 2026. The code must have mechanisms to detect when the edge has decayed.

### What to check

**No performance monitoring**
- Is there a mechanism to track live Sharpe ratio over rolling windows? If the strategy runs indefinitely without performance tracking, a decayed edge will bleed money silently.
- Chan: "I review my strategies' performance at least monthly and compare live Sharpe to backtest Sharpe."
- Severity: **P1** for production strategies with no performance monitoring.

**No kill switch**
- What happens when the strategy loses 5%, 10%, 20%? Is there a max drawdown circuit breaker?
- Check: is there code that halts trading when cumulative losses exceed a threshold? If not, a broken strategy will drain the account.
- Severity: **P1** for strategies with no drawdown limit.

**Stale parameters**
- When were the strategy parameters last recalibrated? If the answer is "at initial deployment" and the strategy has been running for months, the parameters may be stale.
- Chan: parameters should be re-estimated periodically using walk-forward methodology.
- Severity: **P2**

---

## Principle 8: Regime detection is not optional for trend-following

*Chan: "Trend-following strategies lose money in mean-reverting regimes, and mean-reversion strategies lose money in trending regimes. The key is knowing which regime you're in."*

A trend-following bot that runs identically in trending and ranging markets will give back all trend profits during consolidation. The code must either detect regimes and adapt, or acknowledge that it will lose during adverse regimes and size accordingly.

### What to check

**No regime filter**
- Does the strategy apply the same logic regardless of market conditions? A trend-following strategy without a volatility or trend-strength filter will generate false signals in sideways markets.
- Chan's Hurst exponent test: H > 0.5 indicates trending, H < 0.5 indicates mean-reverting. The Variance Ratio test is a simpler alternative.
- Severity: **P2** for production strategies with no regime awareness.

**ADX/volatility filters with arbitrary thresholds**
- Is there an ADX > 25 filter (or similar)? Where did 25 come from? If it was optimised on the same data as the strategy, it adds another overfit parameter rather than reducing risk.
- Better: use the filter's threshold as a separate parameter validated out-of-sample.
- Severity: **P2** for unvalidated regime filter thresholds.

**Regime detection trained on future data**
- Does the regime classifier use information from the future? Example: fitting a Hidden Markov Model to the entire dataset and then labeling regimes retroactively. In live trading, you only know the regime after the fact — the classifier must use only past data.
- Severity: **P1** if the regime label uses future data.

---

## Principle 9: A false signal is more expensive than a missed signal

*Chan: "In trend-following, the biggest cost is not missing a trend — it's the whipsaws from false signals."*

Every false entry incurs fees (entry + exit), slippage (both ways), and opportunity cost (capital locked in a losing position). For a trend-following strategy with 35-40% win rate, the majority of trades are losers. The strategy is profitable only if the winners are large enough to cover the losers. Anything that increases the false signal rate destroys this math.

### What to check

**No signal confirmation**
- Does the strategy enter on a single indicator? Single-indicator entries have high false positive rates. Chan advocates requiring confirmation from at least two independent sources.
- "Independent" means not derived from the same underlying data. EMA(12) and EMA(26) are NOT independent — they both come from price. Price + volume is independent. Price + funding rate is independent.
- Severity: **P2** for single-indicator entries without documented rationale.

**No minimum holding period**
- Can the strategy enter and exit within the same candle or within 1-2 candles? Extremely short holding periods on low-frequency signals (5m, 15m candles) are almost always whipsaws.
- Check: what is the average holding period in backtest? If it's shorter than 3x the candle interval, the strategy is likely churning.
- Severity: **P2**

**Signal frequency vs opportunity**
- How many signals per day/week does the strategy generate? If it generates 50 signals per day on 5m candles, most are noise. If it generates 1 signal per week, each signal must be high quality.
- Chan: "The higher the frequency, the more edge you need to overcome transaction costs."
- Check: is the signal frequency consistent with the stated edge and fee structure?
- Severity: **P2** for high signal frequency without proportional edge documentation.

---

## Principle 10: Position sizing is half the strategy

*Chan: "Kelly criterion tells you the optimal bet size. Most traders ignore it and either bet too much or too little."*

A correct signal with wrong sizing is a losing strategy. Full Kelly is too aggressive (assumes perfect edge estimation). Half-Kelly is the practical standard.

### What to check

**Fixed position sizes**
- Does the strategy bet the same dollar amount on every trade regardless of signal strength or account size? Fixed sizing ignores the fundamental insight that stronger signals deserve larger bets.
- Severity: **P3** if the strategy is in early testing, **P2** for production.

**Kelly fraction calculation**
- If Kelly is used: is the win probability and payoff ratio estimated from out-of-sample data? Using in-sample estimates for Kelly is doubly overfit (overfit signal + overfit sizing).
- Chan: "I recommend using half-Kelly or less, because our estimates of edge are always noisy."
- Severity: **P2** for full Kelly, **P3** for half-Kelly or less.

**No position limit**
- Is there a maximum position size as a fraction of capital? Without a cap, a sequence of correlated signals can concentrate 100% of capital in one position.
- Check: is there a max_position or max_exposure parameter?
- Severity: **P1** for strategies that can allocate 100% to one position without a cap.

---

## Principle 11: Data quality is a prerequisite, not an assumption

*Chan: "Garbage in, garbage out. I have seen strategies that appeared profitable purely because of bad data — missing bars, wrong timestamps, stale prices."*

Before evaluating any signal, verify the data pipeline. A missing candle can create a phantom signal. A wrong timestamp can create look-ahead bias. A stale price can create a phantom profit.

### What to check

**Missing candle handling**
- What happens when a candle is missing from the feed? Does the code use the previous candle's close? Does it skip the bar? Does the indicator calculation handle gaps?
- If missing candles are filled forward, the indicators will flatten artificially, potentially generating false signals at the boundary.
- Severity: **P2**

**Timestamp alignment**
- Are candle timestamps in UTC? Are all data sources aligned to the same timezone? A 1-hour offset between price data and signal data shifts every entry by one bar.
- Check: is there an explicit timezone in the candle struct/parsing?
- Severity: **P1** for misaligned timestamps between data sources.

**Exchange-specific data quirks**
- Different exchanges report OHLCV differently during low-volume periods. Coinbase, Binance, and Polymarket each have different conventions for candles with zero volume.
- Is the code tested against the specific exchange's data format, or against a generic OHLCV assumption?
- Severity: **P2**

---

## Principle 12: Compounding is not free — the backtest must use realistic capital management

*Chan: "A backtest that assumes you can always invest $10,000 per trade regardless of prior losses is not testing a real strategy."*

If the strategy starts with $1,000, loses $200, and the next trade still uses $1,000 notional, the backtest is using infinite capital. Real capital management means position sizes scale with available equity.

### What to check

**Constant bet size regardless of equity**
- Does the backtest reduce position size after losses and increase after gains? If not, the P&L curve is unrealistically smooth and hides the impact of drawdowns.
- Severity: **P2**

**No margin/buying power check**
- Can the strategy enter a position that exceeds available capital? In prediction markets with binary outcomes, the max loss per position is the position size — but can the strategy open more positions than the account can cover?
- Severity: **P1** for strategies that can over-allocate.

**Unrealistic starting capital**
- Is the backtest run with the same capital as the live deployment? A strategy that works with $100K may not work with $1K due to minimum order sizes and proportionally higher fees.
- Severity: **P2**

---

## Principle 13: Mean reversion and momentum are mutually exclusive hypotheses — pick one

*Chan: "You cannot simultaneously believe that price reverts to the mean AND that price trends. Your strategy must commit to one or the other, or explicitly model the transition between them."*

A strategy that uses RSI for mean reversion AND EMA crossover for momentum is internally contradictory unless it has a regime switch. The RSI says "sell the rally" while the EMA says "buy the breakout." Without regime logic, these signals cancel out and the strategy becomes a random entry generator paying fees.

### What to check

**Contradictory indicators**
- Does the strategy combine trend-following indicators (EMA, MACD, Donchian) with mean-reversion indicators (RSI overbought/oversold, Bollinger band reversion) without a regime filter?
- If RSI > 70 is used as a sell signal while the EMA crossover says buy, which wins? Is this conflict handled explicitly in the code?
- Severity: **P2** for contradictory signals without conflict resolution logic.

**RSI as a trend filter (common antipattern)**
- RSI > 70 in a trend-following strategy is often used to "confirm momentum." But RSI > 70 is classically a mean-reversion sell signal. The developer must be explicit about which interpretation they're using and why.
- Severity: **P3** if documented, **P2** if undocumented.

---

## Principle 14: The number of trades in the backtest must be statistically significant

*Chan: "I need at least 100 trades in a backtest to have any confidence in the results. Ideally 1,000+."*

A strategy that generated 12 trades in backtest and 8 were winners has a 67% win rate — but the 95% confidence interval for the true win rate is [35%, 90%]. That's consistent with a coin flip. The sample size is too small to draw any conclusion.

### What to check

**Insufficient trade count**
- How many trades does the backtest produce? Fewer than 30 is statistically meaningless. Fewer than 100 is unreliable. 100-1000 is acceptable. 1000+ is robust.
- Chan: with N trades, win rate w, the standard error is sqrt(w*(1-w)/N). For 30 trades with 60% win rate, SE = 9% — the true win rate could easily be 50%.
- Severity: **P1** for strategies deployed to production with fewer than 100 backtest trades.

**Low-frequency strategies with short backtest periods**
- A strategy that trades once per week needs 2+ years of data to generate 100 trades. If the backtest covers 3 months, it has ~12 trades — meaningless.
- Check: does the backtest period generate enough trades for the stated signal frequency?
- Severity: **P2**

---

## Principle 15: Survivorship-free data is a requirement, not a luxury

*Chan: "If your data only contains assets that are still trading today, your backtest has survivorship bias."*

In prediction markets, this is critical. Markets that resolved to "No" (total loss for "Yes" holders) must be included. If the backtest only runs on markets that are currently active or that resolved to "Yes," it systematically overstates returns.

### What to check

**Backtest universe construction**
- How is the set of tradeable assets/markets selected? Is it "all markets available at time T" or "all markets that exist today"? The latter introduces survivorship bias.
- Severity: **P2**

**Delisted/resolved market handling**
- Are prediction markets that resolved during the backtest period included with their terminal value? A market that resolved to "No" should show a total loss for long positions.
- Severity: **P1** if resolved markets are excluded from the backtest.

---

## Principle 16: Beware the strategy that works on all timeframes

*Chan: "If a strategy works equally well on 1-minute, 5-minute, and daily bars, something is wrong. Different timeframes have different microstructure, different noise levels, and different participant types."*

A strategy that backtests profitably on every timeframe is likely overfitting to something universal (like autocorrelation of close prices, which is an artifact) or is not actually timeframe-dependent (which means it has no edge specific to any timeframe).

### What to check

**Tested on a single timeframe only**
- Was the strategy ONLY tested on 5m candles? What happens on 15m, 1h, 4h? If it completely breaks on adjacent timeframes, the 5m result may be noise.
- Chan: a robust strategy should degrade gracefully on nearby timeframes, not cliff-edge.
- Severity: **P2** for production strategies tested on only one timeframe.

**Identical parameters across timeframes**
- If the same EMA period and RSI threshold work on 5m and 1h, that's suspicious. Different timeframes should require different parameters because the signal-to-noise ratio changes.
- Severity: **P3** as a warning signal for deeper investigation.

---

## Principle 17: The Kelly criterion assumes you know your edge — you don't

*Chan: "The biggest risk in Kelly sizing is overestimating your edge. If your true edge is half what you think, full Kelly will ruin you."*

Kelly optimal sizing is f* = (bp - q) / b, where b is the odds, p is the win probability, q = 1-p. But p is estimated from backtests, which are noisy. An error of 5% in p can turn optimal sizing into reckless sizing.

### What to check

**Edge estimation from in-sample data**
- Is the win probability / edge used for sizing estimated from the same data as the strategy? This double-counts the overfitting.
- Severity: **P2**

**No fractional Kelly**
- Is the code using full Kelly (f*)? Chan recommends half-Kelly (0.5 * f*) as a floor for the estimation error.
- Check: is there a kelly_fraction or similar parameter? Is it < 1.0?
- Severity: **P2** for full Kelly without explicit justification.

**Bet size exceeding Kelly**
- If the position size is larger than Kelly optimal, the strategy has *negative* expected log-wealth growth. It will go to zero given enough time.
- Severity: **P1** for position sizes exceeding Kelly optimal.

---

## Principle 18: Max drawdown in live will exceed max drawdown in backtest

*Chan: "Whatever your worst backtest drawdown is, plan for at least 2x that in live trading."*

The backtest only covers historical regimes. Black swans, flash crashes, and novel market conditions are not in the data. A strategy with 15% max drawdown in backtest should plan for 30% in live.

### What to check

**Drawdown limits calibrated to backtest**
- If the kill switch triggers at 20% drawdown and the backtest max drawdown was 18%, the strategy will trigger its own kill switch under normal conditions.
- The kill switch should be set at 2-3x the backtest max drawdown to avoid stopping during expected drawdown periods.
- Severity: **P2** for kill switches set too close to backtest max drawdown.

**No drawdown recovery analysis**
- How long did it take to recover from the worst drawdown in backtest? If recovery takes 6 months, does the capital allocation plan account for this?
- Severity: **P3**

**Drawdown reported as percentage of peak**
- Drawdown should be measured from the equity peak, not from starting capital. A strategy that grows from $1K to $2K and then drops to $1.5K has a 25% drawdown, not a 50% gain.
- Check: how is drawdown calculated in the code?
- Severity: **P3** for incorrect drawdown calculation.

---

## Principle 19: Correlation to existing positions matters more than standalone returns

*Chan: "A strategy with a Sharpe of 1.0 that is uncorrelated to your existing portfolio is more valuable than a strategy with a Sharpe of 2.0 that is perfectly correlated."*

If the bot trades multiple markets or runs alongside other strategies, the correlation structure determines portfolio-level risk. Two profitable strategies that always lose at the same time provide no diversification.

### What to check

**No correlation analysis**
- If multiple markets or strategies are running simultaneously, is there analysis of their return correlation?
- Two BTC-correlated prediction markets are effectively one position. Sizing them independently overstates diversification.
- Severity: **P2** for multi-market strategies without correlation analysis.

**Concentrated exposure**
- Does the portfolio of active positions have concentrated exposure to one underlying risk factor (e.g., all positions are implicitly long BTC)?
- Check: is there a correlation matrix or exposure analysis in the codebase?
- Severity: **P2**

---

## Principle 20: The time between signal and execution is alpha decay

*Chan: "Every millisecond between signal generation and order execution is alpha leaking away."*

For a low-frequency strategy on 5m candles, this is less critical than for HFT — but the principle still applies. If the signal fires at candle close and the order is placed 30 seconds later, the market may have moved. On prediction markets with thin books, even small delays can mean worse fills.

### What to check

**Signal-to-execution latency**
- How long between signal generation and order submission? Is this measured and logged?
- For 5m candle strategies, anything under 5 seconds is fine. But if the execution pipeline involves multiple API calls (fetch book, compute size, place order), the total latency can exceed 10 seconds.
- Severity: **P3** for low-frequency strategies, **P1** for high-frequency.

**Stale signal acting on new data**
- If the signal was computed on candle N's close but the order fills during candle N+1, the market context has changed. Is the strategy aware of this?
- Severity: **P2** for strategies where the holding period is short relative to execution latency.

---

## Principle 21: A profitable backtest with no losing periods is a lie

*Chan: "Show me a backtest with no losing months and I'll show you an overfit strategy."*

Every legitimate strategy has drawdown periods. Trend-following strategies lose during consolidation. Mean-reversion strategies lose during trends. If the equity curve is monotonically increasing, the strategy was tuned to avoid every historical losing period — which means it was tuned to fit noise.

### What to check

**Suspiciously smooth equity curves**
- Plot the backtest equity curve mentally (or check if the code generates one). Are there losing periods? If not, the strategy has too many degrees of freedom.
- Severity: **P2** for equity curves with no drawdown > 5%.

**Monthly/weekly return distribution**
- Are monthly returns normally distributed around a positive mean with occasional negative months? Or are they all positive? The latter is a red flag.
- Check: does the backtest report return distribution statistics, or only the aggregate?
- Severity: **P3**

---

## Principle 22: Round-trip cost must be less than average profit per trade

*Chan: "A strategy's average profit per trade must exceed the round-trip transaction cost by a comfortable margin."*

This is the most basic viability check. If average profit per trade is 0.3% and round-trip costs (entry fee + exit fee + slippage) are 0.25%, the strategy survives — but one bad month of slippage kills it. The margin should be at least 2x costs.

### What to check

**Average profit per trade vs round-trip cost**
- Calculate: (total P&L after fees) / (number of trades). Compare to (entry fee + exit fee + estimated slippage).
- If profit per trade < 2x round-trip cost, the strategy is fragile.
- Severity: **P1** if profit per trade < round-trip cost. **P2** if between 1x and 2x.

**Fee structure changes**
- Exchange fees change. A strategy calibrated to 0.01% maker fees that encounters 0.05% taker fees (because limit orders didn't fill and fell back to market orders) has a 5x cost increase.
- Check: does the code handle fee tier changes? Is the fee hardcoded or configurable?
- Severity: **P2** for hardcoded fees.

---

## Principle 23: Never optimise on total P&L — optimise on Sharpe or profit factor

*Chan: "Optimizing on total P&L will give you the strategy that took the biggest risks, not the best strategy."*

Total P&L maximisation picks the strategy variant with the highest variance, not the highest risk-adjusted return. A variant that made $50K with 30% drawdown will be chosen over one that made $40K with 10% drawdown. The latter is unambiguously better.

### What to check

**Optimisation objective**
- If there's a parameter search / grid search in the code, what metric is being maximised? Total P&L, Sharpe ratio, profit factor, or something else?
- Acceptable: Sharpe ratio, Sortino ratio, profit factor (gross profits / gross losses), Calmar ratio (CAGR / max drawdown).
- Unacceptable: total P&L, total return, number of winning trades.
- Severity: **P2** for optimisation on total P&L.

**Multi-objective without ranking**
- If the code reports multiple metrics but doesn't rank parameter sets by a single risk-adjusted metric, the developer will cherry-pick the one that looks best — which is implicit optimisation on total P&L.
- Severity: **P3**

---

## Principle 24: Inventory risk is real — the longer you hold, the more you're exposed

*Chan: "Time in the market without edge is time losing money to transaction costs and opportunity cost."*

A trend-following strategy that enters and the trend reverses is now holding a losing position. The longer it holds, the more it loses. Exit logic is as important as entry logic — and usually receives less attention in code reviews.

### What to check

**No exit logic review**
- Entry signals get 90% of the attention. But exit signals determine whether winning trades are large enough to cover the losers.
- Check: is the exit logic as rigorously defined as the entry logic? Are there explicit take-profit AND stop-loss levels?
- Severity: **P2** for strategies with well-defined entries but vague exits.

**Trailing stops that never trigger**
- A trailing stop set too far from the entry will never trigger during normal moves, converting the strategy into a buy-and-hold.
- Check: what percentage of backtest trades hit the trailing stop vs the take-profit vs the signal reversal? If trailing stops trigger in < 5% of trades, they're not functional.
- Severity: **P3**

**Hold time analysis**
- What is the average and maximum holding period? Does it match the stated strategy (e.g., a "short-term momentum" strategy that holds for 3 days on average is not short-term on 5m candles).
- Severity: **P3**

---

## Principle 25: Log everything, but analyse only what matters

*Chan: "The most valuable data is your own live trading data. Analyse it ruthlessly."*

Live trading generates data that backtests cannot: actual fill prices, actual slippage, actual latency, actual market impact. This data is gold for strategy refinement — but only if it's logged and reviewed.

### What to check

**Missing live performance logs**
- Does the code log: entry price, exit price, fill slippage vs expected price, holding period, signal strength at entry, P&L per trade?
- If the bot logs "bought" and "sold" but not the prices and reasons, live performance cannot be analysed.
- Severity: **P1** for production bots without per-trade logging.

**No live vs backtest comparison**
- Is there a mechanism to compare live performance to backtest expectations? If live Sharpe is 50% of backtest Sharpe, the strategy may be decaying or the backtest may have been overfit.
- Chan: "When my live Sharpe drops below 50% of my backtest Sharpe, I reduce position size or stop trading."
- Severity: **P2** for production strategies without live/backtest comparison.

**No attribution analysis**
- When the strategy loses money, can you identify whether the loss came from: bad signal (wrong direction), bad timing (right direction, wrong entry), bad sizing (right direction, too large), or bad exit (right direction, gave back profits)?
- Check: does the logging support this decomposition?
- Severity: **P3**

---

## Severity Guide

| Severity | Meaning | Action |
|----------|---------|--------|
| **P1** | The strategy may lose money due to this issue. Deployment should not proceed. | Fix before deploying. |
| **P2** | The strategy's profitability is uncertain or fragile due to this issue. | Fix before scaling up capital. |
| **P3** | Best practice violation that increases long-term risk. | Track and fix in next iteration. |
