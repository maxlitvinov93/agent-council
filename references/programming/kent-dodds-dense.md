# React Frontend Quality Reference — Kent C. Dodds

Philosophy: Kent C. Dodds. Creator of Testing Library, author of Epic React, authority on React component patterns, testing strategy, and frontend architecture. Key doctrines: AHA Programming, the Testing Trophy, "derive don't sync", colocation, composition over configuration.
Stack context: React 18 SPA (Vite 6) for Dense Club fitness SaaS. TanStack React Query 5 (primary server state). React Hook Form 7 + Zod 3. React Router 6. Radix UI + Tailwind 3.4 + shadcn/ui. Framer Motion 11. Recharts 2.15. Stripe.js. date-fns 3. PWA via vite-plugin-pwa + Workbox + Dexie (IndexedDB). Migrating from @base44/sdk to custom FastAPI API client. 5-role UI (admin, coach, athlete trial/standard/premium). Offline-first workout logging.

Every finding must describe the **concrete failure mode** — not just "bad practice."

---

## Principle 1: TanStack Query is your server state manager — never duplicate it into useState

*Dodds: "I would take the data I got from the server and treat it like it was UI state. I would mix it in with the actual UI state and this resulted in making both more complex."*

Dense Club's state is 95% server state: workouts, performances, PRs, rankings, skills. React Query IS the state management layer. Any `useState` holding API response data is a sync bug waiting to happen.

### What to check

**API data copied into useState**
- `const [workout, setWorkout] = useState(null)` populated from a `useQuery` result via `useEffect`. The cache already holds this data. The `useState` copy goes stale on background refetch, window focus refetch, and cache invalidation — athlete sees yesterday's workout after coach pushes an override.
- The fix: consume `useQuery` return directly. If transformation is needed, use the `select` option.
- Severity: **P1** for workout data, performances, PRs (stale data = wrong exercises shown). **P2** for non-critical display data.

**Missing or inconsistent query keys**
- Query key factory pattern is mandatory:
  ```
  queryKeys.workout.today(userId)
  queryKeys.performances.list(userId, { date, exerciseId })
  queryKeys.rankings.exercise(exerciseId, scheme)
  queryKeys.community.feed({ filter, cursor })
  ```
- Without a factory: `invalidateQueries` after logging a performance targets a key that doesn't match the actual key used by the workout query — today's workout still shows the old state until manual refresh.
- Severity: **P2**

**staleTime left at default (0) for stable data**
- Default `staleTime: 0` means every query refetches on mount. With 30+ queries per page (exercises, skills, templates), this = 30+ requests per navigation. Tune: workout data 5min, exercise definitions 30min, skill definitions Infinity (static), rankings 10min.
- Severity: **P2** — hundreds of unnecessary requests per navigation.

**networkMode not set to offlineFirst**
- Dense Club is offline-first. Default `networkMode: "online"` pauses queries when offline — athlete opens app in gym dead zone, sees loading spinner forever. Set `networkMode: "offlineFirst"` globally + `persistQueryClient` to IndexedDB so cached workout data loads instantly even offline.
- Severity: **P1** — app is unusable in the primary use environment (gym).

---

## Principle 2: Derive state — don't sync it

*Dodds: "Don't Sync State. Derive It." Any piece of stored state that could be computed from other state is a synchronisation bug waiting to happen.*

### What to check

**Derivable values stored in useState**
- `useState` for filtered exercise list when it could be `useMemo(() => exercises.filter(matchesSearch), [exercises, search])`. The `useState` copy goes stale when exercises are refetched (new exercise added by admin) — search results show old list.
- URL search params mirrored into `useState` — the URL IS the source of truth for community feed filters, ranking filters, exercise search. Derive from `useSearchParams()`.
- Severity: **P1** when derivable state controls data display (wrong exercises shown). **P2** for UI-only state.

**useEffect used solely to synchronise two state variables**
- `useEffect(() => { setFilteredPerformances(filter(performances, date)) }, [performances, date])` — causes extra render + one frame of stale data. Compute during render instead.
- Severity: **P2**

**Tonnage/PR display derived on the client**
- Tonnage totals, PR badges, and progress percentages should be computed from React Query cache data during render, not stored in separate state. If stored separately, they go stale when the performance query refetches.
- Severity: **P2**

---

## Principle 3: Base44 migration discipline

*Dodds: "Colocation means code that changes together should live together."*

33/34 source files import `@base44/sdk`. The migration to FastAPI API client is the highest-risk change in the project.

### What to check

**Centralized API client module**
- Create `src/lib/api/client.ts` with typed functions: `api.athlete.getWorkoutToday()`, `api.athlete.logPerformance(data)`, `api.coach.getAthletes()`. Every component imports from this module, never from `@base44/sdk` or raw `fetch`.
- Without centralization: 33 files each implementing their own fetch calls with inconsistent error handling, auth header injection, and offline queue integration.
- Severity: **P1** if any component uses raw `fetch` or `axios` directly instead of the API client.

**No mixed SDK calls**
- During migration, a component must use EITHER base44 calls OR new API calls, never both. Mixing creates: base44 call creates a performance → new API call queries performances → the base44 performance isn't in the new DB → athlete sees missing workout data.
- Severity: **P1** if any component mixes old and new data sources.

**Auth header injection**
- The API client must automatically inject `Authorization: Bearer {token}` on every request. Token refresh must be handled transparently (intercept 401 → refresh → retry). Without this: every component manually managing tokens = one component forgets = silent auth failure.
- Severity: **P1** if auth is not centralized in the API client.

