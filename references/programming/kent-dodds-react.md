# React & Next.js Frontend Quality Reference — Kent C. Dodds

Philosophy: Kent C. Dodds. Creator of Testing Library, author of Epic React, authority on React component patterns, testing strategy, and frontend architecture. Key doctrines: AHA Programming, the Testing Trophy, "derive don't sync", colocation, composition over configuration.
Stack context: Next.js 16 (App Router) + React 19 + TypeScript 5 + TanStack React Query 5 + Tailwind CSS 4 + Recharts 3 + Framer Motion + shadcn/ui + Lucide icons. Custom AuthProvider (no NextAuth). Google OAuth. Direct fetch to FastAPI backend — NO tRPC. NO CSS Modules — Tailwind only. Bilingual English/Russian via custom i18n service.

Every finding must describe the **concrete failure mode** — not just "bad practice."

---

## Principle 1: React Query is your server state manager — never duplicate it into useState

*Dodds: "I would take the data I got from the server and treat it like it was UI state. I would mix it in with the actual UI state and this resulted in making both more complex."*

With 455 useQuery calls, React Query IS the state management layer for server data. Any `useState` holding API response data is a synchronisation bug waiting to happen: the query cache updates, the local state doesn't, the UI shows stale data. This is especially dangerous in a financial product where prices, scores, and alerts change continuously.

### What to check

**API data copied into useState**
- `const [data, setData] = useState(null)` populated from a `useQuery` result via `useEffect`. The cache already holds this data. The `useState` copy will go stale on background refetch, window focus refetch, and cache invalidation — silently showing old prices or scores.
- The fix: consume `useQuery` return directly. If transformation is needed, use the `select` option: `useQuery({ queryKey: [...], queryFn: ..., select: (data) => transform(data) })`.
- Severity: **P1** for financial data (prices, scores, alerts). **P2** for non-critical display data.

**Missing or inconsistent query keys**
- Query keys are React Query's cache identity. If two components fetch the same endpoint with different key structures, they get separate cache entries and separate network requests. With 455 useQuery calls, key consistency is critical.
- Pattern: create a `queryKeys` factory object (`queryKeys.stock.detail(ticker)`, `queryKeys.watchlist.all()`). Every useQuery call references the factory, never inline strings.
- The failure mode: stale data after mutation because `invalidateQueries` targets a key that doesn't match the actual key used by the query.
- Severity: **P2**

**staleTime left at default (0) for stable data**
- React Query's default `staleTime` is 0 — every query is immediately stale and refetches on mount. For data that changes infrequently (company profiles, sector classifications, model descriptions), this causes unnecessary network requests on every page navigation.
- Financial data that updates frequently (prices, alerts) should have short staleTime (30s-60s). Reference data should have long staleTime (5min-30min). Static data (i18n strings, enums) should use `staleTime: Infinity`.
- The cost of wrong defaults: 455 queries all refetching on every mount = hundreds of unnecessary requests per navigation, slow page transitions, wasted bandwidth.
- Severity: **P2**

**Optimistic updates missing on user-initiated mutations**
- Watchlist add/remove, alert create/delete — these are user actions that should feel instant. Without optimistic updates, the UI waits for the round-trip to FastAPI and back before showing the change. The user clicks "Add to Watchlist," nothing happens for 200-500ms, then the button changes.
- Pattern: `useMutation` with `onMutate` (optimistic cache update), `onError` (rollback), `onSettled` (invalidate for consistency).
- Severity: **P2** for interactive features (watchlist, alerts). **P3** for less frequent mutations.

---

## Principle 2: Derive state — don't sync it

*Dodds: "Don't Sync State. Derive It." Any piece of stored state that could be computed from other state is a synchronisation bug waiting to happen.*

The cost isn't theoretical — financial dashboards that show a filtered stock list, a count of filtered results, and a "no results" message from three separate state variables will inevitably show "0 results" while the list still displays stocks from the previous filter.

### What to check

**Derivable values stored in useState**
- `useState` for filtered/sorted/paginated data that could be computed from the source data + current filters during render. `const filtered = useMemo(() => stocks.filter(matchesFilters), [stocks, filters])` has zero sync risk.
- URL search params mirrored into `useState` — the URL IS the source of truth for filter state in a screener app. Derive from `useSearchParams()`, never copy.
- Severity: **P1** when derivable state controls data display (wrong stocks shown, wrong counts). **P2** for UI-only state.

**useEffect used solely to synchronise two state variables**
- `useEffect(() => { setB(computeFromA(a)) }, [a])` — this is the "state sync" anti-pattern. Compute `b` during render instead. The useEffect version causes an extra render cycle AND can show the old `b` value for one frame.
- Severity: **P2**

**State escalation order violated**
- Check that state follows this progression: (1) derive during render, (2) `useState` for independent local state, (3) lift to closest common parent, (4) component composition via `children` to skip intermediate layers, (5) `useReducer` for interdependent state, (6) React Context scoped to subtree, (7) React Query for server state.
- Severity: **P3** for minor violations. **P2** when the wrong level caused prop drilling that triggered a global store.

---

## Principle 3: Server Components are the default — 'use client' is a conscious boundary

*Dodds: "The mental model is that Server Components are the default. You opt into client-side interactivity explicitly."*

