# PWA & Offline-First Quality Reference — Jake Archibald

Philosophy: Jake Archibald. Google Chrome developer advocate, Service Worker specification editor, creator of the offline cookbook, key contributor to Workbox. Core doctrine: "Offline isn't an error state — it's a fact of life. Build for it."
Stack context: PWA for Dense Club fitness SaaS. React 18 SPA (Vite 6) with vite-plugin-pwa + Workbox. Dexie (IndexedDB wrapper) for offline mutation queue. Athletes train in commercial gyms with concrete walls and dead wifi — offline workout logging is a core requirement. Background Sync for deferred mutations. Idempotency keys on every write to prevent duplicate PRs on reconnect. TanStack Query with networkMode: "offlineFirst" and persistQueryClient to IndexedDB.

Every finding must describe the **concrete failure mode** — not just "bad offline support."

---

## Principle 1: Service Worker registration must not block first paint

*Archibald: "The service worker should be invisible on first visit. It sets up for the second visit."*

### What to check

**Register after window.onload**
- SW registration fires network requests (precache manifest). If registered in React's entry point (before render), it competes with the initial API calls (today's workout, user profile) for bandwidth. On gym wifi (1-5 Mbps), this adds 2-3 seconds to first meaningful paint.
- Pattern: `if ('serviceWorker' in navigator) { window.addEventListener('load', () => navigator.serviceWorker.register('/sw.js')) }`
- Severity: **P2** if SW registration blocks first paint.

**Update flow: toast, never auto-reload**
- When a new SW version is detected: show "Update available" toast → athlete taps "Update" → `skipWaiting()` + `clients.claim()` + reload. NEVER auto-reload — athlete mid-set logging loses unsaved form data (weight, reps, effort for current set).
- Severity: **P1** if auto-reload is enabled. Athlete loses workout data mid-training.

**skipWaiting only on user action**
- `self.skipWaiting()` in the SW install event means the new SW activates immediately, potentially breaking in-flight requests. Only call it when the user explicitly accepts the update.
- Severity: **P2** if skipWaiting fires automatically.

---

## Principle 2: Cache strategy per resource type

*Archibald: "There is no one caching strategy. Each resource type demands its own."*

### What to check

**App shell: CacheFirst with precache manifest**
- HTML, JS, CSS bundles: precached by Workbox `injectManifest`. On subsequent visits, the app loads from cache instantly (0ms network). Workbox checks for updates in the background.
- vite-plugin-pwa handles manifest generation. Verify: the manifest includes all route chunks (lazy-loaded admin/coach pages too) so the app works offline regardless of which page was visited first.
- Severity: **P1** if the app shell is not precached — athlete opens app in dead zone, gets blank screen.

**API GET responses: NetworkFirst with cache fallback**
- Today's workout, exercise list, PR history: try network first (fresh data), fall back to cached response if offline. Use Workbox `NetworkFirst` strategy with `cacheName: "api-cache"` and `networkTimeoutSeconds: 3` (don't wait forever on slow gym wifi).
- Without cache fallback: athlete opens app offline → all useQuery calls hang → loading spinner forever → athlete can't see their workout.
- Severity: **P1** if API GET responses have no cache fallback.

**API mutations (POST/PUT/DELETE): NEVER cache**
- Mutations must go through the Dexie offline queue, not the SW cache. Caching a POST response means: athlete logs a set → response cached → next identical POST returns cached response without hitting server → server never records the set.
- Severity: **P1** if any mutation response is cached by the SW.

**Images/icons: CacheFirst**
- Exercise demo thumbnails, profile pictures, UI icons: CacheFirst with `expiration: { maxEntries: 200, maxAgeSeconds: 30 * 24 * 60 * 60 }`. These change rarely; always serve from cache.
- Severity: **P3** — performance optimization, not correctness.

---

## Principle 3: Offline mutation queue via Dexie

*Archibald: "The network is a lie. Store locally first, sync when you can."*

The Dexie queue is the heart of offline-first. Every write operation goes through it.

### What to check

**Queue schema in Dexie**
```javascript
db.version(1).stores({
  mutationQueue: '++id, endpoint, method, idempotencyKey, createdAt, retryCount, status'
});
```
- Fields: `endpoint` (API path), `method` (POST/PUT/DELETE), `body` (JSON), `idempotencyKey` (UUID v4), `createdAt` (timestamp), `retryCount` (int, max 5), `status` (pending/syncing/failed/synced).
- Without `idempotencyKey`: Background Sync retries a mutation that already succeeded → duplicate performance log → duplicate PR detection → athlete gets two "New PR!" notifications for the same set.
- Severity: **P1** if idempotency keys are missing from the queue.

**Queue processing order**
- Mutations must be replayed in order (FIFO by `createdAt`). Out-of-order: athlete logs set 1 (bench 80kg) → logs set 2 (bench 85kg, new PR). If set 2 syncs before set 1: PR detection fires for 85kg → then 80kg arrives → no issue. But if set 1 arrives first and triggers a different code path, ordering matters for tonnage calculations and session totals.
- Severity: **P2** — FIFO queue order.

