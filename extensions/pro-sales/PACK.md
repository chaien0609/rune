---
name: "@rune-pro/sales"
description: Sales enablement skills — account research, call preparation, competitive intelligence, outreach drafting, pipeline review, daily briefings.
metadata:
  author: runedev
  version: "0.1.0"
  layer: L4
  price: "included in Pro ($49)"
  target: Founders, sales engineers, account executives, BDRs
  pro: true
---

# @rune-pro/sales

## Purpose

Sales work in AI coding assistants fails because the AI has zero context about your prospects, competitors, or pipeline. You end up copy-pasting company info from LinkedIn, manually writing follow-up emails, and losing track of which deals need attention. This pack turns your AI assistant into a sales research and preparation machine — it researches accounts before calls, drafts multi-touch outreach sequences, tracks competitive positioning, reviews pipeline health, and generates daily briefings so you start every morning knowing exactly what needs attention.

## Triggers

- Auto-trigger: when `.rune/business/` directory exists (business context loaded)
- Auto-trigger: when task contains "prospect", "outreach", "cold email", "call prep", "pipeline", "deal", "account"
- `/sales research <company>` — deep account research
- `/sales call-prep <company>` — pre-call brief with talking points
- `/sales outreach <prospect>` — draft outreach sequence
- `/sales competitive <competitor>` — competitive intelligence update
- `/sales pipeline` — pipeline health review
- `/sales briefing` — daily morning briefing
- Called by `cook` (L1) when sales context detected in Phase 1.5

## Skills Included (6)

### account-research

Research prospects and companies using publicly available data. Produces structured account briefs that feed into call-prep and outreach.

#### Workflow

**Step 1 — Gather company intelligence**
Use `research` (L3) to find: company website, recent news/press releases, funding rounds (Crunchbase), tech stack (BuiltWith/Wappalyzer clues from job posts), team size, key decision makers. Check `.rune/business/context.md` for existing relationship notes. If `gh` is available and company has public repos, scan for tech stack signals.

**Step 2 — Identify pain points**
Cross-reference company profile with our product capabilities. Look for:
- **Explicit signals**: job postings mentioning problems we solve, complaints on social media, negative reviews of competitors they use
- **Implicit signals**: tech stack gaps (e.g., no analytics tool in stack), rapid hiring (scaling pain), recent funding (budget available)
- **Timing signals**: contract renewal season, new leadership, product launch

**Step 3 — Build account brief**
Save to `.rune/business/accounts/<company-slug>.md`:
- Company snapshot (size, industry, funding, tech stack)
- Key contacts (name, role, LinkedIn — public info only)
- Pain points (ranked by confidence: confirmed > inferred > speculated)
- Our value proposition (specific to their situation)
- Conversation starters (3-5 relevant talking points)
- Red flags (reasons deal might not close)

#### Example

```markdown
# Account Brief: Acme Analytics

## Snapshot
- Industry: B2B SaaS (product analytics)
- Size: 50-100 employees | Series B ($18M, 2024)
- Tech: React, Python, PostgreSQL, AWS
- Current tools: Mixpanel (analytics), Intercom (support)

## Key Contacts
- Sarah Chen, VP Engineering — LinkedIn (built their data pipeline)
- Mike Torres, CTO — LinkedIn (ex-Datadog, infrastructure focus)

## Pain Points
1. **Confirmed**: Hiring 3 data engineers → scaling data pipeline pain
2. **Inferred**: Using Mixpanel but job posts mention "custom analytics" → outgrowing current tool
3. **Speculated**: Series B pressure to show metrics → need better dashboards

## Our Value Proposition
Replace custom data pipeline with our managed solution.
Save 2-3 data engineer headcount ($400-600K/yr).
Migration from Mixpanel takes < 1 week.

## Conversation Starters
- "I noticed you're hiring data engineers — are you building custom analytics infrastructure?"
- "We helped [similar company] migrate from Mixpanel in 4 days"
- "Your Series B announcement mentioned data-driven growth — how's that going?"

## Red Flags
- CTO is ex-Datadog → may prefer building in-house
- Already using Mixpanel → switching cost friction
```

---

### call-preparation

Generate pre-call briefings with talking points, objection handling, and meeting agenda. Uses account-research output as input.

#### Workflow

**Step 1 — Load context**
Read account brief from `.rune/business/accounts/<company>.md`. If no brief exists, run account-research first. Also load: previous meeting notes (if any in `.rune/business/meetings/`), our product's current pricing, recent product updates (from git log or `.rune/releases/`).

