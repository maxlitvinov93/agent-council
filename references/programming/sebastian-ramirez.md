# FastAPI Backend Quality Reference -- Sebastian Ramirez

Philosophy: Sebastian Ramirez (tiangolo). Creator of FastAPI, SQLModel, Typer. Expertise: Pydantic validation, dependency injection, async endpoints, OpenAPI auto-docs, proper API structure, type hints as contracts, middleware patterns.
Stack context: FastAPI backend for stock screener (AlphaStocks). SQLAlchemy 2.0 ORM with SQLite. JWT auth (python-jose) + Google OAuth + Magic Link. 47 endpoints across 19 route files. slowapi rate limiting. SQLAdmin panel. Pydantic for validation. pytest (174 tests).

Every finding must describe the **concrete failure mode** -- not just "bad practice."
Ramirez built FastAPI around a single conviction: Python type hints should be the single source of truth for validation, serialization, and documentation. When the code drifts from this principle, the auto-generated OpenAPI docs lie, validation gaps open, and the API becomes a hand-maintained contract that nobody maintains.

---

## Principle 1: Pydantic response models are contracts -- never return raw ORM objects

*Ramirez: "The response model is the most important part of the endpoint. It defines what the client can rely on."*

When an endpoint returns a SQLAlchemy model directly, the response shape is determined by whatever attributes the ORM loaded -- including relationships, internal fields, and columns the client should never see. A lazy-loaded relationship that works in dev (single request, session still open) returns `null` or raises `DetachedInstanceError` in production when the session is closed.

### What to check

**Endpoints missing `response_model`**
- Every router function must declare `response_model=SomeSchema`. Without it, FastAPI serializes whatever the function returns, bypassing Pydantic validation entirely. The OpenAPI spec shows `200: {}` -- useless for clients.
- If the endpoint returns a list, use `response_model=list[StockSchema]`, not `response_model=list`. The latter provides no schema.
- Severity: **P1** if the endpoint returns user data, auth tokens, or financial data without a response model. **P2** for other endpoints.

**ORM objects passed directly to the response**
- A function that does `return db_stock` instead of `return StockResponse.model_validate(db_stock)` relies on FastAPI's implicit `from_attributes` conversion. This works until someone adds a column to the ORM model that shouldn't be exposed (internal score weights, admin flags, soft-delete timestamps). The new column silently appears in every API response.
- Check: does the Pydantic schema use `model_config = ConfigDict(from_attributes=True)`? If not, ORM-to-schema conversion will fail silently or raise at runtime.
- Severity: **P2** for endpoints that return ORM objects where the schema and ORM model have diverged.

**Schema explosion without inheritance**
- 47 endpoints often need slight variations: `StockBrief`, `StockDetail`, `StockWithScores`, `StockAdmin`. Without a base schema and selective inheritance, you get 15 near-identical classes that drift independently. When a field name changes in the DB, half the schemas get updated and half don't.
- Use a base schema with shared fields and extend it. Use `Field(exclude=True)` or separate schemas for admin vs. public views.
- Severity: **P2** if there are more than 3 schemas for the same entity without a shared base.

---

## Principle 2: Dependency injection is the architecture -- not a convenience

*Ramirez: "Dependencies are the way FastAPI handles shared logic, auth, database sessions, and any reusable component."*

FastAPI's `Depends()` system replaces middleware, decorators, and manual function calls for cross-cutting concerns. When a codebase reaches 19 route files, consistency of dependency usage determines whether the auth layer has gaps.

### What to check

**DB session not injected via `Depends`**
- If any route creates its own `SessionLocal()` instead of using `Depends(get_db)`, that session won't be closed on exceptions, won't participate in request-scoped lifecycle, and won't be mockable in tests.
- The `get_db` dependency must use `yield` (not `return`) to guarantee `session.close()` runs even when the endpoint raises an exception.
- Severity: **P1** if any route manages its own session lifecycle. **P2** if `get_db` uses `return` instead of `yield`.

**Auth dependency not applied uniformly**
- With 19 route files and 47 endpoints, it's easy for one endpoint to miss `Depends(get_current_user)`. The endpoint works in dev (no auth check = no error) but exposes data to unauthenticated users in production.
- Prefer applying auth at the router level: `router = APIRouter(dependencies=[Depends(get_current_user)])` for routes that are entirely protected. Override at the endpoint level only for public routes.
- Check: is there a route that accepts user-specific data (watchlists, portfolios, settings) without an auth dependency?
- Severity: **P1** for any endpoint that reads or writes user-specific data without auth.

**Tier enforcement as a dependency**
- If the app has Free/Pro/Premium tiers, tier checks must be a dependency, not inline `if user.tier < required_tier` scattered across routes. A dependency like `Depends(require_tier(Tier.PRO))` is composable, testable, and impossible to forget.
- Check: are tier checks implemented consistently, or do some endpoints check inline while others use a dependency?
- Severity: **P2** if tier enforcement is inconsistent across endpoints.

**Nested dependencies not leveraging the cache**
- FastAPI caches dependency results within a single request by default. If `get_current_user` depends on `get_db`, and a route depends on both `get_current_user` and `get_db`, the DB session is created once, not twice. But if someone adds `use_cache=False` or restructures the dependency chain, the session can be created twice -- two connections, two potential transaction states.
- Severity: **P2** if dependencies are duplicated due to broken cache chains.

---

## Principle 3: Route organization must match the OpenAPI spec the client reads

*Ramirez: "Tags, prefixes, and response descriptions are not decoration -- they are the API documentation."*

19 route files generating 47 endpoints produce an OpenAPI spec that clients (frontend, mobile, third-party) consume directly. If the spec is messy, every consumer writes ad-hoc parsing code.

### What to check

