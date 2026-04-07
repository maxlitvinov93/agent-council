# Authentication & Security Quality Reference — Troy Hunt

Philosophy: Troy Hunt. Creator of Have I Been Pwned, Microsoft Regional Director, Pluralsight author on web security. Core doctrine: "If a vulnerability is syntactically possible, it statistically exists in your codebase." Combined with OWASP 2025 Top 10 practical checklists.
Stack context: Auth & security for Dense Club fitness SaaS. FastAPI + python-jose (JWT HS256) + passlib[bcrypt]. 5-role system: admin, coach, athlete_trial, athlete_standard, athlete_premium. JWT access tokens (15min) + refresh tokens (30 days) in HttpOnly cookies. Refresh token rotation with reuse detection. Stripe payment data. Coach-athlete relationship (coach sees athlete data). is_also_athlete flag (dual role). Trial expiry logic. slowapi rate limiting. Sentry error tracking.

Every finding must describe the **concrete attack vector** — not just "this is insecure."

---

## Principle 1: Refresh token rotation with reuse detection is non-negotiable

*Hunt: "A stolen refresh token without rotation is a permanent backdoor."*

### What to check

**Rotation on every refresh**
- On `POST /api/auth/refresh`: issue new access token + new refresh token → mark old refresh token as revoked (`revoked_at = now()`) in the `refresh_tokens` table → return both new tokens. The new refresh token goes into an HttpOnly SameSite=Lax Secure cookie.
- Without rotation: a stolen refresh token (XSS, network sniff, malware) grants 30 days of unlimited access. With rotation: the stolen token works exactly once, then the legitimate client's next refresh fails (their token was rotated away), alerting the system.
- Severity: **P1** if refresh tokens are reusable without rotation.

**Reuse detection**
- If a revoked refresh token is presented: this means the token was stolen (legitimate client already rotated past it, attacker replayed old token). Response: revoke ALL refresh tokens for that `user_id`, force logout everywhere, log security event to `audit_log`.
  ```python
  if record.revoked_at is not None:
      # REUSE DETECTED
      await session.execute(
          update(RefreshToken)
          .where(RefreshToken.user_id == record.user_id, RefreshToken.revoked_at.is_(None))
          .values(revoked_at=datetime.utcnow())
      )
      await log_security_event("refresh_token_reuse", user_id=record.user_id)
      raise HTTPException(401, "Session invalidated for security")
  ```
- Without reuse detection: attacker's stolen token works indefinitely in parallel with the legitimate user. Both rotate independently, creating two active token chains.
- Severity: **P1** — the entire JWT security model depends on this.

**Atomic check + revoke**
- The reuse detection query (SELECT revoked_at + UPDATE if revoked) must be in one transaction. Without atomicity: race condition where two concurrent requests with the same token both pass the "not revoked" check → both get new tokens → attacker has a permanent chain.
- Severity: **P1**

---

## Principle 2: Role-based access is a server concern, not a UI concern

*Hunt: "Hiding a button doesn't remove the endpoint. Security is what the server enforces, not what the client renders."*

### What to check

**Every endpoint validates role via Depends**
- `Depends(require_role(["coach", "admin"]))` on every coach/admin endpoint. A missing dependency means: change React component to show the admin page → all admin API calls work → full admin access from an athlete account.
- Severity: **P1** for any endpoint that reads/writes data without role verification.

**Coach-athlete relationship verification**
- Coach endpoints must verify the relationship: `coach_assignments WHERE coach_id = current_user.id AND athlete_id = ? AND revoked_at IS NULL`. Without this: coach A can access athlete B's data even if B is assigned to coach C. In a gym with multiple coaches, this leaks training data, body metrics, and performance history across coaches.
  ```python
  async def verify_coach_assignment(athlete_id: UUID, user: User, session: AsyncSession):
      result = await session.execute(
          select(CoachAssignment).where(
              CoachAssignment.coach_id == user.id,
              CoachAssignment.athlete_id == athlete_id,
              CoachAssignment.revoked_at.is_(None)
          )
      )
      if not result.scalar_one_or_none():
          raise HTTPException(403, "Not assigned to this athlete")
  ```
- Severity: **P1** — data breach across coaches.

**is_also_athlete dual role handling**
- A coach with `is_also_athlete = true` has both coach and athlete permissions. But they should ONLY see their own athlete data through athlete endpoints, and only their assigned athletes through coach endpoints. Don't treat `is_also_athlete` as "admin lite."
- Severity: **P2** if dual role bypasses any permission boundary.

**Admin operations require admin role exclusively**
- Admin endpoints (user management, exercise CRUD, skill definitions, global settings) require `role == "admin"`. Coaches with `is_also_athlete` flag should NOT have admin access. The role check must be exact, not inclusive (coach < admin hierarchy is tempting but dangerous — one compromised coach account = full admin access).
- Severity: **P1** if admin endpoints accept coach role.

---

## Principle 3: JWT claims must be minimal and verified

*Hunt: "A JWT is a signed claim, not an encrypted vault. Treat the payload as public."*

### What to check

**Minimal claims**
- Required: `sub` (user_id UUID), `role` (string), `is_also_athlete` (bool), `exp` (expiry), `iat` (issued at), `jti` (JWT ID for revocation).
- NEVER include: email, name, subscription_status, bodyweight, stripe_customer_id. JWTs are base64-encoded (NOT encrypted). Anyone with the token can decode it. An athlete's bodyweight in a JWT = bodyweight visible to anyone who intercepts the token.
- Severity: **P1** if sensitive PII or payment data is in the JWT payload.