In Next.js App Router, every component is a Server Component unless marked `'use client'`. The boundary should be drawn as deep in the tree as possible. A page that is entirely `'use client'` ships its entire component tree as JavaScript to the browser — defeating the purpose of App Router.

### What to check

**'use client' placed too high in the tree**
- A page-level component marked `'use client'` when only a child (e.g., a chart, a filter form, a watchlist button) needs interactivity. Push `'use client'` down to the leaf that actually uses state, effects, or browser APIs.
- The cost: the entire subtree below the boundary becomes client JavaScript. For a stock detail page, the company profile, description, and financial tables could be Server Components — only the interactive chart and watchlist button need the client.
- Severity: **P2** for pages where significant static content is unnecessarily client-rendered. **P3** for small components.

**Data fetching in Client Components that could be Server Components**
- `useQuery` calls for data that is known at request time and doesn't need client-side reactivity (e.g., initial stock data on a detail page). This data could be fetched in a Server Component and passed as props, with React Query used only for client-side mutations and polling.
- The tradeoff: with 455 useQuery calls, migrating all fetching to Server Components is unrealistic. Focus on page-level initial data — fetch in the Server Component, pass to client children, let React Query handle subsequent updates.
- Severity: **P3** — this is an architectural improvement, not a correctness bug.

**Non-serialisable props crossing the boundary**
- Functions, class instances, Dates, Maps, Sets passed from Server Components to Client Components. This is a hard RSC error. Only JSON-serialisable data can cross the boundary.
- Severity: **P1** if it causes runtime errors.

---

## Principle 4: Every component must handle loading, error, and empty states

*Dodds: "By using a status variable rather than a simple isLoading indicator we enable our users to know exactly what the state is at any given point in time."*

Financial data components that show stale data during an error, or show nothing during loading, or show a broken chart when the API returns an empty array — these are bugs, not cosmetic issues. A user making investment decisions based on stale or missing data is the worst failure mode for this product.

### What to check

**Components that only handle the happy path**
- A `ScoreCard`, `PriceChart`, or `CompareTable` that renders `data.map(...)` without checking for `isLoading`, `isError`, or empty data. The concrete failure: component crashes with "cannot read property of undefined" when the query is still loading.
- Every component consuming a `useQuery` result must handle all four states: `pending` (skeleton/spinner), `error` (error message with retry), `success` with data (render), `success` with empty data (empty state message).
- Severity: **P1** for components displaying financial data without error handling. **P2** for non-critical UI components.

**Error boundaries missing around data-fetching subtrees**
- A single unhandled error in a `PriceChart` crashes the entire page to a white screen. Error Boundaries catch render-phase errors and display a fallback. Use `react-error-boundary` with `FallbackComponent` that shows what went wrong and offers a retry.
- Minimum: one Error Boundary per page layout. Better: one around each independent data-fetching section (chart, table, sidebar).
- Severity: **P1** for no Error Boundary at all. **P2** for only a single top-level boundary.

**Loading states that block the entire page**
- A full-page spinner while one slow query loads, instead of streaming content as each query resolves. Use `<Suspense>` boundaries around independent sections so fast queries render immediately. The stock screener main page should show the filter panel instantly even if the results table is still loading.
- Severity: **P2**

---

## Principle 5: Composition over configuration — avoid the prop apocalypse

*Dodds: "Take advantage of component composition. One of the things that really aggravates problems with prop drilling is breaking out your render method into multiple components unnecessarily."*

A `<DataTable>` that accepts 20 configuration props (columns, sorting, filtering, pagination, row click handler, empty message, loading state, error state...) is an abstraction that will calcify and resist change. Compound components and children-based composition keep the API flexible.

### What to check

**Components with 10+ props**
- Any component accepting more than 10 props is a signal that composition is needed. Break it into compound components: `<Table>`, `<Table.Header>`, `<Table.Body>`, `<Table.Row>`, `<Table.Empty>`.
- The concrete cost of prop-heavy components: every new feature requires adding another prop, updating the type, threading it through, and praying it doesn't break existing uses.
- Severity: **P3** unless the component is already causing bugs due to prop combinations (**P2**).

**Prop drilling through 3+ intermediate layers**
- A value passed from a page through a layout through a container through a card to a button — where the intermediates don't use the value. Before reaching for Context, check: could the parent render the button directly via `children`?
- Dodds: "One of the things that really aggravates problems with prop drilling is breaking out your render method into multiple components unnecessarily." Sometimes the fix is to inline, not abstract.
- Severity: **P2** when the drilling has already caused intermediates to accept props they don't understand.

**shadcn/ui components over-wrapped**
- Wrapping shadcn/ui's `<Button>`, `<Card>`, `<Dialog>` in project-specific wrappers that just forward props with slightly different defaults. Unless the wrapper adds real logic (auth checks, analytics tracking, i18n), it's an unnecessary indirection layer. Consume shadcn/ui directly.
- Severity: **P3**

---

## Principle 6: TypeScript is armour — wear it or don't bother having it

*Dodds: "Static analysis catches entire categories of bugs and costs nearly nothing." He places static analysis at the base of the Testing Trophy.*

With direct fetch to FastAPI (no tRPC to generate types), the API boundary is the most dangerous type gap. An `any` on an API response propagates through every component that consumes it — the entire type system downstream is theater.

