---
name: "@rune-pro/product"
description: Product management skills — PRD writing, roadmap planning, KPI tracking, release communications, user research synthesis, competitive analysis.
metadata:
  author: runedev
  version: "0.1.0"
  layer: L4
  price: "included in Pro ($49)"
  target: Product managers, technical founders, dev leads
  pro: true
---

# @rune-pro/product

## Purpose

Product work in AI coding assistants fails because the AI writes code without product context: features get built without clear acceptance criteria, roadmaps exist in someone's head but never reach the codebase, release notes are an afterthought copy-pasted from git log, and competitive analysis is a one-time Google doc that's outdated within a week. This pack bridges the PM-developer gap — structured PRDs that feed directly into `plan` → `cook`, roadmaps with ICE scoring that prioritize what to build next, KPI definitions that connect metrics to code, and release communications generated from actual git diffs.

## Triggers

- Auto-trigger: when `.rune/business/` directory exists (business context loaded)
- Auto-trigger: when task contains "PRD", "spec", "roadmap", "release notes", "KPI", "metrics"
- `/product write-prd` — generate product requirements document
- `/product roadmap` — create or update feature roadmap
- `/product metrics` — define and track product KPIs
- `/product release-notes` — auto-generate release communications
- `/product user-research` — synthesize user feedback into insights
- `/product competitive` — competitive analysis and positioning
- Called by `cook` (L1) when product context detected in Phase 1.5

## Skills Included (6)

### feature-spec

Write structured Product Requirements Documents (PRDs) from loose requirements. Output feeds directly into `rune:plan` for implementation planning.

#### Workflow

**Step 1 — Gather raw input**
Read user's feature request. Check for existing `.rune/business/context.md` (company info), `.rune/business/glossary.md` (terminology), and `.rune/business/people.md` (stakeholders). If business context exists, load it to align the PRD with company strategy and terminology.

**Step 2 — Structure the PRD**
Produce a structured PRD with these sections:
- **Problem Statement**: What problem, who has it, current workaround, cost of inaction
- **User Stories**: Primary (1) + secondary (2-3), format: `As a [persona], I want [action] so that [benefit]`
- **Acceptance Criteria**: `GIVEN [context] WHEN [action] THEN [result]` — happy path + error cases + edge cases
- **Scope**: In scope / Out of scope / Deferred (future) / Non-goals
- **Success Metrics**: 2-3 measurable KPIs with targets (e.g., "reduce onboarding time from 5min to 2min")
- **Dependencies**: external APIs, data sources, team resources
- **Open Questions**: unresolved decisions that need stakeholder input

**Step 3 — Validate and save**
Check: every user story has acceptance criteria, success metrics are measurable (not "improve UX"), scope explicitly lists what's NOT included. Save to `.rune/features/<name>/prd.md`. Announce: "PRD ready for review. Run `/rune plan` to create implementation plan from this spec."

#### Example

```markdown
# PRD: User Onboarding Flow

## Problem Statement
New users drop off at 68% during onboarding (Step 3: connect data source).
Current flow requires 7 steps. Competitors (PostHog, Amplitude) do it in 3.
Cost: ~$12K/month in lost conversions (based on $45 CAC × 270 drop-offs).

## User Stories
**Primary**: As a new user, I want to connect my data source in under 2 minutes
so that I can see my first dashboard without reading documentation.

**Secondary**: As an admin, I want to pre-configure data sources for my team
so that new members skip the connection step entirely.

## Acceptance Criteria
- GIVEN a new user WHEN they reach onboarding THEN they see max 3 steps
- GIVEN a user selects PostgreSQL WHEN they paste a connection string
  THEN the system validates and shows sample data within 5 seconds
- GIVEN an invalid connection string WHEN validation fails
  THEN the error message explains what's wrong and how to fix it

## Success Metrics
- Onboarding completion rate: 32% → 65% (within 30 days)
- Time to first dashboard: 5min → 2min (p50)
- Support tickets about onboarding: 45/week → 15/week

## Scope
**In**: Simplified 3-step flow, auto-detect DB type, connection validation
**Out**: OAuth-based connections (Phase 2), SSO onboarding, mobile onboarding
**Deferred**: Team onboarding templates, guided tours
```

