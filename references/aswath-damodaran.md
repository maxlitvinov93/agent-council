# Financial Scoring & Valuation Reference — Carmack x Damodaran

Philosophy: John Carmack. Valuation expertise: Aswath Damodaran (NYU Stern, "Dean of Valuation," author of *Investment Valuation*, *The Little Book of Valuation*, *Narrative and Numbers*).
Stack context: Python / FastAPI / SQLite / financial scoring models (Piotroski, Graham, Buffett, Lynch, Greenblatt) / composite 0-10 scoring / sector-aware calibration.

Every finding must describe the **concrete valuation error** — not just "this metric is wrong."
Security patterns are in security.md (Hunt). Backend patterns are in quality-backend.md (Collina). This doc covers: metric selection, sector-specific calibration, score composition, fair value methodology, financial statement quality, and backtest validity.

---

## Principle 1: Every metric has a sector where it lies — never apply a universal screen

*Carmack: "If a mistake is possible, it will eventually happen."*
*Damodaran: "The biggest mistake in valuation is using one approach for all companies. You cannot value a bank the way you value a manufacturing company."*

P/E applied to a bank is meaningless because earnings are after provisions and heavily regulated. EV/EBITDA applied to a financial company is wrong because debt is operating capital, not financing — EV itself is undefined. Revenue multiples applied to a utility are absurd because revenue is rate-regulated and meaningless as a growth signal.

### What to check

**P/E for financials (banks, insurance, asset managers)**
- Banks earn the spread between borrowing and lending rates. Their "earnings" are after loan loss provisions — a management judgment that can swing net income by 30%+. Screening banks on P/E without adjusting for provision cycles will flag cheap banks that are about to write off loans.
- Correct metric: Price/Book (P/B) or Price/Tangible Book (P/TBV) as the primary multiple. ROE as the quality filter. Net Interest Margin (NIM) for operational efficiency.
- Check: does the code have a sector dispatch (`if sector == "Financials"`) that switches the primary valuation metric? If every company runs through the same `calculate_pe_score()`, flag it.
- Severity: **P1** — produces systematically wrong valuations for ~20% of the S&P 500.

**EV/EBITDA for financials**
- Enterprise Value = Market Cap + Debt - Cash. For a bank, debt IS the business — a bank's deposits, interbank borrowings, and bonds are its raw materials. EV for JPMorgan includes trillions in deposits, making the ratio meaningless.
- Damodaran: "Enterprise value multiples don't work for financial service firms because debt for a bank is raw material, not a source of capital."
- Check: does the code compute `ev = market_cap + total_debt - cash` for all companies, then use `ev / ebitda` as a valuation signal? Banks, insurance companies, and broker-dealers must be excluded or given a sector-specific alternative.
- Severity: **P1**

**EV/EBITDA for capital-light tech and asset-heavy industries**
- For capital-light SaaS, EBITDA ≈ operating income because D&A is trivial. EV/EBITDA and EV/EBIT converge. But for telecom, airlines, or oil & gas, D&A is massive and reflects real economic cost. EV/EBITDA makes an airline look cheap by ignoring the depreciation of its fleet.
- Check: is there a fallback to EV/EBIT or EV/FCF for capital-intensive sectors? If `ebitda` is used universally, the screener overvalues asset-heavy businesses.
- Severity: **P2**

**Revenue multiples without growth context**
- EV/Revenue is only meaningful for high-growth pre-profit companies where other metrics don't exist yet. Applying it to a mature utility or REIT produces nonsense rankings.
- Check: is `ev / revenue` used as a scoring component for companies with stable, low-growth revenue? If yes, it should be gated by growth rate or excluded for mature sectors.
- Severity: **P2**

---

## Principle 2: Normalize earnings — the cycle is not a signal

*Carmack: "If you can't see the cost, you can't reason about correctness."*
*Damodaran: "A common mistake in valuation is to use the current year's earnings as normal earnings. For cyclical companies, earnings at the peak of the cycle are not normal earnings."*

A commodity company with record earnings at the top of the cycle will look cheap on trailing P/E. A screener that doesn't cyclically adjust will recommend buying at the peak and selling at the trough — the exact opposite of value investing.

### What to check

**Trailing earnings without normalization**
- Does the scoring code use `ttm_earnings` (trailing twelve months) directly? For cyclical sectors (energy, materials, industrials, autos, semiconductors), TTM earnings at cycle peak inflate quality scores and deflate valuation multiples.
- Damodaran's fix: use average earnings over 5-10 years (Shiller CAPE approach), or normalize margins to mid-cycle averages.
- Check: is there a `normalize_earnings()` or `cyclical_adjustment()` function? If all models consume raw TTM data, cyclical stocks will be systematically mispriced.
- Severity: **P1** for screeners that recommend cyclicals based on peak earnings.

**One-time items in earnings**
- Asset sales, litigation settlements, restructuring charges, tax law changes — these inflate or deflate a single year's earnings. A company that sold a division for $2B will show a massive P/E improvement that vanishes next quarter.
- Check: does the code strip `extraordinary_items`, `restructuring_charges`, or `discontinued_operations` from earnings? If using `net_income` directly from financial statements without adjustment, one-time items contaminate every earnings-based metric.
- Severity: **P2**

**Negative earnings**
- P/E is undefined for loss-making companies. `earnings / price` when earnings < 0 produces a negative ratio that breaks percentile-based scoring.
- Check: how does the code handle `eps <= 0`? Common bugs: division by zero, negative P/E ranked as "cheapest," or NaN propagating through composite scores. Correct approaches: exclude from P/E scoring, use EV/Revenue or EV/Gross Profit, or assign a floor score.
- Severity: **P1** — NaN in a composite score silently corrupts the final output.

---