**Missing or inconsistent `tags` on routers**
- Each `APIRouter` should have a `tags=["stocks"]` that groups its endpoints in the OpenAPI UI. Without tags, all 47 endpoints appear in a flat list in Swagger UI, making it unusable.
- Tags should match the resource, not the file name. `tags=["stock-routes"]` is a file system artifact; `tags=["Stocks"]` is an API concept.
- Severity: **P2** if endpoints appear untagged in the OpenAPI spec. **P3** if tags exist but are inconsistent in naming style.

**Prefixes that don't form clean REST paths**
- Router prefixes should be `/stocks`, `/watchlists`, `/auth` -- resource nouns, plural, lowercase. Not `/api/v1/getStockData` or `/stock_routes/list`. Every prefix becomes a permanent part of the client contract.
- Check: `app.include_router(stocks_router, prefix="/stocks")`. Are there routes with verb-based paths (`/getStock`, `/updateWatchlist`) instead of HTTP-method + resource-noun (`GET /stocks/{id}`, `PATCH /watchlists/{id}`)?
- Severity: **P2** for verb-based paths. **P3** for minor naming inconsistencies.

**Duplicate or shadowed routes**
- With 19 route files included in the same app, it's possible for two routers to register the same path. FastAPI matches the first registered route -- the second is silently unreachable. No warning is raised.
- Check: run the app and inspect `/openapi.json`. Are there paths that appear only once despite being defined in multiple files?
- Severity: **P1** if a shadowed route silently hides functionality.

**Missing `response_description` and status codes**
- Endpoints should declare `responses={404: {"description": "Stock not found"}}` for error cases. Without this, the OpenAPI spec shows only `200` for every endpoint, and clients have no contract for error handling.
- Check: do endpoints that raise `HTTPException(404)` also declare 404 in their response spec?
- Severity: **P3** for missing response descriptions. **P2** if clients are building against the OpenAPI spec and need accurate error contracts.

---

## Principle 4: HTTPException is the only error the client should see

*Ramirez: "FastAPI converts HTTPException to a clean JSON response. Any other exception becomes a 500 with no useful information."*

When an unhandled SQLAlchemy `IntegrityError` or a `KeyError` from business logic propagates to FastAPI, the client gets `{"detail": "Internal Server Error"}` with no context. The server logs the traceback, but the client can't distinguish a duplicate email from a database outage.

### What to check

**Raw exceptions leaking as 500s**
- Every code path that can raise a known exception (duplicate entry, not found, permission denied, validation failure) must be caught and re-raised as `HTTPException` with an appropriate status code and message.
- Check: are there `try/except` blocks in route handlers? If not, unhandled ORM exceptions will produce generic 500s.
- SQLAlchemy `IntegrityError` should become `409 Conflict`. `NoResultFound` should become `404 Not Found`.
- Severity: **P1** if user-facing operations (signup, watchlist creation) can return generic 500s for predictable failures.

**Inconsistent error response format**
- Some endpoints return `{"detail": "Not found"}`, others return `{"error": "not_found", "message": "Stock not found"}`, others return `{"msg": "error"}`. Clients must handle every variation.
- Define a single error schema and enforce it via a custom exception handler or a base `HTTPException` subclass.
- Severity: **P2** if error formats vary across endpoints. **P3** if the inconsistency is cosmetic (same structure, different wording).

**Exception handlers that swallow context**
- A global `@app.exception_handler(Exception)` that returns `JSONResponse(status_code=500, content={"detail": "Something went wrong"})` hides the actual error from logging if it doesn't also `log.exception()`. The 500 becomes un-debuggable.
- Check: does the global exception handler log the full traceback before returning the sanitized response?
- Severity: **P2** if exceptions are caught globally without logging.

**`detail` field leaking internal information**
- `HTTPException(status_code=400, detail=str(e))` where `e` is a SQLAlchemy error exposes table names, column names, and constraint names. The `detail` must be a human-readable message, never a raw exception string.
- Severity: **P1** if `detail` contains SQL fragments, table names, or stack traces.

---

## Principle 5: Query optimization -- SQLAlchemy makes N+1 invisible

*Ramirez: "SQLAlchemy is powerful, but it hides the SQL. You must know what queries your ORM is generating."*

With SQLite, N+1 queries are less catastrophic than with network-attached databases (no network round-trip per query). But they still multiply execution time linearly with result set size, and they become a migration blocker when the project moves to Postgres.

### What to check

**Lazy-loaded relationships accessed in loops**
- A query returns 50 stocks. The route handler iterates over them and accesses `stock.scores` for each. SQLAlchemy issues 50 separate `SELECT` queries for scores. The fix: `selectinload(Stock.scores)` or `joinedload(Stock.scores)` in the original query.
- Concrete detection: enable `echo=True` on the engine in test mode and count the queries per endpoint. An endpoint returning a list should issue a bounded number of queries, not O(N).
- Severity: **P1** if any list endpoint has confirmed N+1 behavior in production. **P2** if relationships exist but eager loading strategy is not explicitly chosen.

**Session scope extending beyond the request**
- If the `get_db` dependency yields a session that outlives the request (stored in a global, passed to a background task without its own session), subsequent requests see stale data or raise `DetachedInstanceError`.
- Background tasks must create their own sessions via `SessionLocal()`, not reuse the request session.
- Severity: **P1** if background tasks share the request session. **P2** if session lifecycle is unclear.

**Missing indexes on filter columns**
- The stock screener filters by sector, market_cap range, score range, and ticker. If these columns lack indexes, every filter query is a full table scan. SQLite performs table scans faster than Postgres, but at 10K+ rows, the difference becomes noticeable.
- Check: are columns used in `WHERE`, `ORDER BY`, and `JOIN` clauses indexed in the SQLAlchemy model definition?
- Severity: **P2** for unindexed columns used in user-facing filter queries. **P3** for admin-only queries.