---

### roadmap

Prioritize features into milestones using ICE scoring (Impact × Confidence × Ease). Produces a living roadmap document that updates as features ship.

#### Workflow

**Step 1 — Inventory features**
Scan for feature sources: `.rune/features/*/prd.md` (existing PRDs), GitHub issues (if `gh` available), TODO/FIXME comments in code (via Grep for `TODO|FIXME|HACK`), and user-provided feature list. Deduplicate and normalize into a feature list with: name, one-line description, source.

**Step 2 — Score with ICE**
For each feature, assess:
- **Impact** (1-10): How much does this move the needle on core metrics? 10 = directly drives revenue/retention, 1 = nice-to-have cosmetic
- **Confidence** (1-10): How sure are we about the impact? 10 = data-backed (analytics, user research), 1 = gut feeling
- **Ease** (1-10): How easy is this to build? 10 = < 1 day, single file. 1 = multi-sprint, requires new infra

Calculate ICE score = Impact × Confidence × Ease. Sort descending.

**Step 3 — Group into milestones**
- **Now** (Milestone 1): Top 3-5 features by ICE score. These ship in current sprint/cycle.
- **Next** (Milestone 2): Next 3-5 features. Planned but not started.
- **Later** (Backlog): Everything else. Revisit quarterly.

Flag features where Impact is high (8+) but Confidence is low (< 5) → these need user research before committing.

**Step 4 — Write roadmap**
Save to `.rune/roadmap.md`. Include: milestone table with ICE scores, dependency arrows between features, "needs research" callouts, and last-updated timestamp. If a previous roadmap exists, diff and highlight what changed.

#### Example

```markdown
# Product Roadmap — Updated 2025-03-15

## Now (Sprint 12)
| Feature | Impact | Confidence | Ease | ICE | Status |
|---------|--------|------------|------|-----|--------|
| Simplified onboarding | 9 | 8 | 6 | 432 | 🔄 In Progress |
| Dashboard export PDF | 7 | 9 | 8 | 504 | ⬚ Ready |
| Billing page redesign | 8 | 7 | 5 | 280 | ⬚ Ready |

## Next (Sprint 13-14)
| Feature | Impact | Confidence | Ease | ICE | Notes |
|---------|--------|------------|------|-----|-------|
| Team workspaces | 9 | 6 | 3 | 162 | Needs arch design |
| Webhook integrations | 7 | 8 | 4 | 224 | |
| ⚠️ AI query builder | 10 | 3 | 2 | 60 | Needs user research |

## Later (Backlog)
- White-label dashboard (ICE: 48)
- Mobile app (ICE: 36)
- SOC2 compliance (ICE: depends on enterprise pipeline)
```

---

### metrics-tracking

Define product KPIs, connect them to code instrumentation, and generate weekly metrics review templates.

#### Workflow

**Step 1 — Define metrics framework**
Classify metrics into the AARRR funnel (Pirate Metrics):
- **Acquisition**: How do users find us? (signups, landing page visits)
- **Activation**: Do they have a good first experience? (onboarding completion, time-to-value)
- **Retention**: Do they come back? (DAU/MAU, week-1 retention, churn rate)
- **Revenue**: Do they pay? (MRR, ARPU, conversion rate, LTV)
- **Referral**: Do they tell others? (NPS, invite rate, viral coefficient)

For each metric: define the calculation formula, data source (analytics event, DB query, API), target value, and alert threshold.

**Step 2 — Generate tracking code**
For each metric, emit the analytics event or DB query needed to compute it. Check if the project uses an analytics provider (PostHog, Mixpanel, GA4, custom) by scanning for analytics imports. Generate typed event definitions that match the provider's SDK.