## Principle 3: DCF is a framework, not a formula — garbage in, garbage out

*Carmack: "Understand every number that comes out of your code."*
*Damodaran: "A DCF valuation is only as good as its inputs. If you don't think about what goes into the model, you're just doing accounting with a discount rate."*

A DCF model with a 5% terminal growth rate and a 15% revenue CAGR projected for 10 years will produce a spectacularly wrong intrinsic value. The code can be mathematically correct and economically absurd.

### What to check

**Terminal growth rate > GDP growth**
- The terminal value typically represents 60-80% of total DCF value. If `terminal_growth_rate` exceeds long-term nominal GDP growth (~4-5% in nominal, ~2-3% real), the model implies the company will eventually become larger than the economy.
- Damodaran: "No firm can grow faster than the economy forever. The terminal growth rate should be less than or equal to the growth rate of the economy."
- Check: is `terminal_growth_rate` a hardcoded constant, a user input without bounds, or capped at a sensible maximum? If the code allows `terminal_g = 0.08`, the DCF is fantasy.
- Severity: **P1**

**Revenue growth projected without fade**
- High-growth companies decelerate. A 30% grower today will not grow at 30% in year 10. Damodaran's base rate: of companies growing revenue >25%/year, fewer than 10% sustain it beyond 5 years.
- Check: does the projection model have a `growth_fade()` function that tapers growth toward a terminal rate? If `projected_revenue[year] = revenue * (1 + growth_rate) ** year` with a constant rate, the model is systematically overvaluing growth stocks.
- Severity: **P2**

**WACC estimation errors**
- Common bugs: using book value weights instead of market value weights for WACC, using trailing beta calculated on monthly returns (noisy) for cost of equity, assuming a constant debt/equity ratio in perpetuity.
- Damodaran: "If you're going to use a cost of capital, at least get the weights right. The market value of equity is what matters, not book value."
- Check: is `wacc = (equity_weight * cost_of_equity) + (debt_weight * cost_of_debt * (1 - tax_rate))` using market cap for equity weight and market value of debt (or a reasonable approximation) for debt weight? If using `total_equity` from the balance sheet, the WACC is wrong.
- Severity: **P2**

---

## Principle 4: Composite scores need economic justification — weights are claims about the world

*Carmack: "Every number your code produces should be traceable to a reason."*
*Damodaran: "If you're going to create a scoring system, you need to be able to explain why each component is there and why it has the weight you give it."*

A composite like Quality 40% + Value 15% + Momentum 25% + Timing 20% is not a neutral engineering choice — it's a claim that quality matters 2.67x more than value and that momentum matters 1.67x more than value. This claim needs empirical backing or at minimum explicit acknowledgment.

### What to check

**Weight justification**
- Are the composite weights documented with a rationale? "Quality is 40% because Novy-Marx (2013) showed gross profitability predicts cross-section returns with t-stat > 5" is defensible. "Quality is 40% because it felt right" is not.
- Check: look for a `WEIGHTS` dict or config. Are the values explained in comments or docs? If weights are magic numbers, the composite score is arbitrary.
- Severity: **P2**

**Value at 15% is suspiciously low for a stock screener**
- If the tool is marketed or used as a value-oriented screener (Graham, Buffett, Piotroski models are all value paradigms), giving Value only 15% weight while Momentum gets 25% is internally contradictory. The models say "buy cheap quality" but the composite says "buy trending stocks."
- Damodaran: "Know what you're screening for. A value screen that gives most of its weight to momentum is not a value screen."
- Check: do the underlying model scores (Piotroski, Graham, etc.) feed into a composite where their contribution is diluted below the momentum component? If yes, document the tension.
- Severity: **P3** — this is a design choice, but it should be deliberate and documented.

**Double-counting across factors**
- If Quality uses ROE and Value uses P/E, both are driven by earnings. A company with temporarily high earnings scores well on both, and the "diversification" between factors is illusory.
- Check: compute the correlation matrix between factor scores across the universe. If Quality and Value have >0.5 correlation, they're partially measuring the same thing, and their combined weight overstates a single signal.
- Severity: **P2**