**`SELECT *` when only a few columns are needed**
- `db.query(Stock).all()` loads every column of every row. If the endpoint only needs `ticker`, `name`, and `score`, this wastes memory and bandwidth. Use `db.query(Stock.ticker, Stock.name, Stock.score)` or load the full model but ensure the response schema filters it.
- Severity: **P3** for list endpoints. **P2** if the table has large text/blob columns being loaded unnecessarily.

---

## Principle 6: Input validation happens in Pydantic -- not in route logic

*Ramirez: "If you define the types correctly, FastAPI validates the input for you. You never write validation code in the route function."*

When validation logic lives in the route body (`if len(ticker) > 10: raise HTTPException(...)`) instead of the Pydantic schema (`ticker: str = Field(max_length=10)`), it doesn't appear in the OpenAPI spec, isn't reusable across endpoints, and is easy to forget in new routes.

### What to check

**Filter ranges validated inline instead of in schemas**
- A screener endpoint accepts `min_price`, `max_price`, `min_score`, `max_score`. If validation (`min_price >= 0`, `max_price > min_price`) happens in the route body, the OpenAPI spec shows no constraints, and every endpoint that reuses these parameters must duplicate the checks.
- Define a `ScreenerFilters` schema with `Field(ge=0)`, `@model_validator` for cross-field checks, and use it as a dependency or body parameter.
- Severity: **P2** if validation is duplicated across multiple routes. **P3** if it's in one route only.

**Pagination parameters without bounds**
- `skip: int = 0, limit: int = 100` without `Query(ge=0, le=1000)` allows a client to request `limit=999999`, dumping the entire database in one response. It also allows `skip=-1`, which may produce unexpected SQL behavior.
- Severity: **P1** if `limit` has no upper bound. **P2** if bounds exist but are inconsistent across endpoints.

**Ticker validation allowing arbitrary strings**
- A ticker parameter accepted as `ticker: str` allows SQL-injection-shaped strings, excessively long inputs, and meaningless queries. Tickers should be validated: uppercase, 1-5 characters, alphanumeric. Use `Field(pattern=r"^[A-Z]{1,5}$")` or a custom Pydantic type.
- Severity: **P2** if ticker parameters have no format validation. SQLAlchemy parameterizes queries, so SQL injection is unlikely, but garbage input wastes database round-trips.

**Enum parameters as raw strings**
- If `sector` or `sort_by` accept arbitrary strings, clients can send `sector=helloworld` and get an empty result with no error. Define enums in Pydantic: `sector: SectorEnum | None = None`. Invalid values produce a 422 with a clear message listing valid options.
- Severity: **P2** if string parameters should be constrained to a known set but aren't.

---

## Principle 7: Auth dependencies must form a strict hierarchy

*Ramirez: "Build your auth as layers of dependencies. Each layer adds a guarantee."*

JWT verification, user loading, tier checking, and admin verification are a chain: each step depends on the previous. When they're implemented as independent checks, it's possible to check the tier without verifying the JWT, or check admin status without loading the user.

### What to check

**Auth dependency chain**
- The correct chain: `get_token()` (extracts and verifies JWT) -> `get_current_user(token)` (loads user from DB) -> `require_tier(tier)(user)` (checks subscription level) -> `require_admin(user)` (checks admin flag).
- Each level should depend on the previous via `Depends()`. If `require_admin` accepts a raw token string and decodes it independently, JWT verification logic is duplicated and can diverge.
- Severity: **P1** if admin checks bypass JWT verification. **P2** if the chain exists but has redundant decoding.

