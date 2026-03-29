# Financial Data Visualization Reference — Tufte

Authority: Edward Tufte. Author of *The Visual Display of Quantitative Information*, *Envisioning Information*, *Visual Explanations*, and *Beautiful Evidence*. The foremost authority on presenting quantitative data clearly and truthfully.

Stack context: Next.js / React / TypeScript / Recharts / CSS Modules. Financial data product displaying composite scores (0-10), price charts, comparison tables, sector analysis, and multi-model scoring matrices. Data-dense analytical interface.

Every finding must describe the **concrete data-communication failure** — not just "this violates Tufte's principles." The question is always: does the reader understand the data faster and more accurately, or does the design impede understanding?

**Scope boundary:** This doc covers data visualization quality: chart design, table design, number presentation, data density, and truthful data representation. It does NOT cover: general UI patterns (quality-ui.md), UX flows (quality-ux.md), accessibility beyond color-blindness in charts, or backend data correctness.

---

## Principle 1: Maximise the data-ink ratio — every pixel must earn its place

*Tufte: "Above all else show the data." The Visual Display of Quantitative Information, p. 92.*
*Tufte: "A large share of ink on a graphic should present data-information, the ink changing as the data change. Data-ink is the non-erasable core of a graphic, the non-redundant ink arranged in response to variation in the numbers represented."*

The data-ink ratio is the proportion of a graphic's ink devoted to the non-redundant display of data-information. Maximise it. Every gridline, background fill, border, legend box, and axis tick mark that does not help the reader decode a specific data value is wasted ink — visual noise that competes with the signal. In Recharts components, the defaults are maximally decorated: gridlines on both axes, filled legend boxes, thick axis lines, cartesian grids. The reviewer must verify that these defaults have been stripped back.

### What to check

**Recharts default gridlines left intact**
- `<CartesianGrid>` renders a full grid by default. For most financial charts (price history, score trends), horizontal gridlines alone are sufficient — the reader's eye tracks horizontally to the Y axis. Vertical gridlines rarely help because the X axis (time) is read sequentially. Check: is `<CartesianGrid>` present, and if so, does it use `horizontal={true} vertical={false}` with a muted stroke like `strokeDasharray="3 3"` and low opacity?
- The concrete cost: full grids create a cage that the data line sits inside, flattening the visual hierarchy between the data and the scaffolding.
- Severity: **P2** for full cartesian grids on line/area charts. **P3** for bar charts where vertical gridlines may occasionally aid comparison.

**Axis lines that duplicate the data range**
- Recharts renders `<XAxis>` and `<YAxis>` with visible axis lines and tick marks by default. When gridlines are present, the axis line itself is redundant — the gridline at zero or at the axis edge already marks the boundary. Check: `axisLine={false}` and `tickLine={false}` when gridlines provide the reference.
- Severity: **P3**

**Decorative chart borders and backgrounds**
- Chart containers with visible borders, background fills, or drop shadows. A score trend chart does not need a card border to distinguish it from surrounding content — spacing and typography do that work. Check: is the chart wrapped in a container with decorative styling that could be removed?
- Severity: **P3** for borders. **P2** if a background fill reduces contrast with the data.

**Legend boxes with borders and symbols**
- Recharts `<Legend>` renders boxed entries with colored squares. Tufte's approach: integrate the label directly into the data — label the line itself at its endpoint, or place the label near the most recent data point. When a legend is necessary (multiple series), it should be minimal: colored text with no enclosing box, positioned near the data it describes.
- Severity: **P3** for standard legends. **P2** if the legend is far from the data it describes (e.g., centered below a chart with 5+ series), forcing the reader to repeatedly cross-reference.

---

## Principle 2: Erase chartjunk — decoration that does not convey data is harm

*Tufte: "Chartjunk does not achieve the goals of its propagators. The overwhelming fact of data graphics is that they are looked at and decoded in the course of analytical thinking about the substance." — VDQI, p. 121.*
*Tufte: "Graphical decoration... comes cheaper than the hard work required to produce intriguing numbers and selectively display them."*

Chartjunk includes moiré patterns, gratuitous 3D effects, heavy fills, unnecessary icons inside charts, gradient fills that imply a third dimension that doesn't exist, and animated transitions that celebrate the visualization instead of revealing the data. In financial dashboards, chartjunk is especially costly: users are making decisions based on what they see. A gradient fill on a price chart that makes the bottom of the area darker than the top implies that lower prices are "heavier" — a meaningless visual metaphor that could bias perception.

### What to check

**Gradient fills on area charts**
- Recharts `<Area>` components commonly use `<linearGradient>` fills fading from a colour to transparent. This creates a visual weight at the top of the filled area that has no data meaning. For a stock price area chart, the fill should either be a uniform low-opacity colour (communicating the range) or absent entirely (line chart). Check: do `<Area>` components use gradient fills? If so, does the gradient direction imply something about the data that isn't true?
- Severity: **P2** for gradients that could mislead (e.g., suggesting volume or density). **P3** for purely decorative gradients that don't mislead but add no information.

**Animated chart transitions on load**
- Charts that animate line-drawing or bar-growing on initial render. Tufte: the graphic should be instantly readable. Animation that delays comprehension by 500ms-1000ms while bars grow from zero serves the developer's pride, not the reader's understanding. Check: `isAnimationActive={false}` on Recharts components for data that should be immediately readable. Animation is acceptable only for data transitions (e.g., switching time ranges) where it helps the reader track what changed.
- Severity: **P2** for initial-load animations on primary data displays. **P3** for secondary/supplementary charts.

**Decorative icons or illustrations inside charts**
- Emoji, icons, or illustrations placed inside chart areas (e.g., a bull/bear icon overlaid on a price chart). These compete with the data for the reader's attention and add zero quantitative information. Check: is any non-data visual element rendered inside the chart's data area?
- Severity: **P2**

**3D effects or perspective distortion**
- Any chart rendered with a perspective or 3D effect. 3D pie charts, 3D bar charts, or any projection that distorts the data plane. These make accurate comparison impossible because foreshortening changes the perceived size of data marks. Recharts does not natively support 3D, but custom SVG effects or CSS transforms can create the illusion. Check: does any chart use `transform: perspective()` or 3D SVG effects?
- Severity: **P1** — 3D distortion on financial data is a data integrity issue.

---

## Principle 3: Control the lie factor — represent data proportions truthfully

*Tufte: "The representation of numbers, as physically measured on the surface of the graphic itself, should be directly proportional to the numerical quantities represented." — VDQI, p. 56.*
*Tufte defines the Lie Factor as: (size of effect shown in graphic) / (size of effect in data). A Lie Factor of 1.0 is truthful. Values substantially above or below 1.0 distort.*

