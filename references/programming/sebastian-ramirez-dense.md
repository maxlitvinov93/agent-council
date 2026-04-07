# FastAPI Backend Quality Reference — Sebastian Ramirez

Philosophy: Sebastian Ramirez (tiangolo). Creator of FastAPI, SQLModel, Typer. Expertise: Pydantic validation, dependency injection, async endpoints, OpenAPI auto-docs, proper API structure, type hints as contracts, middleware patterns.
Stack context: FastAPI 0.115+ backend for Dense Club fitness SaaS. Python 3.12 / SQLAlchemy 2.0 async / asyncpg / PostgreSQL 16 / Pydantic v2. JWT auth (python-jose) with refresh token rotation. 5-role system (admin, coach, athlete_trial/standard/premium). ~50 endpoints across role-prefixed routers (/api/athlete/*, /api/coach/*, /api/admin/*). APScheduler for cron. Stripe webhooks. Resend email. Cloudflare R2 via aioboto3. structlog logging. slowapi rate limiting.

Every finding must describe the **concrete failure mode** — not just "bad practice."
Ramirez built FastAPI around a single conviction: Python type hints should be the single source of truth for validation, serialization, and documentation. When the code drifts from this principle, the auto-generated OpenAPI docs lie, validation gaps open, and the API becomes a hand-maintained contract that nobody maintains.

---

## Principle 1: Pydantic response models are contracts — never return raw ORM objects

*Ramirez: "The response model is the most important part of the endpoint. It defines what the client can rely on."*

Dense Club's API response shapes diverge from DB shapes. `GET /api/athlete/workouts/today` returns a resolved workout combining assignments + overrides + micro blocks — that is not a DB table. Returning ORM objects leaks internal structure to the React frontend.

### What to check

**Endpoints missing `response_model`**
- Every router function must declare `response_model=SomeSchema`. Without it, FastAPI serializes whatever the function returns — the OpenAPI spec shows `200: {}`, useless for the React frontend consuming it.
- If the endpoint returns a list, use `response_model=list[PerformanceResponse]`, not `response_model=list`.
- Severity: **P1** if the endpoint returns user data, auth tokens, body metrics, or payment status without a response model. **P2** for other endpoints.

**ORM objects passed directly to the response**
- A function that does `return db_performance` instead of `return PerformanceResponse.model_validate(db_performance)` relies on implicit conversion. This works until someone adds `bodyweight_contribution_pct` to the Exercise ORM model — the internal training coefficient silently appears in every API response visible to athletes.
- Check: does every Pydantic schema use `model_config = ConfigDict(from_attributes=True)`? Without it, ORM→schema conversion fails at runtime.
- Severity: **P2** for endpoints where schema and ORM model have diverged.

**Schema explosion without inheritance**
- 5 roles see different data. An exercise needs: `ExerciseBrief` (athlete list), `ExerciseDetail` (workout view), `ExerciseAdmin` (admin CRUD), `ExerciseCoach` (assignment builder). Without a base schema and selective inheritance, you get near-identical classes that drift independently. When `bodyweight_contribution_pct` changes from float to Decimal, half the schemas get updated and half don't.
- Use a base schema with shared fields and extend per role. Use `Field(exclude=True)` for role-specific filtering.
- Severity: **P2** if there are more than 3 schemas for the same entity without a shared base.

---

## Principle 2: Dependency injection is the architecture — not a convenience

*Ramirez: "Dependencies are the way FastAPI handles shared logic, auth, database sessions, and any reusable component."*

Dense Club has 5 roles and ~50 endpoints. Consistency of dependency usage determines whether the auth layer has gaps.

### What to check

**DB session not injected via `Depends`**
- If any route creates its own session instead of using `Depends(get_db)`, that session won't be closed on exceptions, won't participate in request-scoped lifecycle, and won't be mockable in tests.
- The `get_db` dependency must use `yield` (not `return`) to guarantee `session.close()` runs even when the endpoint raises. With asyncpg, this means `async with async_session() as session: yield session`.
- Severity: **P1** if any route manages its own session lifecycle. **P2** if `get_db` uses `return` instead of `yield`.

**Auth dependency not applied uniformly**
- With 50 endpoints across role-prefixed routers, it's easy for one endpoint to miss `Depends(get_current_user)`. The endpoint works in dev (no auth check = no error) but exposes athlete data to unauthenticated users in production.
- Apply auth at the router level: `router = APIRouter(dependencies=[Depends(get_current_user)])`. Override at the endpoint level only for public routes (register, login, health check).
- Check: is there a route that accepts user-specific data without an auth dependency? Especially dangerous: coach endpoints accessing athlete performances without verifying the coach-athlete relationship.
- Severity: **P1** for any endpoint that reads or writes user-specific data without auth.

**Role enforcement as a dependency**
- Dense Club has 5 roles. Role checks must be a composable dependency: `Depends(require_role(["coach", "admin"]))`. Inline `if user.role != "coach"` scattered across routes will miss cases — especially the `is_also_athlete` dual role.
- Coach endpoints need TWO checks: role + relationship. `require_role(["coach"])` confirms the role, but a separate `Depends(verify_coach_assignment(athlete_id))` must confirm this coach is assigned to this athlete.
- Severity: **P1** if role enforcement is missing. **P2** if role checks exist but coach-athlete relationship is not verified.

**Trial expiry as middleware, not per-route check**
- Trial expiry (7-day limit) must be checked on every authenticated request for athlete_trial users. If implemented as a per-route dependency, someone will forget it on one endpoint. Implement as middleware that runs after auth: if `user.role == "athlete_trial"` and trial expired → 403. Exempt: profile, payment, and logout endpoints.
- Severity: **P1** if trial expiry is only checked on the payment page (client-side bypass = unlimited free access).

---

## Principle 3: Route organization must match the OpenAPI spec the client reads

*Ramirez: "Tags, prefixes, and response descriptions are not decoration — they are the API documentation."*

### What to check

**Role-prefixed routing**
- Router structure: `/api/athlete/*`, `/api/coach/*`, `/api/admin/*`, `/api/auth/*`, `/api/public/*`. Each prefix maps to a role boundary. The React frontend team uses the OpenAPI spec to build API client calls — messy routes = messy client code.
- Tags should match the role prefix: `tags=["Athlete - Workouts"]`, `tags=["Coach - Programs"]`. Not `tags=["workout-routes"]`.
- Severity: **P2** if endpoints appear untagged or with inconsistent naming in OpenAPI spec.

**RESTful path conventions**
- Resource nouns, plural, lowercase: `/api/athlete/performances`, not `/api/athlete/getPerformances`. Nested resources: `/api/coach/athletes/{athlete_id}/performances`, not `/api/coach/athlete-performances?athlete_id=X`.
- Severity: **P3** for naming inconsistencies.

**Admin routes lazy-loaded**
- Admin endpoints (user management, exercise CRUD, skill definitions) should be in a separate router included conditionally or always included but protected. The critical point: admin endpoints must NEVER appear in the public OpenAPI spec in production. Set `docs_url=None` in production or exclude admin routes from the schema.
- Severity: **P2** if admin endpoints are visible in production API docs.

---

## Principle 4: The hot write path must be one transaction

*Ramirez: "FastAPI gives you async — use it to keep the event loop free, not to split transactions."*

Dense Club's performance logging is the hottest path: every set the athlete logs triggers performance write → loaded_volume computation → PR detection (upsert) → bodyweight log upsert → activity_event creation → ranking invalidation. This MUST be one database transaction.

### What to check

**Transaction boundary wrapping the entire chain**
- The service function must use `async with session.begin():` to wrap all steps. If any step fails (e.g., PR upsert conflict), the entire transaction rolls back. Without this: a failed PR detection leaves an orphaned performance record with no corresponding activity_event — the community feed is inconsistent.
- Severity: **P1** if the hot write path has multiple independent commits.

**No HTTP calls inside the transaction**
- Stripe API calls, Resend email, R2 uploads — these must happen AFTER the transaction commits. A slow Stripe call inside a transaction holds a database connection for seconds, starving other requests. Pattern: commit DB transaction → then fire-and-forget via `BackgroundTasks.add_task(send_pr_email, ...)`.
- Severity: **P1** if any outbound HTTP call is inside a `session.begin()` block.

**Tonnage calculation correctness**
- `total_tonnage = (weight_kg + bodyweight_kg * exercise.bodyweight_contribution_pct) * total_reps`. If `bodyweight_kg` is None (athlete hasn't weighed in), the calculation must use the latest known bodyweight from `bodyweight_logs`, not default to 0. Defaulting to 0 means pull-up tonnage (100% BW contribution) shows as just the added weight — athlete's 20kg weighted pull-up shows 20kg instead of 90kg.
- Severity: **P1** if bodyweight fallback is missing in tonnage calculation.

---

## Principle 5: Background tasks vs in-request work

*Ramirez: "BackgroundTasks is for fire-and-forget. If you can't afford to lose it, use a proper queue."*

### What to check

**BackgroundTasks for email only**
- Welcome email after signup, PR notification, trial expiry warning — these are fire-and-forget. If Resend is down, retry 3 times, then log to `email_failures` table. Acceptable loss.
- Severity: **P2** if email sending blocks the response.

**APScheduler for reliable cron**
- Trial expiry checks (hourly), program transitions (every 15 min), Stripe status sync (every 6 hours), ranking cache refresh (daily at 3 AM), old refresh token cleanup (daily at 4 AM). These use PostgreSQL advisory locks for multi-worker dedup.
- Severity: **P1** if any cron job runs without advisory lock protection (double-firing on 2-worker Gunicorn).

**NEVER use BackgroundTasks for Stripe webhooks**
- Stripe webhook processing must be synchronous within the request handler. If you push it to a background task and the response returns 200 before processing completes, a crash loses the webhook. Stripe will retry, but the 200 told Stripe "I got it" — retry behavior is unreliable after 200.
- Severity: **P1** if webhook processing is deferred to BackgroundTasks.

---

## Principle 6: Async discipline — no sync calls in async handlers

*Ramirez: "One blocking call in an async handler blocks the entire event loop for every other request."*

### What to check

**No `requests` library**
- All outbound HTTP must use `httpx.AsyncClient`. A single `requests.post()` in an async handler blocks the event loop for the duration of the HTTP call. With 2 Gunicorn workers, two slow Stripe calls = zero capacity.
- Severity: **P1** if `requests` is imported anywhere in the async codebase.

**CPU-bound work via run_in_executor**
- Pillow image resizing (profile pictures) is CPU-bound sync. Must use `await loop.run_in_executor(None, resize_sync, image_bytes)`. Without this: a 5-second image resize blocks all other requests for 5 seconds.
- Severity: **P2** if any CPU-bound operation runs directly in an async handler.

**structlog async logging**
- Use `await log.ainfo(...)` in async handlers, not `log.info(...)`. The sync version is technically safe (structlog doesn't do I/O by default) but establishes a bad pattern. When a custom processor adds I/O (e.g., writing to a file), the sync call blocks.
- Severity: **P3** for sync logging in async handlers.

---

## Principle 7: Fitness-specific validation at the Pydantic boundary

*Ramirez: "Pydantic validators are the firewall. Everything inside is trusted."*

### What to check

**Dense scheme regex validation**
- `dense_scheme` must match `^\d+D(\d+(-\d+-\d+)?)?$`. Examples: "10D5" (valid), "10D" (valid — bodyweight), "10D5-3-2" (valid — drop set). Without the regex: "10D-1" passes validation, breaks the frontend parser, shows NaN in the timer.
- Severity: **P1** if dense_scheme is accepted as free-text string.

**Weight sanity caps**
- `weight_kg`: 0-500 (world record deadlift is ~501kg). `bodyweight_kg`: 20-300. `total_reps`: 1-1000. `effort` (RPE): 1-10. Without caps: a typo of "8000" kg gets stored, blows up tonnage calculation, corrupts rankings. The athlete shows as #1 in the world with a fake 8-ton squat.
- Severity: **P1** if numeric fields have no upper bounds.

**Exercise nature enum validation**
- 7 valid natures: `barbell`, `dumbbell`, `kettlebell`, `bodyweight`, `machine`, `cable`, `band`. Using a string field without enum validation means "Barbell" (capitalized) or "barbell " (trailing space) passes — then fails to match in tonnage calculation logic that checks `exercise.nature == "barbell"`.
- Severity: **P2** if exercise_nature is a free-text string instead of a Python Enum.

**Workout date as DATE, not datetime**
- `workout_date` must be a `date` type, not `datetime`. Athletes log workouts for a specific date regardless of timezone. Storing as datetime with timezone creates the bug: athlete in UTC+8 logs at 11 PM → server stores as next day in UTC → workout appears on wrong date in dashboard.
- Severity: **P1** if workout_date accepts or stores datetime with timezone.