**Token expiration not checked**
- `python-jose` decodes the token but does not automatically reject expired tokens unless `options={"verify_exp": True}` is set (it's the default, but worth verifying). If someone passes `options={"verify_exp": False}` to simplify testing and forgets to revert, expired tokens work forever.
- Check: is there a test that explicitly sends an expired token and asserts 401?
- Severity: **P1** if token expiration is not verified. **P2** if it's verified but not tested.

**Google OAuth token exchange without email verification**
- Google's OAuth returns an ID token with `email_verified: bool`. If the backend accepts any Google-authenticated email without checking this field, an attacker can sign up with Google using an unverified email and potentially access another user's account.
- Severity: **P1** if `email_verified` is not checked during Google OAuth flow.

**Magic link tokens stored without expiration or single-use enforcement**
- A magic link token that works forever or can be used multiple times is a persistent credential. Tokens must expire (15-30 minutes) and be invalidated after first use.
- Severity: **P1** if magic link tokens don't expire or aren't single-use.

---

## Principle 8: Background tasks must not trust the request context

*Ramirez: "BackgroundTasks run after the response is sent. The request is done. The database session is closed."*

FastAPI's `BackgroundTasks` execute after the response has been returned to the client. Any dependency that was injected via `Depends()` and uses `yield` has already been cleaned up.

### What to check

**Background task using the request's DB session**
- If a background task receives the `db` session from `Depends(get_db)`, that session is closed by the time the task runs. The task will raise `DetachedInstanceError` on any ORM access or silently use a stale session.
- Background tasks must create their own session: `with SessionLocal() as db:` inside the task function.
- Severity: **P1** for background tasks that accept `db` from the request dependency.

**Background task failure is silent**
- If a background task raises an exception, FastAPI logs it but the client has already received a 200. There's no retry, no alert, no dead-letter queue. For critical tasks (sending email, updating scores, triggering pipelines), silent failure means data loss.
- Check: do background tasks have try/except with logging? Is there a retry mechanism for critical tasks? Consider Celery or ARQ for tasks that must not be lost.
- Severity: **P2** for critical background tasks without error handling. **P3** for non-critical tasks.

**Background task blocking the event loop**
- If the app runs with `uvicorn` (async), a background task that performs synchronous I/O (file writes, network calls via `requests` instead of `httpx`) blocks the event loop, stalling all concurrent requests.
- Use `asyncio.to_thread()` for CPU-bound or synchronous-I/O work in background tasks, or use `run_in_executor`.
- Severity: **P2** if the app is async and background tasks use synchronous I/O.

---

## Principle 9: CORS must be applied before error responses -- not just success responses

*Ramirez: "CORS middleware must wrap everything, including exception handlers. If a 422 validation error doesn't have CORS headers, the browser blocks it and the client sees a network error instead of the validation message."*

This is a past bug in this codebase. The failure mode is insidious: everything works in development (same origin), and in production the frontend gets opaque `TypeError: Failed to fetch` on every validation error.

### What to check

**CORSMiddleware position in the middleware stack**
- `CORSMiddleware` must be the outermost middleware (last added, first to execute). If it's added before a custom error-handling middleware, errors raised by that middleware won't have CORS headers.
- Check: in the `main.py` or app factory, is `add_middleware(CORSMiddleware, ...)` the last middleware registration?
- Severity: **P1** if CORS headers are missing on 4xx/5xx responses in a cross-origin deployment.

**Wildcard origins in production**
- `allow_origins=["*"]` allows any domain to make authenticated requests to the API. With JWT in headers, this is less dangerous than with cookies, but it still enables third-party sites to probe the API using a user's browser.
- Production should list explicit origins: `["https://alphastocks.app", "https://www.alphastocks.app"]`.
- Severity: **P2** if origins are wildcarded in production. **P3** in development.

**CORS and credentials**
- If `allow_credentials=True`, `allow_origins` cannot be `["*"]` -- the CORS spec forbids it. The browser silently drops the `Access-Control-Allow-Origin` header. The symptom: authenticated requests fail in production with no useful error message.
- Severity: **P1** if `allow_credentials=True` and `allow_origins=["*"]` are set simultaneously.

---

## Principle 10: Rate limiting must distinguish authenticated from anonymous traffic

*Ramirez: "Rate limiting is a dependency. It should compose with auth, not fight it."*

`slowapi` provides rate limiting via decorators, but the default configuration uses client IP. Behind a reverse proxy or CDN, all clients share the same IP. Authenticated users should have higher limits than anonymous users, and rate-limit keys should be user IDs, not IPs.

### What to check

**Rate limit key function**
- Check the `key_func` used by slowapi. If it's `get_remote_address`, every user behind the same corporate NAT or Cloudflare IP shares one rate limit bucket. A single power user can exhaust the limit for everyone.
- For authenticated endpoints, key by `user.id`. For anonymous endpoints, key by IP but ensure `X-Forwarded-For` is parsed correctly (and not spoofable).
- Severity: **P2** if all endpoints use IP-based rate limiting. **P1** if the API is behind a proxy and `X-Forwarded-For` is not configured.

**Rate limit headers not returned**
- Clients need `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers to implement backoff. Without them, clients retry aggressively, amplifying the load.
- Check: does slowapi configuration include `headers_enabled=True`?
- Severity: **P3** for missing rate limit headers.

**Rate limit on write vs. read endpoints**
- Login attempts, magic link requests, and signup should have aggressive rate limits (5-10/minute). Read-only endpoints like stock screener queries can be more generous (100-200/minute). A single limit applied everywhere is either too tight for reads or too loose for auth.
- Severity: **P2** if auth endpoints have the same rate limit as data endpoints.

---

## Principle 11: Response caching must invalidate correctly -- stale data is a bug

*Ramirez: "Caching is easy. Cache invalidation is the only hard problem."*

ETag middleware caches responses based on content hash. This works for data that changes on known schedules (daily score updates) but fails for data that changes on user action (watchlist modifications, settings changes).

### What to check

**ETag on user-specific endpoints**
- If the ETag middleware applies to `/watchlists` and the user adds a stock, the next request may return a cached 304 with the old watchlist. ETags must be scoped per-user (include `user_id` in the hash) and invalidated on writes.
- Check: does the ETag middleware distinguish between per-user and global responses?
- Severity: **P1** if user-specific endpoints return stale cached data after mutations.

**Cache-Control headers on dynamic data**
- Endpoints returning real-time or frequently updated data (current prices, live scores) should set `Cache-Control: no-store` or `max-age=0`. Without it, intermediate caches (CDNs, browser) may serve stale data.
- Check: do endpoints serving time-sensitive data set appropriate `Cache-Control` headers?
- Severity: **P2** if price or score data can be cached by intermediaries.

**Background task updates not clearing the cache**
- If a daily batch pipeline updates scores and the ETag cache is computed from the old response body, the cache serves stale scores until the hash naturally changes on the next different request.
- Check: does the batch pipeline trigger a cache clear or version bump?
- Severity: **P2** if batch updates don't invalidate cached responses.

---

## Principle 12: Middleware ordering determines behavior -- order is not optional

*Ramirez: "Middleware in FastAPI executes in reverse order of addition. The last middleware added is the first to process the request."*

This is counterintuitive and leads to subtle bugs. The middleware stack for auth, rate limiting, CORS, and caching must execute in a specific order.

### What to check

**Correct middleware execution order**
- The request should flow through: CORS (add headers) -> Rate Limit (reject excess) -> Auth (verify identity) -> Caching (check ETag) -> Route handler.
- In FastAPI, this means adding in reverse: first the route handler (implicit), then `CacheMiddleware`, then auth middleware (if implemented as middleware rather than dependency), then rate limiting, then `CORSMiddleware` last.
- Check: trace the middleware stack in `main.py`. Is the order intentional and documented?
- Severity: **P1** if CORS runs after error-producing middleware (see Principle 9). **P2** for other ordering issues.

**Auth as middleware vs. dependency**
- Auth can be a middleware (runs on every request) or a dependency (runs on routes that declare it). With 19 route files, some containing public endpoints and some private, middleware-based auth requires a skip-list for public routes -- which is fragile. Dependency-based auth is explicit per-route.
- Check: if auth is implemented as middleware, is the skip-list for public routes complete and tested?
- Severity: **P2** if middleware-based auth silently blocks public endpoints or silently allows private ones.

**Middleware that catches exceptions and changes response codes**
- A middleware that wraps the endpoint in `try/except Exception` and returns a 500 response will prevent FastAPI's own exception handlers from running. This breaks `HTTPException` handling, validation error formatting, and CORS header injection.
- Severity: **P1** if custom middleware intercepts exceptions that FastAPI should handle.

---

## Principle 13: Async endpoints must be async all the way down

*Ramirez: "If you define an endpoint as async def, every I/O call inside must be awaited. A synchronous call inside an async endpoint blocks the event loop."*

FastAPI runs `async def` endpoints on the main event loop and `def` endpoints in a threadpool. Mixing sync I/O inside an async endpoint freezes all concurrent requests.

### What to check

**Synchronous ORM calls inside `async def` endpoints**
- SQLAlchemy 2.0 supports async sessions, but if the project uses synchronous `Session`, all DB calls are blocking. An `async def` endpoint calling `db.query(Stock).all()` blocks the event loop for the duration of the query.
- Fix: either use `def` endpoints (FastAPI runs them in a threadpool) or use SQLAlchemy's async engine with `AsyncSession`.
- Check: if any endpoint is declared `async def`, are all its I/O calls non-blocking?
- Severity: **P1** if `async def` endpoints make synchronous DB calls under load. **P2** for low-traffic endpoints.

**`requests` library used inside async endpoints**
- The `requests` library is synchronous. Using it inside an `async def` endpoint blocks the event loop. Use `httpx.AsyncClient` instead.
- Check: is `import requests` present in any module that defines `async def` route handlers?
- Severity: **P2** if sync HTTP calls exist inside async endpoints.

**File I/O without `aiofiles` in async endpoints**
- `open()` and `write()` are synchronous. In an async endpoint handling file uploads or OG image generation, these block the event loop.
- Use `aiofiles` for file operations in async contexts, or declare the endpoint as `def`.
- Severity: **P2** for file operations in async endpoints.

---

## Principle 14: Environment configuration must fail fast on missing secrets

*Ramirez: "Use Pydantic Settings. It validates all environment variables at startup, not when the first request hits a missing key."*

With 20+ environment variables (JWT secret, Google OAuth credentials, database URL, email service keys, admin passwords), a missing variable discovered at runtime produces an opaque error for the user who triggered the code path.

### What to check

**Settings class using `pydantic-settings`**
- All environment variables should be declared in a `Settings(BaseSettings)` class with types and defaults. The class is instantiated at import time -- missing required variables crash the app at startup, not at the first request that needs them.
- Check: is there a centralized settings class, or are `os.getenv()` calls scattered across modules?
- Severity: **P2** if environment variables are read with `os.getenv()` in route handlers. **P1** if a missing env var causes a runtime crash in production that could have been caught at startup.

**Secrets with default values**
- `JWT_SECRET: str = "changeme"` is a default that will silently be used in production if the env var isn't set. Secrets must be required (no default) so that a missing value prevents startup.
- Severity: **P1** if any secret (JWT key, OAuth client secret, database password) has a non-empty default.

**No `.env.example` file**
- With 20+ env vars, a new developer (or a fresh deployment) has no way to know what's needed. An `.env.example` with every variable listed (values blanked or placeholder) is the contract.
- Severity: **P3** for missing `.env.example`. **P2** if the deployment fails silently due to missing variables and there's no documentation.

**SQLite path hardcoded vs. configurable**
- If the database path is hardcoded (`sqlite:///./app.db`), tests and production share the same file. The path must come from settings, with tests overriding to `:memory:` or a temporary file.
- Severity: **P2** if the DB path is hardcoded.

---

## Principle 15: Testing must use dependency overrides -- not monkeypatching

*Ramirez: "FastAPI's dependency override system exists specifically for testing. It replaces dependencies cleanly without touching production code."*

174 tests must run in isolation with predictable data. FastAPI's `app.dependency_overrides[get_db] = override_get_db` is the intended mechanism for swapping the real DB, auth, and external services in tests.

### What to check

**Tests using `app.dependency_overrides` for DB**
- The test DB session should be an in-memory SQLite with tables created fresh per test (or per module). The override replaces `get_db` with a function yielding the test session.
- Check: is `app.dependency_overrides[get_db]` set in conftest? If not, tests may hit the real database.
- Severity: **P1** if tests can modify production data. **P2** if tests use a separate file-based DB that isn't cleaned between runs.

**Auth overrides for testing protected endpoints**
- Testing a protected endpoint should override `get_current_user` to return a known test user, not create a real JWT and pass it in headers. The latter tests JWT decoding, not the endpoint logic.
- When JWT decoding itself needs testing, those tests should be separate and explicit.
- Severity: **P2** if auth testing is tangled with endpoint testing (failures in JWT logic break unrelated tests).

**Test isolation -- database state leaking between tests**
- If tests share a DB session and one test creates a stock, subsequent tests see that stock. Use transactions with rollback per test: begin a transaction before each test, roll back after.
- Check: does the test session use `begin_nested()` and `rollback()` to isolate tests?
- Severity: **P2** if test order affects test results (flaky tests).

**No tests for error paths**
- With 174 tests across 47 endpoints, the ratio is ~3.7 tests per endpoint. If most tests cover the happy path, error paths (invalid input, unauthorized, duplicate entry, not found) are untested. The error responses are part of the contract.
- Check: for critical endpoints (auth, watchlist mutations, screener), is there at least one test for each expected error code?
- Severity: **P2** if error paths are untested for auth endpoints. **P3** for other endpoints.

---

## Principle 16: File upload and generation must not block or leak

*Ramirez: "File operations should use UploadFile, which spools to disk automatically for large files. Never read the entire file into memory."*

OG image generation and file uploads require careful resource management. A leaked file handle or an in-memory buffer of a large file can exhaust server resources.

### What to check

**UploadFile not read into memory**
- `contents = await file.read()` loads the entire file into memory. For large files, this causes OOM. Use streaming: `async for chunk in file` or configure `SpooledTemporaryFile` size limits.
- Check: if any endpoint accepts `UploadFile`, how is the content consumed?
- Severity: **P2** if file content is fully loaded into memory without size limits. **P1** if the endpoint is publicly accessible (DoS vector).

**OG image generation blocking the event loop**
- Image generation (Pillow, Cairo) is CPU-bound and synchronous. If the endpoint is `async def`, it blocks the event loop. Run image generation in a threadpool via `asyncio.to_thread()` or declare the endpoint as `def`.
- Severity: **P2** if OG image generation runs synchronously inside an async endpoint.

**Generated files not cleaned up**
- If OG images are written to disk and served via `FileResponse`, they must be cleaned up after serving. A cron-less server accumulates files indefinitely.
- Use `background` parameter of `FileResponse` to delete after sending, or generate in-memory with `BytesIO` and return `StreamingResponse`.
- Severity: **P3** for missing cleanup of generated files. **P2** if the volume is high enough to fill disk.

---

## Principle 17: SQLAdmin must not bypass application security

*Ramirez: "Admin panels are powerful tools. They need their own auth layer, independent of the API."*

SQLAdmin provides a full CRUD interface to the database. If its auth is weaker than the API's, it becomes a backdoor.

### What to check

**SQLAdmin auth independent of API JWT**
- SQLAdmin typically uses session-based auth, not the same JWT the API uses. If the admin panel's login is a simple username/password without rate limiting, it's brute-forceable.
- Check: does the SQLAdmin auth backend enforce rate limiting, strong passwords, and session expiration?
- Severity: **P1** if SQLAdmin has no authentication or weak authentication. **P2** if auth exists but lacks rate limiting.

**SQLAdmin accessible in production without IP restriction**
- The admin panel should be accessible only from whitelisted IPs or behind a VPN. Exposing it publicly means any discovered auth vulnerability gives full database access.
- Check: is the `/admin` route protected by IP filtering or network-level access control?
- Severity: **P2** if SQLAdmin is publicly accessible in production.

**SQLAdmin actions not audited**
- Changes made through SQLAdmin bypass the application's business logic (no Pydantic validation, no event triggers, no audit logging). An admin who changes a user's tier through SQLAdmin won't trigger the application's tier-change logic.
- Check: are SQLAdmin modifications logged? Is there a risk of data inconsistency from direct DB edits?
- Severity: **P2** for missing audit logs on admin actions. **P3** if SQLAdmin is read-only.

---

## Principle 18: The daily batch pipeline must be idempotent

*Ramirez: "If it runs twice, the result should be the same. If it fails halfway, you should be able to restart it."*

A daily pipeline that updates scores, fetches new financial data, and sends emails will inevitably be run twice (operator error, cron overlap, crash-and-restart). If it's not idempotent, the second run corrupts data.

### What to check

**Score updates are upserts, not inserts**
- If the pipeline inserts new score records without checking for existing ones, a double-run creates duplicate entries. Use `INSERT ... ON CONFLICT UPDATE` (SQLAlchemy's `merge()` or an explicit upsert pattern).
- Check: what happens if the pipeline runs twice in the same day?
- Severity: **P1** if double-runs create duplicate data that affects user-visible scores.

**Email sending is not guarded by a sent flag**
- If the pipeline sends daily digest emails and is run twice, users receive duplicate emails. Track sent status per user per day and check before sending.
- Severity: **P2** if emails can be sent in duplicate.

**Pipeline failure leaves partial state**
- If the pipeline updates stocks A-M and crashes on N, is there a way to resume from N? Or does the next run re-process A-M (wasting API calls and time)?
- Use a progress marker (last processed ID, checkpoint timestamp) to enable resumption.
- Severity: **P2** if the pipeline cannot resume from failure. **P3** if the full re-run is fast enough to be acceptable.

**Pipeline holding a DB session for the entire run**
- A batch pipeline that opens one session, processes 5000 stocks, and commits at the end holds a long-running transaction. In SQLite, this blocks all writes from the API for the duration.
- Process in batches: commit every N records. Use WAL mode for SQLite to allow concurrent reads during writes.
- Severity: **P2** if the pipeline blocks API writes. **P1** if the pipeline runs during business hours and causes user-visible timeouts.

---

## Principle 19: OpenAPI spec is a deployment artifact -- it must be versioned

*Ramirez: "FastAPI generates the OpenAPI spec automatically. But automatic doesn't mean unmanaged."*

The OpenAPI spec at `/openapi.json` is what code generators, frontend clients, and third-party integrations consume. A change to a response model or a renamed field silently breaks every consumer.

### What to check

**Breaking changes without version bump**
- Removing a field from a response model, renaming a field, or changing a field's type is a breaking change. Clients that depend on the field will fail.
- Check: is there a mechanism to detect breaking changes? Even a simple approach: snapshot the OpenAPI JSON in version control and diff it on PRs.
- Severity: **P2** if there's no process for detecting breaking changes. **P1** if the API has external consumers.

**Inconsistent field naming conventions**
- Some schemas use `snake_case` (`market_cap`), others use `camelCase` (`marketCap`). With Pydantic v2, configure `model_config = ConfigDict(alias_generator=to_camel)` and `populate_by_name=True` globally to enforce consistency.
- Check: are field names in the OpenAPI spec consistent in casing?
- Severity: **P3** for internal-only APIs. **P2** for APIs with external consumers.

**Missing descriptions on schema fields**
- `Field(description="...")` populates the OpenAPI spec with human-readable descriptions. Without them, `score: float` in the spec tells the consumer nothing about range, units, or meaning.
- Check: do schemas for complex financial data include descriptions for non-obvious fields (score range, currency, percentage vs. ratio)?
- Severity: **P3** for most fields. **P2** for financial data fields where misinterpretation causes errors.

---

## Principle 20: Startup and shutdown events must manage resources explicitly

*Ramirez: "Use lifespan context managers. They guarantee cleanup even on shutdown signals."*

Database connections, background schedulers, and HTTP clients opened at startup must be closed on shutdown. FastAPI's `lifespan` context manager (replacing deprecated `on_event`) is the correct pattern.

### What to check

**Using `lifespan` instead of `@app.on_event`**
- `@app.on_event("startup")` and `@app.on_event("shutdown")` are deprecated in newer FastAPI. They also don't guarantee shutdown runs if the process receives SIGKILL. The `lifespan` async context manager ensures cleanup via `finally`.
- Severity: **P3** for using deprecated `on_event` (functional but not future-proof). **P2** if shutdown cleanup is missing entirely.

**Database engine disposed on shutdown**
- If the SQLAlchemy engine isn't disposed on shutdown, open connections linger. In SQLite, this can leave the database locked.
- Check: does the lifespan's teardown call `engine.dispose()`?
- Severity: **P2** for missing engine disposal.

**Background scheduler stopped on shutdown**
- If a scheduler (APScheduler, custom cron) runs background tasks, it must be stopped on shutdown. A scheduler that continues after the app is "stopped" can cause database corruption (writes after connections are closed).
- Severity: **P2** if the scheduler isn't tied to the app's lifecycle.

---

## Principle 21: Path parameters must be typed -- string IDs are a trap

*Ramirez: "Use type annotations for path parameters. FastAPI converts and validates them automatically."*

A path parameter declared as `stock_id: str` accepts anything: UUIDs, integers, emoji, empty strings. If the database uses integer primary keys, the SQLAlchemy query `db.get(Stock, "abc")` may raise, return None unexpectedly, or in some drivers, cast silently.

### What to check

**Path parameters matching DB key types**
- If the primary key is `int`, the path parameter must be `stock_id: int`. FastAPI returns 422 automatically for non-integer values, before the route logic even runs.
- Check: do path parameters use the correct Python type matching the database schema?
- Severity: **P2** for type mismatches between path params and DB keys.

**UUID path parameters without validation**
- If using UUIDs, `stock_id: UUID` (from `uuid` module) validates format automatically. `stock_id: str` allows `"not-a-uuid"` to reach the DB layer.
- Severity: **P2** for UUID fields accepted as unvalidated strings.

**Composite lookups via path vs. query**
- `/stocks/{ticker}` is cleaner than `/stocks?ticker=AAPL` for single-resource lookups. But if `ticker` is not unique in the DB, the path-based route must handle duplicates (return 409 or choose a resolution strategy).
- Severity: **P3** for style preferences. **P2** if a path-based lookup silently returns arbitrary results for non-unique keys.

---

## Principle 22: Status codes are part of the contract -- use them precisely

*Ramirez: "FastAPI lets you declare status codes explicitly. Use the right one -- clients build logic around them."*

Returning 200 for everything (including errors encoded in the body) is an anti-pattern that prevents clients from using HTTP-level error handling. Returning 400 for everything that isn't 200 prevents clients from distinguishing between bad input and server errors.

### What to check

**`201 Created` for successful POST that creates a resource**
- `@router.post("/watchlists", status_code=201)` signals to clients that the resource was created. If it returns 200, clients can't distinguish "created" from "already existed."
- Check: do POST endpoints that create resources return 201?
- Severity: **P3** for wrong-but-functional status codes. **P2** if the API documentation claims REST compliance.

**`204 No Content` for successful DELETE**
- A DELETE endpoint that returns the deleted object with 200 is acceptable but chatty. If it returns nothing, it should be 204, not 200 with an empty body.
- Severity: **P3**

**`409 Conflict` for duplicate entries**
- When a user tries to add a stock that's already in their watchlist, 409 is more informative than 400 (which means "your request was malformed" -- it wasn't, it was valid but conflicting).
- Check: are duplicate-entry errors returned as 409 or as generic 400?
- Severity: **P3** for incorrect error codes. **P2** if the client distinguishes between 400 and 409 for UX purposes.

**`422 Unprocessable Entity` only for Pydantic validation**
- FastAPI uses 422 for request validation failures (Pydantic). Custom business logic validation (e.g., "score must be positive") should use 400, not 422 -- to help clients distinguish "malformed request" from "valid request, invalid business state."
- Severity: **P3**

---

## Principle 23: Logging must be structured and must never contain secrets

*Ramirez: "Use Python's logging module. Structure your logs. Never log tokens, passwords, or user data."*

When debugging a production issue across 47 endpoints, unstructured print statements are unsearchable. And a log line containing a JWT or an email address is a compliance violation.

### What to check

**`print()` in production code**
- `print()` is not logging. It doesn't include timestamps, levels, or structured data. It can't be filtered, routed, or suppressed.
- Check: are there `print()` statements in route handlers, dependencies, or middleware?
- Severity: **P2** for `print()` in production code paths. **P3** for print in scripts or CLI tools.

**Secrets in log output**
- `logger.info(f"User logged in with token {token}")` logs the JWT in plaintext. `logger.debug(f"Request body: {body}")` may log passwords, API keys, or PII.
- Check: do log statements include any of: `token`, `password`, `secret`, `key`, `authorization`, or raw request bodies?
- Severity: **P1** for logging tokens or passwords. **P2** for logging PII (emails, names).

**Request ID for tracing**
- With 47 endpoints serving concurrent requests, correlating logs to a specific request requires a request ID. Add a middleware that generates a UUID per request and attaches it to the logger context.
- Check: can you trace all log entries for a single request?
- Severity: **P3** for missing request IDs. **P2** if production debugging is regularly hampered by inability to trace requests.

---

## Principle 24: Pydantic v2 migration traps -- the silent behavior changes

*Ramirez: "Pydantic v2 is faster but has breaking changes. The migration guide lists them, but the subtle ones are the dangerous ones."*

If the codebase migrated from Pydantic v1 to v2 (or uses v2 from the start but follows v1 tutorials), several behavior changes can cause silent data corruption.

### What to check

**`orm_mode` renamed to `from_attributes`**
- Pydantic v1: `class Config: orm_mode = True`. Pydantic v2: `model_config = ConfigDict(from_attributes=True)`. If the old syntax is used with v2, it's silently ignored -- ORM-to-schema conversion raises `ValidationError` at runtime.
- Severity: **P1** if any schema uses the v1 `orm_mode` config with Pydantic v2.

**`.dict()` replaced by `.model_dump()`**
- Pydantic v1: `schema.dict()`. Pydantic v2: `schema.model_dump()`. `.dict()` still works in v2 but is deprecated and may be removed. More critically, `.dict()` in v2 may handle nested models differently than v1.
- Severity: **P3** for using deprecated `.dict()`. **P2** if the behavior differs for nested models in the codebase.

**`validator` replaced by `field_validator`**
- Pydantic v1's `@validator` has different method signature expectations than v2's `@field_validator`. Mixing them causes runtime errors or silent validation skips.
- Check: are all validators using v2 syntax (`@field_validator`, `@model_validator`)?
- Severity: **P2** if v1-style validators are present in a v2 codebase.

**Default serialization of `datetime`**
- Pydantic v2 serializes `datetime` to ISO 8601 strings by default. V1 serialized to the default `str()` representation. If clients parse the datetime string, the format change breaks them.
- Severity: **P2** if clients depend on a specific datetime format and it changed during migration.

---

## Principle 25: Health check and readiness probes must verify actual dependencies

*Ramirez: "A health endpoint that returns 200 without checking the database is lying."*

A `/health` endpoint that returns `{"status": "ok"}` unconditionally tells the load balancer the app is healthy even when the database is unreachable, the batch pipeline has crashed, or a required external service is down.

### What to check

**Health endpoint checks database connectivity**
- The health endpoint should execute a trivial query (`SELECT 1`) and return 500 if it fails. This catches: SQLite file locked, connection pool exhausted, database file missing.
- Severity: **P2** if `/health` doesn't check the database.

**Readiness vs. liveness separation**
- Liveness: "is the process running?" (always return 200 unless the process should be restarted). Readiness: "can the process serve traffic?" (check DB, check required services). Kubernetes and most orchestrators distinguish these.
- If only one endpoint exists, it should be the readiness probe (checks dependencies), not just liveness.
- Severity: **P3** for missing separation. **P2** if the app is deployed in an orchestrated environment.

**Health endpoint not rate-limited but not expensive**
- Load balancers hit `/health` every 5-30 seconds. If the health check runs a complex query, it becomes a constant load source. `SELECT 1` or equivalent is sufficient.
- Check: is the health check lightweight? Does it avoid full table scans or expensive joins?
- Severity: **P3** for expensive health checks.

---

## Quick Reference: Severity Guide

| Severity | Pattern | Examples |
|----------|---------|----------|
| **P1 -- Fix Now** | Security hole, data leak, silent data corruption, or production crash | Missing auth on user endpoints, secrets with defaults, CORS blocking error responses, logging tokens, N+1 confirmed in production, background task using closed session, admin panel without auth |
| **P2 -- Fix Soon** | Contract violation, inconsistency, or degraded reliability | Missing response models, inline validation instead of schemas, no dependency overrides in tests, rate limiting by IP behind proxy, no cache invalidation, middleware ordering bugs, unstructured logging |
| **P3 -- Improve** | Style, documentation, or minor optimization | Missing field descriptions, deprecated Pydantic syntax, wrong status codes that still work, missing `.env.example`, health check without dependency verification |

### The Overriding Filter

Before writing any finding, apply the Ramirez synthesis:

1. **Does the OpenAPI spec match reality?** If the auto-generated spec lies about what the endpoint accepts or returns, every consumer is building on a false contract. This is always P2 or higher.
2. **Is validation happening in the right layer?** If Pydantic schemas don't enforce it, it's not validated -- it's hoped. Type hints that aren't enforced at runtime are documentation, not contracts.
3. **Is the dependency chain unbroken?** If any cross-cutting concern (auth, DB session, rate limiting, tier check) is implemented outside the dependency system, it will be forgotten in at least one of 47 endpoints.
4. **Would this survive a second developer?** If the pattern requires reading the original developer's mind to use correctly (which middleware goes first? which endpoints need auth? which schemas include sensitive fields?), it will break when a second person touches the code.
5. **Is the error response as useful as the success response?** Clients spend more time handling errors than successes. A 500 with no context is a support ticket. A 409 with `{"detail": "Stock AAPL is already in your watchlist"}` is self-service.
