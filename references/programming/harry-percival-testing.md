# Python TDD & Testing Quality Reference — Harry Percival

Philosophy: Harry Percival. Author of "Test-Driven Development with Python" (O'Reilly, "Obey the Testing Goat!") and "Architecture Patterns with Python" (with Bob Gregory). Authority on TDD methodology, test architecture, and pragmatic testing in Python web applications. Core doctrine: "Write the test first. Watch it fail. Write the minimum code to make it pass. Refactor. Repeat."
Stack context: Testing for Dense Club fitness SaaS. pytest 8.x + pytest-asyncio (asyncio_mode="auto"). httpx AsyncClient with ASGITransport for async API testing. PostgreSQL (real DB, not mocks) for integration tests. factory-boy + faker for fixtures. pytest-cov for coverage. syrupy for snapshot tests. Target: 85% coverage on business logic (workout resolution, PR detection, trial expiry, ranking calculation), 60% on route handlers.

Every finding must describe the **concrete failure mode** — not just "insufficient testing."

---

## Principle 1: Test the behavior, not the implementation

*Percival: "A test that mirrors the implementation is just the code written twice. It adds no value and breaks when you refactor."*

### What to check

**Assert outcomes, not method calls**
- PR detection test should assert: "when athlete logs 80kg and previous PR was 77.5kg → a new PR record exists AND an activity event was created." NOT: "session.add was called with PersonalRecord(pr_value=80) AND session.add was called with ActivityEvent(type='new_pr')."
- The first test survives refactoring (switch from session.add to bulk_insert, change the service function signature). The second breaks on any implementation change.
- Failure mode: tests pass but production is broken because the mock asserts matched the old implementation that no longer exists.
- Severity: **P1** if business logic tests mock the database and assert on ORM calls.

**Test the API contract, not internal functions**
- For route tests: `response = await client.post("/api/athlete/performances", json={...})` → assert `response.status_code == 201` and `response.json()["pr_detected"] == True`. This tests the entire stack: validation, auth, service logic, DB, response serialization.
- Testing internal `_detect_pr(session, user_id, exercise_id, value)` directly is acceptable for complex logic but doesn't catch integration bugs (missing auth, wrong response schema, transaction boundaries).
- Severity: **P2** if there are no API-level integration tests for critical paths.

---

## Principle 2: Integration tests for the hot path, unit tests for logic

*Percival: "If it touches the database, test it with the database."*

### What to check

**Performance logging path tested against real PostgreSQL**
- The hot write path (INSERT performance → tonnage → PR upsert → activity_event → ranking invalidation) MUST be tested against real PostgreSQL. Mocking the DB here hides:
  - Incorrect `ON CONFLICT` behavior (the `WHERE EXCLUDED.pr_value > personal_records.pr_value` clause)
  - Transaction isolation bugs (concurrent athlete logs)
  - asyncpg-specific behavior (binary protocol, type coercion)
  - Trigger or constraint side effects
- Severity: **P1** if the hot write path only has mocked tests.

**Unit test pure functions**
- Tonnage calculation: `calculate_tonnage(weight_kg=80, bodyweight_kg=75, contribution_pct=0.85, total_reps=5)` → pure math, no DB needed. Unit test with edge cases: contribution_pct=0 (bench), contribution_pct=1.0 (pull-up), None bodyweight.
- Dense scheme parsing: `parse_dense_scheme("10D5")` → `(10, 5)`. Test valid patterns, invalid patterns, edge cases.
- Workout resolution logic: given assignments + overrides → resolved exercise list. Use snapshot tests (syrupy) for the output shape.
- Severity: **P2** if pure functions lack unit tests.

**factory-boy factories for every model**
```python
class UserFactory(factory.Factory):
    class Meta:
        model = User
    id = factory.LazyFunction(uuid4)
    email = factory.Faker("email")
    role = "athlete_standard"
    hashed_password = factory.LazyFunction(lambda: hash_password("testpass123"))

class ExerciseFactory(factory.Factory):
    class Meta:
        model = Exercise
    name = factory.Faker("word")
    nature = "barbell"
    bodyweight_contribution_pct = Decimal("0")

class PerformanceFactory(factory.Factory):
    class Meta:
        model = Performance
    user_id = factory.LazyAttribute(lambda o: o.user.id if o.user else uuid4())
    exercise_id = factory.LazyAttribute(lambda o: o.exercise.id if o.exercise else uuid4())
    dense_scheme = "10D5"
    weight_kg = Decimal("80.00")
```
- Without factories: every test creates models with 15 fields inline. One field changes → update 50 tests. Factories set sensible defaults; tests override only what they care about.
- Severity: **P2** if test setup is copy-pasted across tests.

---

## Principle 3: Fixture strategy — truncate, don't recreate

*Percival: "Tests must be independent. Each test gets a clean slate."*

### What to check

**TRUNCATE ... RESTART IDENTITY CASCADE per test module**
```python
@pytest.fixture(autouse=True)
async def clean_db(session):
    yield
    for table in reversed(Base.metadata.sorted_tables):
        await session.execute(text(f"TRUNCATE TABLE {table.name} RESTART IDENTITY CASCADE"))
    await session.commit()
```
- Faster than DROP/CREATE database. Resets auto-increment IDs (important for deterministic tests). CASCADE handles FK dependencies.
- Severity: **P2** if tests share state (test B depends on data created by test A).

**Transaction rollback is unreliable with asyncpg**
- The common pattern (wrap each test in a transaction, rollback at end) has edge cases with asyncpg + pytest-asyncio: nested transactions (SAVEPOINTs) don't always rollback correctly, DDL statements autocommit, and async session lifecycle interacts with pytest's fixture teardown order.
- TRUNCATE is simpler and more reliable. Slightly slower but not enough to matter at Dense Club's test suite size.
- Severity: **P2** if using transaction rollback and seeing intermittent test failures.

**Module-scoped fixtures for static data**
- Exercise definitions (148 exercises) and skill definitions (110 skills) don't change per test. Load them once per module: `@pytest.fixture(scope="module")`. Re-creating them per test wastes ~200ms per test × 100 tests = 20 seconds.
- Severity: **P3** — performance optimization.

---

## Principle 4: Snapshot tests for complex response shapes

*Percival: "When the output is too complex to assert field by field, snapshot it."*

### What to check

**Workout resolution output**
- `resolve_workout(assignment, date, overrides)` returns a complex structure: list of exercises with resolved schemes, weights, override markers, micro program interleaving. Asserting 30+ fields individually is fragile and unreadable.
- Use syrupy: `assert resolved_workout == snapshot`. On first run, syrupy saves the output as `.ambr` file. On subsequent runs, it compares. When the algorithm changes, `pytest --snapshot-update` regenerates.
- The snapshot diff in code review shows exactly what changed — much clearer than "AssertionError: 80 != 85."
- Severity: **P2** for workout resolution. **P3** for simpler outputs.

**Snapshot tests are not a substitute for behavior tests**
- A snapshot that says "the output is this JSON blob" doesn't explain WHY it should be that blob. Pair snapshot tests with named behavior tests: `test_override_removes_exercise_from_workout`, `test_micro_program_interleaved_by_sort_order`.
- Severity: **P2** if snapshot tests are the ONLY tests for complex logic.

---

## Principle 5: Test the auth boundaries

*Percival: "The auth test matrix is the most important table in your test suite."*

Every endpoint needs at minimum: (1) 401 without token, (2) 403 with wrong role, (3) 200/201 with correct role. Coach endpoints add: (4) 403 when coach accesses unassigned athlete.

### What to check

**Parametrize the auth matrix**
```python
@pytest.mark.parametrize("role,expected_status", [
    ("athlete_standard", 403),
    ("athlete_trial", 403),
    ("coach", 200),
    ("admin", 200),
])
async def test_get_coach_athletes(client, role, expected_status, make_token):
    token = make_token(role=role)
    response = await client.get("/api/coach/athletes", headers=auth_header(token))
    assert response.status_code == expected_status
```
- These tests catch the "forgot `Depends(require_role)`" bug. Without them: one endpoint accidentally becomes public, discovered by a user, reported months later.
- Severity: **P1** for coach and admin endpoints. **P2** for athlete endpoints.

**Coach-athlete boundary tests**
- Coach A requests athlete B's data (B is assigned to coach C) → must return 403. This is a cross-tenant data leak test.
  ```python
  async def test_coach_cannot_access_unassigned_athlete(client, coach_a_token, athlete_b):
      response = await client.get(
          f"/api/coach/athletes/{athlete_b.id}/performances",
          headers=auth_header(coach_a_token)
      )
      assert response.status_code == 403
  ```
- Severity: **P1** if cross-coach boundary tests don't exist.

**Expired trial tests**
- `athlete_trial` with expired `trial_ends_at` → 403 on protected endpoints, 200 on profile and payment page. Parametrize with both valid and expired trial states.
- Severity: **P1** if trial expiry is not tested.

---

## Principle 6: Coverage targets are floors, not ceilings

*Percival: "100% coverage is a vanity metric. 85% coverage on the right code is worth more than 100% on everything."*

### What to check

**Coverage by directory**
| Directory | Target | Why |
|---|---|---|
| `app/services/` | 85% | Business logic: PR detection, tonnage, workout resolution, rankings |
| `app/core/auth.py` | 100% | Security-critical: any untested path is a potential bypass |
| `app/routes/` | 60% | Route handlers delegate to services; test the happy path + auth |
| `app/models/` | N/A | Models are data classes; testing them directly adds no value |
| `app/core/config.py` | N/A | Config loading; tested implicitly by app startup |

- Severity: **P2** if `app/services/` is below 70%. **P1** if `app/core/auth.py` is below 90%.

**pragma: no cover for genuinely unreachable code**
- Acceptable: error handlers that catch "impossible" conditions (e.g., `except IntegrityError` after a check that guarantees uniqueness). Use `# pragma: no cover` so they don't drag coverage down.
- Not acceptable: using `pragma: no cover` to hide untested business logic.
- Severity: **P3** if `pragma: no cover` is overused.

**CI enforces coverage minimums**
- `pytest --cov=app --cov-fail-under=75` in CI pipeline. Prevents gradual coverage erosion. Start at 75%, raise to 80% after initial development stabilizes.
- Severity: **P2** if CI doesn't check coverage.
