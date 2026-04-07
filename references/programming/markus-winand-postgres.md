# PostgreSQL Quality Reference — Markus Winand

Philosophy: Markus Winand. Author of "SQL Performance Explained" and use-the-index-luke.com. Authority on SQL indexing, query optimizer behavior, execution plans, and database-agnostic performance tuning. Core doctrine: "An index makes a slow query fast; the wrong index makes a fast query slow."
Stack context: PostgreSQL 16 for Dense Club fitness SaaS. 32 tables (users, exercises, skills, performances, personal_records, activity_events, rankings, assignments, templates, etc.). asyncpg driver. Write-heavy logging (every set → performance + PR detection + tonnage + activity_event in one TX). Rankings via window functions (percentiles per exercise per scheme). ~560 MB/year growth at 1K users. Alembic migrations. Railway managed Postgres.

Every finding must describe the **concrete failure mode** — not just "missing index."

---

## Principle 1: Index the access path, not the column

*Winand: "A single-column index does not help a multi-column query. Index the access path as a whole."*

Dense Club's hot queries define the index strategy. Wrong column order in a composite index = full table scan despite the index existing.

### What to check

**PR lookup: (user_id, exercise_id, scheme)**
- The PR detection query: `SELECT pr_value FROM personal_records WHERE user_id = ? AND exercise_id = ? AND scheme = ?`. This needs a composite index `(user_id, exercise_id, scheme)` — in that order, because user_id is always present. A single-column index on `user_id` forces a filter scan across all that user's PRs (could be hundreds).
- Add `INCLUDE (pr_value)` for an index-only scan — the query never touches the heap.
- Severity: **P1** if PR lookup lacks the composite index — every set logged triggers this query.

**Today's workout: (user_id, workout_date)**
- `SELECT * FROM performances WHERE user_id = ? AND workout_date = ?` — the most common read query. Index on `(user_id, workout_date)` with most recent dates at the end (default B-tree ascending = fine, latest dates at right edge).
- Severity: **P1** — this query runs on every app open.

**Rankings: (exercise_id, scheme) for window functions**
- Rankings compute `PERCENT_RANK() OVER (PARTITION BY exercise_id, scheme ORDER BY pr_value DESC)`. The window function needs the partition columns + sort column indexed: `CREATE INDEX idx_pr_rankings ON personal_records (exercise_id, scheme, pr_value DESC)`.
- Without this index: PostgreSQL sorts the entire personal_records table to compute one exercise's percentile. At 10K users × 50 exercises = 500K rows, this takes seconds.
- Severity: **P1** if ranking queries exceed 100ms.

**Partial indexes for active users**
- 83 users now, but users churn. A partial index `WHERE is_active = true` on frequently-queried tables (assignments, performances) reduces index size by excluding churned accounts. Useful once inactive users > 50% of total.
- Severity: **P3** — premature at current scale, plan for later.

---

## Principle 2: The write path determines the index budget

*Winand: "Every index you add makes INSERT slower. The question is not 'what queries do I want to optimize' but 'what index cost can I afford on the write path.'"*

### What to check

**Count indexes on the hot write path tables**
- `performances`: every set logged = INSERT. Each index on this table adds ~10-20% write overhead. If there are 5 indexes on performances, the INSERT is 50-100% slower than with 1 index.
- Rule of thumb for Dense Club: max 3-4 indexes per high-write table. `performances` needs: PK, `(user_id, workout_date)`, and `(user_id, exercise_id)` for duplicate detection. A 4th on `(exercise_id, scheme)` for rankings is acceptable. A 5th is suspicious.
- Severity: **P2** if any hot write table has more than 5 indexes.

**Monitor unused indexes**
- `SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0 AND idx_tup_read = 0 ORDER BY pg_relation_size(indexrelid) DESC` — shows indexes that exist but are never used. Every unused index wastes write throughput and disk.
- Severity: **P3** — run quarterly as maintenance.