**Score normalization method**
- Are raw metrics converted to scores via percentile rank, z-score, min-max, or fixed thresholds? Each choice has consequences. Percentile rank forces a uniform distribution — exactly 10% of stocks score 9-10, regardless of whether any are actually good. Z-scores are distorted by outliers. Min-max is dominated by extremes.
- Damodaran prefers fixed thresholds with economic meaning: P/E < 15 is "cheap" (Graham's number), ROE > 15% is "quality" (Buffett's threshold). These are interpretable but don't adapt to market regimes.
- Check: is the normalization method documented? Does it produce sensible outputs when the entire market is expensive (all stocks should score low on value) or when outliers are present (one stock with P/E = 2000 shouldn't dominate)?
- Severity: **P2**

---

## Principle 5: Sector-specific calibration — banks are not tech, utilities are not REITs

*Carmack: "Generic solutions hide domain-specific failure modes."*
*Damodaran: "I have sector-specific approaches for a reason. A bank's balance sheet is fundamentally different from a tech company's. A REIT's cash flow is not the same as a manufacturer's."*

A screener that applies the same thresholds, metrics, and weights across all sectors will systematically misflag certain sectors. Banks always look "cheap" on P/E (they trade on book value). Utilities always look "slow growth" (they're rate-regulated). REITs always look "over-leveraged" (their debt funds real estate, not operations).

### What to check

**Financials (banks, insurance, diversified financials)**
- Primary metrics: P/B, P/TBV, ROE, efficiency ratio, NIM, CET1 ratio (capital adequacy).
- Must NOT use: EV/EBITDA, EV/Revenue, Debt/Equity (debt is the business), Free Cash Flow (undefined — a bank's "investment" is making loans).
- Piotroski F-Score: the asset turnover and leverage signals are meaningless for banks. A bank increasing its leverage ratio may be growing its loan book (good), not overleveraging.
- Check: does the Piotroski implementation have a financial-sector variant that replaces leverage/turnover signals?
- Severity: **P1**

**REITs**
- Primary metrics: P/FFO (Funds From Operations), P/AFFO (Adjusted FFO), dividend yield, NAV discount/premium.
- Must NOT use: P/E (depreciation distorts earnings — a REIT with old, fully depreciated buildings shows high earnings but low economic value), FCF (capex for property acquisition is the core business).
- Check: does the code detect REIT sector and switch to FFO-based metrics? If using `eps` for REITs, every REIT valuation is wrong.
- Severity: **P1**

**Utilities**
- Rate-regulated businesses have capped returns. Growth screening is meaningless — a utility growing revenue at 8% may simply have gotten a rate increase, not a competitive advantage.
- Primary metrics: dividend yield, payout ratio, regulated vs unregulated mix, rate base growth.
- Check: are utilities penalized in growth scores? If the model rewards revenue growth equally across sectors, utilities will always score low and be ignored.
- Severity: **P2**

**High-growth tech (pre-profit or early-profit)**
- Many high-quality tech companies have negative earnings, negative free cash flow, and high stock-based compensation. Graham, Buffett, and Piotroski models will all score them 0-2 because these models were designed for mature value companies.
- Primary metrics: revenue growth, gross margin trajectory, rule of 40, net dollar retention, FCF margin trend.
- Check: does the screener acknowledge that classic value models are structurally incapable of scoring growth-stage tech? If a $50B SaaS company with 40% revenue growth, 75% gross margins, and negative net income scores 1/10, the screener is correct within its framework but misleading to users.
- Severity: **P2** — not a bug, but a known limitation that should be documented prominently.

---

## Principle 6: Growth rates revert to the mean — always

*Carmack: "Extrapolation from short-term data is the most common engineering failure mode."*
*Damodaran: "If there is one lesson I have learned in 30 years of valuation, it is that growth rates revert to the mean. High growth does not last forever."*

### What to check

**Unbounded growth projections**
- Does any growth-based score extrapolate recent growth indefinitely? If `growth_score = min(10, revenue_cagr_3y * 20)` gives a perfect 10 for 50% CAGR with no decay assumption, it overvalues high-growth stocks.
- Damodaran's base rates: of companies growing >25%/year, <15% sustain it for 5 years, <5% sustain it for 10 years. Growth fades toward the economy's growth rate.
- Check: does the growth scoring function incorporate any mean-reversion assumption? If it scores purely on trailing growth rate, it assumes persistence that doesn't exist.
- Severity: **P2**

**Compounding small-sample CAGR**
- CAGR calculated from 2 data points (start and end) can be wildly misleading. If revenue went $100M -> $50M -> $200M, the 2-year CAGR is 41% but the business nearly died.
- Check: does the code compute `(end/start) ** (1/years) - 1` on just two data points? It should also check the intermediate trajectory, or use regression-based growth to smooth volatility.
- Severity: **P2**

**SBC-inflated growth**
- Stock-based compensation inflates reported revenue growth in tech. A company that acqui-hires aggressively shows revenue growth funded by shareholder dilution, not organic business expansion.
- Check: is share dilution factored into growth metrics? Revenue growth of 30% with 10% annual share dilution is really 20% per-share growth.
- Severity: **P3**

---

## Principle 7: Margin of safety is not a fixed discount — it depends on uncertainty

*Carmack: "Safety margins should scale with the uncertainty of your inputs."*
*Damodaran: "The margin of safety should be a function of the uncertainty you feel about your valuation. For a stable utility, a 10% discount may be sufficient. For a young growth company, you might need 40% or more."*

Graham's blanket "buy at 2/3 of intrinsic value" is a starting point, not a universal rule. A 33% margin of safety on a utility with predictable cash flows is excessive. The same 33% margin on an early-stage biotech is insufficient.

### What to check

**Fixed margin of safety across all companies**
- Does the code use a single threshold like `if price < intrinsic_value * 0.7: undervalued = True`? This treats a utility and a biotech with equal uncertainty.
- Check: is `margin_of_safety` a function of something — sector volatility, earnings predictability, business age, or analyst forecast dispersion? If it's a constant, the screener is overconfident on uncertain companies and overly demanding on stable ones.
- Severity: **P2**

**Intrinsic value precision**
- An intrinsic value of $147.32 implies 5 significant figures of precision. The inputs (growth rate, discount rate, terminal value) are lucky to have 1 significant figure.
- Damodaran: "I don't believe in a single intrinsic value number. I believe in a range."
- Check: does the code produce a point estimate or a range? If a point estimate, is the downstream logic aware that `intrinsic_value = 147.32` really means "somewhere between $110 and $190"?
- Severity: **P3** — presentation issue, but creates false confidence.

---

## Principle 8: Financial statement quality — the numbers can lie

*Carmack: "Never trust input that hasn't been validated."*
*Damodaran: "The first thing I do before valuing a company is read the footnotes. Revenue recognition choices, operating lease capitalization, pension fund assumptions — these are where companies hide their true economics."*

### What to check

**Revenue recognition games**
- Bill-and-hold revenue (booking revenue before delivery), channel stuffing (shipping excess inventory to distributors), long-term contract revenue pulled forward — these inflate current revenue and deflate future revenue.
- Check: does the code use raw `total_revenue` without any quality adjustment? At minimum, compare revenue growth to cash flow from operations growth. If revenue grows 20% but operating cash flow grows 5%, revenue quality is suspect.
- Severity: **P2**

**Stock-based compensation treatment**
- Many tech companies exclude SBC from "adjusted" metrics. SBC is a real cost to shareholders — it dilutes ownership. Adding back SBC to get "adjusted earnings" overstates profitability.
- Damodaran: "Stock-based compensation is a real expense. If you add it back, you're double-counting — the shares are already being diluted."
- Check: does the code use `adjusted_ebitda` or `adjusted_earnings` from the data provider that adds back SBC? If yes, the quality and value scores are inflated for heavy-SBC companies (most of tech).
- Severity: **P2**

**Operating lease capitalization**
- Pre-ASC 842 (and for companies still reporting under older standards internationally), operating leases were off-balance-sheet. Damodaran capitalizes them by treating lease payments as debt. Post-ASC 842, this is largely handled, but check the data source.
- Check: is `total_debt` from the data provider inclusive of lease liabilities? If not, retailers, airlines, and restaurants appear less levered than they are.
- Severity: **P3** for post-2019 US data (ASC 842 handles it). **P2** for international or historical data.

**Share dilution masking per-share growth**
- Total revenue growing 25% while shares outstanding grow 15% means per-share revenue only grew ~9%. A screener that scores on total metrics ignores dilution.
- Check: are valuation metrics computed on a per-share basis where appropriate? `revenue_growth` should ideally be checked against `revenue_per_share_growth`. If diluted share count isn't used, flag it.
- Severity: **P2**

---

## Principle 9: Comparable selection — wrong peers destroy relative valuation

*Carmack: "Comparing two things that aren't comparable gives you an answer that's confidently wrong."*
*Damodaran: "The key to using multiples well is to pick the right comparable companies. Same sector is necessary but not sufficient — you need similar growth, risk, and cash flow profiles."*

### What to check

**Sector-only comparables**
- Grouping all "Technology" companies together puts Apple (2T market cap, 25% margins, hardware + services) in the same bucket as a pre-revenue SaaS startup. Their "average P/E" is meaningless.
- Check: does the code use SIC/GICS sector codes as the sole grouping key? Subsector (GICS industry or sub-industry) is the minimum. Even then, controlling for size, growth rate, and margins improves comparisons dramatically.
- Severity: **P2**

**Survivorship bias in comparables**
- If the comparable universe is the current S&P 500, you're comparing against survivors. Companies that went bankrupt, were acquired, or delisted are excluded — biasing the "average" upward.
- Check: is the comparable universe static (current constituents) or does it include historical constituents at each point in time? For backtest purposes, this is critical (see Principle 14).
- Severity: **P2** for scoring, **P1** for backtests.

**Size mismatch**
- Comparing a $500M small cap to the sector median when the median is dominated by $50B+ companies. Small caps trade at persistent discounts due to illiquidity, lower analyst coverage, and higher risk. A small cap at P/E = 12 vs sector median P/E = 20 is not necessarily cheap — it may be appropriately discounted.
- Check: does the code segment comparables by market cap band? If a $200M company is scored against a universe including FAANG, it will always look "undervalued."
- Severity: **P2**

---

## Principle 10: Return on capital — the right metric for the right question

*Carmack: "Precision in naming prevents precision errors."*
*Damodaran: "ROIC is my preferred measure because it strips out the effects of leverage and focuses on the operating business. ROE mixes operating performance with capital structure decisions."*

### What to check

**ROE as a standalone quality metric**
- ROE = Net Income / Shareholders' Equity. A company can boost ROE by taking on more debt (reducing equity through buybacks or leveraging up). High ROE with high leverage is risk, not quality.
- Damodaran: "A company that earns a 20% ROE with a debt-to-equity ratio of 3 is not the same quality as one that earns 18% ROE with no debt."
- Check: is ROE used without an accompanying leverage check? If the quality score uses ROE alone, highly levered companies will score as "high quality." Pair ROE with Debt/Equity or use ROIC instead.
- Severity: **P2**

**ROIC calculation correctness**
- ROIC = NOPAT / Invested Capital, where NOPAT = Operating Income * (1 - Tax Rate) and Invested Capital = Total Equity + Total Debt - Cash (or equivalently, Fixed Assets + Net Working Capital).
- Common bugs: using `net_income` instead of NOPAT (includes interest and one-time items), using `total_assets` instead of invested capital (includes non-operating assets and excess cash), not adjusting the tax rate (using statutory rate instead of effective rate).
- Check: trace the ROIC calculation step by step. Is `nopat = operating_income * (1 - tax_rate)`? Is `invested_capital` computed correctly? If ROIC is computed as `net_income / total_assets`, it's ROA, not ROIC.
- Severity: **P2**

**ROA for financials**
- For banks, ROA (Return on Assets) is more meaningful than ROIC because invested capital is undefined — a bank's assets ARE its investments (loans).
- Check: does the code use ROIC for financial companies? If yes, the denominator is wrong. Switch to ROA or ROE with an explicit leverage context.
- Severity: **P2**

---

## Principle 11: Debt is not a number — it's a structure

*Carmack: "A single number hides the distribution."*
*Damodaran: "Looking at total debt tells you nothing. You need to know: when does it mature, what rate is it at, what currency is it in, and can the company service it?"*

### What to check

**Debt/Equity ratio without context**
- D/E = 2.0 for a utility with stable regulated cash flows and 30-year fixed-rate bonds is conservative. D/E = 2.0 for a cyclical manufacturer with floating-rate debt maturing in 18 months is dangerous. The number is the same; the risk is vastly different.
- Check: does the leverage score use D/E as a single number, or does it incorporate interest coverage ratio (EBIT/Interest Expense), debt maturity profile, and fixed vs floating rate mix? A D/E-only score is incomplete.
- Severity: **P2**

**Interest coverage ratio thresholds**
- Damodaran uses interest coverage as the primary input to his synthetic credit rating model. Coverage < 1.25x is distressed territory. Coverage > 8.5x is roughly AAA/AA.
- Check: is interest coverage computed and used? If the debt analysis only looks at leverage ratios without checking whether the company can actually pay its interest, the score misses imminent distress.
- Severity: **P2**

**Net debt vs gross debt**
- A company with $5B debt and $4B cash has $1B net debt. Scoring on gross debt alone penalizes cash-rich companies. Conversely, scoring on net debt alone can hide the rollover risk of $5B in debt maturities even if cash covers it today.
- Check: which measure does the code use? Both have legitimate uses. The key is consistency and awareness of the trade-off.
- Severity: **P3**

---

## Principle 12: Dividend analysis must use total shareholder return, not just yield

*Carmack: "Optimizing a partial metric leads to globally wrong conclusions."*
*Damodaran: "Dividends are just one form of cash return. Buybacks have become equally important, and ignoring them gives you a distorted picture of how much a company returns to shareholders."*

### What to check

**Dividend yield without buyback consideration**
- A company with 1% dividend yield and 5% buyback yield returns 6% total to shareholders — more than a company with 4% dividend yield and no buybacks. Scoring only on dividend yield penalizes buyback-heavy companies (most of modern tech and financials).
- Check: does the dividend/income score incorporate `total_shareholder_yield = dividend_yield + buyback_yield`? If only `dividend_yield`, the model is biased toward traditional dividend payers and against buyback-heavy companies.
- Severity: **P2**

**Dividend sustainability**
- A 10% dividend yield is not a gift — it's often a warning. Either the stock price has crashed (yield is high because denominator fell) or the payout ratio is unsustainable.
- Check: does the code check `payout_ratio = dividends / earnings`? Payout > 100% means the company is paying dividends from debt or reserves. For REITs, use FFO-based payout ratio instead of earnings-based.
- Severity: **P2**

**Price return vs total return in backtests**
- If the backtest measures strategy performance using price returns only, it systematically undervalues dividend-paying stocks. A strategy that selects high-yield stocks and shows 5% annual return is really producing 5% + ~3% yield = 8%.
- Check: does the backtest engine use `total_return = price_return + dividend_return`, or just price changes? If only price, the backtest is biased against income strategies.
- Severity: **P1** for backtests, **P3** for real-time scoring.

---

## Principle 13: Backtest validity — most backtests lie

*Carmack: "If you can't define the failure mode, you can't trust the success case."*
*Damodaran: "Most backtests are exercises in data mining. You look at enough variables over enough time periods, and something will look significant."*

### What to check

**Survivorship bias**
- If the backtest universe is current S&P 500 constituents applied to historical data, every company that went bankrupt, was delisted, or was acquired before today is excluded. The survivors have an upward bias — of course a portfolio of "today's winners tested over 20 years" looks good.
- Check: does the backtest use point-in-time constituent lists? If `universe = get_sp500_tickers()` returns today's list and applies it to 2015 data, the backtest has survivorship bias.
- Severity: **P1**

**Look-ahead bias**
- Using data that wasn't available at the time of the simulated decision. Financial statements are available weeks to months after the period end. If the backtest uses Q4 earnings to make a January 1 decision, it's using data that wasn't public yet.
- Check: does the code respect reporting lag? Is there a `as_of_date` or `report_date` field that the backtest uses to filter available data? If it uses `fiscal_period_end_date` as the availability date, it's using data ~45 days before it was public.
- Severity: **P1**

**Data snooping / multiple testing**
- If 50 scoring weights were tried and the best one is reported, the "best" result is likely overfit. Without out-of-sample validation, the reported performance is the maximum of 50 random walks, not a predictive signal.
- Check: is there a train/test split in the backtest? Is performance reported on held-out data? If the backtest optimizes parameters on the full dataset, the results are unreliable.
- Severity: **P1**

**Transaction cost and slippage**
- A strategy that trades 200 small-cap stocks weekly incurs massive transaction costs and slippage. A backtest that ignores these costs will show unrealistic returns.
- Check: does the backtest include a `transaction_cost` parameter? Is slippage modeled for illiquid names? If returns are computed as `target_price / entry_price - 1` without costs, the backtest overstates performance.
- Severity: **P2**

---

## Principle 14: Market cap normalization — small cap scoring requires different lenses

*Carmack: "Treating heterogeneous data as homogeneous produces homogeneously wrong results."*
*Damodaran: "Small companies look different on every dimension — higher growth, higher risk, lower margins, less data. Comparing them to large caps on the same scale is unfair to both."*

### What to check

**Percentile scoring across all market caps**
- If scores are percentile-ranked across the full universe, large caps cluster in the middle (low volatility, moderate growth, moderate value) and dominate the comfortable 4-7 range, while small caps dominate both extremes (best and worst). This isn't signal — it's a mechanical artifact of small-cap volatility.
- Check: are percentile ranks computed within market cap bands (mega, large, mid, small, micro) or across the entire universe? If across, a $100M micro cap and a $500B mega cap are directly compared.
- Severity: **P2**

**Data coverage gaps**
- Small caps have less analyst coverage, fewer reported metrics, and more frequent data gaps. A score that requires 10 metrics but only 6 are available for a small cap will either NaN out or impute — both are problems.
- Check: how does the code handle missing data? Is there a `min_data_completeness` threshold? Are missing metrics imputed (with what?), excluded from the composite (changing effective weights), or defaulted to a penalty value?
- Severity: **P2**

**Liquidity-adjusted scoring**
- A $50M market cap stock scoring 9/10 may be uninvestable — daily volume might be $200K, meaning any meaningful position takes weeks to build and moves the price.
- Check: does the scoring system factor in liquidity (average daily volume, bid-ask spread, market cap)? If not, top-scored stocks may be practically untradeable.
- Severity: **P3** for scoring, **P1** for portfolio construction or trade execution.

---

## Principle 15: Model-specific integrity — Piotroski, Graham, Buffett, Lynch, Greenblatt each have precise definitions

*Carmack: "If you name something, implement exactly what the name promises."*
*Damodaran: "These models are well-defined. If you're implementing Piotroski, implement Piotroski — don't invent your own variation and call it Piotroski."*

### What to check

**Piotroski F-Score (9 binary signals)**
- The F-Score has exactly 9 components: (1) ROA > 0, (2) Operating Cash Flow > 0, (3) Delta ROA > 0, (4) Cash Flow > Net Income (accruals), (5) Delta Leverage < 0, (6) Delta Current Ratio > 0, (7) No new shares issued, (8) Delta Gross Margin > 0, (9) Delta Asset Turnover > 0.
- Check: does the implementation match all 9? Common errors: using ROE instead of ROA, ignoring the accruals signal (#4), computing delta on wrong time periods (should be YoY, not QoQ). Each signal is binary (0 or 1), sum is 0-9. If the code produces fractional F-Scores, the implementation is wrong.
- Severity: **P2**

**Graham Number**
- Graham Number = sqrt(22.5 * EPS * BVPS). This gives an upper bound on price. The 22.5 is not arbitrary — it's 15 (max P/E) * 1.5 (max P/B).
- Check: is the formula correct? Is it using trailing EPS (not forward)? Is book value per share (not total book value)? Is the square root taken? Common bug: `graham = 22.5 * eps * bvps` without the sqrt.
- Severity: **P2**

**Greenblatt Magic Formula**
- Ranks companies by two factors: earnings yield (EBIT/EV) and return on capital (EBIT / (Net Fixed Assets + Net Working Capital)). The magic is in the ranking, not the raw values — each stock gets a rank on each factor, and the combined rank determines the score.
- Check: is the ranking mechanism implemented correctly? Common error: scoring on absolute EBIT/EV values instead of ranking. The formula specifically uses EBIT (not net income), EV (not market cap), and a specific definition of invested capital (not total assets).
- Severity: **P2**

**Lynch classification**
- Lynch classifies companies into: slow growers, stalwarts, fast growers, cyclicals, turnarounds, asset plays. The PEG ratio (P/E / growth rate) is central but only applies to growers. A cyclical at the bottom of its cycle has infinite PEG (low/negative earnings, no growth) — applying PEG to cyclicals is a misuse.
- Check: does the Lynch implementation classify companies before scoring? If PEG is applied to all companies regardless of Lynch category, it will produce nonsense for cyclicals and turnarounds.
- Severity: **P2**

---

## Principle 16: Currency and accounting standard consistency

*Carmack: "Unit mismatches are the deadliest silent bugs."*
*Damodaran: "When comparing companies across countries, you must ensure you're comparing in the same currency and adjusting for differences in accounting standards."*

### What to check

**Mixed currencies in comparisons**
- If the screener covers global stocks, a Japanese company reporting in JPY and a US company reporting in USD cannot be compared on raw earnings or revenue numbers. Even ratios like P/E can differ due to accounting standard differences.
- Check: are all monetary values converted to a common currency before comparison? If the code computes sector median P/E across companies reporting in different currencies, the P/E itself is unit-free but the underlying components (EPS computation) may differ due to accounting standards.
- Severity: **P2** for global screeners, **P3** for US-only.

**GAAP vs IFRS differences**
- R&D is expensed under US GAAP but can be capitalized under IFRS. This means an IFRS company's earnings are higher and its assets are larger than the identical company under GAAP. Operating income, total assets, and book value are all affected.
- Check: does the data source normalize across accounting standards, or does the code need to handle this? If mixing GAAP and IFRS companies in the same score computation without adjustment, margins and returns are not comparable.
- Severity: **P2**

---

## Principle 17: Time period sensitivity — the window changes the answer

*Carmack: "Parameters that silently control output must be explicit."*
*Damodaran: "A company that looks cheap on trailing earnings can look expensive on forward earnings. Always be clear about which number you're using."*

### What to check

**Trailing vs forward metrics**
- TTM P/E and forward P/E can diverge massively. A cyclical at peak earnings has low TTM P/E (looks cheap) and high forward P/E (analysts expect earnings decline). Using TTM exclusively biases toward cyclicals at peaks.
- Check: does the code clearly label whether it uses `ttm_pe` or `forward_pe`? Are both available? If only TTM, the screener has a systematic cyclical bias.
- Severity: **P2**

**Lookback period for growth**
- 1-year growth is noisy, 5-year growth misses regime changes. A company that grew 50% last year after 4 years of decline shows high 1Y growth but near-zero 5Y CAGR.
- Check: does the growth score use a single lookback period or multiple? If `growth_score = score(revenue_cagr_3y)`, it misses the recent trajectory. Using multiple periods (1Y, 3Y, 5Y) with weights gives a more robust signal.
- Severity: **P3**

---

## Principle 18: The data quality chain — your model is only as good as your data source

*Carmack: "Never trust input you haven't validated."*
*Damodaran: "Garbage in, garbage out applies to valuation more than to any other field. If the data is wrong, the most sophisticated model will give you wrong answers."*

### What to check

**Data provider discrepancies**
- Different providers compute "EBITDA," "free cash flow," and "operating income" differently. Yahoo Finance EBITDA may differ from Bloomberg EBITDA by 10-20% for the same company due to different add-back conventions.
- Check: does the code document which data provider it uses and how each metric is defined? If switching from one provider to another, are the score distributions validated to be consistent?
- Severity: **P2**

**Stale data**
- Financial data can be stale by quarters. If the code uses `last_reported_eps` without checking the reporting date, a company that hasn't reported in 8 months has 8-month-old earnings powering today's score.
- Check: is there a staleness check? Something like `if report_date < today - timedelta(days=120): flag_as_stale`. If stale data silently feeds into scores, the output is unreliable.
- Severity: **P2**

**Null handling and default values**
- Financial datasets are sparse. Some companies don't report certain line items. How the code handles `None` / `NaN` determines whether the score is correct, degraded, or corrupted.
- Check: trace the path of a `None` value through the scoring pipeline. Does it produce `NaN` in the final score? Does it default to 0 (which may mean "worst" or "best" depending on the metric)? Does it exclude the metric from the composite (changing effective weights)?
- Severity: **P1** for NaN propagation into final scores. **P2** for silent default values that distort scoring.

---

## Principle 19: The narrative check — does the number tell a coherent story?

*Carmack: "If the output doesn't make intuitive sense, the code is wrong."*
*Damodaran: "Every valuation should be accompanied by a narrative. If the numbers say a company is worth $500 per share but the story doesn't make sense, the numbers are wrong."*

### What to check

**Sanity bounds on outputs**
- Can the model produce a score of 10/10 for a company losing money, burning cash, and diluting shareholders? If yes, the scoring logic has a bug or the composite weights allow extreme factor scores to dominate.
- Check: run the model on known edge cases — a company in bankruptcy (should score near 0), a company like Berkshire Hathaway at its best (should score high), a pre-revenue biotech (most models shouldn't apply). Do the outputs match economic intuition?
- Severity: **P2**

**Contradictory factor scores**
- A company scoring 9/10 on Value and 9/10 on Quality is either a generational bargain or a data error. In efficient markets, high quality trades at a premium (high valuations), making simultaneous high-quality + high-value rare.
- Check: does the system flag or log when a company scores >8 on both Value and Quality? This doesn't mean it's wrong — it means it warrants attention. An automated sanity alert for unusual score combinations catches data errors.
- Severity: **P3**

---

## Principle 20: Beware of model vintage — markets evolve, factors decay

*Carmack: "Code that was correct last year may be wrong today if the environment changed."*
*Damodaran: "Factor premiums are not constant. The value premium has shrunk, the momentum premium has shifted, and any published factor is partially arbitraged away once it becomes known."*

### What to check

**Hardcoded thresholds from decades-old models**
- Graham's "P/E < 15 and P/B < 1.5" was calibrated for the 1970s. The S&P 500 median P/E hasn't been below 15 for sustained periods in decades. A rigid Graham screen finds zero stocks in most market environments.
- Check: are thresholds adaptive or fixed? If fixed, are they documented as "classic" thresholds vs. updated ones? A screener that returns 0 stocks because the market doesn't match 1970s conditions is technically correct but practically useless.
- Severity: **P3** if documented. **P2** if the screener silently returns empty results.

**Factor crowding**
- Published factors (value, momentum, quality) get crowded as more capital chases them. The value premium post-publication is roughly half of the pre-publication premium. A backtest showing 5% annual alpha from value in 1960-2000 will not replicate in 2020-2030.
- Check: does the documentation or scoring methodology acknowledge factor decay? If backtested alpha is presented as expected future alpha without a haircut, the user is being misled.
- Severity: **P3**

---

## Principle 21: Never score what you can't explain — interpretability is not optional

*Carmack: "If you can't explain why a number is what it is, you don't understand your system."*
*Damodaran: "A black box valuation is not a valuation. If you can't decompose the number into its drivers, you shouldn't trust it."*

### What to check

**Score decomposition**
- Can a user see WHY a stock scored 7.3/10? Which factors contributed, which detracted? If the composite score is a single opaque number, the user can't validate it and can't learn from it.
- Check: does the API or UI provide a breakdown — e.g., `{quality: 8.5, value: 4.2, momentum: 7.8, timing: 6.1}` — alongside the composite? If only the composite is exposed, flag the lack of transparency.
- Severity: **P2**

**Metric attribution**
- Within each factor, which metrics drove the score? Quality = 8.5 because ROE = 22% (good), margin trend positive (good), but D/E = 2.3 (concerning). Without this level of detail, the score is a magic number.
- Check: does the scoring code produce intermediate results that are stored or returned? If `calculate_quality_score()` returns a single float with no trace of its inputs, debugging and explanation are impossible.
- Severity: **P2**

---

## Principle 22: Handle regime changes — what works in bull markets fails in bear markets

*Carmack: "Test under adversarial conditions, not just the happy path."*
*Damodaran: "Valuation approaches that work in stable markets break down in crises. Relative valuation during a bubble tells you everything is fairly priced — relative to other overpriced stocks."*

### What to check

**Relative valuation in market extremes**
- If every stock in a sector trades at P/E > 30, the "cheapest" at P/E = 25 scores as "undervalued" in a percentile-rank system. But it's still expensive in absolute terms. Conversely, in a crash, even P/E = 8 may rank as "average" because everything is cheap.
- Check: does the scoring system include any absolute valuation anchor (e.g., earnings yield vs bond yield, absolute P/E vs historical average) in addition to relative ranking? If purely relative, the screener cannot identify regime-wide overvaluation or undervaluation.
- Severity: **P2**

**Momentum in crashes**
- Momentum (buying recent winners) works in trending markets and fails catastrophically in reversals. A 25% weight on momentum means the composite score will recommend stocks at the top of their run right before a crash.
- Check: does the momentum scoring include any mean-reversion or volatility adjustment? Is there a regime detector that reduces momentum weight during high-volatility periods? If momentum is always 25%, the screener is vulnerable to crash-induced drawdowns.
- Severity: **P2**

---

## Principle 23: Free cash flow is not free — define it precisely

*Carmack: "Ambiguous definitions produce ambiguous results."*
*Damodaran: "There are at least three definitions of free cash flow in common use, and they give different answers. If you don't specify which one you're using, you don't know what you're measuring."*

### What to check

**FCF definition consistency**
- FCFF (Free Cash Flow to Firm) = Operating Cash Flow - Capex + Interest*(1-Tax). FCFE (Free Cash Flow to Equity) = Operating Cash Flow - Capex + Net Borrowing. Simple FCF = Operating Cash Flow - Capex. These differ significantly for leveraged companies.
- Check: which definition does the code use? Is it consistent across all uses — the same FCF definition in the value score, quality score, and DCF model? If `value_score` uses FCFF but `quality_score` uses FCFE, the signals are based on different economics.
- Severity: **P2**

**Capex classification**
- Maintenance capex (keeping assets running) and growth capex (expanding capacity) have different economic meanings. Subtracting total capex from operating cash flow penalizes companies investing for growth. But separating them is difficult — most companies don't disclose the split.
- Damodaran: "I use the depreciation charge as a rough proxy for maintenance capex. It's imperfect but better than treating all capex the same."
- Check: does the code distinguish maintenance from growth capex? If not, is the limitation documented? A company spending heavily on growth capex will show low FCF but may be creating future value.
- Severity: **P3**

---

## Principle 24: Timing scores are the hardest to get right — acknowledge the uncertainty

*Carmack: "The most dangerous code is the code that gives confident answers to uncertain questions."*
*Damodaran: "I don't believe in market timing. But I do believe that intrinsic value provides an anchor, and when price deviates far enough from that anchor, the odds shift in your favor."*

### What to check

**Technical indicators as valuation signals**
- RSI, MACD, moving average crossovers — these are price pattern indicators, not valuation tools. Including them in a composite score alongside fundamental models (Graham, Buffett, Piotroski) mixes two incompatible frameworks. The fundamental models say "this company is worth X." The technical indicators say "the price is trending Y." These are independent claims.
- Check: if a Timing component uses technical indicators, is it clearly separated from the fundamental score? Does the UI make clear that Timing is a momentum/sentiment signal, not a valuation judgment? If Timing can override a low fundamental score (or vice versa), the interaction should be explicit.
- Severity: **P3** if separated. **P2** if blended into a single score without clear labeling.

**Timing weight in composite**
- A 20% weight on Timing means a stock with perfect fundamentals (8/10 on Quality, Value, Momentum) but bad technical timing (2/10) gets penalized to ~6.8/10. The timing signal has a half-life of days to weeks. The fundamental signal has a half-life of quarters to years. Giving them comparable weight implies comparable reliability.
- Check: is the Timing weight justified by the signal's demonstrated persistence? If timing signals decay within days but the screener updates weekly, the timing score is mostly noise by the time the user acts on it.
- Severity: **P3**

---

## Gaps: What This Doc Doesn't Cover

- **Macroeconomic factors**: interest rate sensitivity, inflation adjustment, currency risk for international portfolios. Damodaran covers this in his country risk premium work, but it's deep enough for its own reference doc.
- **Option-adjusted valuation**: companies with significant option pools, convertible debt, or warrants. Damodaran's approach to adjusting for dilutive securities is important but specialized.
- **Private company valuation**: illiquidity discounts, control premiums, key-person risk. Not relevant for a public stock screener.
- **ESG scoring integration**: increasingly demanded but methodologically immature. Damodaran is skeptical: "ESG ratings are more about the rating agency's preferences than about the company's actual environmental or social impact."
- **Real-time data feeds**: streaming quotes, real-time financials. This doc assumes batch-processed fundamental data.

---

## Quick Reference: Severity Guide

| Severity | Pattern | Examples |
|----------|---------|----------|
| **P1 — Fix Now** | Produces systematically wrong valuations, corrupted outputs, or misleading backtests | P/E for banks, EV/EBITDA for financials, NaN propagation in composite scores, survivorship bias in backtests, look-ahead bias, data snooping without holdout, terminal growth > GDP, negative earnings crashing P/E, total return missing from backtests |
| **P2 — Fix Soon** | Distorts scores or hides important nuance, but doesn't produce outright wrong numbers | Cyclical earnings not normalized, SBC added back, missing sector calibration for REITs/utilities, wrong ROIC formula, single-period CAGR, no interest coverage in debt analysis, fixed margin of safety, stale data, relative-only valuation, undocumented weights, ROE without leverage context |
| **P3 — Consider** | Design choices that should be documented and deliberate, not accidental | Fixed historical thresholds, factor decay acknowledgment, intrinsic value precision, timing weight justification, net debt vs gross debt choice, SBC-inflated growth, capex classification |

### The Overriding Filter

Before writing any finding, apply the Carmack-Damodaran synthesis:

1. **Does the metric match the sector?** If a universal metric is applied to a sector where it's meaningless (EV/EBITDA for banks, P/E for REITs), flag it as P1. (Damodaran: sector-specific approaches exist for a reason.)
2. **Is the number normalized?** If raw trailing data is used without cyclical adjustment, one-time item removal, or dilution correction, flag it. (Damodaran: current earnings are not normal earnings.)
3. **Is the methodology transparent?** If a score, weight, or threshold can't be explained to a user in one sentence, it's a black box. (Both: if you can't explain it, you don't understand it.)
4. **Does the backtest respect time?** If future data leaks into historical decisions, if dead companies are excluded, or if parameters were optimized on the test set, the results are fiction. (Both: test under real conditions.)
5. **Does the output pass the smell test?** If a bankrupt company scores 8/10 or Berkshire scores 2/10, something is wrong. Trust economic intuition over model output. (Damodaran: numbers without narrative are just numbers.)