### What to check

**`any` on API response boundaries**
- `fetch('/api/stocks').then(r => r.json())` returns `Promise<any>`. Without explicit typing, every property access downstream is unchecked. Create response types matching the FastAPI schema and validate at the boundary: `const data: StockResponse = await res.json()` — or better, use a runtime validator like Zod: `StockResponseSchema.parse(await res.json())`.
- The failure mode: FastAPI changes a field name from `price` to `current_price`, TypeScript doesn't catch it, the frontend silently shows `undefined` where a price should be.
- Severity: **P1** for API boundaries. This is the single highest-risk type gap in a non-tRPC stack.

**`as` type assertions that suppress real errors**
- `(data as StockData).price` — this tells TypeScript "trust me" and skips validation. If the data doesn't match, the error surfaces at runtime as `undefined` property access, not at compile time.
- Severity: **P2** for data boundaries. **P3** for internal type narrowing where the assertion is provably safe.

**Catch blocks assuming `error instanceof Error`**
- JavaScript can throw anything. `catch (error) { setError(error.message) }` crashes if the thrown value is a string or an object without `.message`. Use a utility: `function getErrorMessage(error: unknown): string`.
- Severity: **P2** for catch blocks in data fetching paths. **P3** for UI-only error handling.

**Generic component props typed as `any`**
- `function ScoreCard({ data }: { data: any })` — every consumer can pass anything, and the component's internal assumptions are unchecked. Use specific types or generics.
- Severity: **P2**

---

## Principle 7: Custom hooks must earn their extraction

*Dodds (AHA Programming): "Prefer duplication over the wrong abstraction." Apply AHA to hooks: don't extract until reuse is real.*

A custom hook extracted for "separation of concerns" with a single call site adds a file, an import, and an indirection — with no actual benefit. The cost compounds when the hook needs to be modified: the developer must find the hook file, understand its context, check all consumers (even if there's only one), and ensure the change doesn't break the interface.

### What to check

**Single-use hooks extracted prematurely**
- `useStockData(ticker)` used in exactly one component. The fetch logic could be inline in the component. Extract only when the same fetching pattern appears in two or more components.
- Exception: hooks that wrap Context consumers (`useAuth()`, `useI18n()`) are always appropriate — they enforce the provider boundary and provide a fail-fast error if used outside the provider.
- Severity: **P3**

**Hooks with too many responsibilities**
- `useScreenerState()` that manages filters, sorting, pagination, and URL sync — a god hook. Break into focused concerns: URL sync is one hook, filter logic is derived state, pagination is a separate concern.
- The failure mode: changing pagination logic accidentally breaks filter reset because they share state inside the hook.
- Severity: **P2** when the hook is actively causing bugs from tangled state.

**Polling hooks without cleanup**
- AlertBell polling every 15 seconds — is the interval cleared on unmount? Is it paused when the tab is hidden? Does it use React Query's `refetchInterval` (which handles these cases) or a manual `setInterval` (which doesn't)?
- React Query pattern: `useQuery({ ..., refetchInterval: 15_000, refetchIntervalInBackground: false })`. This automatically pauses when the tab is hidden and cleans up on unmount.
- Severity: **P2** for manual polling without cleanup (memory leak, wasted requests). **P1** if the leak causes cumulative performance degradation.

---

## Principle 8: Test behaviour, not implementation — start with the highest-risk gaps

*Dodds: "The more your tests resemble the way your software is used, the more confidence they can give you."*
*Dodds: "Write tests. Not too many. Mostly integration."*

The codebase currently has ZERO frontend tests. The priority is not to achieve coverage — it's to protect the highest-risk paths first. Dodds' heuristic: "What part of your untested codebase would be really bad if it broke?"

### What to check

**Missing tests on critical user flows (ZERO tests currently)**
- Priority 1: Auth flow — login, token refresh, redirect after login, logout. Past bug: login loops. This is the first thing to test.
- Priority 2: Watchlist CRUD — add, remove, list. User-visible data mutation that must not lose data.
- Priority 3: Screener filters — filter application, URL sync, reset. The core product feature.
- Priority 4: Data display correctness — does ScoreCard show the right score? Does PriceChart render without crashing on empty data?
- Use Testing Library (not Enzyme, not shallow rendering). Use MSW to mock the FastAPI backend at the network level.
- Severity: **P1** for zero tests on auth flow. **P2** for zero tests on core CRUD and filter flows.

**Implementation detail tests (when tests are added)**
- Tests asserting on internal state, component structure, or hook return values. Dodds: "With shallow rendering, I can refactor my component's implementation and my tests break. With shallow rendering, I can break my application and my tests say everything's still working."
- The litmus test: (a) will this test break when there's a real bug? (b) will this test survive a backward-compatible refactor? Implementation-detail tests fail both.
- Severity: **P2**

**Wrong Testing Library query priority (when tests are added)**
- Prefer `getByRole` > `getByLabelText` > `getByText` > `getByTestId`. `data-testid` is the escape hatch, not the default. Dodds: "If you can't get something with getByRole, there's a good chance your UI is inaccessible."
- Severity: **P3**

---

## Principle 9: The auth flow must be airtight — no silent failures, no infinite loops

