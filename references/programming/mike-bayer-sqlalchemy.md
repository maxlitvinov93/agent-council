# SQLAlchemy 2.0 Async Quality Reference — Mike Bayer

Philosophy: Mike Bayer. Creator and BDFL of SQLAlchemy and Alembic since 2006. Authority on ORM design patterns, identity map, unit of work, relationship loading strategies, and async session management. Core doctrine: "The ORM's job is to translate between the relational world and the object world — it is NOT a SQL generation tool."
Stack context: SQLAlchemy 2.0 async with Mapped[]/mapped_column() declarative style for Dense Club fitness SaaS. AsyncSession via asyncpg. PostgreSQL 16. 32 tables with deep FK relationships (User → Assignment → Template → TemplateDay → TemplateItem → Exercise). Coach views loading 30 athletes × current assignment × latest performance. selectinload/joinedload for N+1 prevention. Alembic async migrations. Pydantic v2 DTOs separate from ORM models.

Every finding must describe the **concrete failure mode** — not just "ORM misuse."

---

## Principle 1: AsyncSession lifecycle must be request-scoped

*Bayer: "A session is a workspace. A request is a unit of work. They should live and die together."*

The #1 async SQLAlchemy bug is `DetachedInstanceError` — accessing a lazy-loaded relationship after the session is closed. In async mode, this manifests as `MissingGreenlet` which is even more confusing.

### What to check

**Yield-based dependency for session injection**
```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session
```
- The `async with` ensures `session.close()` runs even if the endpoint raises an exception. If using `return` instead of `yield`, the session is never closed on error → connection leak → pool exhaustion after ~15 failed requests → entire API freezes.
- Severity: **P1** if `get_db` uses `return` or if sessions are created inside service functions.

**Sessions must not escape the request**
- A session passed to a background task or stored in a global variable outlives the request → `DetachedInstanceError` when the task accesses relationships. Background tasks that need DB access must create their own session.
- Severity: **P1** if `session` is passed to `BackgroundTasks.add_task()`.

**No session.execute() after yield**
- In the `get_db` dependency, no code runs after `yield` except cleanup. If there's logic after `yield` that uses the session (e.g., logging, audit), it runs during response teardown when the session may already be committed/rolled back.
- Severity: **P2**

---

## Principle 2: Explicit loading strategies prevent N+1

*Bayer: "Lazy loading is a convenience for scripts. In a web application, it's a performance trap."*

Coach dashboard: 30 athletes × current assignment × latest performance. Without eager loading, this is 30 + 30 + 30 = 90+ queries. With joinedload/selectinload, it's 3-4 queries.

### What to check

**selectinload for collections, joinedload for single FK**
- Collection relationships (User.assignments, Template.template_days): use `selectinload`. This fires a separate `SELECT ... WHERE id IN (...)` query — one query per relationship level, regardless of parent count.
- Single FK (Assignment.template, TemplateItem.exercise): use `joinedload`. This adds a JOIN to the parent query — one query total. But NEVER use joinedload for collections — it produces a cartesian product (30 athletes × 5 assignments × 7 template days = 1050 rows in one query).
- Severity: **P1** if coach dashboard queries produce N+1 (>30 queries for one page load). **P2** if joinedload used on collections.

**Lazy loading raises MissingGreenlet in async**
- In sync SQLAlchemy, accessing `user.assignments` without eager loading fires a lazy query. In async SQLAlchemy, this raises `MissingGreenlet: greenlet_spawn has not been called`. The error is opaque and happens at attribute access, not at query time.
- Every relationship access must be either eagerly loaded or explicitly loaded via `await session.execute(select(Assignment).where(...))`.
- Severity: **P1** — async lazy loading is not "slow", it's "broken."

**Deep loading chains for workout resolution**
- Workout resolution requires: Assignment → Template → TemplateWeeks → TemplateDays → TemplateItems → Exercise. That's 5 levels deep. A single query with nested selectinloads:
  ```python
  selectinload(Assignment.template)
    .selectinload(Template.template_weeks)
    .selectinload(TemplateWeek.template_days)
    .selectinload(TemplateDay.template_items)
    .selectinload(TemplateItem.exercise)
  ```
- Without this chain: 5 separate queries at best, MissingGreenlet at worst.
- Severity: **P1** for the workout resolution query path.

---

## Principle 3: ORM models are not API schemas

*Bayer: "The ORM model represents the database. The API schema represents the client contract. They serve different masters."*

Dense Club's API response shapes diverge from DB shapes. The resolved workout is a computed structure combining assignments + overrides + micro blocks — it has no corresponding table.

### What to check

**Pydantic DTOs with from_attributes**
- Every ORM → API conversion goes through a Pydantic schema with `model_config = ConfigDict(from_attributes=True)`. This enables `PerformanceResponse.model_validate(db_performance)`.
- Without `from_attributes`: `model_validate()` fails because it tries dict-style access on an ORM object. The error is a validation error, not a clear "you forgot from_attributes."
- Severity: **P2**

