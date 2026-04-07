# Fitness Data Architecture Reference — Charles Poliquin × Mike Israetel × Andy Galpin

Philosophy: Charles Poliquin (Strength Sensei — structural balance, exercise classification, tempo prescription, strength standards). Mike Israetel (Renaissance Periodization — volume landmarks, hypertrophy metrics, fatigue management). Andy Galpin (Exercise Physiology — performance testing, adaptation tracking, data-driven periodization).

Stack context: Dense Club fitness SaaS. EMOM-based training (Dense methodology). PostgreSQL database tracking exercises, performances, PRs, tonnage, skill progressions. 5-role system. ~150 exercises, ~4000 performance logs. Goal: maximize data granularity from every workout to enable intelligent analytics, progression tracking, structural balance monitoring, and personalized programming.

**The Dense methodology is sacred — never suggest changing the training approach itself. Only optimize what data we collect, how we classify it, and what analytics we derive from it.**

Every finding must describe the **concrete analytical value** — not just "nice to have data."

---

## Principle 1: Exercise Classification Must Be Multi-Dimensional

*Poliquin: "An exercise is not just a name. It's a movement pattern, a muscle recruitment sequence, a joint action, a force curve, and a fatigue profile."*

A flat exercise list with just `name` and `exercise_nature` loses 80% of analytical value. Every exercise must be classified along multiple axes to enable cross-exercise comparisons and intelligent programming.

### Required Classification Axes

**1. Movement Pattern** (PRIMARY — determines programming slot)
- `horizontal_push` (bench press, push-up)
- `vertical_push` (OHP, handstand push-up)
- `horizontal_pull` (row, face pull)
- `vertical_pull` (chin-up, lat pulldown)
- `hip_hinge` (deadlift, RDL, hip thrust)
- `squat` (back squat, front squat, pistol)
- `lunge` (walking lunge, Bulgarian split squat)
- `carry` (farmer walk, overhead carry)
- `rotation` (pallof press, wood chop)
- `isolation_push` (tricep extension, lateral raise)
- `isolation_pull` (curl, reverse fly)
- `skill` (handstand, muscle-up, planche)

**Why**: Without movement pattern, you cannot detect: "This athlete does 12 sets of horizontal push per week but 0 vertical pull — shoulder impingement incoming." Structural balance requires movement-level volume tracking.

**2. Movement Length / Range of Motion Profile**
- `lengthened` — exercise emphasizes the stretched position (incline curl, Romanian deadlift, overhead tricep extension, deficit push-up)
- `mid_range` — exercise is strongest in the middle (bench press, squat, chin-up)  
- `shortened` — exercise emphasizes the contracted position (lateral raise top, cable crossover, leg extension lockout)
- `full_range` — significant load through entire ROM (pull-up, dip, pistol squat)

*Israetel: "Lengthened partials produce more hypertrophy per set than shortened partials. If we track ROM profile, we can quantify stimulus quality, not just volume."*

**Why**: 10 sets of cable crossover (shortened) ≠ 10 sets of incline dumbbell fly (lengthened) for chest growth. Without this, volume tracking is misleading.

**3. Compound vs Isolation**
- `compound_primary` — multi-joint, high systemic fatigue (squat, deadlift, bench)
- `compound_secondary` — multi-joint, moderate fatigue (row, lunge, dip)
- `isolation` — single-joint (curl, extension, raise)
- `skill_movement` — calisthenics skill (handstand, planche, L-sit)

*Poliquin: "A set of squats generates 3-5x more systemic fatigue than a set of leg extensions at equivalent local stimulus. Programming that ignores this creates recovery debt."*

**Why**: Fatigue tracking. 20 sets of compounds ≠ 20 sets of isolation for recovery cost.

**4. Primary/Secondary/Stabilizer Muscle Involvement**
Already modeled in `exercise_muscles` junction table with `involvement` enum. But must be POPULATED for every exercise. An exercise without muscle mapping is analytically dead.

**Why**: Volume per muscle group calculations. "Your chest got 16 sets this week (10 primary from bench, 6 secondary from dips)" — impossible without mappings.

**5. Force Curve / Resistance Profile**
- `constant` — cable, band (constant tension throughout ROM)
- `ascending` — barbell free weight (hardest at lockout for squat/bench)
- `descending` — bodyweight pull-up (hardest at bottom)
- `bell_curve` — dumbbell fly (hardest at mid-range)

*Galpin: "Matching resistance profile to muscle length-tension relationship is how you target specific adaptations. Track it."*

**Why**: Advanced programming — if athlete only does ascending-curve exercises for chest, they miss lengthened-position stimulus.

### What to check in the database

