# Stripe Integration Quality Reference — Brandur Leach

Philosophy: Brandur Leach. Former Stripe engineer, author of foundational blog posts on idempotency patterns, webhook reliability, and robust API design. Core doctrine: "Exactly-once delivery is impossible over a network. Build for at-least-once with idempotent handlers."
Stack context: Stripe integration for Dense Club fitness SaaS. stripe-python 11.x (async). Subscription model: 7-day free trial → monthly Stripe subscription (standard/premium tiers). Checkout Session for initial signup. Customer Portal for plan management. Webhooks for lifecycle events. FastAPI webhook endpoint. Idempotency keys on all Stripe write calls. PostgreSQL stores subscription_status synced from Stripe. APScheduler syncs Stripe statuses every 6 hours as safety net.

Every finding must describe the **concrete failure mode** — not just "bad Stripe practice."

---

## Principle 1: Webhooks are the source of truth, not API responses

*Leach: "The redirect after checkout is best-effort. The webhook is the contract."*

### What to check

**Update subscription status from webhooks, not checkout redirect**
- After `checkout.session.completed`, update `users.subscription_status` and `users.subscription_tier` from webhook data. The checkout redirect (user returns to success page) can fail: browser closes, network drops, user navigates away.
- Failure mode: athlete pays → redirect fails → user shows as `athlete_trial` → blocked from premium features → support ticket → coach and athlete both frustrated.
- Severity: **P1** if subscription activation depends on the checkout redirect.

**Critical webhook events to handle**
- `checkout.session.completed` — activate subscription
- `invoice.paid` — renew access (recurring billing success)
- `invoice.payment_failed` — set `past_due`, show warning banner
- `customer.subscription.updated` — tier change (standard → premium or downgrade)
- `customer.subscription.deleted` — set `canceled`, access until period end
- Missing any of these: subscription state in Dense Club's DB diverges from Stripe → athlete loses access despite paying, or retains access without paying.
- Severity: **P1** for checkout.session.completed and invoice.paid. **P2** for the rest.

**APScheduler safety net every 6 hours**
- `sync_stripe_statuses` job polls `stripe.Subscription.list()` and reconciles with DB. Catches any webhook that was lost (Stripe retry exhausted, server was down during all retries).
- Without safety net: a single lost webhook means permanent state divergence until manual intervention.
- Severity: **P2** if no reconciliation job exists.

---

## Principle 2: Webhook endpoint must be idempotent

*Leach: "At-least-once delivery means exactly that. Your handler sees the same event more than once."*

### What to check

**Store processed event IDs**
- `processed_stripe_events` table with `stripe_event_id` (unique). Before processing any webhook: check if event_id exists. If yes → return 200 immediately (already processed). If no → process and store.
  ```python
  existing = await session.execute(
      select(ProcessedStripeEvent).where(
          ProcessedStripeEvent.stripe_event_id == event.id
      )
  )
  if existing.scalar_one_or_none():
      return {"status": "already_processed"}
  ```
- Without dedup: `invoice.paid` fires twice → two activity_events "Subscription renewed" in community feed. Or worse: `checkout.session.completed` fires twice → two Stripe customers created for the same user → double billing.
- Severity: **P1** if webhook handlers lack idempotency checks.

**Idempotency check + processing in one transaction**
- The check (event exists?) and the processing (update user, insert event record) must be in one transaction. Without atomicity: two concurrent deliveries of the same event both pass the "not exists" check → both process → duplicate side effects.
- Severity: **P2**

**Return 200 even for events you don't handle**
- Stripe sends many event types. Return 200 for unrecognized types — don't return 400/500. A non-200 response tells Stripe to retry, and it will retry failed events up to 3 days, polluting your logs and Stripe dashboard with false failures.
- Severity: **P2**

---

## Principle 3: Verify webhook signatures, always

*Leach: "An unauthenticated webhook endpoint is a backdoor with a sign on it."*

### What to check

**Signature verification with raw body**
```python
@router.post("/api/webhooks/stripe")
async def stripe_webhook(request: Request):
    raw_body = await request.body()
    sig_header = request.headers.get("stripe-signature")
    try:
        event = stripe.Webhook.construct_event(
            raw_body, sig_header, settings.stripe_webhook_secret.get_secret_value()
        )
    except stripe.error.SignatureVerificationError:
        raise HTTPException(400, "Invalid signature")
```
- CRITICAL: use `await request.body()` (raw bytes), NOT `await request.json()`. The signature is computed over the raw body. JSON parsing then re-serializing changes whitespace/encoding → signature mismatch on valid webhooks, or worse: signature validation passes on tampered JSON that happened to serialize identically.
- Severity: **P1** if signature verification is missing or uses parsed JSON.

