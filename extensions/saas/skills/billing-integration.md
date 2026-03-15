---
name: "billing-integration"
pack: "@rune/saas"
description: "Billing integration — Stripe and LemonSqueezy (Stripe alternative for Vietnam/non-US sellers). Subscription lifecycle, webhook handling, usage-based billing, dunning management, and tax handling."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# billing-integration

Billing integration — Stripe and LemonSqueezy (Stripe alternative for Vietnam/non-US sellers). Subscription lifecycle, webhook handling, usage-based billing, dunning management, and tax handling.

> **Vietnam note**: Stripe requires a US/EU entity and is unavailable for direct signup from Vietnam. LemonSqueezy acts as Merchant of Record — handles VAT, tax compliance, and payouts globally. Prefer LemonSqueezy for solo founders and small teams in Vietnam/Southeast Asia.

#### Workflow

**Step 1 — Detect billing provider**
Use Grep to find billing code: `stripe`, `lemonsqueezy`, `@stripe/stripe-js`, webhook endpoints (`/webhook`, `/billing/webhook`), subscription models. Read payment configuration and webhook handlers.

**Step 2 — Audit webhook reliability**
Check for: missing webhook signature verification, no idempotency handling, missing event types (subscription deleted, payment failed, invoice paid), no dead-letter queue for failed webhook processing, subscription state stored only in payment provider (no local sync).

**Step 3 — Emit robust billing integration**
Emit: webhook handler with signature verification, idempotent event processing (store processed event IDs), subscription state sync (local DB mirrors provider state).

**Step 4 — Usage-based billing (metered)**
For products where billing scales with usage (API calls, seats, storage): create a Stripe Meter, report usage records incrementally using `stripe.billing.meterEvents.create`, and handle overage pricing in the subscription's price tiers. Display current-period usage in the billing portal. For LemonSqueezy, use quantity-based subscriptions with a per-unit price and update quantity on usage checkpoints.

**Step 5 — Dunning management flow**
When `invoice.payment_failed` fires: Day 0 — notify customer, retry in 3 days. Day 3 — retry + second email. Day 7 — retry + urgent email + in-app warning banner. Day 14 — suspend account (read-only mode), email with payment link. Day 21 — cancel subscription, archive data with 30-day recovery window. Never hard-delete on cancellation.

#### Example

```typescript
// Stripe webhook — verified, idempotent, full lifecycle
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

app.post('/billing/webhook/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  const sig = req.headers['stripe-signature']!;
  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return res.status(400).json({ error: 'Invalid signature' });
  }

  const processed = await db.webhookEvent.findUnique({ where: { eventId: event.id } });
  if (processed) return res.json({ received: true, skipped: true });

  switch (event.type) {
    case 'customer.subscription.created':
    case 'customer.subscription.updated':
      await syncSubscription(event.data.object as Stripe.Subscription); break;
    case 'customer.subscription.deleted':
      await cancelSubscription(event.data.object as Stripe.Subscription); break;
    case 'invoice.payment_failed':
      await startDunningFlow(event.data.object as Stripe.Invoice); break;
    case 'invoice.payment_succeeded':
      await clearDunningState((event.data.object as Stripe.Invoice).customer as string); break;
  }

  await db.webhookEvent.create({ data: { eventId: event.id, type: event.type, processedAt: new Date() } });
  res.json({ received: true });
});

// LemonSqueezy webhook — alternative for Vietnam-based sellers
import crypto from 'crypto';

app.post('/billing/webhook/lemonsqueezy', express.raw({ type: 'application/json' }), async (req, res) => {
  const secret = process.env.LEMONSQUEEZY_WEBHOOK_SECRET!;
  const hmac = crypto.createHmac('sha256', secret);
  const digest = Buffer.from(hmac.update(req.body).digest('hex'), 'utf8');
  const signature = Buffer.from(req.headers['x-signature'] as string ?? '', 'utf8');

  if (!crypto.timingSafeEqual(digest, signature)) {
    return res.status(400).json({ error: 'Invalid signature' });
  }

  const payload = JSON.parse(req.body.toString());
  const eventName: string = payload.meta.event_name;

  switch (eventName) {
    case 'subscription_created':
    case 'subscription_updated':
      await syncLSSubscription(payload.data); break;
    case 'subscription_cancelled':
      await cancelLSSubscription(payload.data); break;
    case 'subscription_payment_failed':
      await startDunningFlow({ customerId: payload.data.attributes.customer_id }); break;
  }

  res.json({ received: true });
});

// Usage-based billing — report metered usage to Stripe
const reportUsage = async (tenantId: string, quantity: number) => {
  const subscription = await db.subscription.findUnique({ where: { tenantId } });
  await stripe.billing.meterEvents.create({
    event_name: 'api_call',
    payload: { stripe_customer_id: subscription!.stripeCustomerId, value: String(quantity) },
  });
};

// Dunning state machine
const startDunningFlow = async ({ customer }: { customer?: string | null; customerId?: string }) => {
  const tenantId = await getTenantByCustomer(customer ?? '');
  await db.tenant.update({ where: { id: tenantId }, data: { dunningStartedAt: new Date(), status: 'PAYMENT_FAILED' } });
  await emailQueue.add('dunning-day0', { tenantId }, { delay: 0 });
  await emailQueue.add('dunning-day3', { tenantId }, { delay: 3 * 24 * 60 * 60 * 1000 });
  await emailQueue.add('dunning-day7', { tenantId }, { delay: 7 * 24 * 60 * 60 * 1000 });
  await emailQueue.add('dunning-suspend', { tenantId }, { delay: 14 * 24 * 60 * 60 * 1000 });
  await emailQueue.add('dunning-cancel', { tenantId }, { delay: 21 * 24 * 60 * 60 * 1000 });
};
```

**Tax handling:**
- **Stripe Tax** — enable in Stripe dashboard, set `automatic_tax: { enabled: true }` on checkout sessions. Handles US state tax, EU VAT automatically.
- **Paddle** — acts as Merchant of Record (same as LemonSqueezy), handles all tax obligations. Good alternative if LemonSqueezy doesn't support your use case.
- **EU VAT** — if selling direct (not through MoR): collect VAT registration number, validate via VIES API, apply reverse charge for B2B EU transactions.