**Step 2 — Build call brief**
Structure the brief for quick scanning (you'll read this 5 minutes before the call):
- **Objective**: One sentence — what's the ONE thing we want from this call?
- **Agenda** (3-4 items, 30min default): intro → discovery → demo/value → next steps
- **Discovery questions**: 5-7 questions that uncover pain. Open-ended, not yes/no. Ordered from safe → probing → closing.
- **Talking points**: 3 key messages mapped to their specific pain points
- **Objection handling**: Top 3 likely objections with responses (Acknowledge → Reframe → Evidence → Ask)
- **Competitive landmines**: If they mention competitor X, here's our response
- **Next step options**: What to propose depending on how the call goes (warm → demo, cold → follow-up content, hot → trial setup)

**Step 3 — Save and announce**
Save to `.rune/business/meetings/<date>-<company>.md`. Announce: "Call prep ready. Key objective: [X]. Top risk: [Y]."

#### Example

```markdown
# Call Prep: Acme Analytics — 2025-03-15

## Objective
Get agreement to run a 2-week proof-of-concept with their real data.

## Agenda (30 min)
1. (5 min) Intro — confirm their current data challenges
2. (10 min) Discovery — understand pipeline complexity and pain
3. (10 min) Value — show migration path from Mixpanel
4. (5 min) Next steps — propose POC with success criteria

## Discovery Questions
1. "Walk me through how data gets from your app to a dashboard today."
2. "How many people touch the data pipeline weekly?"
3. "When was the last time a dashboard was wrong and you didn't know?"
4. "What would you do with 3 extra data engineers if you didn't need them on pipeline?"
5. "What does success look like for your data team this quarter?"

## Objection Handling
**"We're happy with Mixpanel"**
→ Acknowledge: "Mixpanel is great for event analytics."
→ Reframe: "The gap is usually in custom metrics — things Mixpanel can't track natively."
→ Evidence: "[Similar company] used Mixpanel + us side-by-side, eventually consolidated."
→ Ask: "What's one metric you wish you could track but can't today?"

**"We don't have bandwidth to migrate"**
→ Acknowledge: "Migration is always a concern."
→ Reframe: "Our migration takes 4 days, not 4 months. We handle 90% of it."
→ Evidence: "Here's a case study — [company] migrated in 3 days with zero downtime."
→ Ask: "Would a 2-week POC running parallel to Mixpanel reduce the risk?"
```

---

### competitive-intel

Track competitor pricing, features, positioning, and recent moves. Produces actionable battle cards for sales conversations.

#### Workflow

**Step 1 — Define competitors**
Check `.rune/business/competitive/` for existing analysis. If present, load latest report. If not, ask user to list 3-5 direct competitors. Use `research` (L3) to find: pricing pages, feature comparison pages, recent changelog/blog posts, G2/Capterra reviews (sentiment + complaints).

**Step 2 — Build battle cards**
For each competitor, create a sales-ready battle card:
- **Positioning**: Their one-liner vs ours
- **Strengths** (be honest — credibility matters): What they do better
- **Weaknesses**: Where we win
- **Common objections**: "Why not use [Competitor]?" with responses
- **Win stories**: customers who switched from them to us (if available)
- **Pricing comparison**: Their tiers vs ours, total cost of ownership
- **Trigger phrases**: If prospect says X, they're comparing us to this competitor

**Step 3 — Save battle cards**
Save to `.rune/business/competitive/battlecards/<competitor>.md`. Include last-updated date. Flag any cards > 30 days old as potentially stale.

#### Example

```markdown
# Battle Card: Mixpanel

**Last updated**: 2025-03-15 | Status: Current

## Their Positioning
"Product analytics that helps you understand your users"

## Our Positioning
"Custom analytics infrastructure — own your data pipeline"

## Where They Win
- Easier setup (SDK drops in, no infrastructure)
- Larger free tier (20M events vs our 10K)
- More pre-built reports (funnels, retention, flows)

## Where We Win
- Custom metrics (anything you can SQL, we can track)
- Data ownership (your database, not their cloud)
- No event limits on paid plans (flat pricing)
- SQL access (analysts can query directly, no export needed)

## When Prospect Mentions Mixpanel
Ask: "What percentage of your metrics are custom vs standard?"
If > 30% custom → strong fit for us (Mixpanel struggles with custom)
If < 10% custom → weak fit, they're better off with Mixpanel (be honest)
```

---

### outreach-drafting

Draft multi-touch cold outreach sequences (email + LinkedIn). Personalized to prospect using account-research data.

#### Workflow

**Step 1 — Load prospect context**
Read account brief from `.rune/business/accounts/<company>.md`. If no brief exists, run account-research first. Identify: prospect's role, likely pain points, any mutual connections, recent activity (blog posts, tweets, talks they gave).

**Step 2 — Draft sequence**
Generate a 4-touch sequence over 14 days:
- **Touch 1 (Day 1)**: Email — personalized opener referencing something specific about them, one pain point, one value prop, soft CTA (ask a question, not book a demo)
- **Touch 2 (Day 3)**: LinkedIn connection request — short, no pitch, reference something genuine
- **Touch 3 (Day 7)**: Email — different angle, share relevant content (case study, blog post, data point), harder CTA (offer specific time for a call)
- **Touch 4 (Day 14)**: Email — breakup email, acknowledge they're busy, leave the door open, no guilt

Rules:
- Max 150 words per email (respect their time)
- No "I hope this email finds you well" or "Just following up" — banned phrases
- Every email must have ONE clear ask
- Personalization must be genuine (reference real things, not "I love your company")

**Step 3 — Save and review**
Save to `.rune/business/outreach/<prospect-slug>.md`. Mark as DRAFT — user MUST review before sending. Never auto-send.

#### Example

```markdown
# Outreach: Sarah Chen (VP Eng, Acme Analytics)

## Touch 1 — Email (Day 1)
Subject: Your data engineer hiring sprint

Sarah — saw you're hiring 3 data engineers focused on pipeline infrastructure.

We built [product] specifically for teams outgrowing Mixpanel's custom metrics limits.
[Similar company] cut their pipeline team from 5 to 2 after switching.

Quick question: are you building custom analytics on top of Mixpanel, or replacing it entirely?

[signature]

## Touch 2 — LinkedIn (Day 3)
Hi Sarah — fellow infrastructure nerd here. Your talk at DataEng Conf
on pipeline reliability was spot on, especially the circuit breaker pattern.
Would love to connect.

## Touch 3 — Email (Day 7)
Subject: How [similar company] saved $400K on data infrastructure

Sarah — sharing a case study that might be relevant given your
current data engineering buildout.

[Similar company] (Series B, similar stack) migrated from
Mixpanel + custom pipeline to us in 4 days. Saved 2 headcount.

Worth a 20-minute call next Tuesday or Thursday?

## Touch 4 — Email (Day 14)
Subject: Closing the loop

Sarah — I know pipeline projects are all-consuming.

If the timing isn't right, no worries at all. I'll check back
in Q3 when your new engineers are ramped.

If data infrastructure is a pain point before then, I'm here.
```

---

### pipeline-review

Analyze deal pipeline health, flag at-risk deals, and generate weekly pipeline summary.

#### Workflow

**Step 1 — Load pipeline data**
Check for pipeline data in `.rune/business/pipeline.md` or `.rune/business/pipeline.csv`. If no structured data exists, ask user to provide: deal name, company, stage (prospect → qualified → proposal → negotiation → closed), deal value, last activity date, expected close date, owner.

**Step 2 — Health analysis**
For each deal, assess risk signals:
- **Stale**: No activity > 14 days → flag as at-risk
- **Slipping**: Past expected close date → flag as overdue
- **Single-threaded**: Only one contact → flag as fragile (need multi-threading)
- **No next step**: No clear next action defined → flag as drifting
- **Value mismatch**: Deal value > historical average for stage → verify qualification

Generate pipeline metrics:
- Total pipeline value (weighted by stage probability)
- Coverage ratio (pipeline / quota — healthy = 3x+)
- Average deal age by stage
- Conversion rates between stages
- Forecast accuracy (if historical data available)

**Step 3 — Generate review**
Save to `.rune/business/pipeline-review-<date>.md`:
- Dashboard: total value, deal count, weighted forecast
- At-risk deals (sorted by value, highest first)
- Action items: specific next steps for each at-risk deal
- Win/loss patterns: what's working, what's not

#### Example

```markdown
# Pipeline Review — Week of 2025-03-15

## Dashboard
| Metric | Value | Health |
|--------|-------|--------|
| Total pipeline | $340K | - |
| Weighted forecast | $142K | 🟡 Below $180K target |
| Coverage ratio | 2.1x | 🔴 Need 3x |
| Deals in pipeline | 12 | - |
| Average age | 28 days | 🟡 Target < 21 days |

## At-Risk Deals
1. **Acme Analytics** ($80K) — STALE: Last activity 18 days ago.
   Action: Send Sarah the case study, propose POC timeline.
2. **Beta Corp** ($45K) — SINGLE-THREADED: Only talking to junior PM.
   Action: Ask for intro to VP Eng or CTO.
3. **Gamma Inc** ($30K) — SLIPPING: Expected close was March 10.
   Action: Call to understand blocker. Offer extended trial if budget timing.

## Wins This Week
- Delta Co ($25K) closed — 22-day cycle, champion was CTO
- Pattern: CTO-led deals close 2x faster than bottom-up
```

---

### daily-briefing

Generate a morning briefing summarizing key accounts, tasks, and priorities for the day.

#### Workflow

**Step 1 — Collect daily context**
Scan `.rune/business/` for:
- Pipeline data: deals with activity due today or overdue
- Meeting notes: any follow-ups from yesterday's calls
- Account briefs: recently updated accounts
- Outreach sequences: touches due today
- Calendar hints: if user mentions meetings in conversation

**Step 2 — Prioritize**
Rank items by urgency × value:
- **Red** (do now): overdue follow-ups on high-value deals, meeting prep needed today
- **Yellow** (do today): scheduled outreach touches, pipeline updates to log
- **Green** (this week): research tasks, content to review, low-priority follow-ups

**Step 3 — Generate briefing**
Keep it scannable — max 1 page. Format:

```markdown
# Daily Briefing — Monday, March 15

## Today's Priority (Red)
- [ ] Follow up with Acme Analytics (Sarah Chen) — send POC proposal. Deal: $80K, stale 18 days.
- [ ] Prep for 2pm call with Beta Corp — review account brief, bring multi-thread strategy.

## Due Today (Yellow)
- [ ] Send Touch 3 email to Epsilon Labs (Jake) — case study angle.
- [ ] Log Delta Co win in pipeline — update stage to Closed Won.

## This Week (Green)
- [ ] Research Zeta Inc (new prospect from conference).
- [ ] Update competitive battlecard for Mixpanel (30+ days old).
- [ ] Weekly pipeline review (Friday).
```

Save to `.rune/business/briefings/<date>.md`.

## Connections

```
Calls → research (L3): prospect/company data, competitive intelligence
Calls → trend-scout (L3): market context for positioning conversations
Calls → marketing (L2): align outreach messaging with marketing campaigns
Calls → @rune-pro/product (L4): product roadmap info for customer conversations
Calls → scout (L2): scan codebase for integration/API details to mention in calls
Called By ← cook (L1): Phase 1.5 when sales context detected
Called By ← ba (L2): when requirements gathering involves sales stakeholder input
```

## Business Memory Integration

This pack extends `.rune/business/` with sales-specific state:

```
.rune/business/
├── accounts/             — account research briefs
│   └── <company-slug>.md
├── meetings/             — call prep and meeting notes
│   └── <date>-<company>.md
├── outreach/             — outreach sequences (DRAFT until reviewed)
│   └── <prospect-slug>.md
├── competitive/
│   └── battlecards/      — sales-ready battle cards per competitor
│       └── <competitor>.md
├── pipeline.md           — deal pipeline data
├── pipeline-review-<date>.md — weekly pipeline reviews
└── briefings/            — daily morning briefings
    └── <date>.md
```

Session-bridge (L3) auto-loads `.rune/business/context.md` at session start when Pro packs are detected.

## Constraints

1. MUST save all outputs to `.rune/business/` — sales artifacts are versioned alongside code
2. MUST mark outreach sequences as DRAFT — never auto-send emails or messages
3. MUST use only publicly available information for account research — no scraping private data
4. MUST flag stale competitive data (> 30 days) — outdated intel is worse than no intel
5. MUST include red flags in account briefs — honest assessment prevents wasted effort
6. MUST personalize outreach with genuine references — no generic "I love your company" filler
7. MUST define clear next steps for every pipeline deal — no deal without an action item

## Sharp Edges

| Failure Mode | Severity | Mitigation |
|---|---|---|
| Outreach sent without human review → brand damage | CRITICAL | All sequences saved as DRAFT, user must review |
| Account research uses private/scraped data → legal risk | CRITICAL | Step 1 explicitly limits to public sources |
| Competitive intel presented as fact when speculated → lost credibility | HIGH | Confidence labels: confirmed / inferred / speculated |
| Pipeline review without fresh data → false confidence | HIGH | Timestamps on all data, staleness warnings |
| Generic outreach that sounds AI-generated → ignored | HIGH | Banned phrases list, genuine personalization required |
| Call prep without objection handling → unprepared | MEDIUM | Step 2 mandates top 3 objections with AREA framework |

## Done When

- Account brief covers company snapshot, pain points, value prop, and red flags
- Call prep includes objective, discovery questions, objection handling, and next step options
- Outreach sequence has 4 touches over 14 days with genuine personalization
- Competitive battle cards cover strengths (honest), weaknesses, and trigger phrases
- Pipeline review identifies at-risk deals with specific action items
- Daily briefing prioritizes tasks by urgency × deal value
- All outputs saved to `.rune/business/` and marked appropriately (DRAFT for outreach)

## Cost Profile

~2000-4000 tokens input, ~1500-3000 tokens output per skill invocation. Sonnet default for most skills. Opus recommended for competitive-intel (deeper reasoning about positioning) and pipeline-review (pattern recognition across deals).