- Does `exercises` table have columns for `movement_pattern`, `movement_length`, `compound_type`, `force_curve`?
- Is `exercise_muscles` populated for all 183 exercises?
- Can the system aggregate volume by movement pattern and by muscle group?
- Severity: **P1** if movement_pattern is missing (structural balance impossible). **P2** for other axes.

---

## Principle 2: Volume Tracking Must Be Per-Muscle, Not Per-Exercise

*Israetel: "Volume is the primary driver of hypertrophy. But volume must be counted per muscle group, not per exercise. A set of bench press is a chest set AND a front delt set AND a tricep set."*

### Volume Metrics to Derive (per muscle group per week)

1. **Total sets** — count of performance logs where this muscle is primary or secondary
2. **Effective sets** — sets within 5-0 RIR (Reps In Reserve). Sets at RPE <6 are "junk volume"
3. **Tonnage** — weight × reps × sets, split by muscle involvement percentage
4. **Frequency** — how many distinct days this muscle was trained
5. **Stimulus-to-fatigue ratio** — (effective sets × lengthened bias) / (compound fatigue cost)

### Volume Landmarks per Muscle (Israetel)

| Landmark | Meaning | Dense Context |
|----------|---------|---------------|
| MV (Maintenance Volume) | Minimum to not lose gains | 4-6 sets/week |
| MEV (Minimum Effective Volume) | Minimum to grow | 8-10 sets/week |
| MAV (Maximum Adaptive Volume) | Sweet spot | 12-20 sets/week |
| MRV (Maximum Recoverable Volume) | Beyond = overtraining | 20-25 sets/week |

**Why**: Dashboard can show: "Your quads are at 22 sets/week (approaching MRV) but your hamstrings are at 6 (below MEV) — structural imbalance detected."

### What to check

- Can the system compute weekly volume per muscle group from logged performances?
- Does effort/RPE data allow filtering for "effective sets" (RPE ≥ 6)?
- Does the system track sets per performance or just total reps?
- Severity: **P1** if volume per muscle cannot be computed. **P2** if volume landmarks are missing.

---

## Principle 3: Progression Must Be Multi-Metric, Not Just PR Weight

*Poliquin: "Strength is a skill. Measuring only 1RM or best weight misses 90% of progression signal."*

### Progression Metrics to Track

**1. Absolute Strength** (current PR tracking — already implemented)
- Best weight at each Dense scheme per exercise
- This is good but insufficient alone

**2. Relative Strength** (weight / bodyweight)
- Essential for bodyweight athletes and weight class comparison
- Must track bodyweight at time of PR, not just current bodyweight
- `relative_pr = pr_value / bodyweight_at_pr`

**3. Volume PR** (most total reps at a weight in a single session)
- Example: "Best session: 10D5 @ 80kg = 50 total reps" vs previous "10D5 @ 80kg = 45 reps"
- Indicates work capacity improvement even without weight increase

**4. Density PR** (same work in less time, or more work in same time)
- Dense methodology is TIME-BASED. If athlete completes 10D5 @ 80kg but finishes each set in 20s instead of 30s, that's progression.
- Currently NOT tracked. Would need `set_duration_seconds` or `rest_time_seconds` field.

**5. Consistency PR** (longest streak of completing prescribed scheme)
- "You've hit your prescribed 10D5 for 8 consecutive sessions — consistency PR!"
- Indicates program adherence and neural adaptation

**6. Effort Progression** (same weight feels easier)
- If athlete logs 80kg @ RPE 9 in week 1 and 80kg @ RPE 7 in week 4 — strength gained without weight increase
- Requires RPE/effort tracking per set (currently tracked per performance, which is good)

### What to check

- Does the PR system track only weight-based PRs or also volume/relative PRs?
- Is bodyweight stored with each performance (for relative strength calculation)?
- Is per-set timing data captured (for density progression)?
- Can the system detect "same weight, lower effort" as progression?
- Severity: **P1** if relative strength is not derivable. **P2** for density/volume PRs.

---

## Principle 4: Structural Balance Ratios Are Non-Negotiable

*Poliquin: "I can predict injury risk from 5 ratios. Every athlete should know where they stand."*

### Poliquin Structural Balance Ratios

| Ratio | Target | What It Reveals |
|-------|--------|-----------------|
| Close-Grip Bench : Bench Press | 87% | Tricep vs chest balance |
| Behind-Neck Press : Bench Press | 66% | Shoulder health |
| External Rotation : Internal Rotation | >66% | Rotator cuff health |
| Scott Curl : Close-Grip Bench | 50% | Bicep vs tricep |
| Squat : Deadlift | 80% | Quad vs posterior chain |
| Front Squat : Back Squat | 85% | Quad dominance vs posterior |
| Chin-Up (weighted) : Bench Press | 100% | Push-pull balance |
| Romanian Deadlift : Back Squat | 75% | Hamstring vs quad |

### How to Implement