*Dodds: "Context without the fail-fast pattern" is a class of bug that's hard to trace. A custom AuthProvider is a Context — apply the same discipline.*

Custom auth (no NextAuth) means the team owns every edge case: token expiry, refresh race conditions, redirect loops, and the gap between "no token" and "token expired." Past bug: login loops — this is a known failure mode that must be structurally prevented.

### What to check

**AuthProvider without fail-fast consumer hook**
- `createContext(undefined)` for auth without a `useAuth()` hook that throws if used outside the provider. Any component that calls `useContext(AuthContext)` directly gets `undefined` when the provider is missing — and silently renders as if the user is logged out.
- The fix: `function useAuth() { const ctx = useContext(AuthContext); if (!ctx) throw new Error('useAuth must be used within AuthProvider'); return ctx; }`
- Severity: **P1**

**Token refresh race condition**
- Multiple components mount simultaneously, each detects an expired token, each triggers a refresh request. The first succeeds, the second fails (refresh token already consumed), the third gets a 401. The user is randomly logged out.
- The fix: a single refresh promise that all concurrent callers await. If a refresh is in progress, subsequent calls should not start a new one.
- Severity: **P1**

**Redirect loop prevention**
- After login, the user is redirected to the original page. If the original page checks auth and redirects to login before the token is set, infinite loop. Past bug in this codebase.
- The fix: a `returnTo` parameter in the login URL, combined with a guard that waits for the auth state to resolve (not just check synchronously) before redirecting.
- Severity: **P1** — known past bug.

**Auth state loading treated as "not authenticated"**
- During the initial auth check (token validation on app load), the auth state is indeterminate. If components treat "loading" as "not logged in," they flash login prompts or redirect before the token is validated. Use a three-state model: `loading | authenticated | unauthenticated`.
- Severity: **P2**

---

## Principle 10: Recharts performance with large datasets requires explicit constraints

*Dodds: "Apply the AHA Programming principle and wait until the abstraction/optimization is screaming at you before applying it."*

Recharts renders SVG DOM nodes — one `<circle>`, `<line>`, or `<rect>` per data point. A PriceChart with 1250+ data points generates 1250+ SVG elements, each tracked by React's reconciler. On mobile, this causes visible jank on mount and interaction.

### What to check

**Raw data passed to Recharts without downsampling**
- 1250 daily price points rendered as 1250 SVG elements. The user's screen is ~375px wide on mobile — you physically cannot distinguish 1250 data points. Downsample to match the pixel width: one data point per 2-3 pixels, max ~200 points for a mobile chart.
- The failure mode: chart mount takes 200-500ms, scroll jank during animation, browser memory pressure from 1000+ SVG nodes.
- Severity: **P2** for desktop (tolerable). **P1** for mobile if causing visible jank.

**Tooltip rendering on every mouse move**
- Recharts `<Tooltip>` re-renders on every `mousemove` event by default. With 1250 data points, each move triggers a nearest-point calculation. Use `throttle` on the tooltip or implement a custom tooltip with debounced positioning.
- Severity: **P2** if causing visible lag on hover.

**Multiple Recharts instances on a single page**
- A comparison page with 10 stock charts, each with 500+ data points. That's 5000+ SVG nodes plus 10 independent Recharts render cycles. Consider: lazy rendering (only render charts in viewport), shared axis computation, or a single chart with overlaid series.
- Severity: **P2** for pages with 5+ charts simultaneously.

**Animation enabled on large datasets**
- `isAnimationActive={true}` (Recharts default) on a chart with 1000+ points. The animation interpolates every point, causing a multi-second paint on mount. Disable animation for datasets above ~200 points: `isAnimationActive={data.length < 200}`.
- Severity: **P2**

---

## Principle 11: Forms should be controlled by the URL — not hidden state

*Dodds: "The URL is the best place to store UI state that should be shareable and bookmarkable."*

A stock screener's filter form is the product. If the user applies 5 filters, shares the URL, and the recipient sees an unfiltered page — the product is broken. URL-driven state also means browser back/forward works as expected, and React Query can use search params as query keys for automatic cache separation.

### What to check

**Filter state in useState instead of URL searchParams**
- Screener filters (sector, market cap range, score threshold, sort order) stored in `useState`. The user applies filters, copies the URL, sends it — the recipient sees the default view. Filter state must live in the URL.
- Pattern: `useSearchParams()` as the source of truth, `router.push` or `router.replace` to update. Derive filter values from params during render.
- Severity: **P1** for core screener filters. **P3** for ephemeral UI state (modal open, tooltip visible).

**Search params not used as React Query keys**
- If the screener fetches `/api/stocks?sector=tech&sort=score`, the query key should include the search params: `['stocks', { sector, sort }]`. This gives automatic cache separation per filter combination and correct invalidation.
- The failure mode without this: changing filters triggers a refetch, but the cache key is just `['stocks']`, so the old data flashes before the new data arrives, and back-navigation refetches instead of using the cache.
- Severity: **P2**

**Form submission without debouncing**
- A search autocomplete that fires a query on every keystroke. With a 200ms round-trip to FastAPI, typing "apple" sends 5 requests, 4 of which are wasted. Debounce with 300ms delay, or use React Query's `keepPreviousData` to show stale results while the new query loads.
- Severity: **P2** for search inputs. **P3** for filter dropdowns (discrete selections, no debounce needed).