**personal_records UPSERT index cost**
- `INSERT ... ON CONFLICT (user_id, exercise_id, scheme) DO UPDATE` — the unique index on the conflict target is both a constraint and a performance tool. This single index serves PR detection, uniqueness, and lookup. It's the most cost-effective index in the schema.
- Severity: **P1** if the unique constraint on personal_records is missing.

---

## Principle 3: Window functions for rankings must be bounded

*Winand: "Window functions are elegant but dangerous. They operate on the result set — if the result set is large, the window function is slow."*

### What to check

**Rankings query scope**
- Dense Club computes percentile rankings per exercise per scheme. The query:
  ```sql
  SELECT user_id, pr_value,
         PERCENT_RANK() OVER (PARTITION BY exercise_id, scheme ORDER BY pr_value DESC) as percentile
  FROM personal_records
  WHERE exercise_id = ? AND scheme = ?
  ```
- This is already bounded by the WHERE clause — good. The window function only operates on one partition. But if someone writes `SELECT ... PERCENT_RANK() OVER (ORDER BY pr_value DESC) FROM personal_records` without WHERE — the function scans all 500K rows.
- Severity: **P1** if any ranking query lacks a WHERE clause bounding the partition.

**Materialized views for global leaderboards**
- If a leaderboard shows "Top 50 squatters globally", the window function runs on ALL squat PRs for ALL users. At scale, this exceeds 100ms. Solution: materialized view refreshed by cron (daily at 3 AM).
  ```sql
  CREATE MATERIALIZED VIEW mv_rankings AS
  SELECT exercise_id, scheme, user_id, pr_value,
         PERCENT_RANK() OVER (...) as percentile
  FROM personal_records;
  CREATE UNIQUE INDEX ON mv_rankings (exercise_id, scheme, user_id);
  ```
- Severity: **P2** — needed when any ranking query exceeds 100ms consistently.

**No minimum user threshold per spec**
- The spec says rankings are computed "regardless of user count." This means an exercise with 1 user shows that user at 100th percentile. Acceptable, but document that percentiles are meaningless below ~20 users per exercise/scheme.
- Severity: **P3** — UX issue, not performance.

---

## Principle 4: UPSERT (ON CONFLICT) patterns for PR detection

*Winand: "ON CONFLICT DO UPDATE is an atomic read-modify-write. Get the conflict target wrong and it silently inserts duplicates."*

### What to check

**Conflict target must match the unique constraint exactly**
- `INSERT INTO personal_records (user_id, exercise_id, scheme, pr_value) VALUES (...) ON CONFLICT (user_id, exercise_id, scheme) DO UPDATE SET pr_value = EXCLUDED.pr_value WHERE EXCLUDED.pr_value > personal_records.pr_value`
- The `WHERE` clause is critical — without it, every performance log overwrites the PR, even if the new value is LOWER. Athlete sets 100kg PR, then does a lighter set at 60kg → PR is now 60kg. Career PR destroyed.
- Severity: **P1** if the WHERE clause is missing in the PR upsert.

**The EXCLUDED table reference**
- `EXCLUDED.pr_value` refers to the row that WOULD have been inserted. `personal_records.pr_value` refers to the EXISTING row. Swapping them: `WHERE personal_records.pr_value > EXCLUDED.pr_value` — only updates when new value is LOWER. Silently inverts PR logic.
- Severity: **P1** — subtle bug that passes casual testing.

**Return whether the PR was actually updated**
- Use `RETURNING (xmax = 0) AS is_new_record` — `xmax = 0` means a fresh INSERT (first ever PR for this exercise/scheme), `xmax != 0` means UPDATE (new PR beat old PR). The application needs this to decide whether to create an activity_event and send a notification.
- Severity: **P2** if the application can't distinguish "new PR" from "no change."

---

## Principle 5: Transaction isolation for the hot path

*Winand: "The default READ COMMITTED is correct for 99% of applications. SERIALIZABLE is correct for 0.1% and dangerous for the rest."*

### What to check