**Step 3 — Create review template**
Save weekly metrics review template to `.rune/business/metrics-review.md`:
- Current values vs targets (with traffic-light indicators)
- Week-over-week trends (↑ ↓ →)
- Anomaly flags (metric moved > 2 standard deviations)
- Action items for red metrics

#### Example

```typescript
// Metrics definition — connects business KPIs to code
interface MetricDefinition {
  name: string;
  category: 'acquisition' | 'activation' | 'retention' | 'revenue' | 'referral';
  formula: string;
  source: 'analytics' | 'database' | 'api';
  target: number;
  unit: string;
  alertThreshold: number;
}

const PRODUCT_METRICS: MetricDefinition[] = [
  {
    name: 'onboarding_completion_rate',
    category: 'activation',
    formula: 'completed_onboarding / started_onboarding * 100',
    source: 'analytics',
    target: 65,
    unit: '%',
    alertThreshold: 50,
  },
  {
    name: 'weekly_active_users',
    category: 'retention',
    formula: 'COUNT(DISTINCT user_id) WHERE last_active > NOW() - INTERVAL 7 DAY',
    source: 'database',
    target: 1000,
    unit: 'users',
    alertThreshold: 800,
  },
];

// Weekly review query
const METRICS_REVIEW_SQL = `
SELECT
  metric_name,
  current_value,
  previous_value,
  ROUND((current_value - previous_value) / previous_value * 100, 1) AS wow_change,
  target_value,
  CASE
    WHEN current_value >= target_value THEN '🟢'
    WHEN current_value >= target_value * 0.8 THEN '🟡'
    ELSE '🔴'
  END AS status
FROM weekly_metrics
WHERE week = CURRENT_DATE - INTERVAL (DAYOFWEEK(CURRENT_DATE) - 1) DAY
ORDER BY category, metric_name;
`;
```

---

### release-comms

Auto-generate release notes, changelog entries, and stakeholder updates from git history. Understands conventional commits and maps them to user-facing language.

#### Workflow

**Step 1 — Collect changes**
Run `git log --oneline <since>..<until>` to get commit range. Parse conventional commits to extract: type (feat/fix/refactor), scope, description, breaking changes. If PR descriptions exist (`gh pr list --state merged`), include them for richer context.

**Step 2 — Classify for audience**
Map commits to audience-appropriate language:
- **User-facing changelog**: `feat` → "New", `fix` → "Fixed", `perf` → "Improved". Skip `chore`, `ci`, `refactor` (internal only).
- **Stakeholder update**: Group by business impact. Lead with metrics-moving features.
- **Technical release notes**: Include all changes with migration steps for breaking changes.

**Step 3 — Generate outputs**
Produce 3 outputs:
1. **CHANGELOG.md entry**: Keep a Changelog format, appended to existing file
2. **Release notes** (GitHub): markdown for `gh release create`
3. **Stakeholder email**: plain-language summary for non-technical readers

Save drafts to `.rune/releases/<version>/`. User reviews before publishing.

#### Example

```markdown
## [1.4.0] - 2025-03-15

### New
- **Simplified onboarding**: 3-step flow replaces 7-step wizard.
  Connect your data source in under 2 minutes. (#142)
- **Dashboard PDF export**: Export any dashboard as a branded PDF
  with custom date ranges. (#156)

### Fixed
- Charts no longer flicker when switching between date ranges (#163)
- Connection timeout increased to 30s for slow databases (#171)

### Improved
- Dashboard load time reduced by 40% via query caching (#158)

---

**Stakeholder summary**: This release focuses on onboarding and
export — the two highest-impact items from our roadmap. Expected
impact: onboarding completion 32% → 50%+ (based on funnel analysis).
Next release targets team workspaces (Sprint 13).
```

---

### user-research-synthesis

Convert raw user feedback (interviews, surveys, support tickets, NPS responses) into structured insights with actionable recommendations.

#### Workflow

**Step 1 — Ingest feedback**
Accept feedback in any format: pasted text, CSV, markdown notes. Normalize into structured entries: `{ source, date, user_segment, verbatim, sentiment }`. If `.rune/business/people.md` exists, map feedback to known user personas.