---

## Principle 4: Component composition over prop drilling

*Dodds: "Use composition. Instead of passing data through several layers, pass the component that needs the data as children."*

Workout resolution creates deep trees: Today → WorkoutView → ExerciseCard → LogModal → SetInput → WeightInput. Prop drilling through 5 levels makes refactoring impossible.

### What to check

**Workout context for the active workout session**
- The current workout (resolved exercises, logged sets, timer state) should be in a React context scoped to the Today page, not drilled through props. `WorkoutProvider` wraps the page, children consume via `useWorkout()`.
- Severity: **P2** if more than 3 props are drilled through more than 3 levels for workout data.

**Compound components for ExerciseCard + LogModal**
- ExerciseCard and LogModal are tightly coupled (logging a set updates the card's display). Use compound component pattern: `<Exercise.Card>`, `<Exercise.LogModal>`, `<Exercise.PRBadge>` sharing state via context.
- Severity: **P3** for minor prop drilling.

**Avoid global state stores**
- No Redux, no Zustand for server data. React Query handles it. Client-only state (modal open, selected tab, timer running) uses useState/useReducer locally. If global client state is needed (active workout session, unit preference), use React Context scoped to the smallest subtree.
- Severity: **P2** if a global store is introduced for data that React Query already manages.

---

## Principle 5: Offline-first patterns in React

*Dodds: "The network is an implementation detail. Users shouldn't care whether data came from cache or server."*

Athletes train in commercial gyms with concrete walls and dead wifi. Offline workout logging is not a nice-to-have — it's the core requirement.

### What to check

**Mutations queue to Dexie when offline**
- Every `useMutation` that writes data (log performance, save PR, update bodyweight) must check network status. If offline: save to Dexie queue with idempotency key → show optimistic UI → sync on reconnect.
- Without queue: athlete logs 10 sets, wifi drops after set 3, sets 4-10 are lost. Athlete doesn't know until they check the app later.
- Severity: **P1** if any write mutation fails silently when offline.

**Optimistic UI for logged sets**
- After athlete taps "Save Set", the UI must immediately show the logged set (weight, reps, effort) without waiting for network response. Use `useMutation.onMutate` to update the query cache optimistically. Rollback on error.
- In a gym: 200-2000ms network latency is common. Without optimistic UI: athlete taps save, nothing happens for 2 seconds, taps again → duplicate log.
- Severity: **P1** for workout logging mutations.

**Sync status indicator**
- Show a small badge/icon: "3 sets pending sync" when offline mutations exist. On reconnect: show "Syncing..." → "All synced". Without this: athlete has no confidence their workout was saved, logs everything again manually on a different device → duplicate data.
- Severity: **P2**

---

## Principle 6: Role-based UI — hide AND protect

*Dodds: "Never rely on UI hiding for security. The server is the authority."*

5 roles see different views. The React app must both hide unauthorized UI elements AND handle server-side 403 responses gracefully.

### What to check

**Route guards via React Router loaders**
- `/coach/*` routes check `user.role === "coach" || user.role === "admin"` in a loader or layout component. Redirect unauthorized users to `/today` (athlete home).
- Without guards: athlete navigates to `/coach/athletes` via URL bar → sees empty coach page → confusing UX at best, data exposure at worst.
- Severity: **P2**

**Admin/coach routes code-split**
- Use React Router lazy routes: `<Route path="admin/*" lazy={() => import("./pages/Admin")} />`. Admin components (user management, exercise CRUD) should not be in the athlete's JavaScript bundle.
- Without splitting: athletes download 200KB of admin code they'll never use. On gym wifi, this means 3-5 extra seconds of load time.
- Severity: **P2**

**Graceful 403 handling**
- If the server returns 403 (expired trial, wrong role, unassigned athlete), the React app must show a meaningful message, not a blank screen or generic error. Trial expired → redirect to payment page with message. Wrong role → redirect to home.
- Severity: **P2**

---

## Principle 7: Form patterns for workout logging

*Dodds: "Forms are where user intent meets application state. Get this wrong and users lose trust."*

Workout logging is the most-used feature. Every friction point here costs athlete engagement.

### What to check

**React Hook Form + Zod for all inputs**
- Dense scheme input, weight input, reps input, effort RPE — all validated via Zod schemas. Transform form values to API shape: lbs→kg conversion must happen in the Zod `.transform()`, not scattered across components.
- Without Zod: athlete types "abc" in weight field → NaN sent to API → 422 error → athlete confused.
- Severity: **P2**

**Unit preference (kg/lbs) handled in the form layer**
- Backend stores kg. Frontend displays user's preference. The conversion must happen in ONE place: the form's Zod transform (input→API) and the display layer (API→display). NOT in individual components.
- Mixing conversion locations: athlete sees 100lbs displayed, logs it, the form converts to kg (45.36), the display component also converts → shows 100lbs. But if one conversion is wrong, athlete logs 100kg thinking it's lbs — records a fake PR of 220lbs.
- Severity: **P1** if unit conversion logic is duplicated across components.

**Auto-save form drafts during offline**
- If the athlete is mid-set-entry and the app goes to background (phone lock, incoming call), the form state must be preserved. Use `react-hook-form` `watch()` + debounced write to Dexie. On app resume, restore form state.
- Without this: athlete enters weight, phone rings, call ends, opens app → form reset, re-enter everything.
- Severity: **P2**
