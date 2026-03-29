# SEO & LLM Search Optimization Reference — Carmack × Fishkin

Philosophy: John Carmack. Specifics: Rand Fishkin (Moz, SparkToro, "Lost and Founder", zero-click content, SparkToro blog).
Stack context: Next.js App Router / React / TypeScript / FastAPI / SQLite. Finance SaaS (AlphaStocks). 694 indexable pages (503 programmatic stock pages + ~190 editorial/tool pages). YMYL content. Solo founder. $0–79/month marketing budget.

Every finding must describe the **concrete discovery cost of getting it wrong** — not just "this isn't SEO best practice."
Fishkin's core thesis: organic discovery is fragmenting across Google, LLMs (ChatGPT, Perplexity, Claude), social, and niche communities. Optimising for one channel while ignoring others is a strategic error.

---

## Principle 1: Own a topic, not a ranking — build topical authority clusters

*Carmack: reduce a complex domain to its essential structure.*
*Fishkin: "Google doesn't rank pages. It ranks websites that prove they deserve to own a topic." And: "You don't need to be a big brand — you need to be THE source on a specific thing."*

Topical authority means Google (and LLMs) recognise your site as a comprehensive, trustworthy source on a defined subject. A site with 30 deep pages on one topic outranks a site with 300 shallow pages on 30 topics. For a finance SaaS: the topic is "stock analysis for retail investors" — every page should reinforce this positioning.

### What to check

**Missing pillar-cluster structure**
- Is there a clear hierarchy? Pillar pages (broad topic hubs like "/stocks", "/screener", "/learn/fundamental-analysis") linking down to cluster pages (individual stock pages, specific metric explainers, how-to guides).
- Each cluster page must link back to its pillar. Each pillar must link to all its cluster pages. This creates a crawlable, semantically connected structure that signals topical depth.
- For 503 stock pages: they are the cluster. The pillar is the screener/sector overview page. Without this linkage, Google sees 503 orphaned pages, not a comprehensive stock analysis resource.
- Severity: **P1** — without clusters, programmatic pages have no topical context and will be filtered as thin content.

**Topic gaps that break authority**
- Map the full topic space: if you cover stock screeners but not "how to read a balance sheet" or "P/E ratio explained", you have gaps that competitors fill. LLMs especially prefer sources that cover a topic end-to-end.
- Fishkin: "The sites that win topical authority are the ones where searchers never need to go elsewhere." Check: could a user learn everything they need about stock analysis without leaving the site?
- Severity: **P2** — gaps don't penalise directly, but they cap your authority ceiling.

**No semantic consistency across pages**
- Are stock pages using consistent terminology, consistent data labels, consistent section headings? Inconsistency signals auto-generated content without editorial oversight. Google's Helpful Content system specifically targets this.
- Severity: **P2**

---

## Principle 2: EEAT is survival for YMYL — demonstrate real expertise in finance

*Carmack: credibility comes from demonstrated competence, not claimed competence.*
*Fishkin: "For YMYL topics, Google's quality raters are literally checking: who wrote this, why should I trust them, and can this advice hurt someone?"*

EEAT — Experience, Expertise, Authoritativeness, Trustworthiness — is Google's quality framework. For finance content (Your Money or Your Life), EEAT requirements are maximally strict. A stock analysis page without visible author credentials, data sources, and disclaimers will be suppressed regardless of technical SEO quality.

### What to check

**Missing author/founder attribution**
- Every editorial page and data methodology page must have a visible author with credentials. For a solo founder: create an `/about` page with real name, finance background, methodology philosophy. Link to it from every content page.
- Stock pages can attribute to "AlphaStocks Research" with a link to methodology — but pure data pages without ANY attribution signal are EEAT-negative.
- Fishkin: "Anonymous finance content is dead. Google killed it in the August 2023 core update."
- Severity: **P1** for editorial/analysis content without author attribution. **P2** for pure data pages.

**No data source citations**
- Every financial metric displayed must cite its source (SEC filings, exchange data feeds, calculated from X). This isn't just EEAT — it's legal compliance for financial data.
- Add a "Data Sources" section to stock pages or a global `/data-sources` page linked from every stock page footer.
- Severity: **P1** — finance regulators and Google quality raters both require source transparency.

**Missing disclaimers**
- Finance SaaS requires: "Not financial advice" disclaimer visible on every page with financial data or analysis. A `/disclaimer` page linked from the global footer.
- Risk disclaimers on any page that could influence investment decisions.
- Severity: **P1** — legal exposure and EEAT signal simultaneously.

**No "Experience" signal**
- The "E" in EEAT (Experience) means: has the author actually done the thing? For a stock analysis tool: show that the founder uses the tool, share methodology evolution, publish case studies of the analysis in action.
- A `/methodology` or `/how-it-works` page that explains the analytical approach signals both expertise and experience.
- Severity: **P2**

---