---

## Principle 12: Accessibility is not optional — financial data has legal and ethical requirements

*Dodds: "If you can't get something with getByRole, there's a good chance your UI is inaccessible." Testing Library's query priority is an accessibility audit by design.*

Financial data tables without proper headers, charts without text alternatives, and color-only score indicators exclude screen reader users and violate WCAG 2.1. In some jurisdictions, financial tools have legal accessibility requirements.

### What to check

**Data tables without semantic markup**
- Stock screener results rendered as `<div>` grids instead of `<table>` with `<thead>`, `<th scope="col">`, and `<tbody>`. Screen readers cannot navigate a div-grid as a table. The cost: blind users literally cannot use the core product feature.
- Severity: **P1** for the main screener results table. **P2** for secondary data displays.

**Charts without text alternatives**
- A PriceChart or ScoreCard chart is purely visual — screen readers see nothing. Every chart needs: `role="img"` with `aria-label` describing the trend ("AAPL price: $185 to $192 over 30 days, up 3.8%"), or a visually hidden data table equivalent.
- Severity: **P2**

**Color-only indicators**
- Score cards using green/yellow/red to indicate good/neutral/bad without text labels or icons. 8% of men have color vision deficiency. Add text ("Strong", "Neutral", "Weak") or icons alongside color.
- Severity: **P2**

**Interactive elements without accessible names**
- Icon-only buttons (WatchlistButton with just a star icon, AlertBell with just a bell icon) without `aria-label`. Lucide icons render as decorative SVGs — the button has no accessible name.
- Severity: **P2**

**Focus management in modals and drawers**
- shadcn/ui's Dialog and Sheet components handle focus trapping — but custom modal implementations or conditional renders may not. Focus must move into the modal on open and return to the trigger on close.
- Severity: **P2** for modals. **P3** for non-modal overlays.

---

## Principle 13: Lazy load below-the-fold — especially charts and heavy components

*Dodds: "The fastest code is the code that never runs."*

Recharts, Framer Motion, and data-heavy components should not be in the initial bundle for pages where they appear below the fold. A stock detail page loads instantly if the header and key stats render server-side, and the chart lazy-loads as the user scrolls.

### What to check

**Heavy components in the initial bundle**
- Recharts is ~200KB minified. If every page imports it eagerly (even pages without charts), the bundle is bloated. Use `next/dynamic` with `{ ssr: false }` for chart components — they require browser APIs anyway.
- Framer Motion adds ~30KB. If only used on specific pages, dynamic import those pages' animated sections.
- Severity: **P2** for charts imported on pages that don't always show them. **P3** for Framer Motion on specific pages.

**No loading fallback during lazy load**
- `dynamic(() => import('./PriceChart'), { ssr: false })` without a `loading` component. The user sees a layout jump when the chart pops in. Provide a skeleton with the chart's dimensions.
- Severity: **P3**

**Images without next/image**
- Stock logos, profile images, or screenshots loaded with `<img>` instead of `next/image`. Next.js Image handles lazy loading, responsive sizing, and format optimization. Without it: large images block initial paint, no WebP/AVIF conversion, no srcset for different screen sizes.
- Severity: **P2** for above-the-fold images. **P3** for below-the-fold.

---

## Principle 14: React Context is for dependency injection — not global state

*Dodds: "Not all of your context needs to be globally accessible!" Place Context providers as close to their consumers as possible.*

The codebase uses React Context for auth (appropriate — app-wide concern) and likely for i18n (also appropriate). The temptation is to add more app-root Contexts for theme, filters, watchlist state, and layout preferences. Each root-level Context update triggers a re-render of the entire tree beneath it.

### What to check

**Context providers at the app root that serve a single feature**
- A `<ScreenerFilterProvider>` at the root when only the screener page uses it. Move it to the screener layout or page component. The cost of root placement: every Context value change re-renders the entire app tree, forcing `React.memo` everywhere as a band-aid.
- Severity: **P2** when causing measurable re-render cascades. **P3** when merely misplaced but not causing harm.

**Context used for server-state that React Query should manage**
- A `<StocksContext>` that fetches and provides stock data via Context. This duplicates React Query's job and loses its caching, background refetch, and stale-while-revalidate behavior. Use React Query for server data, Context for true client-side concerns (auth, i18n, theme).
- Severity: **P2**

**Context without stable value references**
- `<AuthContext.Provider value={{ user, login, logout }}>` — this creates a new object on every render, causing all consumers to re-render even if the values haven't changed. Memoize the value: `const value = useMemo(() => ({ user, login, logout }), [user, login, logout])`.
- Severity: **P2** for Contexts with many consumers. **P3** for Contexts consumed by 1-2 components.

---

## Principle 15: Responsive design with heavy data tables requires a different approach — not just CSS breakpoints

*Dodds: "Inline as much as you can." For responsive data tables, the problem isn't CSS — it's information architecture.*

A stock screener table with 15 columns on desktop cannot be made responsive by shrinking font size. On mobile, the user needs a fundamentally different data presentation — card layout, prioritised columns, or horizontal scroll with pinned first column.

### What to check

