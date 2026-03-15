---
name: "onboarding-flow"
pack: "@rune/saas"
description: "User onboarding patterns — progressive disclosure, setup wizards, product tours, activation metrics (AARRR), empty states, re-engagement, and invite flows."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# onboarding-flow

User onboarding patterns — progressive disclosure, setup wizards, product tours, activation metrics (AARRR), empty states, re-engagement, and invite flows.

#### Workflow

**Step 1 — Detect onboarding state**
Use Grep to find onboarding code: `onboarding`, `setup`, `wizard`, `tour`, `welcome`, `getting-started`, `empty-state`, `invite`. Read the signup/post-registration flow to understand what happens after account creation.

**Step 2 — Audit activation funnel**
Check for: signup → empty dashboard (no guidance), missing setup wizard for critical config, no progress indicator during multi-step setup, empty states without action prompts, invite flow that doesn't pre-populate team context, no activation metric tracking.

**Step 3 — Emit onboarding patterns**
Emit: multi-step setup wizard with progress persistence (resume on reload), context-aware empty states with primary action, team invite flow with role selection, activation checklist component, and analytics event tracking for funnel steps.

**Step 4 — Activation metric framework (AARRR)**
Define your "Aha moment" — the single action that correlates with long-term retention. Common patterns: "created first project + invited one teammate" (Slack), "connected data source" (analytics tools), "ran first workflow" (automation tools). Instrument this event explicitly: `analytics.track('activation_achieved', { userId, tenantId, daysFromSignup })`. Track activation rate weekly. If <40% of signups activate in 7 days, the onboarding is broken.

**Step 5 — Re-engagement for dormant users**
Detect dormant: user signed up but never achieved activation, OR activated user with no activity in 14 days. Trigger: Day 3 after signup with no activation → in-app banner + email tip. Day 7 → personalized email with "here's what you haven't tried yet". Day 14 → offer a guided setup call or live demo. Track re-engagement conversion rate separately from organic activation.

#### Example

```typescript
// Onboarding wizard with progress persistence + analytics
const ONBOARDING_STEPS = ['profile', 'workspace', 'invite_team', 'first_project'] as const;
type Step = typeof ONBOARDING_STEPS[number];

function useOnboarding() {
  const [progress, setProgress] = useLocalStorage<Record<Step, boolean>>('onboarding', {
    profile: false, workspace: false, invite_team: false, first_project: false,
  });

  const currentStep = ONBOARDING_STEPS.find(step => !progress[step]) ?? null;
  const complete = (step: Step) => {
    setProgress(prev => ({ ...prev, [step]: true }));
    analytics.track('onboarding_step_complete', { step, totalSteps: ONBOARDING_STEPS.length });
  };
  const isComplete = currentStep === null;
  const percentComplete = (Object.values(progress).filter(Boolean).length / ONBOARDING_STEPS.length) * 100;
  return { currentStep, complete, isComplete, percentComplete, progress };
}

// Empty state library — 5 common SaaS empty states
const EMPTY_STATES = {
  no_projects: {
    icon: 'FolderIcon',
    title: 'No projects yet',
    description: 'Create your first project to get started.',
    cta: { label: 'Create Project', href: '/projects/new' },
  },
  no_team_members: {
    icon: 'UsersIcon',
    title: 'You\'re working alone',
    description: 'Invite your team to collaborate.',
    cta: { label: 'Invite Teammates', href: '/settings/members' },
  },
  no_data: {
    icon: 'ChartIcon',
    title: 'No data yet',
    description: 'Connect your first data source to see analytics.',
    cta: { label: 'Connect Source', href: '/integrations' },
  },
  no_integrations: {
    icon: 'PlugIcon',
    title: 'No integrations connected',
    description: 'Connect your tools to unlock automation.',
    cta: { label: 'Browse Integrations', href: '/integrations' },
  },
  no_billing: {
    icon: 'CreditCardIcon',
    title: 'No payment method',
    description: 'Add a payment method to unlock Pro features.',
    cta: { label: 'Add Payment Method', href: '/settings/billing' },
  },
} as const;

// Product tour — step-by-step spotlight with dismiss/snooze
interface TourStep { target: string; title: string; description: string; position: 'top' | 'bottom' | 'left' | 'right'; }

function useProductTour(tourId: string, steps: TourStep[]) {
  const [state, setState] = useLocalStorage<{ completed: boolean; dismissed: boolean; step: number }>(`tour:${tourId}`, {
    completed: false, dismissed: false, step: 0,
  });

  const advance = () => {
    if (state.step + 1 >= steps.length) {
      setState(s => ({ ...s, completed: true }));
      analytics.track('product_tour_completed', { tourId });
    } else {
      setState(s => ({ ...s, step: s.step + 1 }));
    }
  };

  const dismiss = (snoozeMinutes?: number) => {
    if (snoozeMinutes) {
      const snoozeUntil = Date.now() + snoozeMinutes * 60_000;
      setState(s => ({ ...s, dismissed: true }));
      localStorage.setItem(`tour:${tourId}:snooze`, String(snoozeUntil));
    } else {
      setState(s => ({ ...s, dismissed: true }));
      analytics.track('product_tour_dismissed', { tourId, atStep: state.step });
    }
  };

  const isSnoozed = () => {
    const snoozeUntil = Number(localStorage.getItem(`tour:${tourId}:snooze`) ?? 0);
    return Date.now() < snoozeUntil;
  };

  const active = !state.completed && !state.dismissed && !isSnoozed();
  return { active, currentStep: steps[state.step], stepIndex: state.step, advance, dismiss };
}

// Re-engagement detection — server-side cron
const detectDormantUsers = async () => {
  const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const dormant = await prisma.user.findMany({
    where: {
      createdAt: { lt: sevenDaysAgo },
      activatedAt: null, // never completed activation
      lastReEngagementEmailAt: null,
    },
    take: 500,
  });
  for (const user of dormant) {
    await emailQueue.add('re-engagement', { userId: user.id });
    await prisma.user.update({ where: { id: user.id }, data: { lastReEngagementEmailAt: new Date() } });
  }
};
```