## Principle 3: Optimise for LLM citation — get mentioned by ChatGPT, Perplexity, Claude

*Carmack: adapt to the platform's actual constraints, not your assumptions about them.*
*Fishkin: "The next generation of organic traffic isn't from Google — it's from AI answers that cite your content. And the rules are completely different."*

LLM search (ChatGPT Browse, Perplexity, Claude web search) is a new discovery channel. LLMs prefer: clear definitions, structured data, unique datasets, and pages that directly answer questions without requiring interpretation. They cite sources that are authoritative AND machine-parseable.

### What to check

**Content not structured for extraction**
- LLMs extract answers from pages. Content must have clear question-answer patterns: H2 as a question, first paragraph as a direct answer, then supporting detail. If your pages are walls of text, LLMs will skip them for competitors with cleaner structure.
- Stock pages should have extractable facts: "Apple (AAPL) has a P/E ratio of X as of [date]" — not buried in a data table without textual context.
- Severity: **P2**

**No unique data that LLMs can't get elsewhere**
- Fishkin: "LLMs cite you when you have something they can't synthesise from other sources." For AlphaStocks: proprietary scores, custom rankings, composite metrics, screening results — these are unique. If your stock pages only show data available on Yahoo Finance, there's no reason for an LLM to cite you.
- Ensure every stock page prominently features your proprietary analysis (scores, signals, composite metrics) with clear labelling.
- Severity: **P1** — without unique data, LLM citation probability approaches zero.

**Missing definition pages**
- LLMs heavily cite definitional content. Create `/learn/[metric-name]` pages for every financial metric you use (P/E ratio, debt-to-equity, free cash flow yield, etc.). These pages become citation targets when users ask LLMs "what is [metric]?"
- Each definition page should: define the term in the first sentence, explain why it matters, show how AlphaStocks uses it, link to stock pages that demonstrate it.
- Severity: **P2**

**No structured FAQ content**
- Add FAQ sections to key pages (screener, pricing, methodology). Use `FAQPage` schema markup. LLMs and Google's featured snippets both preferentially extract FAQ content.
- Severity: **P3**

---

## Principle 4: Implement llms.txt — make your product machine-readable

*Carmack: make the system's interface explicit and unambiguous.*
*Fishkin: "If you're not telling AI crawlers what your product does in their language, you're invisible to the fastest-growing discovery channel."*

`llms.txt` is an emerging standard (proposed by Jeremy Howard / Answer.AI) — a machine-readable file at your domain root that describes your product, its capabilities, and its content structure for LLM crawlers. Think of it as `robots.txt` for AI comprehension.

### What to check

**No /llms.txt file**
- Create `/public/llms.txt` with: product name, one-line description, what the product does, key features, content categories, data freshness, and links to key pages.
- Format: Markdown-like, concise, factual. Not marketing copy — LLMs ignore superlatives.
- Example structure:
  ```
  # AlphaStocks
  > Stock analysis platform for retail investors with proprietary scoring

  ## Features
  - Stock screener with 50+ fundamental filters
  - Proprietary composite scores for 503 stocks
  - Financial metric definitions and education

  ## Content
  - /stocks/[ticker] — individual stock analysis pages (503 pages)
  - /screener — filterable stock screener
  - /learn/[topic] — financial education articles
  ```
- Severity: **P2** — early adopter advantage while the standard gains traction.

**No /llms-full.txt for detailed context**
- The extended version: a more detailed document that LLMs can use to deeply understand your product. Include data methodology, metric definitions, update frequency.
- Severity: **P3**

---

## Principle 5: Create zero-click value — win attention even without the visit

*Carmack: deliver the payload immediately, don't make users work for it.*
*Fishkin coined "zero-click content": "The best content marketing doesn't need the click. It delivers value in the SERP, in the social feed, in the AI answer — and THAT builds the brand that eventually converts."*