Financial data is consequential. A chart that makes a 2% price decline look like a crash, or a chart that makes a 30% gain look flat, can mislead investment decisions. The lie factor operates through truncated axes, inconsistent scales across comparison charts, area/volume encoding of linear data, and aspect ratios that exaggerate or compress trends.

### What to check

**Truncated Y-axis that exaggerates variation**
- A score chart showing values from 7.0 to 8.5 with the Y-axis starting at 7.0 makes a 0.5-point difference fill the entire chart height, visually equating it with a full-scale change. For scores on a 0-10 scale, the Y-axis should either start at 0 (showing true proportion) or explicitly mark the truncation with a broken axis indicator. Check: does `<YAxis domain={[min, max]}` use a range that exaggerates variation? Compute: (visual range / full data range) vs (displayed range / full data range).
- The most common violation: Recharts `domain={['dataMin', 'dataMax']}` auto-fits the Y-axis to the data, which for a stock that moved from $148 to $152 creates a chart that looks like a 50% swing.
- Severity: **P1** for price charts with truncated axes and no truncation indicator. **P2** for score charts where the scale (0-10) is known and visible.

**Inconsistent scales across comparison charts**
- When two stocks or two models are displayed side-by-side for comparison, their Y-axes must share the same scale. If Stock A is shown on a 0-200 scale and Stock B on a 0-50 scale, a visual comparison is meaningless — equal heights represent different values. Check: in any comparison layout (small multiples, side-by-side charts), are the Y-axis domains identical?
- Severity: **P1** — this is a direct data integrity failure in a comparison context.