**Separate Create/Update/Response schemas**
- `PerformanceCreate` (what the client sends), `PerformanceUpdate` (partial update), `PerformanceResponse` (what the client receives). The response includes computed fields (tonnage, is_pr, rank) that don't exist on the ORM model or create schema.
- If using one schema for both create and response: the client can send `tonnage: 99999` and the server trusts it — bypassing server-side tonnage calculation.
- Severity: **P1** if a single schema is used for both input and output, especially for fields with server-computed values.

**Never expose internal ORM fields**
- ORM models may have: `hashed_password`, `stripe_customer_id`, `refresh_token_hash`, internal `_sa_instance_state`. Returning the ORM object directly leaks these to the client. The Pydantic response schema acts as a whitelist of allowed fields.
- Severity: **P1** if any auth-related or payment-related field is exposed via API response.

---

## Principle 4: Transaction boundaries define data consistency

*Bayer: "A transaction is a promise that either everything happens or nothing happens. Break the promise and you break the data."*

Dense Club's hot write path: performance log → tonnage → PR upsert → activity_event → ranking invalidation — all in one transaction.

### What to check

**async with session.begin() wrapping the entire chain**
```python
async with session.begin():
    performance = Performance(**data)
    session.add(performance)
    await session.flush()  # get performance.id
    
    tonnage = calculate_tonnage(performance, exercise)
    performance.total_tonnage = tonnage
    
    is_new_pr = await upsert_personal_record(session, performance)
    if is_new_pr:
        session.add(ActivityEvent(type="new_pr", ...))
```
- If any step fails, the entire transaction rolls back. Without `session.begin()`: autocommit mode commits each `session.add()` independently → failed PR detection leaves an orphaned performance with no activity_event.
- Severity: **P1** if the hot write path has multiple independent commits.

**session.flush() vs session.commit()**
- `flush()` writes to DB but doesn't commit — useful for getting auto-generated IDs mid-transaction. `commit()` ends the transaction. Calling `commit()` mid-chain breaks the transaction boundary.
- Severity: **P2** if `session.commit()` is called before the full chain completes.

**No auto-flush surprises**
- By default, SQLAlchemy auto-flushes before queries. This is usually fine but can cause `IntegrityError` at unexpected points. If the performance INSERT violates a constraint, the auto-flush during the PR SELECT raises the error — confusing stack trace.
- For the hot path: explicit `await session.flush()` after each step makes error locations predictable.
- Severity: **P3**

---

## Principle 5: Relationship modeling for Dense Club's domain

*Bayer: "back_populates is explicit. backref is magic. Explicit wins in a codebase maintained by humans."*

### What to check

**back_populates on every relationship**
- Use `back_populates`, never `backref`. `backref` creates the reverse relationship implicitly — the IDE can't find it, grep can't find it, new developers don't know it exists.
  ```python
  class User(Base):
      assignments: Mapped[list["Assignment"]] = relationship(back_populates="user")
  class Assignment(Base):
      user: Mapped["User"] = relationship(back_populates="assignments")
  ```
- Severity: **P2** if `backref` is used.

**Cascade deletes: soft delete on users, hard delete on template items**
- Users and assignments use soft delete (`deleted_at` timestamp). Template structural items (TemplateDay, TemplateItem) use hard delete with cascade: deleting a Template cascades to its days and items.
- Dangerous: `cascade="all, delete-orphan"` on User.performances — deleting a user hard-deletes all their performance history. Dense Club should soft-delete users and keep historical data.
- Severity: **P1** if hard cascade delete is configured on User → Performances or User → PersonalRecords.

**Foreign key constraints in the ORM model match the DB**
- Every `Mapped["User"] = relationship()` with a FK column must have a corresponding `ForeignKey("users.id")` in the column definition. Missing FK in the model = no constraint in the DB = orphaned rows when a user is deleted.
- Severity: **P2**

---

## Principle 6: Alembic async migration patterns

*Bayer: "Alembic autogenerate is a starting point, not a finished migration."*

### What to check

**env.py configured for async**
```python
async def run_async_migrations():
    connectable = create_async_engine(DATABASE_URL)
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()
```
- Without async env.py: Alembic tries to use the sync engine with asyncpg driver → `NotImplementedError` on first migration.
- Severity: **P1** if env.py is not configured for async.

**Autogenerate misses these (always review)**
- Enum value additions: adding a new exercise nature to the PostgreSQL ENUM type is not detected.
- CHECK constraints: adding `CHECK (effort BETWEEN 1 AND 10)` is not detected.
- Partial indexes: `CREATE INDEX ... WHERE is_active = true` — condition is not detected.
- JSONB column schema changes: content changes inside JSONB are invisible.
- INCLUDE columns: `CREATE INDEX ... INCLUDE (pr_value)` — the INCLUDE clause is not detected.
- Severity: **P2** — false confidence in autogenerate leads to missing constraints in production.

**Migration naming convention**
- `alembic revision --autogenerate -m "add_bodyweight_contribution_pct_to_exercises"` — descriptive, lowercase, underscored. Not `001`, not `update`, not `fix`.
- Severity: **P3** — maintainability issue.