Over 65% of Google searches end without a click (Fishkin's SparkToro/Datos research). Fighting this is futile. Instead: make your brand visible in the zero-click result itself. When someone sees "according to AlphaStocks, AAPL's composite score is 87/100" in a featured snippet or AI answer, that's brand building even without a site visit.

### What to check

**Meta descriptions that don't deliver value**
- Meta descriptions should contain a usable fact, not just a teaser. Bad: "Learn about Apple stock analysis." Good: "AAPL composite score: 87/100. P/E 28.4 vs sector avg 22.1. Free cash flow yield 3.8%. Updated daily."
- For stock pages: programmatically generate meta descriptions with the most compelling data point. The user gets value from the SERP itself, and the brand name is attached.
- Severity: **P2**

**No social-optimised previews (Open Graph)**
- When stock pages are shared on X/LinkedIn/Discord, does the preview card show useful data? Generate OG images dynamically with the stock ticker, key metric, and your brand.
- In Next.js: use `generateMetadata()` in stock page layouts to create dynamic OG tags. Use `@vercel/og` or `satori` for dynamic OG image generation.
- Severity: **P3**

**Content that only works on-site**
- If a data visualisation, ranking, or insight only makes sense on your site, it can't spread. Create shareable, embeddable formats: summary cards, comparison tables, single-metric callouts that work in screenshots and social shares.
- Severity: **P3**

---

## Principle 6: Scale with programmatic SEO — treat 503 stock pages as a system

*Carmack: when you see 503 instances of the same pattern, build a system, not 503 individual solutions.*
*Fishkin: "Programmatic SEO is powerful but dangerous. Google will index 10,000 pages if each one earns its existence. It will deindex 10,000 pages if they're thin templates with swapped variables."*

503 stock pages are a programmatic SEO asset — but only if each page provides genuine, unique value that justifies its existence. Google's Helpful Content system specifically targets template pages where the template adds no insight beyond the raw data.

### What to check

**Thin template pages**
- Does each stock page have content beyond data fields? At minimum: a textual summary sentence ("AAPL trades at a P/E of 28.4, above the Technology sector median of 22.1, suggesting..."), comparative context (vs sector, vs historical), and actionable framing (what the data means for the screener criteria).
- Fishkin: "Every programmatic page must pass the test: would a knowledgeable human have written this page, or is it obviously a template with swapped values?"
- Severity: **P1** — thin programmatic pages risk site-wide quality demotion under Helpful Content.

**No unique value per page**
- Each stock page needs at least one element that's unique to that stock AND unique to your site. Your proprietary scores are this element. If they're not prominently displayed with explanation, the page has no reason to rank above Yahoo Finance or Finviz.
- Severity: **P1**

**Missing index management**
- Not all 503 pages may deserve indexing on day one. If some stock pages are too thin (penny stocks with minimal data, recently added tickers with incomplete analysis), `noindex` them until they have sufficient content. A smaller, high-quality index beats a large, thin one.
- Use Next.js `generateMetadata()` to conditionally set `robots: { index: false }` for pages below a quality threshold.
- Severity: **P2**

**No XML sitemap segmentation**
- 503 stock pages + editorial pages + tool pages should be in separate sitemap segments (`sitemap-stocks.xml`, `sitemap-learn.xml`, `sitemap-tools.xml`). This lets you monitor crawl rates per content type in Google Search Console.
- In Next.js: use `app/sitemap.ts` or multiple sitemap files with a sitemap index.
- Severity: **P2**

---

## Principle 7: Build internal linking architecture that distributes authority

*Carmack: make the dependency graph explicit and intentional.*
*Fishkin: "Internal links are the one ranking lever you have 100% control over. Most sites waste it by linking randomly or not at all."*

Internal links distribute PageRank (link equity) and establish topical relationships. For a site with 694 pages, the linking architecture IS the architecture. A stock page that links to its sector page, related metrics definitions, and the screener with that stock's filters creates a web of semantic meaning. A stock page that only links to the homepage wastes its authority.

### What to check

**Orphaned pages**
- Run a crawl (Screaming Frog, or simply trace from the sitemap): can every indexable page be reached within 3 clicks from the homepage? Pages with no internal links pointing to them are effectively invisible to both Google and users.
- Common orphans in programmatic sites: stock pages not linked from any sector page or screener result.
- Severity: **P1** for pages in the sitemap but unreachable via internal links.

**No contextual cross-linking**
- Stock pages should link to: their sector/industry page, definition pages for displayed metrics, similar stocks (by score, by sector), the screener pre-filtered to that stock's characteristics.
- Editorial pages should link to: relevant stock pages as examples, related educational content, the screener.
- These links must be contextual (in-content), not just in a sidebar or footer. Contextual links carry more weight.
- Severity: **P2**

**Flat link hierarchy**
- If every page links to every other page, no page has priority. Structure links to flow authority: Homepage → Pillar pages → Cluster pages. The screener page and sector overview pages should receive the most internal links because they're the pillar pages.
- Severity: **P2**

**Broken or redirect-chain links**
- Internal links pointing to 404s or through redirect chains waste crawl budget and dilute authority. In Next.js: if you renamed a stock page URL, update all internal links — don't rely on redirects.
- Severity: **P2** for redirect chains, **P1** for internal 404s.

---

## Principle 8: Implement finance-specific structured data (Schema.org)

*Carmack: tell the machine exactly what the data is — don't make it guess.*
*Fishkin: "Structured data is table stakes for rich results. In finance, it's also how LLMs understand your numbers aren't random text."*

Schema.org markup helps Google generate rich results (star ratings, FAQs, breadcrumbs) and helps LLMs understand data semantics. For finance content, structured data disambiguates "P/E: 28.4" from random text into a typed financial metric.

### What to check

**Missing base schemas on every page**
- `WebSite` schema on the homepage with `SearchAction` (enables sitelinks searchbox).
- `Organization` schema with name, logo, URL, social profiles.
- `BreadcrumbList` schema on every page — critical for both rich results and LLM navigation.
- In Next.js: add JSON-LD via `<script type="application/ld+json">` in layout or page components.
- Severity: **P2**

**No structured data on stock pages**
- Use `FinancialProduct` or `Dataset` schema (or the more specific `schema.org/MonetaryAmount` for financial values).
- At minimum per stock page: `name`, `ticker`, `exchange`, `description`, `url`, and key metrics as structured properties.
- Add `dateModified` to signal freshness — critical for finance where stale data is dangerous.
- Severity: **P2**

**Missing FAQ schema on educational pages**
- `/learn/*` pages with Q&A content should use `FAQPage` schema. This enables FAQ rich results in Google and makes the content directly extractable by LLMs.
- Severity: **P3**

**No Article schema on editorial content**
- Blog posts, guides, and analysis pages should use `Article` or `NewsArticle` schema with `author`, `datePublished`, `dateModified`. This directly feeds EEAT signals.
- Severity: **P2**

---

## Principle 9: Eliminate content cannibalization — one intent, one page

*Carmack: a system should have exactly one canonical path for each operation.*
*Fishkin: "The fastest way to kill your own rankings is to have three pages competing for the same query. Google picks one — and it's usually the wrong one."*

Content cannibalization happens when multiple pages target the same search intent, forcing Google to choose between them. For a finance SaaS: if `/stocks/AAPL` and `/blog/apple-stock-analysis` both target "Apple stock analysis", they compete with each other instead of competitors.

### What to check

**Stock page vs editorial page overlap**
- Is there a clear content boundary? Stock pages = current data + proprietary scores. Editorial pages = analysis, opinion, methodology, how-to. If a blog post duplicates what the stock page shows, consolidate or differentiate.
- The test: search `site:alphastocks.app [ticker]` — if multiple pages appear for the same ticker-related query, you have cannibalization.
- Severity: **P2**

**Screener vs category pages overlap**
- If `/screener?sector=Technology` and `/sectors/technology` exist, which one ranks for "technology stocks"? Define one as canonical, use `rel=canonical` or merge them.
- Severity: **P2**

**Multiple pages targeting the same metric definition**
- If "P/E ratio" is explained on `/learn/pe-ratio`, in a blog post, and in a tooltip on stock pages — the blog post and tooltip should NOT be indexable standalone pages competing with the canonical definition page.
- Severity: **P2**

**No canonical tags on filtered/sorted views**
- Screener with URL parameters (`/screener?sort=pe&sector=tech`) creates infinite URLs. Use `rel=canonical` pointing to the base screener URL, or `noindex` filtered views.
- In Next.js: set canonical in `generateMetadata()` for pages with searchParams.
- Severity: **P1** — uncanonicalized parameter URLs can bloat the index with duplicate content.

---

## Principle 10: Earn backlinks at $0 — data assets, partnerships, directories

*Carmack: solve real problems and people come to you.*
*Fishkin: "At SparkToro we've never done outreach for links. We publish original research and data that people can't get elsewhere, and the links come because the data is useful."*

Backlinks remain a top ranking factor. At $0 budget, the only viable strategy is creating link-worthy assets — not outreach, not guest posting at scale, not buying links. For a finance SaaS with proprietary data: publish the data in formats that others want to cite.

### What to check

**No linkable data assets**
- Create shareable, citable content: annual/quarterly "state of" reports using your data ("Top 50 Undervalued Stocks by AlphaStocks Composite Score — Q1 2026"), sector comparison studies, methodology white papers.
- These should be standalone pages with clear citations (not gated PDFs) so bloggers, journalists, and other finance sites can link directly.
- Fishkin: "Original data is the only sustainable link magnet for bootstrapped companies."
- Severity: **P2**

**Not listed in finance directories and aggregators**
- Submit to: Product Hunt, AlternativeTo, G2/Capterra (free tiers), finance tool directories, IndieHackers, relevant subreddit sidebars.
- Each listing is a backlink + a discovery channel. At $0, this is the highest-ROI activity.
- Severity: **P3**

**No data partnerships**
- Offer free data embeds or API access to finance bloggers/newsletter authors in exchange for attribution links. "Powered by AlphaStocks" with a dofollow link on a finance blog is worth more than 100 directory listings.
- Severity: **P3**

**Missing citations in competitor comparison content**
- When finance publications or bloggers compare stock screeners, is AlphaStocks in the comparison? Monitor competitor mentions (set up Google Alerts for "best stock screener", "Finviz alternative", etc.) and reach out when you're not included.
- Severity: **P3**

---

## Principle 11: Write titles and meta descriptions that earn the click

*Carmack: the first thing the user sees must communicate the essential information.*
*Fishkin: "Your title tag is an ad, not a description. It competes against 9 other results for attention. If it doesn't earn the click, nothing else matters."*

In SERPs, you control two things: the title tag and the meta description. CTR is a ranking signal (debated, but Fishkin is in the "yes" camp). More importantly: a better CTR at position 5 can outperform a worse CTR at position 3 in absolute traffic.

### What to check

**Generic, keyword-stuffed titles**
- Bad: "AAPL Stock Analysis | Apple Stock Price | AlphaStocks". Good: "AAPL: Score 87/100 — Overvalued vs Sector | AlphaStocks".
- The title should contain: the primary keyword, a unique data point (your proprietary score), and the brand name. In that order.
- For programmatic stock pages: generate titles with `generateMetadata()` using the stock's current score/signal. A dynamic title with a number outperforms a static template.
- Severity: **P2**

**Meta descriptions that are empty or auto-generated**
- Next.js will not auto-generate meta descriptions. If `generateMetadata()` doesn't return a `description`, Google will pull random page text — often badly.
- Every page type needs a meta description template. Stock pages: "[TICKER] scores [X]/100 on AlphaStocks composite analysis. P/E [X], FCF yield [X]%. Updated [date]."
- Severity: **P2**

**Title truncation**
- Google displays ~55-60 characters of the title tag. Test that programmatically generated titles don't truncate the important part. Ticker + score should be in the first 40 characters.
- Severity: **P3**

---

## Principle 12: Page experience signals — Core Web Vitals, mobile, HTTPS

*Carmack: "Performance is not optional. Users leave before they consciously decide to leave."*
*Fishkin: "Core Web Vitals are a tiebreaker, not a ranking factor. But in competitive finance SERPs, every tiebreaker matters."*

Google's page experience signals: Core Web Vitals (LCP, INP, CLS), mobile-friendliness, HTTPS, no intrusive interstitials. These are baseline requirements, not differentiators — but failing them is an active penalty.

### What to check

**Largest Contentful Paint (LCP) > 2.5s**
- For stock pages: is the primary data table/score above the fold and rendered server-side? Client-fetched data that pops in after 3 seconds fails LCP.
- In Next.js: use Server Components for initial data render. Use `loading.tsx` for Suspense boundaries that show skeleton UI immediately.
- Severity: **P2**

**Cumulative Layout Shift (CLS) > 0.1**
- Dynamic financial data loading into a page causes layout shifts. Reserve explicit dimensions for data containers, charts, and ad slots.
- Common culprit: stock charts that load asynchronously and push content down.
- Severity: **P2**

**Interaction to Next Paint (INP) > 200ms**
- Screener filter interactions must respond instantly. Heavy client-side filtering on 503 stocks can block the main thread. Use `startTransition` for non-urgent filter updates, or move filtering to the server.
- Severity: **P2** for the screener page specifically.

**Not mobile-responsive**
- Finance data tables are notoriously bad on mobile. Test every page type on a 375px viewport. Stock pages should adapt: stack metrics vertically, collapse secondary data, maintain tap targets ≥ 48px.
- Severity: **P1** if stock pages are unusable on mobile (Google uses mobile-first indexing).

**Missing HTTPS or mixed content**
- All resources (API calls, images, scripts) must load over HTTPS. Any `http://` reference is mixed content.
- Severity: **P1** for missing HTTPS, **P2** for mixed content.

---

## Principle 13: Financial content freshness — update frequency as a ranking signal

*Carmack: stale data is wrong data.*
*Fishkin: "For finance content, freshness isn't optional — it's a core quality signal. Google timestamps financial results and prefers recently updated pages."*

Google's QDF (Query Deserves Freshness) model weighs heavily for finance queries. A stock page last updated 6 months ago will be suppressed in favour of one updated yesterday. `dateModified` in structured data and visible "Last updated" timestamps are critical.

### What to check

**No visible "Last Updated" timestamp**
- Every stock page must show when its data was last refreshed. Format: "Data as of March 29, 2026" — human-readable AND in `dateModified` structured data.
- The timestamp must reflect actual data freshness, not page deployment date. If you deploy code daily but data updates weekly, the timestamp should reflect data updates.
- Severity: **P1** for stock pages without visible freshness indicators.

**Stale sitemap lastmod dates**
- `<lastmod>` in the sitemap should reflect actual content changes, not build timestamps. If all 503 pages show the same `lastmod`, Google ignores the field entirely.
- Severity: **P2**

**No content refresh cadence**
- Define update frequency per content type: stock data pages (daily or real-time), editorial analysis (monthly review), educational content (quarterly review), methodology (on change).
- Pages that haven't been updated in 6+ months need either a refresh or a visible "This content was reviewed on [date]" note.
- Severity: **P2**

---

## Principle 14: Infiltrate competitor citations — appear where they appear

*Carmack: study the terrain before you build.*
*Fishkin: "Find where your competitors are mentioned and you aren't. That's your backlink gap, your content gap, and your brand awareness gap all in one."*

Competitor citation analysis reveals where the audience already looks for products like yours. If Finviz is mentioned in "best stock screeners" listicles and you're not, that's a specific, actionable gap.

### What to check

**No competitor citation audit**
- Search for: "best stock screener", "Finviz alternative", "free stock analysis tool", "[competitor] review". Document every page where competitors appear. These are your target placement opportunities.
- Fishkin's SparkToro approach: use SparkToro to find what podcasts, publications, and social accounts your target audience follows. Those are your outreach targets.
- Severity: **P3** — strategic, not technical.

**No comparison pages**
- Create `/compare/alphastocks-vs-finviz`, `/compare/alphastocks-vs-yahoo-finance`. These rank for high-intent "[tool] vs [tool]" and "[tool] alternative" queries.
- Be honest in comparisons — acknowledge competitor strengths. Fishkin: "Comparison pages that are obviously biased get penalised by users and algorithms alike."
- Severity: **P2** — comparison queries are high-intent, low-competition for smaller brands.

**Not present in "alternative to" databases**
- AlternativeTo.net, G2, Capterra — these rank for "[competitor] alternative" queries. Create profiles with accurate descriptions and screenshots.
- Severity: **P3**

---

## Principle 15: Optimise for Google Discover — finance mobile traffic

*Carmack: go where the users are.*
*Fishkin: "Google Discover is the biggest traffic source nobody optimises for. It's interest-based, not query-based — and finance is one of its strongest verticals."*

Google Discover surfaces content to mobile users based on their interests, not searches. Finance is a top Discover category. Requirements: high-quality images (≥1200px wide), compelling titles, recent content, and the user must have shown interest in the topic.

### What to check

**No high-quality feature images**
- Discover requires images ≥1200px wide. Stock pages and editorial content should have OG images that meet this threshold.
- Use `max-image-preview:large` in the robots meta tag to allow Google to show large image previews.
- In Next.js: add to `generateMetadata()`: `robots: { 'max-image-preview': 'large' }`.
- Severity: **P2** for editorial content. **P3** for programmatic pages (Discover favours editorial).

**No editorial content with Discover appeal**
- Discover favours: market commentary, "stocks to watch" lists, sector analysis, methodology deep-dives. Pure data pages rarely surface.
- Publish 2-4 editorial pieces per month with timely angles ("3 Undervalued Tech Stocks Based on AlphaStocks Composite Score — March 2026").
- Severity: **P2** — Discover can be a significant traffic source for finance editorial.

**Missing Web Stories**
- Google Discover prominently features Web Stories (AMP-based visual stories). Finance summaries, market highlights, or stock spotlights in Web Story format can capture Discover real estate.
- Severity: **P3** — optional channel, but low effort with high Discover visibility.

---

## Principle 16: Robots.txt and crawl budget — guide crawlers efficiently

*Carmack: don't waste cycles on work that doesn't contribute to the output.*
*Fishkin: "With 694 pages, crawl budget isn't your problem — but crawl efficiency is. Don't let Google waste time on your admin pages when it could be crawling stock pages."*

Crawl budget matters less for sub-1000-page sites, but crawl efficiency always matters. Every URL Google crawls that shouldn't be indexed is a wasted opportunity for a URL that should be.

### What to check

**Overly permissive robots.txt**
- Block: `/api/*`, `/admin/*`, `/_next/static/*` (served via CDN, doesn't need crawling), any internal tool routes.
- Allow: all public content routes, sitemaps.
- Severity: **P3**

**Search/filter URLs crawlable**
- If the screener generates URLs like `/screener?sort=pe&dir=asc&sector=tech`, these should be blocked via robots.txt or marked `noindex`. They create near-infinite URL space from Google's perspective.
- Severity: **P2** for parameter-heavy pages without noindex/disallow.

**No robots meta per page type**
- Use `generateMetadata()` to set granular robots directives: stock pages (`index, follow`), screener with params (`noindex, follow`), utility pages (`noindex, nofollow`).
- Severity: **P2**

---

## Principle 17: URL structure — clean, hierarchical, permanent

*Carmack: naming is design.*
*Fishkin: "URLs are permanent. Change them and you lose whatever equity they've built. Get the structure right the first time."*

### What to check

**Non-descriptive URLs**
- Good: `/stocks/AAPL`, `/learn/pe-ratio`, `/sectors/technology`. Bad: `/stock?id=123`, `/page/article-42`.
- Every URL should be human-readable and describe its content without needing to visit the page.
- Severity: **P2**

**No URL consistency across page types**
- Define patterns and enforce them: `/stocks/[TICKER]`, `/learn/[slug]`, `/sectors/[slug]`, `/compare/[slug]`. Inconsistency signals a site without editorial control.
- Severity: **P3**

**Changing URLs without redirects**
- If a URL has ever been live, changing it requires a 301 redirect from old to new. In Next.js: configure in `next.config.js` `redirects()` or middleware.
- Severity: **P1** for live pages that changed URL without redirect (immediate ranking loss).

---

## Principle 18: Local and vertical search — finance-specific channels

*Carmack: be where the audience looks, not where you wish they looked.*
*Fishkin: "Google is just one channel. The sites that win long-term diversify across Google, LLM search, finance communities, newsletters, and social."*

### What to check

**No presence on financial data aggregators**
- Submit to: Crunchbase (as a company), AngelList, IndieHackers, relevant finance subreddits (as a community member, not a spammer).
- Create a free API or embeddable widget — every embed is a backlink.
- Severity: **P3**

**No Google Search Console setup**
- GSC is free and essential. Verify the domain, submit sitemaps, monitor: index coverage, Core Web Vitals, search performance by page type.
- If not set up, you're flying blind on every SEO metric.
- Severity: **P1**

**Not registered with Bing Webmaster Tools and Yandex**
- Bing feeds ChatGPT's search. Bing Webmaster Tools lets you submit sitemaps and monitor how Microsoft's ecosystem sees your site. At $0 budget, this is free traffic.
- Severity: **P2** — Bing → ChatGPT citation pipeline is real and growing.

---

## Principle 19: Content velocity vs content quality — the solo founder tradeoff

*Carmack: do fewer things, but do them completely.*
*Fishkin: "One exceptional piece of content per month beats four mediocre ones. Google's Helpful Content system was built to enforce this."*

As a solo founder with a $0-79/month budget, you cannot compete on volume. You compete on: data uniqueness (programmatic pages with proprietary analysis), depth (comprehensive coverage of your niche), and consistency (regular updates to existing content over new content creation).

### What to check

**Publishing cadence unsustainable for one person**
- Is there a content calendar demanding daily or weekly posts? That leads to quality decay. For a solo founder: 2-4 editorial pieces/month + continuous data freshness on programmatic pages.
- Severity: **P2**

**New content prioritised over updating existing content**
- Fishkin: "Updating your top 20 pages is almost always higher ROI than creating page 21." Check: are high-traffic pages being refreshed with current data, or are they stale while new content is created?
- Use GSC to identify top pages by impressions and prioritise refreshing those.
- Severity: **P2**

**No content repurposing**
- One data analysis can become: a stock page update, a `/learn/` article, a social post, an email newsletter section, an LLM-citable data point. If each piece of content exists in only one format, you're leaving 80% of its distribution value on the table.
- Severity: **P3**

---

## Principle 20: Technical SEO hygiene — the baseline that makes everything else work

*Carmack: fix the foundation before decorating the building.*
*Fishkin: "Technical SEO isn't sexy, but a single misconfigured robots tag can undo months of content work."*

### What to check

**No self-referencing canonical tags**
- Every page must have a `<link rel="canonical" href="...">` pointing to itself (or to the canonical version if duplicates exist). In Next.js: set via `generateMetadata()` `alternates.canonical`.
- Severity: **P2**

**Missing or incorrect hreflang (if multi-language)**
- If the site will serve multiple languages, implement `hreflang` from the start. Retrofitting is painful.
- Severity: **P3** (unless already multi-language, then **P2**).

**No 404 page with recovery paths**
- A custom 404 page should: show helpful navigation (search, popular stocks, screener link), return a proper 404 status code (not a soft 404 that returns 200), and be styled consistently with the site.
- Severity: **P2**

**JavaScript rendering issues**
- Next.js Server Components render server-side by default — good. But verify: are critical content and metadata present in the initial HTML response? Use "View Page Source" (not DevTools) or Google's URL Inspection tool.
- Client-only rendered content (behind `use client` with no SSR) may not be indexed.
- Severity: **P1** if primary content (stock data, scores) is client-rendered only.

---

## Principle 21: Monitoring and iteration — measure what matters

*Carmack: if you can't measure it, you can't improve it.*
*Fishkin: "Most SEO failures aren't strategy failures — they're measurement failures. People optimise the wrong metric or don't measure at all."*

### What to check

**Not tracking the right metrics**
- Track per page type: impressions (awareness), clicks (traffic), CTR (title/description effectiveness), average position (ranking), and — critically — branded search volume over time (brand awareness).
- Fishkin: "Branded search is the ultimate measure of content marketing success. If more people search for your brand name, your content strategy is working."
- Severity: **P2**

**No segmented GSC analysis**
- Don't look at site-wide averages. Segment by: stock pages, editorial, learn pages, tool pages. Each content type has different performance benchmarks and different optimisation levers.
- Severity: **P3**

**No LLM citation monitoring**
- Periodically ask ChatGPT, Perplexity, and Claude: "What are the best stock screener tools?" "What is [metric]?" "Analyse [ticker]." Track whether AlphaStocks appears in responses. This is the emerging metric that no one measures yet.
- Severity: **P2** — early measurement creates early advantage.

---

## Principle 22: Avoid SEO anti-patterns that trigger manual penalties

*Carmack: don't be clever — be correct.*
*Fishkin: "The sites that get penalised aren't the ones doing nothing. They're the ones doing something Google explicitly told them not to."*

### What to check

**Hidden text or cloaking**
- Content visible to crawlers but hidden from users (CSS `display: none` on SEO text, white text on white background, off-screen positioning). This is a manual penalty trigger.
- Severity: **P1**

**Keyword stuffing**
- Repeating target keywords unnaturally in content, meta tags, alt text, or URL slugs. For programmatic pages: check that template text doesn't repeat the ticker name excessively.
- Severity: **P2**

**Purchased or manipulative links**
- At $0 budget this is unlikely, but: sponsored content without `rel="sponsored"`, link exchanges, PBN links — all trigger manual actions.
- Severity: **P1** if present.

**Doorway pages**
- Multiple pages targeting slight variations of the same keyword (e.g., "/best-stock-screener", "/top-stock-screener", "/stock-screener-tool") that all lead to the same product. Google's doorway pages algorithm specifically targets this.
- Severity: **P1**

---

## Principle 23: AI-generated content policy — stay on the right side of Google's guidelines

*Carmack: use tools to augment, not to replace, understanding.*
*Fishkin: "Google doesn't penalise AI content. It penalises unhelpful content. If your AI-generated stock summaries are more useful than a human-written wall of text, they'll rank. If they're generic slop, they won't."*

### What to check

**AI content without human editorial review**
- Programmatic content (stock summaries, metric explanations) generated by LLMs should be reviewed for accuracy, especially in YMYL finance context. A single factual error in financial data erodes EEAT for the entire site.
- Severity: **P1** for unreviewed AI finance content.

**No disclosure of methodology**
- If AI assists in generating analysis or summaries, disclose the methodology: "Analysis generated using AlphaStocks proprietary models" is sufficient. Full transparency about what's automated and what's editorial.
- Severity: **P2**

**Generic AI content with no proprietary angle**
- If an LLM could generate the same stock summary without access to your data, the content has no moat. Every AI-assisted page must incorporate your proprietary data (scores, signals, rankings) that the LLM couldn't produce independently.
- Severity: **P2**

---

## Gaps: What This Doc Doesn't Cover

- **Paid search/SEM**: This doc covers organic discovery only. At $0-79/month, paid search is not viable for competitive finance keywords.
- **Social media strategy depth**: Fishkin covers zero-click content for social, but platform-specific algorithms (X, LinkedIn, Reddit) require dedicated strategy.
- **International SEO**: hreflang, multi-language content, country-specific compliance. Relevant if/when AlphaStocks expands beyond English.
- **Video SEO**: YouTube optimisation for finance content. Fishkin acknowledges video's importance but his expertise is primarily text/web.
- **E-commerce SEO patterns**: Pricing page optimisation, conversion funnels. Partially relevant for the SaaS pricing page but not AlphaStocks' primary SEO challenge.

---

## Quick Reference: Severity Guide

| Severity | Pattern | Examples |
|----------|---------|----------|
| **P1 — Fix Now** | Discovery-blocking issues or YMYL compliance failures | Thin programmatic pages, missing EEAT signals on finance content, no financial disclaimers, client-rendered primary content, internal 404s, mobile-unusable pages, uncanonicalized parameter URLs, no GSC setup, hidden text/cloaking |
| **P2 — Fix Soon** | Missed ranking opportunities or suboptimal signals | No LLM-optimised content structure, missing structured data, no competitor comparison pages, generic titles/meta, stale content timestamps, missing sitemap segmentation, no Bing/Yandex registration, no linkable data assets |
| **P3 — Consider** | Strategic improvements that compound over time | llms-full.txt, OG image generation, finance directory listings, Web Stories, content repurposing, hreflang preparation, density of internal cross-links |

### The Overriding Filter

Before writing any finding, apply the Carmack-Fishkin synthesis:

1. **Does this page earn its existence?** If it's a template with swapped variables and no unique value, flag it. (Carmack: eliminate unnecessary complexity. Fishkin: every page must justify its index slot.)
2. **Is the expertise visible?** If there's financial data without attribution, methodology, or disclaimers, flag it. (Both: credibility is demonstrated, not claimed.)
3. **Can a machine extract the value?** If the content is useful but not structured for Google/LLM extraction, flag it. (Carmack: make it explicit. Fishkin: if the crawler can't parse it, it doesn't exist.)
4. **Would this earn a citation?** If the content has no unique data, no original analysis, nothing a competitor doesn't also offer, flag it. (Fishkin: LLMs and humans cite sources that add something new.)
5. **Is the freshness signal accurate?** If financial data shows no update timestamp, or the timestamp is misleading, flag it. (Both: stale data in finance is worse than no data.)