**Area encoding of linear quantities**
- Using circle area or bar width to represent a linear value (e.g., a company's score) without proper sqrt-scaling. Tufte: "the number of information-carrying (variable) dimensions depicted should not exceed the number of dimensions in the data." A score of 8 shown as a circle with radius 8 (area 201) next to a score of 4 with radius 4 (area 50) makes the 8 look four times as big, not twice. If using sized marks: area, not radius, must be proportional to the value.
- Severity: **P1** for financial metrics encoded in unscaled area/volume. **P2** for decorative sized elements that don't invite precise comparison.

**Aspect ratio that distorts trends**
- Tufte (in *The Visual Display*) recommends banking to 45 degrees: the average absolute slope of the line segments should be approximately 45 degrees. A very wide chart compresses volatility; a very tall chart exaggerates it. For a score trend over 12 months, a 3:1 or 4:1 width-to-height ratio typically produces honest slopes. Check: is the chart aspect ratio reasonable for the data range and time period? A chart showing 5 years of daily prices at 1:1 will flatten everything.
- Severity: **P2** for aspect ratios that visibly compress or exaggerate the data story.

---

## Principle 4: Use small multiples for comparison — the same design structure, repeated

*Tufte: "Small multiples are economical: once viewers understand the design of one slice, they have immediate access to the data in all the other slices. Small multiples... answer the most frequently asked question: 'compared to what?'" — Envisioning Information, p. 67.*

Financial comparison is the core use case: how does this stock compare to its sector peers? How does Model A's score compare to Model B? Small multiples — the same chart design repeated across a grid, changing only the data — are the most effective structure for these comparisons. They exploit the reader's ability to detect pattern differences across a consistent frame, rather than forcing them to decode overlay colours or remember values from one chart to the next.

### What to check

**Overlaid multi-series where small multiples would be clearer**
- A single `<LineChart>` with 5+ companies overlaid in different colours. Beyond 3-4 series, overlays become a tangle — colours are hard to distinguish, lines cross and obscure each other, and the reader can't track any individual series. Check: when comparing more than 3-4 entities, are small multiples used instead of a single overlaid chart? A grid of 6 identically-scaled sparkline charts is more informative than a single chart with 6 coloured lines.
- Severity: **P2** for 4+ overlaid series. **P1** for 6+ overlaid series where individual series are unreadable.

**Small multiples with inconsistent scales**
- Small multiples only work when the visual frame is identical. If each panel auto-scales its Y-axis to its own data range, the reader cannot compare heights across panels — the entire purpose of small multiples is defeated. Check: in any grid of repeated charts, do all panels share the same `domain` on both axes?
- Severity: **P1** — inconsistent scales in a comparison grid is worse than no comparison at all, because it invites false conclusions.

**Small multiples with excessive labelling**
- Each panel in a small-multiple grid does not need its own full axis labels, tick marks, and title. The frame is shared — label the axes once (on the left panel and bottom panel of the grid) and let interior panels inherit the structure. Check: are axis labels and ticks repeated on every panel in a grid, wasting space and adding clutter?
- Severity: **P3**

---

## Principle 5: Deploy sparklines for data-dense inline display

*Tufte: "Sparklines are datawords: data-intense, design-simple, word-sized graphics." — Beautiful Evidence, p. 47.*
*Tufte: "Sparklines have obvious application in financial data... a sparkline of price history next to a stock ticker symbol gives immediate context that no number alone can provide."*

A score of 7.2 tells you the current state. A sparkline showing the score's trajectory over 90 days tells you whether 7.2 is a recovery, a decline, or stable. In tables and lists where space is constrained, sparklines deliver trend context in the space of a single table cell. They should be word-sized — the same height as the text around them — and require no axes, labels, or interaction.

### What to check

**Scores displayed as bare numbers without trend context**
- A table column showing "Score: 7.2" with no visual indication of trajectory. The reader cannot distinguish a stable 7.2 from a crashing-toward-7.2 from a recovering-toward-7.2 without navigating to a detail view. Check: do tables displaying scores include inline sparklines or micro-trend indicators (even a simple up/down/flat arrow derived from recent data)?
- Severity: **P2** for primary score displays in tables. **P3** for secondary/supplementary scores.

**Sparklines with excessive decoration**
- A sparkline with axes, gridlines, tooltips, or hover interactions. These transform a data-word into a chart, defeating the purpose. A sparkline should be a bare polyline (or area) at text height, with at most two reference marks: the starting value and the current value, or the high and low. Check: do sparkline components include `<XAxis>`, `<YAxis>`, `<Tooltip>`, or `<CartesianGrid>`?
- Severity: **P2** for sparklines with chart furniture. **P3** for sparklines with only tooltips.

**Sparklines not aligned with surrounding text baseline**
- A sparkline that vertically overflows its table cell or sits at a different vertical position than the text in adjacent cells. Tufte: sparklines are "word-sized" — they should participate in the typographic baseline. Check: is the sparkline height set to match the line-height of the surrounding text (typically 16-20px), and is it vertically centered or baseline-aligned within its cell?
- Severity: **P3**

---

## Principle 6: Format numbers for human comprehension — appropriate precision and units

*Tufte: "Tables usually outperform graphics in reporting on small data sets of 20 numbers or less." The corollary: when you use numbers, make them instantly readable.*
*Tufte: "Confusion and clutter are failures of design, not attributes of information."*

Financial interfaces routinely display numbers that are unreadable: raw penny-precise market caps ($14,293,847,291.00), scores with meaningless decimal places (7.142857), percentages with four decimal places (12.3456%). Every unnecessary digit is a cognitive burden that slows scanning and obscures the meaningful magnitude. The reviewer must check that every displayed number uses the precision and format appropriate to its context.

### What to check

**Raw large numbers without magnitude suffixes**
- Market cap displayed as "$14,293,847,291" instead of "$14.3B". Revenue displayed as "$2,847,391,000" instead of "$2.85B". Trading volume displayed as "48,291,847" instead of "48.3M". The reader is forced to count digits to determine magnitude. Check: do all numbers above one million use K/M/B suffixes? Is the precision appropriate (typically 1-2 decimal places after the suffix)?
- Severity: **P1** for financial values (market cap, revenue, volume) displayed as raw digits. **P2** for non-financial large numbers.

**Excessive decimal precision**
- Scores displayed as "7.142857" when "7.1" conveys the same usable information. Percentages displayed as "12.3456%" when "12.3%" is sufficient. The number of decimal places should reflect the meaningful precision of the underlying data, not the floating-point representation. For scores on a 0-10 scale: one decimal place. For percentages: one decimal place. For dollar values: zero decimals for values > $1000, two decimals for values < $10. Check: does each number type have a consistent formatting rule?
- Severity: **P2** for inconsistent or excessive precision. **P3** for minor formatting inconsistencies.

**No alignment of numbers in tables**
- Numbers in table columns that are left-aligned or center-aligned. Tufte (and every typographic authority): numeric columns must be right-aligned so that digits of the same magnitude line up vertically. This lets the reader instantly compare magnitudes by visual length. In React/CSS: `text-align: right` and monospace font (`font-variant-numeric: tabular-nums` or a monospace typeface like JetBrains Mono). Check: are numeric table columns right-aligned with tabular figures?
- Severity: **P2** for left-aligned numeric columns in comparison tables. **P3** for non-tabular numeric displays.

**Inconsistent number formats across the interface**
- Market cap shown as "$14.3B" in one view and "$14,293M" in another. Score shown as "7.1" in a card and "7.14" in a table. The reader should never have to reconcile different representations of the same number. Check: is there a shared formatting utility (e.g., `formatLargeNumber()`, `formatScore()`) used consistently across all components?
- Severity: **P2**

---

## Principle 7: Design tables as data displays, not as decorated containers

*Tufte: "Tables are clearly the best way to show exact numerical values." — VDQI, p. 178.*
*Tufte: "See the data." A well-designed table is itself a graphic — the structure of rows, columns, alignment, and whitespace communicates relationships.*

A comparison table showing 5 models x N companies is the core analytical tool in a scoring product. Tufte's table design principles: eliminate heavy rules and fills, use whitespace and light rules to separate, right-align numbers, left-align text, remove units from cells (put them in the header), and make the data — not the table furniture — dominate. Heavy grid borders, alternating row colours, bold headers that outweigh the data: all impede the reader.

### What to check

**Heavy grid borders on data tables**
- Tables with visible borders on every cell (the HTML default or `border: 1px solid` on `<td>`). Tufte eliminates most rules, keeping only a top rule, a rule below headers, and a bottom rule. Internal horizontal rules are light or absent — whitespace separates rows. Vertical rules are almost never needed (column alignment does the separation). Check: does the table use heavy borders? Can they be replaced with subtle horizontal rules and spacing?
- Severity: **P2** for full-grid borders on a primary data table. **P3** for secondary tables.

**Alternating row colours (zebra striping)**
- A common pattern that Tufte would question: does the reader need the stripe to track across the row? If columns are close together and the table is narrow, stripes are unnecessary — and they add visual noise that competes with the data. For wide tables where row-tracking is needed, very subtle alternation (2-3% opacity difference) is acceptable. Check: are zebra stripes adding value or just noise? Is the opacity difference minimal?
- Severity: **P3** for subtle stripes. **P2** for high-contrast stripes that visually dominate the data.

**Units repeated in every cell instead of the header**
- A column where every cell reads "$14.3B", "$2.1B", "$892M" instead of a header "Market Cap ($B)" with cells reading "14.3", "2.1", "0.89". Repeating the unit in every cell is redundant ink. Move the unit to the column header. Exception: when mixed magnitudes (M and B) make a single unit impractical, use magnitude-aware formatting but keep it consistent.
- Severity: **P3** for unit repetition. **P2** if the repetition makes cells wider and forces horizontal scrolling or truncation.

**Header styling that dominates the data**
- Bold, large, dark header text with a heavy background fill, followed by lighter, smaller data cells. This inverts the hierarchy — the reader came for the data, not the column names. Headers should be lighter, smaller, or uppercase-muted. The data should be the visually dominant element. Check: is the header row styled more prominently than the data rows?
- Severity: **P2** for headers that visually outweigh the data. **P3** for mildly heavy headers.

**Model comparison matrix without clear reading order**
- A 5-model x N-company matrix with no visual hierarchy. The reader should be able to scan by company (across models) or by model (across companies) with equal ease. Check: is the highest score in each row/column visually indicated (subtle bold or background)? Is there a sort control? Can the reader quickly identify which model scored highest for a given company?
- Severity: **P2** for matrices with no visual scanning aids. **P3** for matrices that are readable but lack highlights.

---

## Principle 8: Use colour to encode data, not to decorate

*Tufte: "Above all, do no harm." Colour should reveal data, not obscure it.*
*Tufte: "Colour used redundantly with position, as an information-reinforcing element, works well — colour used to encode a new variable is far more difficult to decode."*

Colour in financial visualizations has two legitimate roles: encoding a data variable (red/green for loss/gain, a sequential scale for score magnitude) and distinguishing categorical groups (different companies in the same chart). Decorative colour — chart series in rainbow colours chosen for aesthetics, colourful backgrounds, gradient fills for visual interest — adds no information and can actively mislead by implying categorical or quantitative differences that don't exist.

### What to check

**Red/green as the only loss/gain encoding**
- Approximately 8% of men have red-green colour vision deficiency. If positive/negative is encoded solely by red vs green, these users cannot read the data. Check: does the design use a secondary encoding — direction (up arrow / down arrow), position (above/below a baseline), or a perceptually distinct palette (blue/orange, or Tufte's recommended blue/red)? This is not a general accessibility check — it's specific to the quantitative reading of financial gains and losses.
- Severity: **P1** for gain/loss encoding using only red/green with no secondary channel. **P2** if colour-blind-safe colours are used but without a shape/direction backup.

**Rainbow or categorical colours on ordered data**
- Score values (0-10) encoded in a rainbow palette (red, orange, yellow, green, blue). Rainbow palettes imply no ordering — is purple higher or lower than green? For ordered numerical data, use a sequential single-hue or two-hue diverging scale (e.g., red-to-green through neutral for scores, or a single hue with varying lightness). Check: do colour scales match the data type? Sequential data gets sequential colours. Categorical data gets distinct categorical colours.
- Severity: **P2** for rainbow on ordered data. **P3** for suboptimal but still readable colour scales.

**Too many colours in one view**
- Tufte recommends restraint: "use colour only when it is needed to serve a particular communication goal." More than 5-7 distinguishable colours in a single chart or table makes accurate decoding impossible — the reader cannot reliably match a colour to its legend entry. For a sector analysis chart with 11 sectors: group into top 5 + "Other", or use a small-multiples approach. Check: does any single chart use more than 6-7 distinct colours?
- Severity: **P2** for 7+ colours in a single chart. **P3** for 5-6 colours that are distinguishable but approaching the limit.

**Colour with no legend or label**
- A coloured element in a chart or table with no explanation of what the colour encodes. The reader must guess or inspect code. Every colour that encodes data must be labelled, whether via a legend, direct annotation, or a header label. Check: can the reader determine what every colour means without hovering or clicking?
- Severity: **P1** for unlabelled colour encoding on primary data displays.

---

## Principle 9: Design comparison layouts to answer the reader's question

*Tufte: "At the heart of quantitative reasoning is a single question: Compared to what?" — Visual Explanations, p. 35.*

Stock comparison is the product's core analytical function. The layout must be designed around the specific comparison the reader is making: same metric across companies (use a single chart with shared axis), same company across metrics (use aligned rows), temporal comparison (overlay or small multiples). The layout choice determines what comparisons the reader can make at a glance vs what requires memorisation and mental arithmetic.

### What to check

**Side-by-side charts with independent scales**
- Two price charts placed next to each other but with different Y-axis ranges. The reader's eye naturally compares heights, but equal heights represent different dollar values. Either share the Y-axis domain or use an index/percentage-change normalisation so the comparison is valid. Check: in any side-by-side chart layout, are Y-axis domains identical or is the data normalised for comparison?
- Severity: **P1** for price/score comparison charts with independent scales.

**Comparison requiring memory across page scroll**
- Company A's details on one screen, Company B's details requiring scroll. The reader must memorise Company A's numbers to compare. Tufte: bring the data into a single view. For stock comparison, the numbers being compared should be visible simultaneously — either in a shared table, side-by-side cards above the fold, or an overlay chart. Check: can the reader see both comparison targets without scrolling?
- Severity: **P2** for key comparison metrics that require scroll. **P3** for supplementary details below the fold.

**Overlay charts without clear visual distinction**
- Two lines on the same chart with similar colours or no labels. The reader cannot tell which line belongs to which entity. Check: do overlaid series use visually distinct strokes (different colour + different dash pattern for safety) and direct labels (Tufte: label the line at its endpoint, not in a separate legend)?
- Severity: **P2** for overlaid series that are hard to distinguish.

**No normalisation option for different-magnitude comparisons**
- Comparing AAPL ($170) and MSFT ($420) on the same absolute-price chart — MSFT's line dominates visually, but the comparison the reader wants is relative performance. Check: is there a percentage-change or indexed view (base = 100 at start date) for price comparisons? Without this, the chart answers "which stock is more expensive" (not useful) rather than "which performed better" (the actual question).
- Severity: **P2** for comparison views with no normalisation option.

---

## Principle 10: Design time series to reveal temporal patterns, not obscure them

*Tufte: "The time series is the most common form of graphic design." — VDQI, p. 28.*
*Tufte: "Graphics reveal data. Indeed graphics can be more precise and revealing than conventional statistical computations."*

Score histories and price charts are the backbone of a financial data product. The design must reveal the patterns the reader cares about: trends, volatility, regime changes, and the current position relative to history. Common failures: axis choices that hide the meaningful range, time ranges that are too short or too long for the pattern, missing reference lines for benchmarks, and line styles that hide the data density.

### What to check

**Default time range that hides the meaningful pattern**
- A score chart that defaults to showing all available data (2+ years) when the meaningful pattern is the last 90 days. Or a chart that defaults to 7 days when the meaningful pattern is seasonal over 12 months. Check: is the default time range appropriate for the data's typical decision-making context? Are range selectors available (1W, 1M, 3M, 6M, 1Y, ALL)?
- Severity: **P2** for default ranges that hide the actionable pattern. **P3** for missing range selectors when a reasonable default is chosen.

**No current-value annotation**
- A price chart where the reader must trace the rightmost point horizontally to the Y-axis to determine the current value. Tufte would directly annotate the data: place the current value as a label at the right edge of the line. This eliminates the most common lookup task. Check: is the current/latest value directly labelled on the chart, or must the reader decode it from axis position?
- Severity: **P2**

**Time axis labels that don't match the displayed range**
- A 1-month chart with daily data showing year labels on the X-axis. Or a 5-year chart showing every month label, creating an unreadable bar of overlapping text. Time axis labels should adapt to the displayed range: hours for intraday, days for weekly, months for yearly, years for multi-year. Check: does `<XAxis tickFormatter>` adapt to the selected time range? Do labels overlap or get cut off?
- Severity: **P2** for overlapping or inappropriate time labels. **P3** for slightly suboptimal tick frequency.

**Missing reference lines for context**
- A score trend chart with no indication of where "good" or "bad" scores lie. A price chart with no indication of market benchmarks. Tufte: show the context. A horizontal reference line at score 5.0 (neutral), or at the sector average, or at a 52-week high/low, gives the reader immediate context for interpreting the current value. Check: do time-series charts include at least one contextual reference (average, benchmark, threshold)?
- Severity: **P2** for primary score/price charts with no reference context. **P3** for secondary charts.

---

## Principle 11: Maximise information density — a dense display is not a cluttered display

*Tufte: "The quantity of detail is an issue completely separate from the difficulty of reading. Clutter and confusion are failures of design, not attributes of information." — Envisioning Information, p. 51.*
*Tufte: "High-density designs also allow viewers to select, to narrate, to recast and personalise data for their own uses."*

Financial professionals expect density. A Bloomberg terminal packs hundreds of data points into a single screen, and its users find it perfectly readable. The goal is not minimalism — it is clarity at high density. A dashboard with generous whitespace showing four metrics per screen wastes the reader's scroll budget and forces memorisation across views. The target: maximum data per viewport, with the structure (alignment, grouping, typography, colour) doing the work of making it scannable.

### What to check

**One metric per card when many could be grouped**
- A grid of cards where each card displays a single number: "P/E Ratio: 24.5", "Market Cap: $14.3B", "Revenue: $2.85B". This uses the space of an entire card (padding, border, title) for a single value. Tufte would group related metrics into a compact display — a single card or table row containing all key metrics for one entity. Check: is the information density appropriate? Could 4 cards be replaced by 1 compact display?
- Severity: **P2** for primary analytical views with low density. **P3** for overview/summary views where simplicity is intentional.

**Charts that could be tables**
- Tufte: "Tables usually outperform graphics in reporting on small data sets of 20 numbers or less." A bar chart showing 5 models' scores for a single company (5 bars) contains less information than a table row and takes 10x the vertical space. Check: when there are fewer than 10-15 data points, would a table be more space-efficient and more precise? Charts add value for trend, distribution, and relationship patterns — not for a handful of exact values.
- Severity: **P2** for charts displaying fewer than 6 data points where a table would suffice. **P3** for 6-15 data points where either format is reasonable.

**Excessive padding and whitespace in data areas**
- Card padding of 24-32px, large gaps between data sections, generous margins around charts. In a marketing page, whitespace creates elegance. In an analytical tool, it wastes the reader's most precious resource: simultaneous visibility of related data. Check: is the vertical space budget spent on data or on breathing room? Can the reader see all the data they need for a comparison without scrolling?
- Severity: **P2** for data views that require scrolling when the data could fit above the fold with tighter spacing. **P3** for mild padding excess.

---

## Principle 12: Design axes to support the data, not to dominate it

*Tufte: "Axis lines and grid lines do the work of reference, but they can also easily dominate the data. Make them light, thin, and recessive."*

Axes are scaffolding. They help the reader decode values from position, then they should recede. Heavy axis lines, thick ticks, bold axis labels, and axis titles that repeat information already in the page title — all compete with the data. The ideal axis is barely noticed: light-weight ticks at meaningful intervals, labels that don't overlap, and a range that fits the data without wasting space.

### What to check

**Axis tick values that require mental arithmetic**
- Y-axis ticks at 0, 2.5, 5.0, 7.5, 10.0 for scores where the meaningful thresholds are 0, 2, 4, 6, 8, 10. Or price axis ticks at $147.50, $148.75, $150.00, $151.25 when round numbers ($148, $149, $150, $151) would be more scannable. Tufte: ticks should fall on round numbers that the reader can process without arithmetic. Check: does `<YAxis ticks>` use explicit round values appropriate to the data?
- Severity: **P3** for suboptimal tick intervals. **P2** if tick values obscure meaningful thresholds (e.g., no tick at score 5.0, the neutral point).

**Rotated or overlapping axis labels**
- Recharts X-axis labels rotated 45 or 90 degrees because there are too many ticks. Rotated text is dramatically harder to read. Fix by reducing tick count (show every 2nd or 3rd label), abbreviating (Jan, Feb, Mar instead of January, February, March), or adjusting chart width. Check: are any axis labels rotated? Are labels overlapping?
- Severity: **P2** for rotated labels. **P2** for overlapping labels. **P3** for slightly cramped but readable labels.

**Axis range that wastes space**
- A score chart with Y-axis from 0 to 10 when all values fall between 6.5 and 8.0. While starting at 0 is honest (Principle 3), if the chart's purpose is to show relative changes between scores in the 6-8 range, the reader needs resolution in that range. Solution: use a truncated axis with a clear break indicator, or show two views (full-scale context + zoomed detail). Check: is the axis range appropriate for the chart's purpose?
- Severity: **P2** for charts where meaningful variation is invisible due to axis range.

**Redundant axis titles**
- A Y-axis labelled "Score" on a chart titled "Company Score History". The title already communicates the metric. Remove the axis title and let the tick values speak for themselves. Check: do axis titles repeat information available in the chart title, section heading, or page context?
- Severity: **P3**

---

## Principle 13: Use typography to serve data readability — tabular figures, right-alignment, appropriate weight

*Tufte: "In tables and displays of data, the type should be designed to present the information clearly and unambiguously."*

Financial data typography is not body text typography. Numbers must align vertically for comparison. Decimal points must line up. Data should dominate labels. Monospace or tabular-figure fonts prevent column misalignment caused by proportional digit widths. These are not stylistic preferences — they determine whether the reader can accurately compare values by scanning vertically.

### What to check

**Proportional digits in numeric columns**
- Standard proportional fonts render "1" narrower than "8", causing numeric columns to misalign. The fix: `font-variant-numeric: tabular-nums` (forces equal-width digits in OpenType-capable fonts like Inter) or an explicitly monospace font (JetBrains Mono). Check: do numeric columns use tabular figures? Quickly verified by checking if "$111.11" and "$888.88" would occupy the same width.
- Severity: **P2** for comparison tables. **P3** for standalone number displays.

**Decimal alignment in columns**
- A column of values: 7.2, 8.15, 6.0, 9.85. With right-alignment and tabular figures, the decimal points align vertically. Without: the reader must read each number individually rather than scanning the column. Check: are decimal-containing numeric columns right-aligned? Are decimal places consistent within a column (all 1 decimal, or all 2 decimals)?
- Severity: **P2** for score or financial columns with inconsistent decimal places or non-right-alignment.

**Labels styled heavier than data values**
- A display where "P/E Ratio:" is bold and "24.5" is regular weight. The reader came for the number, not the label. The number should be the most visually prominent element. Check: in key-value displays and table cells, is the data value styled with at least equal visual weight to its label?
- Severity: **P2** for labels that visually outweigh data in primary analytical views.

**Number signs and symbols stealing attention**
- Every number prefixed with "$", suffixed with "%", wrapped in parentheses for negatives, plus colour, plus bold — the visual noise per data point explodes. Simplify: move currency and unit indicators to headers/labels, use a minus sign (not parentheses) for negatives when the audience is not accounting-specific, use colour sparingly. Check: is each individual number clean and scannable, or loaded with symbolic decoration?
- Severity: **P3** for mild symbol clutter. **P2** if symbols make column scanning noticeably harder.

---

## Principle 14: Provide context — the data is meaningless without a frame of reference

*Tufte: "To be truthful and revealing, data graphics must bear on the question of compared to what?" — Visual Explanations.*
*Tufte: "Graphical displays should show the data in their context."*

A score of 7.2 for Company X is meaningless without context. Is 7.2 good? What's the sector average? What was it last month? What's the range across all scored companies? Context turns a number into information. Tufte's principle of "adjacent in space rather than stacked in time" means: show the context alongside the data, not behind a click or hover.

### What to check

**Scores displayed without benchmarks**
- A company score of 7.2 with no indication of sector average, percentile ranking, or historical average. The reader cannot evaluate the score without switching to another view. Check: does the score display include at least one benchmark — sector average, universe median, or the company's own historical average?
- Severity: **P2** for primary score displays. **P3** for secondary displays where context is available nearby.

**Price data without market/sector context**
- A stock price chart showing the company in isolation. A 15% gain looks impressive until the sector gained 20%. Check: is there an option to overlay or compare against a benchmark (sector ETF, S&P 500, peers)?
- Severity: **P2** for company price charts with no benchmark option.

**Change values without base period**
- "+12.3%" displayed without indicating the base period. Is this today's change? This week? This year? Check: is every percentage change labelled with its time period? Is the base period consistent within a view?
- Severity: **P1** for unlabelled percentage changes on financial data. The reader may make decisions based on misinterpreting the time frame.

**Range indicators missing**
- A score of 7.2 displayed without indicating the possible range (0-10). New users don't know the scale. Check: is the scoring range indicated at least once on any view showing scores? This can be as minimal as axis labels showing 0-10, or a "/ 10" suffix on the score.
- Severity: **P3** for users who already know the system. **P2** for new-user-facing views.

---

## Principle 15: Annotate directly — integrate text with data, not in separate legends

*Tufte: "The principle of data/text integration is: 'data graphics are paragraphs about data and should be treated as such.'" — VDQI, p. 180.*
*Tufte: "A most unconventional design strategy is revealed: words, numbers and drawing are woven together."*

The most common separation in Recharts-based dashboards: the chart is here, the legend is below, the explanation is in a tooltip, and the insight is in a paragraph above. The reader must assemble meaning from four separate locations. Tufte's alternative: annotate the data directly. Label the line at its endpoint. Place the key insight as a text annotation at the relevant point on the chart. Merge the legend into the data display.

### What to check

**Separate legends for direct-labellable charts**
- A chart with 2-3 lines and a separate legend below. Each line should be labelled at its right endpoint with the series name and current value. This eliminates the legend entirely and reduces the reader's cross-referencing work to zero. Check: for charts with 4 or fewer series, can the legend be replaced with direct line labels?
- Severity: **P3** for standard legend use. **P2** for legends placed far from the chart data (e.g., separate card).

**Tooltips as the only way to read exact values**
- A chart where the reader must hover over each data point to see its value. This makes the chart useless on mobile, in screenshots, and in print. Key values (current, max, min, start) should be directly annotated. Tooltips are appropriate for detail-on-demand on non-key data points. Check: are the most important data points (current value, extremes, notable changes) readable without interaction?
- Severity: **P2** for primary charts where key values require hover. **P3** for supplementary charts.

**Insights separated from the data they describe**
- A text paragraph stating "Company X's score declined sharply in February" positioned above or below a chart where the reader must locate February independently. Tufte: place the annotation at the relevant point on the chart. A small label or marker at the February dip is more effective than a paragraph. Check: when insights or annotations exist, are they spatially adjacent to the data they reference?
- Severity: **P3**

---

## Principle 16: Show causality and mechanism, not just correlation

*Tufte: "The task of the designer is to give visual access to the subtle and the difficult — that is, the revelation of the complex." — Visual Explanations.*
*Tufte devotes chapters to showing causal mechanisms: the Challenger launch decision, cholera maps, evidence of causation.*

A composite score (0-10) derived from multiple models hides its mechanism. The reader sees a number but not why. A score decomposition — showing which model components contributed to the final score and in what proportion — reveals the mechanism. This is especially important when scores change: did the score drop because of one model's re-evaluation, or a broad decline across all models?

### What to check

**Composite scores without decomposition**
- A composite score displayed as a single number with no breakdown by contributing model. The reader must navigate to a separate view to understand why. Check: is there an inline or expandable decomposition showing individual model scores alongside the composite?
- Severity: **P2** for primary score displays. **P3** for list/table contexts where space is constrained (but the detail view must have decomposition).

**Score changes without attribution**
- "Score changed from 7.2 to 6.8" with no indication of which model(s) drove the change. A waterfall or delta breakdown (Model A: -0.2, Model B: -0.1, Model C: -0.1) tells the reader exactly what happened. Check: when score changes are displayed, is the change attributed to contributing factors?
- Severity: **P2** for significant score changes. **P3** for minor changes.

**Correlations shown without caveats**
- A scatter plot or overlay suggesting a relationship between two variables (e.g., score vs price performance) with no indication of sample size, time period, or statistical significance. Tufte: showing possible causal mechanisms requires honest representation of uncertainty. Check: do correlation displays include sample size, time period, and appropriate caveats?
- Severity: **P2** for correlation displays that invite causal interpretation without caveats.

---

## Principle 17: Respect the reader's intelligence — don't dumb down, edit down

*Tufte: "If the statistics are boring, then you've got the wrong numbers." Readers of financial data are capable of processing dense, complex displays — the designer's job is to structure the information, not simplify it away.*
*Tufte: "Clear and precise seeing becomes as one with clear and precise thinking."*

Oversimplified financial data is more dangerous than complex financial data. A traffic-light colour (green/yellow/red) for a composite score reduces a rich signal to three buckets and invites binary decision-making. The solution is not simplification but editing: show the complete picture with careful structure, hierarchy, and progressive disclosure.

### What to check

**Traffic-light reductions of continuous data**
- A score mapped to three colours (green = good, yellow = caution, red = bad) with no visible underlying number. The thresholds are arbitrary, and the reader loses all granularity. A score of 6.9 (yellow) and 7.1 (green) are treated as categorically different when they're essentially the same. Check: are continuous scores shown as continuous values with colour as a secondary encoding, or reduced to categorical buckets?
- Severity: **P2** for traffic-light-only displays with no underlying number. **P3** if colour supplements a visible numeric score.

**Hiding data behind progressive disclosure that should be visible**
- Model scores collapsed behind an "expand" button when they're the core analytical content. Tooltips hiding essential context. Accordions collapsing data that the reader needs for comparison. Tufte: show the data. Progressive disclosure is appropriate for supplementary detail, not for the primary data the reader came to see. Check: is any primary analytical data hidden behind a click/expand/hover?
- Severity: **P2** for primary data hidden behind interaction. **P3** for supplementary data that could be shown inline.

**Percentage-only changes without absolute values**
- "+15.3%" without the absolute change ($12.50) or without the base value ($81.70). Percentages mislead when the base is unknown — a 50% gain on a $2 stock is $1, while a 2% gain on a $200 stock is $4. Check: for financial changes, are both percentage and absolute values available?
- Severity: **P2** for financial values where the base matters. **P3** for scores where the scale is fixed and known.

---

## Principle 18: Design for the smallest effective difference

*Tufte: "Make all visual distinctions as subtle as possible, but still clear and effective." — Envisioning Information, p. 73.*

Visual encoding should use the minimum contrast necessary to communicate the distinction. Heavy borders when a hairline will do. Saturated colours when a tint will do. Large size differences when a 2px difference will do. This principle prevents the "shouting" effect where every element demands equal attention, creating visual chaos. In a data-dense financial dashboard, restraint in visual encoding allows the actual data variation to be the most prominent visual signal.

### What to check

**Gridlines and reference lines competing with data lines**
- Chart gridlines at the same stroke weight or opacity as the data line. The data should be the heaviest element. Gridlines should be barely perceptible — `strokeOpacity: 0.15` or lighter. Check: compare the visual weight of gridlines to data lines. The data line should be 2-3x heavier visually.
- Severity: **P2** for gridlines competing with data. **P3** for slightly heavy but clearly secondary gridlines.

**Over-saturated colours for secondary distinctions**
- Fully saturated colours used for sector categorisation or model identification where muted tones would suffice. Reserve full saturation for alerts and critical states (large score drops, significant events). Normal categorical distinctions should use desaturated colours. Check: are colours saturated proportionally to their importance?
- Severity: **P3**

**Borders, shadows, and dividers used where spacing suffices**
- Tufte's erasing principle: "Erase non-data-ink, within reason." If two chart sections can be separated by 16px of whitespace, a border adds nothing. Check: can any visible border or divider be replaced by spacing alone?
- Severity: **P3**

---

## Principle 19: Avoid moiré and vibration effects in patterns and colours

*Tufte: "Moiré vibration is an undisciplined ambiguity, with the eyes most attracted to the moire patterns rather than the data." — VDQI, p. 111.*

Dense hatching patterns, fine striping in bar charts, and closely spaced repeated elements create optical vibration that physically interferes with reading. In digital interfaces this manifests as: dense dashed lines at small scales, tight alternating-row colours in large tables, overlapping gridlines and data marks, and thin stripes in stacked bar/area charts.

### What to check

**Dense dashed or dotted lines at small scale**
- A sparkline or small chart using `strokeDasharray="2 2"` at 16px height creates a vibrating dotted line that's harder to read than a solid line. Dashes work at chart scale; they fail at sparkline scale. Check: are dashed/dotted lines used at sizes where the dash pattern creates visual noise rather than serving as a line-style distinction?
- Severity: **P3**

**Thin slices in stacked charts**
- A stacked area or bar chart where some segments are 1-2 pixels tall, creating a dense stripe pattern. Below a readable threshold, segments should be grouped into "Other". Check: are all visible segments in stacked charts thick enough to read (minimum ~4px)?
- Severity: **P2** for unreadable thin segments. **P3** for slightly thin but identifiable segments.

---

## Principle 20: Show multivariate data — escape flatland

*Tufte: "The world is complex, dynamic, multidimensional; the paper is static, flat. How are we to represent the rich visual world of experience and measurement on mere flatland?" — Envisioning Information, p. 9.*

Financial data is inherently multivariate: a company has a price, multiple scores, a sector, a market cap, momentum, volatility — all simultaneously relevant. Displaying these one at a time (one chart per metric, one page per dimension) forces the reader to hold dimensions in memory. Effective design encodes multiple variables simultaneously through position, size, colour, and small multiples.

### What to check

**One variable per chart when multiple could be integrated**
- A price chart on one tab, a score chart on another tab, volume on a third. The reader cannot see how score changes relate to price moves. Check: can related time-series be vertically stacked with shared X-axes (synchronized brushing) so temporal relationships are visible? A price chart above a score chart with aligned time axes is far more revealing than two separate pages.
- Severity: **P2** for core related metrics on separate views. **P3** for supplementary metrics.

**Table columns that should be a scatter plot**
- A table with columns for "Score" and "Price Change %" that the reader must mentally correlate. A scatter plot of score vs. price change instantly reveals the relationship (or lack thereof). Check: when two numeric columns invite comparison, is a graphical view available that reveals their relationship?
- Severity: **P3** — tables are fine for exact values, but the option for a graphical view should exist for pattern discovery.

**No ability to sort/filter by multiple dimensions**
- A company table sortable by one column at a time but unable to show "high score AND high momentum AND small cap" simultaneously. Multi-dimensional filtering is a form of showing multivariate data. Check: can the reader filter or sort by combinations of variables to surface multivariate patterns?
- Severity: **P2** for a screening product where multi-dimensional filtering is the core use case.

---

## Principle 21: Design for the specific genre — financial charts have established conventions

*Tufte: "The best designs are intriguing and curiosity-provoking, drawing the viewer into the wonder of the data."*
*However, Tufte also respects convention: "If design is contrary to the readers' expectations, the designs may seem wrong regardless of their virtues."*

Financial data has well-established visual conventions: candlestick charts for price action, green/red for gain/loss (with accessibility alternatives), volume bars below price, tables sorted by market cap. Violating these conventions forces the reader to re-learn a visual language they already know. Innovation should be in data density and clarity, not in reinventing the candlestick.

### What to check

**Price charts missing standard financial affordances**
- A stock price chart with no volume subplot, no time-range selector, no crosshair for precise value reading. Financial users expect these. Check: do price charts include the standard complement of volume bars, range selectors (1D, 1W, 1M, 3M, 1Y, ALL), and a crosshair or reference line on hover?
- Severity: **P2** for missing range selectors or volume. **P3** for missing crosshair.

**Non-standard colour conventions without clear explanation**
- Using blue for gains and orange for losses instead of the conventional green/red (or their accessible equivalents). If departing from convention, the meaning must be immediately explained — not discoverable through trial and error. Check: if non-standard colour associations are used, are they labelled in the immediate context?
- Severity: **P3** for unconventional but labelled colours. **P2** for unconventional and unlabelled.

**Unfamiliar chart types for standard data**
- A radar chart for comparing model scores when a simple bar chart or table would be more precise and more familiar. Radar charts distort area perception and require the reader to learn an unusual geometry. Tufte's preference is clear: use the simplest chart type that reveals the data. Check: is each chart type the simplest one that serves the comparison? Would a bar chart or table be more readable?
- Severity: **P2** for exotic chart types (radar, sunburst, treemap) used where bar charts or tables would be clearer.

---

## Principle 22: Ensure graphical integrity across responsive breakpoints

*Tufte: "What is to be sought in designs for the display of information is the clear portrayal of complexity. Not the complication of the simple."*

A chart designed for a 1440px viewport may become misleading at 768px. Axes compress, labels overlap, small multiples collapse to a single column (breaking simultaneous comparison), and sparklines lose resolution. Graphical integrity is not just about the initial design — it must hold across all viewports where the product is used.

### What to check

**Charts that break at mobile/tablet widths**
- Overlapping axis labels, truncated legends, unreadable sparklines, or compressed charts where data points merge at narrow viewports. Check: do charts render correctly at 768px and 375px widths? Do Recharts `<ResponsiveContainer>` dimensions produce a readable result at all breakpoints?
- Severity: **P2** for data-critical charts that break at tablet width. **P3** for mobile (where financial dashboards are typically supplementary).

**Small multiples that lose their comparison value**
- A 3-column grid of comparison charts that collapses to a single column on mobile. The reader can now only see one chart at a time — the simultaneous comparison that makes small multiples effective is lost. Check: at narrower viewports, do small multiples maintain at least 2-across layout? If not, is an alternative comparison view provided (overlay or table)?
- Severity: **P2** for comparison grids that become single-column with no alternative.

**Number formatting that overflows cells on narrow screens**
- Long formatted numbers ("$14,293,847,291") that overflow table cells or wrap to multiple lines at narrow widths, destroying table alignment. Check: is number formatting (Principle 6) responsive — using more aggressive abbreviation ($14.3B) at narrow widths?
- Severity: **P2** for overflowing numeric cells in data tables.

---

## Principle 23: Design for print and export — the data should survive separation from the interface

*Tufte: "Good design is clear thinking made visible." A well-designed data display works in any medium — screen, print, screenshot, PDF.*

Financial data displays are frequently shared: screenshots in reports, exported PDFs, pasted into presentations, shared in messaging. A chart that relies on hover tooltips for key values, interactive legends for series identification, or animation for data revelation fails completely when separated from its interactive context. The static form of every data display must be self-contained and readable.

### What to check

**Charts that lose essential information when static**
- A chart where the current value is only visible via tooltip, series are only identifiable via hover-highlighting, or data ranges are only visible via interactive brushing. Check: would a screenshot of this chart provide enough information for the reader to understand the data without any interaction? If not, key values must be directly annotated.
- Severity: **P2** for primary charts that are unreadable as screenshots. **P3** for supplementary charts.

**Dark-theme charts without print/export consideration**
- A dark-background chart that, when screenshotted and pasted into a white-background document, looks like a black rectangle with barely visible data. Check: is there a light-theme export option, or do chart colours have sufficient contrast to be readable when composited on any background?
- Severity: **P3** — a nice-to-have for products that expect data export.

---

## Principle 24: Show data provenance and freshness — the reader must know what they are looking at

*Tufte: "Graphical excellence begins with telling the truth about the data."*

Financial data has a temporal dimension that non-financial data often lacks: a stock price from 15 minutes ago may be meaningfully different from the current price. A score computed yesterday from last week's financial filings reflects a different reality than a score computed today. The reader must always know: when was this data last updated, what is the source, and what time period does it cover?

### What to check

**No data freshness indicator**
- Scores and prices displayed with no indication of when they were last computed or fetched. The reader may be acting on stale data. Check: is there a "Last updated: X minutes ago" or "As of [date]" indicator on data displays? Is it positioned near the data, not hidden in a footer?
- Severity: **P1** for real-time or frequently-updated financial data with no freshness indicator. **P2** for daily-computed scores.

**Time period ambiguity**
- A "Performance" percentage without specifying the period. A "Score History" chart with no visible date range. Check: is the time period of every displayed metric unambiguous? Can the reader determine the exact period without interacting with the interface?
- Severity: **P1** for financial performance metrics with no time period. **P2** for scores where the computation period is unclear.

**Mixed data freshness in a single view**
- A view showing real-time price data alongside weekly-computed scores without indicating the different update cadences. The reader may assume all data is equally fresh. Check: when a single view combines data of different freshness, is the difference indicated?
- Severity: **P2**

---

## Principle 25: Every chart must answer: "What am I looking at, and what should I notice?"

*Tufte: "What are the content-reasoning tasks that this display is supposed to help with?"*

The final principle synthesises all others. Every chart, table, and data display must pass a 3-second test: a new reader should be able to determine (1) what data is shown, (2) what the current state is, and (3) what, if anything, is notable. If after 3 seconds the reader is still orienting — trying to decode the legend, figure out the axis units, or determine what the colours mean — the display has failed regardless of its data-ink ratio, truthfulness, or density.

### What to check

**Chart with no title or contextual heading**
- A chart rendered with no title, relying on the page context to explain it. When the chart appears in a grid, is shared as a screenshot, or is viewed by a user who scrolled past the heading, it becomes opaque. Check: does every chart have a concise, descriptive title that identifies the metric, entity, and time period? (Not "Chart 1" but "AAPL Composite Score — 12 Months".)
- Severity: **P2** for untitled charts. **P3** for charts with vague titles.

**Notable patterns that are not called out**
- A score that dropped 30% in one month with no visual annotation. A price chart that crossed a 52-week high with no marker. The data tells the story, but the display should help the reader find the story. Check: are significant events or outliers visually marked (reference line, annotation, highlighted point)?
- Severity: **P3** — this is editorial judgment, but a stock screening product that fails to surface notable patterns misses a key value proposition.

**Visual hierarchy that doesn't match information priority**
- A chart where the gridlines are heavier than the data line, where the legend is larger than the chart, where the title dominates the data, or where decorative elements draw the eye before the numbers do. Check: does the visual weight hierarchy match the information priority? The order should be: data > current values > reference lines > axes > labels > gridlines > chrome.
- Severity: **P2** for inverted hierarchy (chrome heavier than data). **P3** for imperfect but functional hierarchy.