**Step 2 — Theme extraction**
Group feedback into themes using affinity mapping:
- Cluster similar statements (semantic similarity, not keyword matching)
- Name each theme with a user-centric label (e.g., "Onboarding is confusing" not "UX issue")
- Count frequency per theme
- Assign sentiment distribution per theme (positive/neutral/negative)

**Step 3 — Prioritize and recommend**
For each theme, assess:
- **Frequency**: How many users mentioned this? (1-2 = anecdotal, 5+ = pattern, 10+ = systemic)
- **Severity**: How much does this block the user? (cosmetic → frustrating → blocking → churning)
- **Segment**: Which user segment is affected? (free users, paying, enterprise)

Produce prioritized insight cards with: theme name, evidence quotes (3-5 verbatims), frequency, severity, affected segment, and recommended action (build feature / fix UX / update docs / deprioritize).

Save to `.rune/business/research/<date>-synthesis.md`.

#### Example

```markdown
# User Research Synthesis — March 2025
Source: 23 user interviews + 156 NPS responses + 45 support tickets

## Theme 1: "I can't find my dashboards" (Frequency: 18, Severity: Frustrating)
**Segment**: Free users (primarily)
**Verbatims**:
- "I created a dashboard last week and now I can't find it anywhere"
- "The sidebar doesn't show my recently viewed dashboards"
- "Search doesn't work for dashboard names with special characters"

**Recommendation**: Add recent dashboards to sidebar + fix search.
ICE: Impact 7, Confidence 9, Ease 7 = 441. **Add to roadmap NOW.**

## Theme 2: "Pricing is confusing" (Frequency: 12, Severity: Blocking → Churning)
**Segment**: Trial users converting to paid
**Verbatims**:
- "I don't understand the difference between Pro and Enterprise"
- "Why do I need to talk to sales for Enterprise pricing?"

**Recommendation**: Simplify pricing page, add feature comparison table.
ICE: Impact 9, Confidence 8, Ease 6 = 432. **Add to roadmap NOW.**
```

---

### competitive-analysis

Track competitor features, pricing, and positioning. Produces structured comparison matrices and identifies gaps/opportunities.

#### Workflow

**Step 1 — Define competitive landscape**
Identify 3-5 direct competitors and 2-3 indirect competitors. For each: name, URL, category (direct/indirect), primary differentiator. Check `.rune/business/context.md` for existing competitor data. If not present, ask user to list top competitors.