**Retry with exponential backoff**
- Failed mutations retry: 1s, 2s, 4s, 8s, 16s, then give up after 5 retries. Mark as `failed` and surface in UI ("3 sets failed to sync — tap to retry").
- Without backoff: rapid retries on a flaky connection = battery drain + network congestion. Without max retries: a permanently invalid mutation (409 conflict) retries forever.
- Severity: **P2**

**Pending count badge in UI**
- Show "3 pending" badge on the sync indicator when mutations are queued. Athletes need confidence their data is saved. Without this: athlete finishes workout, closes app, mutations are queued but athlete doesn't know → opens app later on wifi, expects data, it's still syncing → panic, logs everything again manually.
- Severity: **P2**

---

## Principle 4: Idempotency keys prevent duplicate PRs

*Archibald: "At-least-once delivery means exactly that. Your server must handle the 'at least' part."*

### What to check

**UUID generated at mutation creation, not at send time**
- The idempotency key is created when the athlete taps "Save Set" — stored in Dexie with the mutation. Every retry of that mutation sends the same key. If the key is generated at send time: each retry gets a new key → server treats each retry as a new set → 3 retries = 3 performance logs.
- Severity: **P1** — this is the single most important offline correctness rule.

**Backend checks idempotency_keys table**
- On every write endpoint: check if `X-Idempotency-Key` exists in `idempotency_keys` table. If yes → return the stored response (200, not 409). If no → process the mutation, store the key + response. Without this: the client sends the correct key, but the server ignores it → duplicates.
- The check + insert must be atomic (INSERT ... ON CONFLICT DO NOTHING + check if inserted).
- Severity: **P1** if the backend doesn't have an idempotency_keys table.

**Key includes enough context for debugging**
- Format: `perf-{userId}-{exerciseId}-{date}-{timestamp}-{random}`. The first 4 parts help debug duplicates in production ("why does this user have two performances for the same exercise on the same date?"). The random suffix ensures uniqueness.
- Severity: **P3** — debugging convenience.

---

## Principle 5: Background Sync is not guaranteed

*Archibald: "Background Sync is best-effort. It fires when the browser thinks you're online. 'Thinks' is the key word."*

### What to check

**Feature detection + fallback**
- Background Sync: Chrome yes, Safari partial (iOS 16.4+ with limitations), Firefox no. Always feature-detect:
  ```javascript
  if ('SyncManager' in window) {
    navigator.serviceWorker.ready.then(reg => reg.sync.register('flush-mutations'));
  }
  ```
- Fallback: on `visibilitychange` (app comes to foreground) + `online` event → flush Dexie queue manually. This catches Firefox, old Safari, and cases where Background Sync doesn't fire.
- Severity: **P1** if Background Sync is the only sync mechanism — Firefox users permanently lose offline data.

**Visibility change flush**
- `document.addEventListener('visibilitychange', () => { if (document.visibilityState === 'visible') flushQueue() })` — every time the athlete opens the app, attempt to flush. This is the most reliable sync trigger across all browsers.
- Severity: **P1** if no foreground flush exists.

**Online event is unreliable**
- `navigator.onLine` returns true on captive portals and false positives. Don't rely solely on the `online` event to trigger sync. The flush function should attempt the request and handle failure gracefully, regardless of `navigator.onLine` status.
- Severity: **P2**

---

## Principle 6: Conflict resolution on reconnect

*Archibald: "Offline-first means accepting that conflicts WILL happen. The question is how you resolve them, not how you prevent them."*

### What to check

**Server wins for structure, client wins for data**
- Coach edits workout template while athlete is offline logging sets. On sync:
  - Structure changes (exercise order, scheme, removed exercises): server/coach version wins. The athlete's logged sets are still valid — they just might reference an exercise that was removed from the template.
  - Performance data (weights, reps, effort): athlete's data always wins. The athlete physically lifted the weight; no server state overrides that.
- Severity: **P2** if there's no conflict resolution strategy defined.

**Orphaned performances on exercise removal**
- Athlete logs sets for exercise X offline. Coach removes exercise X from the template. On sync: the performance log references exercise X — which still exists in the exercises table (exercises are never deleted), just not in the current template. This is fine. Don't reject the performance. Link it to the exercise directly.
- Severity: **P2** if orphaned performances are silently dropped during sync.

**Last-write-wins for non-critical fields**
- Profile updates, bodyweight log, display preferences: last-write-wins with `updated_at` timestamp. No conflict UI needed — these are single-user fields where the latest value is always correct.
- Severity: **P3** — simple strategy for simple fields.

**Conflict UI only when necessary**
- Don't show a merge dialog for every sync conflict. Only show UI when the athlete's logged data would be materially changed by the server state. For Dense Club: this is rare enough that a simple "Your coach updated today's workout while you were offline" toast is sufficient — no merge UI needed at launch.
- Severity: **P3** — over-engineering conflict UI is a common trap.