**READ COMMITTED is correct for Dense Club**
- Performance logging: INSERT performance → SELECT best PR → UPSERT personal_record → INSERT activity_event. Under READ COMMITTED, two athletes logging simultaneously don't interfere — each transaction sees committed data from other transactions.
- SERIALIZABLE would prevent a phantom read where athlete A and athlete B both log at the same time and both think they're #1 — but the deadlock risk with concurrent athletes isn't worth the protection. Rankings are eventually consistent anyway (refreshed periodically).
- Severity: **P2** if SERIALIZABLE is set globally (deadlocks under load).

**Advisory locks for ranking recomputation**
- If two requests trigger ranking recomputation simultaneously, they both compute the same result and write it twice. Use `pg_advisory_xact_lock(exercise_id_hash)` to serialize ranking updates per exercise.
- Severity: **P3** — redundant computation, not data corruption.

**No explicit LOCK TABLE**
- Dense Club should never need `LOCK TABLE`. If you see it, the design is wrong. Row-level locking via `SELECT ... FOR UPDATE` is acceptable for specific rows (e.g., locking a user row during trial expiry check to prevent race conditions).
- Severity: **P2** if `LOCK TABLE` appears anywhere.

---

## Principle 6: Migration discipline with Alembic

*Winand: "A schema migration is a production deployment. Treat it with the same rigor."*

### What to check

**Every migration has upgrade() AND downgrade()**
- Without downgrade: a bug in the new code forces a rollback, but the DB schema can't be rolled back → manual SQL intervention on a production database at 2 AM.
- Severity: **P1** if any migration lacks a downgrade function.

**Destructive changes split into two releases**
- DROP COLUMN, DROP TABLE: first migration adds nullable/tombstone, second migration (next release) drops. Why: if the first release has a bug and needs rollback, the column is still there with data intact. Dropping immediately = data loss on rollback.
- Severity: **P1** for any DROP COLUMN in a single migration.

**ALTER TABLE on large tables acquires ACCESS EXCLUSIVE lock**
- `ALTER TABLE performances ADD COLUMN` locks the entire table. With 3,500 rows this takes milliseconds. At 1M rows, it can take seconds — blocking all reads and writes. For large tables: use `ALTER TABLE ... ADD COLUMN ... DEFAULT NULL` (instant in PG 11+) or `CREATE INDEX CONCURRENTLY` (doesn't lock).
- Severity: **P2** at current scale, **P1** at 100K+ rows per table.

**Autogenerate limitations**
- Alembic autogenerate does NOT detect: enum value additions, CHECK constraint changes, partial index conditions, JSONB column schema changes, index options (INCLUDE, WHERE). Always review the generated migration file manually.
- Severity: **P2** — false confidence in autogenerate.

---

## Principle 7: JSONB vs normalized columns

*Winand: "JSONB is a schema-less escape hatch. Use it when the data structure genuinely varies per row, not when you're too lazy to normalize."*

### What to check

**Workout overrides must be normalized**
- Overrides (remove/replace/edit/add exercises for a specific date) need filtering: `SELECT * FROM overrides WHERE assignment_id = ? AND date = ? AND type = 'add'`. Storing overrides as a JSONB array on the assignments table means: no index on override type/date, full JSONB scan for every workout resolution, no foreign key to exercises.
- Severity: **P1** if overrides are stored as JSONB and queried by attributes.

**Exercise metadata can be JSONB**
- Display-only metadata (demo video URL, muscle groups, equipment needed) that is stored and retrieved as a blob — JSONB is fine. You never query "find all exercises with equipment = dumbbell" from the JSON — that's what the `equipment_type` column is for.
- Severity: **P3** — JSONB for display metadata is acceptable.

**Skill level thresholds as structured columns, not JSONB**
- Each skill has 6 level thresholds (starter, level_1-3, elite_1-2). These are queried: "what level is the athlete at for this skill?" requires comparing the athlete's value against the thresholds. If thresholds are JSONB: `skill_definition->>'level_1'` requires casting to numeric for comparison, defeats indexing, and makes the query fragile (typo in key name = NULL comparison).
- Use 6 NUMERIC columns: `starter`, `level_1`, `level_2`, `level_3`, `elite_1`, `elite_2`.
- Severity: **P2** if skill thresholds are stored as JSONB.