**Attack vector without verification**
- Attacker discovers the webhook URL (common: `/api/webhooks/stripe`) → POSTs `{"type": "customer.subscription.updated", "data": {"object": {"customer": "cus_victim", "status": "active", "items": {"data": [{"plan": {"id": "premium"}}]}}}}` → server updates victim to premium tier → free premium access.
- Severity: **P1** — trivial exploitation.

**Webhook secret in env var, not code**
- `STRIPE_WEBHOOK_SECRET` in `.env` / Railway environment, never in source code. Use Pydantic `SecretStr` so it doesn't appear in logs or error traces.
- Severity: **P1** if webhook secret is hardcoded.

---

## Principle 4: Idempotency keys on all Stripe write calls

*Leach: "An idempotency key turns 'at-least-once' into 'exactly-once' for the caller."*

### What to check

**Deterministic keys, not random UUIDs**
- `stripe.checkout.Session.create_async(..., idempotency_key=f"checkout-{user_id}-{plan}-{date}")`. The key must be deterministic for the same intent: same user, same plan, same day = same key. Random UUID defeats the purpose — every retry gets a new key, Stripe treats each as a new request.
- Without deterministic keys: athlete double-clicks "Subscribe" → two checkout sessions → two subscriptions → double billing. Support refund required.
- Severity: **P1** on checkout session creation and customer creation.

**Keys on every write call**
- `stripe.Customer.create_async(idempotency_key=...)` — prevent duplicate customers
- `stripe.checkout.Session.create_async(idempotency_key=...)` — prevent duplicate checkouts
- `stripe.Subscription.modify_async(idempotency_key=...)` — prevent duplicate plan changes
- Write calls without keys: any network timeout + automatic retry = duplicate operation on Stripe's side.
- Severity: **P1** for customer and checkout creation. **P2** for modifications.

**Key expiration awareness**
- Stripe idempotency keys expire after 24 hours. A key used on Monday won't work on Tuesday. For daily operations (checkout retry on same day), this is fine. For long-lived operations, include the date in the key.
- Severity: **P3**

---

## Principle 5: Subscription lifecycle state machine

*Leach: "Subscription state is a state machine. If you don't model it as one, you'll get impossible states."*

### What to check

**States map to Dense Club access levels**
| Stripe status | Dense Club role | Access |
|---|---|---|
| (no subscription) | athlete_trial | Full access for 7 days |
| active | athlete_standard or athlete_premium | Full access at tier level |
| past_due | (keep current tier) | 3-day grace period, show warning banner |
| canceled | (keep current tier) | Access until `current_period_end` |
| unpaid (after past_due) | athlete_trial-like | Restricted access, payment required |

- Failure mode without state machine: `past_due` immediately blocks access → athlete can't log workout → misses training session → churns. The 3-day grace gives them time to update payment.
- Severity: **P1** if `past_due` immediately blocks access.

**current_period_end for canceled subscriptions**
- When an athlete cancels, they keep access until the end of the billing period. Store `subscription_period_end` from Stripe and check it on each request: `if status == "canceled" and period_end > now() → allow access`.
- Without this: athlete cancels mid-month → immediately locked out → demanded refund for unused days.
- Severity: **P1** if cancellation immediately removes access.

**Tier stored in DB, not derived from JWT**
- The subscription tier (standard/premium) must be stored in the `users` table and updated via webhooks. Do NOT put the tier in the JWT — if the athlete upgrades mid-session, the JWT still says "standard" until it expires in 15 minutes. The server must check the DB tier on each request for tier-gated features.
- Severity: **P2** if tier is only in the JWT.

---

## Principle 6: Never log sensitive Stripe data

*Leach: "Logs are the most-breached asset in any system. Assume they're public."*

### What to check

**structlog must filter Stripe objects**
- Before logging any Stripe-related data, strip: `stripe_customer_id`, `payment_method`, card details, `stripe_subscription_id`. Log only: `stripe_event_id`, `event_type`, `user_id`, `action_taken`.
- Failure mode: structlog logs the full webhook payload → Sentry captures it → Sentry has card-last-four + billing address for all paying users → Sentry breach = PCI compliance violation.
- Severity: **P1** if full Stripe event payloads are logged.

**Sentry before_send hook**
- Configure Sentry's `before_send` to strip Stripe data from breadcrumbs and exception context. Stripe objects captured in exception stacktraces contain customer IDs and subscription details.
- Severity: **P2**

**API keys as SecretStr**
- `stripe_secret_key: SecretStr` and `stripe_webhook_secret: SecretStr` in Pydantic Settings. `SecretStr` repr shows `'**********'` instead of the actual key. Without this: any error log that includes the settings object leaks the Stripe API key.
- Severity: **P1** if Stripe keys are plain strings in settings.
