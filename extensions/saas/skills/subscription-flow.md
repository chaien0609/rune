---
name: "subscription-flow"
pack: "@rune/saas"
description: "Subscription UI flows — pricing page, checkout, plan upgrades/downgrades, plan migration, annual/monthly toggle with proration preview, coupon codes, lifetime deal support, and cancellation with retention."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# subscription-flow

Subscription UI flows — pricing page, checkout, plan upgrades/downgrades, plan migration, annual/monthly toggle with proration preview, coupon codes, lifetime deal support, and cancellation with retention.

#### Workflow

**Step 1 — Detect subscription model**
Use Grep to find plan/tier definitions, feature flags, trial logic, checkout components. Read pricing config to understand: plan tiers, billing intervals, trial duration, feature gates, and upgrade/downgrade rules.

**Step 2 — Audit subscription UX**
Check for: pricing page without annual toggle, checkout without error recovery, no trial-to-paid conversion flow, plan change without proration explanation, cancellation without retention offer, missing feature gates on protected API routes.

**Step 3 — Emit subscription patterns**
Emit: type-safe plan configuration, feature gate middleware/hook, checkout flow with error handling, plan change with proration preview, cancellation flow with feedback collection, and trial expiry handling.

**Step 4 — Plan migration on downgrade**
When a user downgrades to a lower plan that has stricter limits (e.g., Pro 50 projects → Free 3 projects): DO NOT hard-delete over-limit data. Three options: (a) **Read-only grace period** — over-limit items become read-only for 30 days, user prompted to delete or upgrade; (b) **Hard limit** — block new item creation when at limit, existing items preserved; (c) **Grace period + export** — email user with export link, mark items for deletion after 60 days. Default recommendation: option (a) for good UX.

**Step 5 — Annual/monthly toggle + proration + coupons + lifetime deals**
Show annual price with savings badge ("Save 20%"). On plan change, call Stripe's proration preview endpoint and display "You'll be charged $X today" before confirming. For coupon codes: validate via `stripe.promotionCodes.list`, display discount amount/percentage and expiry. For lifetime deals (AppSumo, LemonSqueezy): create a one-time payment product, on `order_created` webhook set `subscription.plan = 'lifetime'` with `expiresAt = null` — lifetime access never expires.

#### Example

```typescript
// Type-safe plan configuration + feature gating
const PLANS = {
  free:     { price: 0,   limits: { projects: 3,   members: 1,   storage: '100MB' }, features: ['basic_analytics'] },
  pro:      { price: 29,  limits: { projects: 50,  members: 10,  storage: '10GB'  }, features: ['basic_analytics', 'advanced_analytics', 'api_access', 'priority_support'] },
  team:     { price: 79,  limits: { projects: -1,  members: -1,  storage: '100GB' }, features: ['basic_analytics', 'advanced_analytics', 'api_access', 'priority_support', 'sso', 'audit_log'] },
  lifetime: { price: 199, limits: { projects: -1,  members: 25,  storage: '50GB'  }, features: ['basic_analytics', 'advanced_analytics', 'api_access', 'priority_support'] },
} as const;

type PlanId = keyof typeof PLANS;
type Feature = typeof PLANS[PlanId]['features'][number];

function useFeatureGate(feature: Feature): { allowed: boolean; upgradeRequired: PlanId | null } {
  const { plan } = useSubscription();
  const allowed = (PLANS[plan].features as readonly string[]).includes(feature);
  if (allowed) return { allowed: true, upgradeRequired: null };
  const requiredPlan = (Object.entries(PLANS) as [PlanId, typeof PLANS[PlanId]][])
    .find(([_, p]) => (p.features as readonly string[]).includes(feature));
  return { allowed: false, upgradeRequired: requiredPlan?.[0] ?? null };
}

// Proration preview before plan change
const getProrationPreview = async (tenantId: string, newPriceId: string): Promise<number> => {
  const sub = await db.subscription.findUnique({ where: { tenantId } });
  const preview = await stripe.invoices.retrieveUpcoming({
    customer: sub!.stripeCustomerId,
    subscription: sub!.stripeSubscriptionId,
    subscription_items: [{ id: sub!.stripeItemId, price: newPriceId }],
    subscription_proration_behavior: 'create_prorations',
  });
  return preview.amount_due / 100; // dollars
};

// Coupon validation
const validateCoupon = async (code: string) => {
  const promos = await stripe.promotionCodes.list({ code, active: true, limit: 1 });
  if (!promos.data.length) throw new Error('Invalid or expired coupon');
  const promo = promos.data[0];
  const coupon = promo.coupon;
  return {
    id: promo.id,
    discount: coupon.percent_off ? `${coupon.percent_off}% off` : `$${(coupon.amount_off! / 100).toFixed(2)} off`,
    duration: coupon.duration,
  };
};

// Lifetime deal — LemonSqueezy one-time payment webhook
app.post('/billing/webhook/lemonsqueezy', express.raw({ type: 'application/json' }), async (req, res) => {
  // ...signature check...
  const payload = JSON.parse(req.body.toString());
  if (payload.meta.event_name === 'order_created') {
    const email = payload.data.attributes.user_email;
    const user = await db.user.findUnique({ where: { email } });
    if (user) {
      await db.subscription.upsert({
        where: { userId: user.id },
        update: { plan: 'lifetime', expiresAt: null },
        create: { userId: user.id, plan: 'lifetime', expiresAt: null },
      });
    }
  }
  res.json({ received: true });
});
```
