---
name: "feature-flags"
pack: "@rune/saas"
description: "Feature flag management — gradual rollouts, kill switches, A/B testing, user-segment targeting, and stale flag cleanup. Supports self-hosted (Unleash, custom Redis) and managed (LaunchDarkly, Statsig, Flagsmith)."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# feature-flags

Feature flag management — gradual rollouts, kill switches, A/B testing, user-segment targeting, and stale flag cleanup. Supports self-hosted (Unleash, custom Redis) and managed (LaunchDarkly, Statsig, Flagsmith).

#### Flag Types

| Type | Use Case | Example |
|---|---|---|
| Boolean | Simple on/off for a feature | `new_dashboard_ui` |
| Percentage rollout | Gradual release 1% → 100% | `redesigned_editor: 25%` |
| User segment | Specific users/orgs first | `beta_users`, `enterprise_plan` |
| A/B test | Compare variants with metrics | `checkout_flow: variant_a / variant_b` |
| Kill switch | Instant disable on failure | `payment_processor_v2` |
| Environment | Dev/staging/prod separation | Auto by `NODE_ENV` |

#### Rollout Pattern: Canary → Gradual → GA

```
1% (internal + beta users) → 10% → 25% → 50% → 100% → cleanup flag after 30 days at 100%
```

#### Workflow

**Step 1 — Identify feature boundary**
Before writing code, define the flag: name (kebab-case, descriptive), default value (false = safe default), targeting rules (who sees it first), and planned cleanup date. Document in your flag provider dashboard.

**Step 2 — Create flag with targeting rules**
In Unleash/LaunchDarkly/Flagsmith: create flag with gradual rollout strategy. Start at 0%. Add a "beta users" segment for internal testing before any percentage rollout. Set environment-specific defaults: always-on in dev, gradual in staging, starts at 0% in prod.

**Step 3 — Implement client/server evaluation**
Client: evaluate flag in a React hook, never inline. Server: evaluate in middleware or at request start, attach result to request context. Never evaluate flags inside hot loops — cache the result for the request lifetime.

**Step 4 — Add analytics event tracking**
Every flag evaluation on a user-facing feature should fire an analytics event: `feature_flag_evaluated` with `{ flag, variant, userId, tenantId }`. This enables funnel analysis by variant and measures the rollout's impact on key metrics.

**Step 5 — Schedule flag cleanup**
Flags that have been at 100% for >30 days are stale. Run a weekly lint job: grep all flag keys used in code, compare against provider's flag list, flag mismatches (code uses a flag that was deleted → runtime error, or flag exists but never referenced → cleanup candidate). Remove stale flags from both code and provider in the same PR.

#### Example

```typescript
// Custom Redis-based flag evaluation (self-hosted, zero SaaS dependency)
import { Redis } from 'ioredis';
const redis = new Redis(process.env.REDIS_URL!);

interface FlagConfig {
  enabled: boolean;
  percentage?: number;          // 0-100 for gradual rollout
  allowedUsers?: string[];      // canary user IDs
  allowedPlans?: string[];      // plan-based targeting
}

const evaluateFlag = async (
  flagKey: string,
  ctx: { userId: string; tenantId: string; plan: string }
): Promise<boolean> => {
  const raw = await redis.get(`flag:${flagKey}`);
  if (!raw) return false; // default off = safe
  const config: FlagConfig = JSON.parse(raw);
  if (!config.enabled) return false;
  if (config.allowedUsers?.includes(ctx.userId)) return true;
  if (config.allowedPlans?.includes(ctx.plan)) return true;
  if (config.percentage !== undefined) {
    // Deterministic: same user always gets same bucket
    const hash = parseInt(ctx.userId.slice(-8), 16) % 100;
    return hash < config.percentage;
  }
  return config.enabled;
};

// React hook — evaluate once per render cycle, never in loops
function useFlag(flagKey: string): boolean {
  const { user } = useAuth();
  const { data: enabled = false } = useQuery({
    queryKey: ['flag', flagKey, user?.id],
    queryFn: () => fetchFlag(flagKey),
    staleTime: 30_000, // cache 30s — flags don't change every millisecond
  });
  return enabled;
}

// Server middleware — evaluate at request boundary, attach to context
const flagMiddleware = (flagKey: string) => async (req: Request, res: Response, next: NextFunction) => {
  req.flags = req.flags ?? {};
  req.flags[flagKey] = await evaluateFlag(flagKey, {
    userId: req.user!.id,
    tenantId: req.tenantId!,
    plan: req.user!.plan,
  });
  next();
};

// Usage in route — flag already evaluated, no async needed
app.get('/api/checkout', flagMiddleware('new_checkout_v2'), (req, res) => {
  if (req.flags['new_checkout_v2']) {
    return checkoutV2Handler(req, res);
  }
  return checkoutV1Handler(req, res);
});

// Stale flag detection — run weekly in CI
import { execSync } from 'child_process';

const findStaleFlags = async () => {
  const flagsInCode = execSync('grep -r "useFlag\\|evaluateFlag" src/ --include="*.ts" -h')
    .toString()
    .match(/(?:useFlag|evaluateFlag)\(['"]([^'"]+)['"]/g)
    ?.map(m => m.match(/['"]([^'"]+)['"]/)?.[1])
    .filter(Boolean) ?? [];

  const flagsInProvider = await redis.keys('flag:*').then(keys => keys.map(k => k.replace('flag:', '')));
  const stale = flagsInProvider.filter(f => !flagsInCode.includes(f));
  const missing = flagsInCode.filter(f => !flagsInProvider.includes(f));
  return { stale, missing };
};
```

**Sharp edges for flags:**
- Never evaluate flags on hot paths (e.g., inside `Array.map` over 1000 items) — cache the flag state at the top of the function.
- In tests: mock flag evaluation at the provider level, not by conditionally skipping flag checks. Every code path should be testable with flags on and off.
- Flag dependency chains (flag A enables flag B) — avoid. If you need compound logic, evaluate both flags independently and combine in application code. Provider-level dependencies are invisible in code review.
- Percentage rollout is not the same as A/B test — percentage rollout has no control group. For A/B tests, always keep a 50/50 split or a defined control group.