**Step 2 — Feature comparison matrix**
Build a feature-by-feature comparison table. For each feature:
- **Us**: ✅ (have it), 🔄 (building), ⬚ (planned), ❌ (no plan)
- **Competitor**: ✅ (have it), ❌ (don't have it), ❓ (unknown)
- **Importance**: Must-have / Nice-to-have / Differentiator

Flag features where ALL competitors have it but we don't → **table stakes gap**.
Flag features where NO competitor has it but we could → **differentiation opportunity**.

**Step 3 — Pricing comparison**
Compare pricing tiers: free tier limits, entry price, per-seat vs per-usage, enterprise pricing model. Calculate price-per-feature ratio for common use cases.

**Step 4 — Generate report**
Save to `.rune/business/competitive/<date>-analysis.md`. Include: landscape summary, feature matrix, pricing table, gap analysis (what to build), positioning recommendation (how to differentiate).

#### Example

```markdown
# Competitive Analysis — March 2025

## Feature Matrix
| Feature | Us | PostHog | Mixpanel | Amplitude |
|---------|-----|---------|----------|-----------|
| Event tracking | ✅ | ✅ | ✅ | ✅ |
| Funnel analysis | ✅ | ✅ | ✅ | ✅ |
| A/B testing | 🔄 | ✅ | ❌ | ✅ |
| Session replay | ❌ | ✅ | ❌ | ✅ |
| SQL query | ✅ | ✅ | ❌ | ❌ |
| AI insights | ⬚ | ❌ | 🔄 | ❌ |

## Gaps
- **Table stakes**: Session replay — all major competitors have it.
  Recommendation: Add to roadmap Q2 (ICE: 7×6×3 = 126).
- **Opportunity**: AI-powered insights — nobody does it well yet.
  Recommendation: Differentiation play. Prioritize over session replay.

## Pricing
| Tier | Us | PostHog | Mixpanel |
|------|-----|---------|----------|
| Free | 10K events | 1M events | 20M events |
| Pro | $49/mo | $0 (usage) | $25/mo |
| Enterprise | Custom | Custom | Custom |

**Warning**: Our free tier is 100x smaller than PostHog.
Recommendation: Increase to 100K events or differentiate on features.
```

## Connections

```
Calls → plan (L2): after PRD complete, feed into implementation planning
Calls → brainstorm (L2): feature ideation, competitive positioning options
Calls → research (L3): competitor data, market trends, user behavior benchmarks
Calls → trend-scout (L3): market intelligence for roadmap prioritization
Calls → marketing (L2): align release comms with marketing strategy
Calls → git (L3): extract commit history for release notes generation
Calls → scout (L2): scan codebase for tracking instrumentation
Called By ← cook (L1): Phase 1.5 when product context detected
Called By ← ba (L2): requirements validation against product strategy
```

## Business Memory Integration

This pack extends `.rune/business/` with product-specific state:

```
.rune/business/
├── context.md          — company info, product lines, market position
├── glossary.md          — company-specific terminology
├── people.md            — stakeholders, roles, preferences
├── processes.md         — business workflows, approval chains
├── metrics-review.md    — weekly KPI review template
├── competitive/         — competitive analysis snapshots
│   └── <date>-analysis.md
└── research/            — user research synthesis
    └── <date>-synthesis.md
```

Session-bridge (L3) auto-loads `.rune/business/context.md` at session start when Pro packs are detected.

## Constraints

1. MUST save all outputs to `.rune/` — product artifacts are versioned alongside code
2. MUST use measurable success metrics — "improve UX" is not a metric, "reduce onboarding time from 5min to 2min" is
3. MUST map PRD acceptance criteria to testable conditions — if it can't be tested, it's not a requirement
4. MUST NOT generate release notes without reading actual git diff — no hallucinated features
5. MUST classify metrics into AARRR framework — prevents vanity metrics
6. MUST flag low-confidence roadmap items for user research — don't build on assumptions
7. MUST include "Out of Scope" in every PRD — prevents scope creep

## Sharp Edges

| Failure Mode | Severity | Mitigation |
|---|---|---|
| PRD without acceptance criteria → vague implementation | CRITICAL | Step 2 enforces GIVEN/WHEN/THEN format for every story |
| Roadmap scored on gut feeling instead of data | HIGH | ICE scoring: Confidence < 5 flags item for user research |
| Release notes list internal refactors to users | HIGH | Step 2 filters: only `feat` and `fix` reach user-facing changelog |
| Competitive analysis based on outdated data | MEDIUM | Timestamp all reports, suggest quarterly refresh |
| Metrics without alert thresholds → silent degradation | MEDIUM | Every metric requires `alertThreshold` alongside `target` |
| User research with < 5 data points presented as "pattern" | MEDIUM | Frequency labeling: 1-2 = anecdotal, 5+ = pattern, 10+ = systemic |

## Done When

- PRD written with problem statement, user stories, acceptance criteria, scope, and success metrics
- Roadmap includes ICE scores with clear NOW / NEXT / LATER buckets
- Metrics framework covers all 5 AARRR categories with formulas and targets
- Release notes generated from actual git history (not hallucinated)
- User research synthesized into prioritized insight cards with evidence
- Competitive matrix identifies table-stakes gaps and differentiation opportunities
- All outputs saved to `.rune/` and announced to user

## Cost Profile

~2000-4000 tokens input, ~1500-3000 tokens output per skill invocation. Sonnet default for most skills. Opus recommended for competitive-analysis (deeper reasoning about positioning) and roadmap (ICE scoring with business context).