**Algorithm explicitly set to HS256**
- `jwt.decode(token, secret, algorithms=["HS256"])`. Without the algorithms parameter, python-jose accepts ANY algorithm — including `alg: none`, which bypasses signature verification entirely. Attacker forges a token with `"alg": "none"` and any claims → server accepts it as valid.
- Severity: **P1** — the `alg: none` attack is automated in security scanners.

**Server-side expiry verification**
- Always check `exp` claim on the server, not just on the client. Client clocks are unreliable (manually set, timezone bugs). The server is the authority on whether a token is expired.
- Severity: **P2**

**jti for optional individual token revocation**
- The `jti` (JWT ID) claim enables revoking specific access tokens without revoking all sessions. Useful for: "revoke the session on the athlete's lost phone." Store `jti` in a redis set or DB table, check on each request. At launch, this is optional — refresh token revocation covers most cases.
- Severity: **P3** — nice-to-have for per-device session management.

---

## Principle 4: Rate limiting on auth endpoints prevents credential stuffing

*Hunt: "Credential stuffing is the #1 attack against consumer applications. It's automated, it's cheap, and it works."*

### What to check

**Login endpoint: 5 attempts per email per 15 minutes**
- Use slowapi with a key function that extracts the email from the request body, not just IP. IP-based limits are bypassed by botnets with thousands of IPs. Email-based limits protect the actual account.
- Without rate limiting: automated credential stuffing against 83 real user emails. If any user reuses a password from a breached site, the attacker gets in.
- Severity: **P1**

**Registration: 3 per IP per hour**
- Prevents mass account creation for spam/abuse. A gym has ~1-2 IPs, so keep the limit reasonable.
- Severity: **P2**

**Refresh endpoint: 30 per user per hour**
- Legitimate apps refresh every 15 minutes = max 4 per hour. 30 gives headroom for page reloads. More than 30 suggests automated abuse.
- Severity: **P2**

**Password reset: 3 per email per hour**
- Prevents reset email spam. An attacker can't flood someone's inbox with 1000 password reset emails by hitting the endpoint in a loop.
- Severity: **P2**

**Return generic errors**
- `/api/auth/login` must return the same error ("Invalid credentials") for wrong email AND wrong password. If "User not found" vs "Wrong password" — attacker enumerates valid emails. With 83 users in a niche fitness app, email enumeration reveals the entire user base.
- Severity: **P1** if login errors distinguish between "user not found" and "wrong password."

---

## Principle 5: Trial security prevents abuse

*Hunt: "Any client-side check can be bypassed. If it matters, check it on the server."*

### What to check

**Server-side trial expiry middleware**
- Trial expiry must be checked on every authenticated request for `athlete_trial` users. Implement as FastAPI middleware that runs after auth extraction:
  ```python
  if user.role == "athlete_trial" and user.trial_ends_at < now_utc():
      raise HTTPException(403, "Trial expired")
  ```
- Exempt endpoints: `/api/auth/*`, `/api/athlete/profile`, `/api/payments/*`, `/api/athlete/subscription-status`.
- Without server check: an athlete modifies the React app (or uses curl) to bypass the trial paywall → unlimited free access forever.
- Severity: **P1** — revenue loss + unfair to paying users.

**trial_ends_at stored in DB, not derived**
- Store `trial_ends_at` as a TIMESTAMPTZ column, not `created_at + 7 days` computed at check time. Why: admin needs to extend trials (support request, promotion). With derived dates, extension is impossible without changing created_at (which breaks audit history).
- Also: "7 days" means end of day 7 in the USER'S timezone, not the server's. Store `trial_ends_at` as `created_at + 7 days at 23:59:59 in user's timezone`.
- Severity: **P2** if trial duration can't be extended by admin.

**Registration fingerprinting**
- After trial expires, user creates new account with new email → new 7-day trial. At launch: acceptable (83 users, manual review). At scale: track device fingerprint or phone number to prevent abuse.
- Severity: **P3** at current scale.

---

## Principle 6: Coach-athlete data boundary is a trust boundary

*Hunt: "Multi-tenancy bugs are data breaches. Every query must be scoped to the tenant."*

### What to check

**Every coach query joins through coach_assignments**
- Coach endpoints that access athlete data MUST filter through the assignment relationship. A query like `SELECT * FROM performances WHERE user_id = ?` without checking `coach_assignments` means any coach can access any athlete by guessing/enumerating user_ids.
- Pattern: every coach service function takes `coach_id` parameter and joins through `coach_assignments`.
- Severity: **P1** — cross-coach data leak.

**Athlete data visible to coach is scoped**
- Coach sees: workout assignments, performances, PRs, bodyweight logs, skill progress for assigned athletes.
- Coach must NOT see: other coaches' athletes, payment/subscription details, auth credentials, admin configuration, athlete's private notes (if any).
- Severity: **P1** if any coach endpoint returns payment or auth data.

**Revoked assignments cut access immediately**
- When a coach-athlete relationship is revoked (`revoked_at IS NOT NULL`), the coach must immediately lose access to that athlete's data. If historical data is needed (coach reviewing past work), use a separate read-only archive permission.
- Without revocation check: ex-coach retains access to ex-athlete's ongoing training data indefinitely.
- Severity: **P1** if revoked assignments don't cut API access.

**Admin sees everything but with audit**
- Admin has unrestricted data access — this is by design. But every admin action (view user, edit exercise, modify subscription) must be logged to `audit_log` with admin_user_id, action, target, timestamp.
- Without audit: a compromised admin account modifies data silently.
- Severity: **P2**