1. Define ratio pairs in a `structural_balance_ratios` table
2. Compute each ratio from user's PRs at the relevant exercises
3. Flag imbalances: >10% deviation from target = warning, >20% = alert
4. Show on dashboard: "Your push-pull balance: 112% (bench 100kg, chin-up 90kg) — slight push dominance"

### What to check

- Are exercises tagged with their structural balance role?
- Can the system find matching exercise pairs for each ratio?
- Is there a mechanism to compute and display ratios?
- Severity: **P2** — advanced feature but high value for injury prevention.

---

## Principle 5: Dense-Specific Analytics

*The Dense methodology adds unique data opportunities that traditional training apps miss.*

### Dense Scheme Performance Curves

Each Dense scheme (e.g., 10D5) has a characteristic performance curve across weeks. Track:

1. **Scheme mastery** — how many sessions until the athlete consistently completes the full scheme
2. **Optimal progression rate** — typical weight increase per week per scheme
3. **Scheme-specific fatigue** — which schemes generate more systemic fatigue (10D10 > 10D1)
4. **Scheme transitions** — when athlete moves from 5D5 to 10D5, how does load drop? Is the drop within expected range?

### Block Analysis (Dense blocks as training units)

A Dense block (one scheme × one exercise × one session) is the atomic training unit. Analytics per block:

1. **Completion rate** — did athlete finish all prescribed reps? (e.g., 10D5 = 50 reps target, logged 47 = 94% completion)
2. **Intra-block fatigue** — if sets_detail is tracked, does performance drop across the 10 minutes? (set 1: 5 reps, set 10: 3 reps = fatigue pattern)
3. **Recovery indicator** — if same block next session has better completion rate = recovered. Worse = accumulated fatigue.

### What to check

- Is `sets_detail` (per-set data within a Dense block) stored or just totals?
- Can the system compute completion rate (actual reps / prescribed reps)?
- Is there any session-to-session comparison for the same exercise+scheme?
- Severity: **P1** for completion rate (essential Dense metric). **P2** for intra-block analysis.

---

## Principle 6: Data Collection — What to Add vs What to Derive

*Galpin: "Collect raw signal, derive metrics. Never ask the athlete to calculate — they'll get it wrong and you'll lose data."*

### Must Collect (athlete input per set/performance)

| Field | Status | Why |
|-------|--------|-----|
| Weight (kg) | ✅ Collected | Primary load metric |
| Reps completed | ✅ Collected | Volume numerator |
| Dense scheme | ✅ Collected | Time structure |
| Effort (RPE/text) | ✅ Collected | Intensity indicator |
| Bodyweight | ✅ Collected (daily) | Relative strength denominator |
| Notes | ✅ Collected | Context (injury, sleep, etc.) |
| Sets detail (per-set reps) | ⚠️ Exists but as JSON | Intra-block fatigue analysis |
| Set duration (seconds) | ❌ Missing | Density progression |
| Rest time between sets | ❌ Missing | Recovery capacity metric |
| Tempo (eccentric/concentric) | ❌ Missing | Time under tension |
| Band level | ⚠️ Exists | Banded exercise progression |

### Must Derive (computed server-side)

| Metric | Status | Formula |
|--------|--------|---------|
| Total tonnage | ✅ Computed | weight × reps (+ BW contribution) |
| PR detection | ✅ Computed | Best weight per exercise per scheme |
| Relative strength | ❌ Not computed | PR / bodyweight_at_pr |
| Volume per muscle | ❌ Not computed | Sum sets × involvement% per muscle |
| Completion rate | ❌ Not computed | actual_reps / scheme_target_reps |
| Effort trend | ❌ Not computed | RPE for same weight over sessions |
| Structural balance | ❌ Not computed | Exercise pair ratios |
| Frequency per muscle | ❌ Not computed | Distinct days per muscle per week |
| Weekly volume trend | ❌ Not computed | Sets per muscle per week over time |
| Fatigue score | ❌ Not computed | Compound sets × fatigue multiplier |

### Recommendations Priority

**P1 (High analytical value, low collection cost)**:
1. Add `movement_pattern` to exercises — enables structural balance
2. Add `compound_type` to exercises — enables fatigue tracking
3. Populate `exercise_muscles` for all exercises — enables per-muscle volume
4. Compute volume per muscle weekly — the killer feature
5. Compute completion rate — essential Dense metric

**P2 (High value, moderate effort)**:
6. Add `movement_length` to exercises — stimulus quality tracking
7. Store per-set data as structured array (not free JSON) — intra-block analysis
8. Compute relative strength PRs — meaningful for bodyweight community
9. Add structural balance ratio computation

**P3 (Advanced, future)**:
10. Set duration/tempo tracking — density progression
11. Force curve classification
12. Machine learning on fatigue patterns