**Data tables with no mobile strategy**
- `<table>` with 10+ columns and no responsive handling. On mobile: horizontal scroll with no indication that more columns exist, or worse, columns overflow the viewport.
- Pattern: on mobile, switch to a card layout showing the 3-4 most important fields (ticker, price, score, change), with a "show more" expand. Or: horizontal scroll with the ticker column pinned.
- Severity: **P2** for the main screener table. **P3** for detail-page tables.

**Tailwind responsive modifiers missing on layout components**
- Fixed-width layouts without responsive breakpoints. Tailwind 4 classes like `grid-cols-1 md:grid-cols-2 lg:grid-cols-3` should be the norm for card grids. Check: are layout components using responsive Tailwind classes?
- Severity: **P2** for pages that are unusable on mobile. **P3** for minor layout issues.

**Touch targets too small for mobile**
- Filter buttons, table row actions, and icon buttons smaller than 44x44px (Apple's HIG minimum). Tailwind: ensure interactive elements have at least `min-h-11 min-w-11` (44px) on touch devices.
- Severity: **P2**

---

## Principle 16: i18n must not leak untranslated strings — fail visibly or not at all

In a bilingual English/Russian app with a custom i18n service, the failure mode is subtle: a missing translation key silently renders the key itself ("screener.filter.sector") in the UI, or worse, renders the English fallback without the developer noticing the Russian translation is missing.

### What to check

**Missing translation keys render raw key strings**
- The i18n service should never silently return the key name. It should either return a fallback language string (English as fallback for missing Russian) with a dev-mode warning, or throw in development.
- The cost: Russian-speaking users see `"watchlist.empty.title"` instead of a real message. This looks broken and unprofessional.
- Severity: **P2**

**Hardcoded strings bypassing i18n**
- User-visible text written directly in JSX instead of going through the i18n service. Every string a user can see must go through translation — including error messages, empty states, button labels, and toast notifications.
- Severity: **P2** for frequently visible strings. **P3** for edge-case error messages.

**Dynamic content not handled by i18n**
- Pluralization ("1 stock" vs "5 stocks"), number formatting (1,250 vs 1.250), date formatting (MM/DD vs DD.MM), and currency formatting. Russian and English have different pluralization rules (1, 2-4, 5+). Check: does the i18n service handle these, or are they hardcoded?
- Severity: **P2** for number/date formatting in a financial app. **P3** for pluralization edge cases.

---

## Principle 17: Mutations need proper error handling and user feedback

*Dodds: "Error handling is not an afterthought — it's a core part of the user experience."*

Watchlist add/remove, alert creation, and settings changes are mutations to the FastAPI backend. Without proper error handling, the user clicks a button, nothing visibly happens, and they don't know if it worked. Worse: the optimistic update shows success, the request fails silently, and the user's data is lost on the next page load.

### What to check

**useMutation without onError handling**
- `useMutation({ mutationFn: addToWatchlist })` without `onError`. If the request fails (network error, 401 expired token, 500 server error), the user sees nothing. Add toast notification on error, rollback optimistic update, and log the error.
- Severity: **P1** for data-mutating operations (watchlist, alerts, settings). **P2** for non-critical mutations.

**No loading indicator during mutation**
- The WatchlistButton should show a pending state while the mutation is in flight. Without it: the user clicks, nothing happens for 200ms, they click again, two requests fire, potential duplicate data.
- Pattern: `const { mutate, isPending } = useMutation(...)` — use `isPending` to disable the button and show a spinner.
- Severity: **P2**

**No retry logic for transient failures**
- Network blips cause a single failed request. React Query's `useMutation` doesn't retry by default (unlike `useQuery`). For idempotent mutations (add to watchlist), configure `retry: 1` with `retryDelay: 1000`.
- Severity: **P3**

---

## Principle 18: Tailwind must be systematic — not ad-hoc utility strings

Without CSS Modules, Tailwind is the entire styling system. The risk is not Tailwind itself — it's ad-hoc, inconsistent utility usage that makes the UI visually incoherent and the code unmaintainable.

### What to check

**Inconsistent spacing and sizing**
- One card uses `p-4`, another `p-5`, a third `px-3 py-6`. Without a spacing system, the UI feels disjointed. Establish spacing tokens via Tailwind config (or shadcn/ui's design tokens) and use them consistently.
- Severity: **P3** for minor inconsistencies. **P2** when the visual inconsistency is noticeable to users.

**Long className strings that should be component abstractions**
- `className="flex items-center justify-between p-4 rounded-lg border border-gray-200 bg-white shadow-sm hover:shadow-md transition-shadow"` duplicated across 10 components. This is a styling pattern that should be a component (`<Card>`) or a Tailwind `@apply` in a utility class — or better, just use shadcn/ui's `<Card>`.
- The concrete cost: changing the card style requires finding and updating 10 separate className strings.
- Severity: **P3**

**Dark mode classes missing or inconsistent**
- If the app supports dark mode: every background, text, and border color needs a `dark:` variant. Missing `dark:` classes cause white-on-white or black-on-black text in dark mode. Check: is dark mode supported? If yes, are `dark:` variants applied consistently?
- Severity: **P2** if dark mode is supported but broken. Not applicable if dark mode is not a feature.

---

## Principle 19: React Query cache invalidation must be deliberate — not accidental

*Dodds: "Server state is fundamentally different from UI state. It's asynchronous, it's owned by someone else, and it can become stale without you knowing."*

With 455 useQuery calls, cache invalidation is the most complex coordination problem in the app. A mutation that invalidates too broadly (clearing the entire cache) causes 455 refetches. A mutation that invalidates too narrowly misses related queries and shows stale data.

### What to check

**Mutations invalidating the entire cache**
- `queryClient.invalidateQueries()` with no filter — this marks every query as stale and triggers refetches for all mounted queries. After adding a stock to a watchlist, only watchlist-related queries need invalidation, not the entire stock universe.
- The cost: visible loading spinners across the entire page, wasted bandwidth, and potential rate limiting from FastAPI.
- Severity: **P2**

**Mutations not invalidating related queries**
- Adding a stock to a watchlist invalidates `['watchlist']` but not `['stock', ticker]` — so the stock detail page still shows the old "not in watchlist" state until the user navigates away and back.
- Pattern: think in terms of data relationships. A watchlist mutation affects: the watchlist list query, the individual stock's watchlist status, and potentially the watchlist count in the nav.
- Severity: **P2**

**Stale closures in mutation callbacks**
- `onSuccess` callbacks that reference stale state variables. Use the `variables` parameter from the mutation and the latest cache state, not closures over component state.
- Severity: **P2** when causing incorrect cache updates. **P3** for cosmetic issues.

---

## Principle 20: Avoid re-render cascades — React 19 makes this easier but not automatic

*Dodds calls re-render cascades "death by a thousand cuts." Each individual re-render is cheap, but 50 components re-rendering on every keystroke is noticeable.*

React 19's compiler (if enabled) auto-memoizes, but the reviewer should not assume it's active. Even with the compiler, structural issues (Context at root, unstable references, inline object props) can cause cascades.

### What to check

**Inline objects and arrays as props**
- `<Chart data={stocks.filter(s => s.sector === 'tech')} />` — this creates a new array on every render, causing Chart to re-render even if the data hasn't changed. Memoize: `const techStocks = useMemo(() => stocks.filter(...), [stocks])`.
- The cost is proportional to component weight. For a text display: negligible. For a Recharts chart with 500 data points: expensive re-mount.
- Severity: **P2** for expensive children (charts, large tables). **P3** for lightweight children.

**Callback functions recreated on every render**
- `<WatchlistButton onClick={() => addToWatchlist(ticker)} />` — new function reference every render. If WatchlistButton is memoized with `React.memo`, the memo is defeated. Use `useCallback` or pass stable references.
- In React 19 with the compiler: this may be auto-handled. Without the compiler: explicit `useCallback` is needed.
- Severity: **P3** in most cases. **P2** if causing visible jank in a list of 100+ items.

**Context value changes triggering full-tree re-renders**
- An auth Context that stores `{ user, token, refreshToken, lastActivity }` — updating `lastActivity` on every API call re-renders every component that consumes auth, even though they only care about `user`. Split into separate Contexts: `AuthUserContext` (changes rarely) and `AuthTokenContext` (changes on refresh).
- Severity: **P2** when measurable. **P3** when theoretical.

---

## Principle 21: API fetch layer should be centralized — not scattered across components

Without tRPC, every API call is a raw `fetch()`. If 455 components each construct their own fetch URL, headers, and error handling, there's no single place to add auth tokens, handle 401 redirects, retry on failure, or switch base URLs between environments.

### What to check

**No centralized API client**
- Components calling `fetch('http://api.example.com/stocks')` with hardcoded URLs. There should be a single `apiClient` module that handles: base URL from environment, auth token injection, response parsing, error normalization, and 401 detection (triggering token refresh).
- The failure mode: changing the API base URL requires finding and updating every fetch call. A token refresh bug requires fixing it in 50 different places.
- Severity: **P1**

**Auth token not automatically injected**
- Components manually reading the token from context/storage and adding it to headers. The API client should handle this — components should not know about tokens.
- Severity: **P2**

**No response normalization**
- Some components check `response.ok`, others don't. Some parse JSON, others use `.text()`. Some handle errors, others swallow them. The API client should normalize: check `response.ok`, parse JSON, throw a typed error on failure.
- `window.fetch` only rejects on network errors — NOT on 4xx/5xx. Any raw fetch without `response.ok` checking silently treats server errors as success.
- Severity: **P2**

---

## Principle 22: Error messages must be user-facing — not developer-facing

*Dodds: "The error message tells you exactly what's wrong." — applied to UI, not just developer errors.*

When a stock data fetch fails, the user should see "Unable to load stock data. Please try again." — not `TypeError: Cannot read properties of undefined (reading 'map')` or `Error: 500 Internal Server Error` or worse, a blank white screen.

### What to check

**Raw error messages shown to users**
- `{error.message}` rendered directly in the UI. API errors from FastAPI may contain Python tracebacks, SQL errors, or internal field names. Map errors to user-friendly messages based on status code and error type.
- Severity: **P2** for any raw error message visible to users. **P1** if the error contains internal system details (security concern).

**No retry mechanism in error UI**
- Error state that shows "Something went wrong" with no action button. Users can't recover without refreshing the page. Every error boundary and error state should include a "Try again" button that calls `refetch()` or `reset()`.
- Severity: **P2**

**Empty states without guidance**
- A watchlist with no items that shows an empty white area. The user doesn't know if it's loading, broken, or intentionally empty. Show "No stocks in your watchlist yet. Use the screener to find stocks and add them here." with a link to the screener.
- Severity: **P2** for core features (watchlist, alerts, screener results). **P3** for secondary features.

---

## Principle 23: Colocation — tests, utilities, types, and hooks live next to their consumers

*Dodds: "Place code as close to where it's relevant as possible. Things that change together should be located as close as reasonable."*

### What to check

**Utility functions in a top-level `utils/` directory with a single consumer**
- `utils/formatScore.ts` used only by `ScoreCard.tsx`. Move it into the ScoreCard file or directory until it's genuinely shared. The cost of non-colocation: orphan utilities that persist after their consumer is deleted, and developers who miss related code when modifying a feature.
- Severity: **P3**

**Types in a global `types/` directory**
- `types/Stock.ts` defining an interface used by one page. Colocate with the page or feature. Global types are appropriate only for genuinely shared types (API response schemas, shared component props).
- Severity: **P3**

**Constants and config far from usage**
- Query key factories, API endpoint strings, and feature flags in a top-level `constants/` file. Colocate with the feature that uses them. When they're shared across features, lift them — but start colocated.
- Severity: **P3**

---

## Principle 24: Framer Motion animations must serve a purpose — not just look pretty

Animation should communicate state changes, guide attention, or provide spatial context. Animation that exists because "it looks cool" adds bundle size, slows interaction, and can cause accessibility issues (motion sickness, vestibular disorders).

### What to check

**Animations without `prefers-reduced-motion` respect**
- Users with vestibular disorders set their OS to reduce motion. Check: is there a `prefers-reduced-motion` media query that disables or reduces Framer Motion animations? Framer Motion supports `useReducedMotion()` hook.
- Severity: **P2** for page transitions and large-scale motion. **P3** for micro-interactions.

**Entry animations that delay content visibility**
- Page content that fades in over 500ms. The user is waiting to see financial data — the fade-in is a 500ms delay disguised as polish. Entry animations should be fast (150-200ms) or eliminated.
- Severity: **P2** for data-heavy pages. **P3** for landing/marketing pages.

**Layout animations causing jank on data updates**
- `layout` prop on Framer Motion components that animate when React Query data changes (new stock added to screener results, watchlist reorder). Every data update triggers a layout animation, which is disorienting when the user is reading data, not interacting with it. Only animate layout changes triggered by user actions.
- Severity: **P2**

---

## Principle 25: Bundle size discipline — every dependency must justify its weight

*Dodds: "The fastest code is the code that doesn't ship to the browser."*

The stack already includes heavy dependencies: Recharts (~200KB), Framer Motion (~30KB), and React Query (~30KB). Each additional dependency compounds the initial load time, especially on mobile.

### What to check

**Duplicate functionality from multiple libraries**
- Using both `date-fns` and `dayjs` for date formatting. Using both native `fetch` and `axios`. Using a charting library AND custom SVG components that replicate chart functionality. Pick one, remove the other.
- Severity: **P2** for functional duplicates. **P3** for minor overlap.

**Dependencies importable tree-shakably but imported whole**
- `import _ from 'lodash'` instead of `import debounce from 'lodash/debounce'`. The former imports the entire 70KB library. Check: are large libraries imported with named/path imports?
- Severity: **P2** for large libraries (lodash, moment, etc.). **P3** for small libraries.

**Client-side dependencies that could be replaced with platform APIs**
- `uuid` when `crypto.randomUUID()` is available. `classnames` when Tailwind's conditional classes or template literals suffice. `is-email` when a simple regex or HTML `type="email"` validation works.
- Severity: **P3**

---

## Quick Reference: Severity Guide

| Severity | Pattern | Examples |
|----------|---------|----------|
| **P1 — Fix Now** | Data corruption, security holes, crashes, known past bugs | API responses typed as `any`, no centralized API client, auth redirect loops, token refresh races, missing auth fail-fast hook, missing error boundaries, financial data without error handling |
| **P2 — Fix Soon** | Stale data, wasted resources, degraded UX, broken accessibility | State sync via useEffect, 455 queries at staleTime=0, cache invalidation too broad/narrow, no mobile table strategy, raw errors shown to users, charts without a11y, missing mutation error handling, re-render cascades, inconsistent query keys |
| **P3 — Consider** | Hygiene that compounds over time | Premature hook extraction, non-colocated utilities, verbose Tailwind strings, unnecessary shadcn/ui wrappers, missing `prefers-reduced-motion`, cosmetic animation delays |

### The Overriding Filter

Before writing any finding, apply the Dodds synthesis:

1. **Is this state necessary?** If derivable, flag it. If it duplicates React Query's cache, flag it.
2. **Is this abstraction necessary?** If single-use, inline it. AHA: wait until duplication screams.
3. **Does the type system protect the API boundary?** With no tRPC, every untyped `fetch` response is a hole in the armour.
4. **Is every component handling all four states?** Loading, error, success, empty — all four, every time.
5. **Would a user notice if this broke?** Test that first. Zero tests means zero confidence.
