---
name: plan
description: Create structured implementation plans from requirements. Produces master plan + phase files for enterprise-scale project management. Master plan = overview (<80 lines). Phase files = execution detail (<150 lines each). Each session handles 1 phase. Uses opus for deep reasoning.
metadata:
  author: runedev
  version: "1.0.0"
  layer: L2
  model: opus
  group: creation
  tools: "Read, Write, Edit, Glob, Grep"
---

# plan

## Purpose

Strategic planning engine for the Rune ecosystem. Produces a **master plan + phase files** architecture — NOT a single monolithic plan. The master plan is a concise overview (<80 lines) that references separate phase files, each containing enough detail (<150 lines) that ANY model can execute with high accuracy.

**Design principle: Plan for the weakest coder.** Phase files are designed so that even an Amateur-level model (Haiku) can execute them with minimal errors. When the plan satisfies the Amateur's needs, every model benefits — Junior (Sonnet) executes near-perfectly, Senior (Opus) executes flawlessly.

This is enterprise-grade project management: BA produces WHAT → Plan produces HOW (structured into phases) → ANY coder executes each phase with full context.

<HARD-GATE>
NEVER produce a single monolithic plan file for non-trivial tasks.
Non-trivial = 3+ phases OR 5+ files OR estimated > 100 LOC total change.
For non-trivial tasks: MUST produce master plan + separate phase files.
For trivial tasks (1-2 phases, < 5 files): inline plan is acceptable.
</HARD-GATE>

## Architecture: Master Plan + Phase Files

```
.rune/
  plan-<feature>.md          ← Master plan: phases overview, goals, status tracker (<80 lines)
  plan-<feature>-phase1.md   ← Phase 1 detail: tasks, acceptance criteria, files to touch (<150 lines)
  plan-<feature>-phase2.md   ← Phase 2 detail
  ...
```

### Why This Architecture

- **Big context = even Opus misses details and makes mistakes**
- **Small context = Sonnet handles correctly, Opus has zero mistakes**
- Phase isolation prevents cross-contamination of concerns
- Each session starts clean with only the relevant phase loaded
- Coder (Sonnet/Haiku) can execute a phase file without needing the full plan

### Size Constraints

| File | Max Lines | Content |
|------|-----------|---------|
| Master plan | 80 lines | Overview, phase table, key decisions, status |
| Phase file | 200 lines | Amateur-proof template: data flow, contracts, tasks, failures, NFRs, rejections, cross-phase |
| Total phases | Max 8 | If > 8 phases, split into sub-projects |

## Modes

### Implementation Mode (default)
Standard implementation planning — decompose task into phased steps with code details.

### Feature Spec Mode
Product-oriented planning — write a feature specification before implementation.

**Triggers:**
- User says "spec", "feature spec", "write spec", "PRD"
- `/rune plan spec <feature>`

### Roadmap Mode
High-level multi-feature planning — organize features into milestones.

**Triggers:**
- User says "roadmap", "milestone", "release plan", "what to build next"
- `/rune plan roadmap`

## Triggers

- Called by `cook` when task scope > 1 file (Implementation Mode)
- Called by `team` for high-level task decomposition
- `/rune plan <task>` — manual planning
- `/rune plan spec <feature>` — feature specification
- `/rune plan roadmap` — roadmap planning
- Auto-trigger: when user says "implement", "build", "create" with complex scope

## Calls (outbound)

- `scout` (L2): scan codebase for existing patterns, conventions, and structure
- `brainstorm` (L2): when multiple valid approaches exist
- `research` (L3): external knowledge lookup
- `sequential-thinking` (L3): complex architecture with many trade-offs
- L4 extension packs: domain-specific architecture patterns
- `neural-memory` | Before architecture decisions | Recall past decisions on similar problems

## Called By (inbound)

- `cook` (L1): Phase 2 PLAN
- `team` (L1): task decomposition into parallel workstreams
- `brainstorm` (L2): when idea needs structuring
- `rescue` (L1): plan refactoring strategy
- `ba` (L2): hand-off after requirements complete
- `scaffold` (L1): Phase 3 architecture planning
- `skill-forge` (L2): plan structure for new skill
- User: `/rune plan` direct invocation

## Data Flow

### Feeds Into →

- `cook` (L1): master plan + phase files → cook's Phase 2-4 execution roadmap
- `team` (L1): task decomposition + wave grouping → team's parallel workstream dispatch
- `fix` (L2): phase file tasks → fix's implementation targets
- `test` (L2): phase file test tasks → test's RED phase targets

### Fed By ←

- `ba` (L2): Requirements Document → plan's primary input (locked decisions, user stories)
- `scout` (L2): codebase analysis → plan's convention/pattern awareness
- `neural-memory` (external): past architectural decisions → plan's precedent context

### Feedback Loops ↻

- `plan` ↔ `brainstorm`: plan requests options when multiple approaches exist → brainstorm generates options → plan selects and structures the chosen approach
- `plan` ↔ `cook`: cook discovers plan gaps during implementation → plan updates phase files → cook resumes with corrected tasks

## Executable Steps (Implementation Mode)

### Step 1 — Gather Context

**Check for Requirements Document first**: Use `Glob` to check for `.rune/features/*/requirements.md`. If a Requirements Document exists (produced by `rune:ba`), read it and use it as the primary input — it contains user stories, acceptance criteria, scope, and constraints. Do NOT re-gather requirements that BA already elicited.

Use findings from `rune:scout` if already available. If not, invoke `rune:scout` with the project root to scan directory structure, detect framework, identify key files, and extract existing patterns. Do NOT skip this step — plans without context produce wrong file paths.

Call `neural-memory` (Recall Mode) to check for past architecture decisions on similar problems before making new ones.

### Step 2 — Classify Complexity

Determine if the task needs master plan + phase files or inline plan:

| Criteria | Inline Plan | Master + Phase Files |
|----------|-------------|---------------------|
| Phases | 1-2 | 3+ |
| Files touched | < 5 | 5+ |
| Estimated LOC | < 100 | 100+ |
| Cross-module | No | Yes |
| Session span | Single session | Multi-session |

If ANY "Master + Phase Files" criterion is true → produce master plan + phase files.

### Step 3 — Decompose into Phases

Group related work into phases. Each phase is a coherent unit that:
- Can be completed in one session
- Has a clear "done when" condition
- Produces testable output
- Is independent enough to execute without other phases loaded

<HARD-GATE>
Each phase MUST be completable by ANY coder model (including Haiku) with ONLY the phase file loaded.
If the coder would need to read the master plan or other phase files to execute → the phase file is missing detail.
Phase files are SELF-CONTAINED execution instructions — designed for the weakest model to succeed.
</HARD-GATE>

Phase decomposition rules:
- **Foundation first**: types, schemas, core engine
- **Dependencies before consumers**: create what's imported before the importer
- **Test alongside**: each phase includes its own test tasks
- **Max 5-7 tasks per phase**: if more, split the phase
- **Vertical slices over horizontal layers**: prefer "auth end-to-end" over "all models → all APIs → all UI"

### Wave-Based Task Grouping (within each phase)

Tasks inside a phase MUST be organized into **waves** based on dependency analysis. Independent tasks within the same wave can execute in parallel.

```
## Tasks

### Wave 1 (parallel — no dependencies)
- [ ] Task 1 — Create types/interfaces
  - File: `src/types.ts` (new)
  - ...
- [ ] Task 2 — Create validation schema
  - File: `src/validation.ts` (new)
  - ...

### Wave 2 (depends on Wave 1)
- [ ] Task 3 — Implement core logic (imports types from Task 1)
  - File: `src/core.ts` (new)
  - depends_on: [Task 1]
  - ...

### Wave 3 (depends on Wave 2)
- [ ] Task 4 — Wire into API endpoint (imports core from Task 3)
  - File: `src/routes/api.ts` (modify)
  - depends_on: [Task 3]
  - ...
- [ ] Task 5 — Write integration tests (tests core from Task 3)
  - File: `tests/core.test.ts` (new)
  - depends_on: [Task 3]
  - ...
```

**Wave rules:**
- Wave 1 = tasks with zero dependencies (types, schemas, configs) — always first
- Subsequent waves: a task goes in the earliest wave where ALL its `depends_on` tasks are in prior waves
- Tasks within the same wave have NO dependencies on each other → safe for parallel dispatch
- `depends_on` field is MANDATORY for Wave 2+ tasks — explicit is better than implicit
- `team` orchestrator can dispatch wave tasks as parallel subagents; solo `cook` executes sequentially within a wave but respects wave ordering

### Step 4 — Write Master Plan File

Save to `.rune/plan-<feature>.md`:

```markdown
# Feature: <name>

## Overview
<1-3 sentences: what and why>

## Phases
| # | Name | Status | Plan File | Summary |
|---|------|--------|-----------|---------|
| 1 | Foundation | ⬚ Pending | plan-X-phase1.md | Types, core engine, basic UI |
| 2 | Interaction | ⬚ Pending | plan-X-phase2.md | Dialogue, combat, items |
| 3 | Polish | ⬚ Pending | plan-X-phase3.md | Effects, sounds, game over |

## Key Decisions
- <decision 1 — chosen approach and why>
- <decision 2>

## Decision Compliance
- Decisions (locked): [list from requirements.md — plan MUST honor these]
- Discretion (agent): [list — agent chose X because Y]
- Deferred: [list — explicitly excluded from this feature]

## Architecture
<brief system diagram or component list — NOT implementation detail>

## Dependencies
- <external dep>: <status>

## Risks
- <risk>: <mitigation>
```

**Max 80 lines.** No implementation details — that's what phase files are for.

### Step 4.5 — Workflow Registry (Complex Features Only)

> From agency-agents (msitarzewski/agency-agents, 50.8k★): "Every route is an entry point. Every worker is a workflow. If it's missing from the registry, it doesn't exist."

For complex features (4+ phases OR 3+ user-facing workflows), build a **4-view Workflow Registry** before writing phase files. This catches missing pieces, dead ends, and integration gaps at plan time — not implementation time.

**Skip conditions**: trivial tasks, inline plans, single-workflow features.

**4 cross-referenced views:**

```markdown
## Workflow Registry

### View 1: By Workflow
| Workflow | Entry Point | Components Touched | Exit Point | Phase |
|----------|-------------|-------------------|------------|-------|
| User signup | POST /auth/register | AuthService, UserRepo, EmailService | 201 + email sent | Phase 1 |
| Password reset | POST /auth/reset | AuthService, EmailService, TokenRepo | 200 + reset email | Phase 2 |

### View 2: By Component
| Component | Used By Workflows | Owner Phase | Status |
|-----------|-------------------|-------------|--------|
| AuthService | signup, login, reset | Phase 1 | Planned |
| EmailService | signup, reset, invite | Phase 2 | Planned |
| TokenRepo | reset, invite | Phase 2 | Missing ← RED FLAG |

### View 3: By User Journey
| Journey | Steps (workflow chain) | Happy Path | Error Path |
|---------|----------------------|------------|------------|
| New user → first action | signup → verify email → login → onboard | 4 steps | signup fail, email bounce |

### View 4: By State
| Step | User Sees | DB State | Logs | Operator Sees |
|------|-----------|----------|------|---------------|
| After signup | "Check your email" | user.status=pending | user.created event | New user in admin |
| After verify | Dashboard | user.status=active | user.verified event | Active user count +1 |
```

**Validation rules:**
- Every component in View 2 MUST appear in at least one workflow in View 1 — orphaned components = dead code
- Every workflow in View 1 MUST map to a phase — unphased workflows will be forgotten
- "Missing" status in View 2 = **red flag** — component needed but not planned in any phase → add to a phase or create new phase
- Every user journey step in View 3 MUST have a corresponding state row in View 4

**Output**: Add the registry to the master plan file (it fits within the 80-line budget when tables are compact). Phase files reference it but don't duplicate it.

### Step 5 — Write Phase Files

For each phase, save to `.rune/plan-<feature>-phase<N>.md`.

Phase files follow the **Amateur-Proof Template** — designed so that even the weakest model can execute without guessing. Every section exists because an Amateur said "I need this to code correctly."

```markdown
# Phase N: <name>

## Goal
<What this phase delivers — 1-2 sentences>

## Data Flow
<5-line ASCII diagram showing how data moves through this phase's components>
```
User Input → validateInput() → calculateProfit() → formatResult() → API Response
                                      ↓
                                 TradeEntry[]
```

## Code Contracts
<Function signatures, interfaces, schemas that this phase MUST implement>
<This is the MOST IMPORTANT section — coder implements these contracts>

```typescript
interface TradeEntry {
  side: 'long' | 'short';
  entryPrice: number;
  exitPrice: number;
  quantity: number;
}

interface ProfitResult {
  netPnL: number;
  totalFees: number;
  winRate: number;
}

function calculateProfit(entries: TradeEntry[]): ProfitResult;
function validateInput(raw: unknown): TradeEntry[];  // throws ValidationError
```

## Tasks

Each task MUST include: **File** (exact path), **Test** (test file or N/A), **Verify** (shell command), **Commit** (semantic message). Granularity: 2-5 min per task. If >10min, decompose.

- [ ] Task 1 — Create calculateProfit function
  - Req: REQ-001 (P&L calculation)
  - File: `src/foo/bar.ts` (new)
  - Test: `tests/foo/bar.test.ts` (new)
  - Verify: `npm test -- --grep "calculateProfit"`
  - Commit: `feat(trading): add calculateProfit with fee calculation`
  - Logic: sum entries by side, apply fees (0.1% per trade), return net P&L
  - Edge: empty array → return { netPnL: 0, totalFees: 0, winRate: 0 }
- [ ] Task 2 — Add input validation
  - Req: REQ-002 (input validation)
  - File: `src/foo/baz.ts` (modify)
  - Test: `tests/foo/baz.test.ts` (new)
  - Verify: `npm test -- --grep "validateInput"`
  - Commit: `feat(trading): add input validation for trade entries`
  - Logic: check side is 'long'|'short', prices > 0, quantity > 0
- [ ] Task 3 — Write integration tests
  - Req: REQ-001, REQ-002 (integration coverage)
  - File: `tests/foo/bar.test.ts` (modify)
  - Test: N/A — this IS the test task
  - Verify: `npm test -- --grep "trading" && npx tsc --noEmit`
  - Commit: `test(trading): add integration tests for edge cases`
  - Cases: happy path, empty input, negative values, overflow

## Failure Scenarios
<What should happen when things go wrong — coder MUST implement these>

| When | Then | Error Type |
|------|------|-----------|
| entries is empty array | return zero-value ProfitResult | No error (valid edge case) |
| entry has negative price | throw ValidationError("price must be positive") | ValidationError |
| entry has quantity = 0 | throw ValidationError("quantity must be > 0") | ValidationError |
| calculation overflows Number.MAX_SAFE_INTEGER | use BigInt or throw OverflowError | OverflowError |

## Performance Constraints
<Non-functional requirements — skip if not applicable>

| Metric | Requirement | Why |
|--------|-------------|-----|
| Input size | Must handle 10,000 entries | Production data volume |
| Response time | < 100ms for 10K entries | Real-time dashboard |
| Memory | < 50MB for 10K entries | Container memory limit |

## Rejection Criteria (DO NOT)
<Anti-patterns the coder MUST avoid — things that seem right but are wrong>

- ❌ DO NOT use `toFixed()` for financial calculations — use Decimal.js or integer cents
- ❌ DO NOT mutate the input array — create new objects (immutability rule)
- ❌ DO NOT use `any` type — full TypeScript strict
- ❌ DO NOT import from Phase 2+ files — this phase is self-contained

## Cross-Phase Context
<What this phase assumes from previous phases / what future phases expect from this one>

- **Assumes**: Phase 1 created `src/shared/types.ts` with base types
- **Exports for Phase 3**: `calculateProfit()` will be imported by `src/dashboard/PnLCard.tsx`
- **Interface contract**: ProfitResult shape MUST NOT change — Phase 3 depends on it

## Acceptance Criteria
- [ ] All tasks marked done
- [ ] Tests pass with 80%+ coverage on new code
- [ ] No TypeScript errors (`tsc --noEmit` passes)
- [ ] Failure scenarios all handled (table above)
- [ ] Performance: calculateProfit(10K entries) < 100ms
- [ ] No `any` types, no mutation, no `toFixed()` for money

## Traceability Matrix
| Req ID | Requirement | Task(s) | Test(s) | Status |
|--------|-------------|---------|---------|--------|
| REQ-001 | P&L calculation with fees | Task 1 | `tests/foo/bar.test.ts` | ⬚ |
| REQ-002 | Input validation | Task 2 | `tests/foo/baz.test.ts` | ⬚ |

Every requirement from BA's Requirements Document MUST appear in this matrix. Missing requirement = incomplete phase. `completion-gate` checks this matrix during verification.

## Files Touched
- `src/foo/bar.ts` — new
- `src/foo/baz.ts` — modify
- `tests/foo/bar.test.ts` — new
```

**Max 200 lines per phase file.** Must be self-contained — coder should NOT need to read master plan or other phases to execute.

<HARD-GATE>
Every phase file MUST include ALL of these sections (Amateur-Proof Checklist):
1. ✅ Data Flow — ASCII diagram of how data moves
2. ✅ Code Contracts — function signatures, interfaces, types
3. ✅ Tasks — with file paths, logic description, edge cases
4. ✅ Failure Scenarios — table of when/then/error for each error case
5. ✅ Rejection Criteria — explicit "DO NOT" anti-patterns
6. ✅ Cross-Phase Context — what's assumed from prior phases, what's exported for future phases
7. ✅ Acceptance Criteria — testable, includes performance if applicable
8. ✅ Test tasks — every code task has corresponding tests
9. ✅ Traceability Matrix — every BA requirement mapped to tasks and tests (skip if no BA requirements exist)

A phase missing ANY of sections 1-7 is INCOMPLETE — the weakest coder will guess wrong.
Performance Constraints section is optional (only when NFRs apply).
</HARD-GATE>

### Step 5.5 — Completeness Scoring (Alternatives)

When presenting alternative approaches (from brainstorm or Step 3 decisions), rate each with **Completeness X/10**:

| Score | Meaning |
|-------|---------|
| 9-10 | Complete — all edge cases, full coverage, production-ready |
| 7-8 | Happy path covered, some edges skipped |
| 4-6 | Shortcut — defers significant work |
| 1-3 | Minimal viable, debt guaranteed |

**Always recommend higher-completeness option.** With AI-assisted coding, the marginal cost of completeness is near-zero. Show dual effort estimates for each approach: `(human: ~X / AI: ~Y)`.

**Anti-pattern**: "Option B saves 70 LOC" → 70 LOC delta is meaningless with AI. Choose complete. The last 10% of coverage is where production bugs hide.

> Source: garrytan/gstack v0.9.0 — "Boil the Lake" principle.

### Step 6 — Present and Get Approval

Present the **master plan** to user (NOT all phase files). User reviews:
- Phase breakdown
- Key decisions
- Risks
- Completeness scores for chosen approach (from Step 5.5)

Wait for explicit approval ("go", "proceed", "yes") before writing phase files.

If user requests changes → revise and re-present.

### Step 7 — Execution Handoff

After approval, the execution flow is:

```
1. Cook loads master plan → identifies current phase (first ⬚ Pending)
2. Cook loads ONLY that phase's file
3. Coder executes tasks in the phase file
4. Mark tasks done in phase file as completed
5. When phase complete → update master plan status: ⬚ → ✅
6. Next session: load master plan → find next ⬚ phase → load phase file → execute
```

**Model selection for execution:**
- Opus plans phases (this skill)
- Sonnet/Haiku executes them (cook → fix)
- If Sonnet makes small errors → fix lightly (cheaper than using Opus for execution)

## Inline Plan (Trivial Tasks)

For trivial tasks (1-2 phases, < 5 files, < 100 LOC):

Skip master plan + phase files. Produce inline plan directly:

```
## Plan: [Task Name]

### Changes
1. [file]: [what to change] — [function signature]
2. [file]: [what to change]

### Tests
- [test file]: [test cases]

### Risks
- [risk]: [mitigation]

Awaiting approval.
```

## Re-Planning (Dynamic Adaptation)

When cook encounters unexpected conditions during execution:

### Trigger Conditions
- Phase execution hits max debug-fix loops (3)
- New files discovered outside the plan scope
- Dependency change alters the approach
- User requests scope change

### Re-Plan Protocol

1. **Read the master plan** + **current phase file**
2. **Read delta context**: what changed, what failed
3. **Assess impact**: which remaining phases are affected?
4. **Revise**:
   - Mark completed phases as ✅ in master plan
   - Modify affected phase files
   - Add new phases if scope expanded
   - **Do NOT rewrite completed phases**
5. **Present revised master plan** with diff summary
6. **Get approval** before resuming

## Feature Spec Mode

When invoked in Feature Spec Mode, produce a structured specification.

### Steps

**Step 1 — Problem Statement**
- What problem? Who has it? Current workaround?

**Step 2 — User Stories**
- Primary story, 2-3 secondary, edge cases
- Format: `As a [persona], I want to [action] so that [benefit]`

**Step 3 — Acceptance Criteria**
- `GIVEN [context] WHEN [action] THEN [result]`
- Happy path + error cases + performance criteria

**Step 4 — Scope Definition**
- In scope / Out of scope / Dependencies / Open questions

**Step 5 — Write Spec File**
Save to `.rune/features/<feature-name>/spec.md`

After spec approved → transition to Implementation Mode.

## Roadmap Mode

When invoked in Roadmap Mode, produce a prioritized feature roadmap.

### Steps

**Step 1 — Inventory**
Scan project for: open issues, TODO/FIXME comments, planned features.

**Step 2 — Prioritize (ICE Scoring)**
Impact × Confidence × Ease (each 1-10). Sort descending.

**Step 3 — Group into Milestones**
- Milestone 1: top 3-5 features by ICE
- Milestone 2: next 3-5
- Backlog: remaining

**Step 4 — Write Roadmap**
Save to `.rune/roadmap.md`

## Output Format

### Master Plan (`.rune/plan-<feature>.md`)
```markdown
# Feature: <name>

## Overview
<1-3 sentences: what and why>

## Phases
| # | Name | Status | Plan File | Summary |
|---|------|--------|-----------|---------|
| 1 | [name] | ⬚ Pending | plan-X-phase1.md | [1-line summary] |

## Key Decisions
- [decision — chosen approach and why]

## Architecture
<brief system diagram — NOT implementation detail>

## Dependencies / Risks
- [dep/risk]: [status/mitigation]
```
Max 80 lines. No implementation details.

### Phase File (`.rune/plan-<feature>-phase<N>.md`)
7 mandatory sections (Amateur-Proof Template):
1. **Goal** — 1-2 sentences
2. **Data Flow** — 5-line ASCII diagram
3. **Code Contracts** — function signatures, interfaces
4. **Tasks** — file paths, logic, edge cases, tests
5. **Failure Scenarios** — when/then/error table
6. **Rejection Criteria** — explicit DO NOTs
7. **Cross-Phase Context** — assumes from prior, exports for future
8. **Acceptance Criteria** — testable conditions

Max 200 lines. Self-contained — coder needs ONLY this file.

### Inline Plan (trivial tasks)
```
## Plan: [Task Name]
### Changes
1. [file]: [what] — [signature]
### Tests
- [test file]: [cases]
### Risks
- [risk]: [mitigation]
```

## Outcome Block (Mandatory Plan Output Section)

Every plan output — whether master plan, phase file, or inline plan — MUST end with an **Outcome Block**. This closes the loop between planning and execution.

```markdown
## Outcome Block

### What Was Planned
<1-2 sentences: what this plan delivers when all phases execute successfully>

### Immediate Next Action
<The SINGLE next action the executor should take right now — not a list, one action>
Example: "Load phase 1 file and execute Wave 1 tasks (create types, validation schema)"

### How to Measure Success
| Metric | Target | How to Check |
|--------|--------|-------------|
| Tests pass | 100% green | `npm test` or `pytest -v` |
| Type errors | 0 | `tsc --noEmit` or `mypy` |
| Phase complete | All tasks ✅ | phase file task list |
| <domain metric> | <value> | <command> |
```

**Why this matters**: Plans without a clear "what to do NOW" cause executor drift — agents re-read the plan, re-analyze, and pick arbitrary starting points. The Immediate Next Action eliminates ambiguity: there is exactly ONE right first move.

**Rules:**
1. Outcome Block is the LAST section in every plan output (master plan, phase file, inline plan)
2. "Immediate Next Action" = ONE action, present tense, imperative mood. Not a list.
3. "How to Measure" table MUST include at least one verifiable shell command
4. For phase files: Immediate Next Action = "Execute Wave 1: [list Wave 1 task names]"
5. For master plans: Immediate Next Action = "Await approval, then load phase 1 file"

> Source: Affitor/affiliate-skills — every output ends with: outcome achieved + immediate next action + measurement metric ("What happened → What to do now → How to measure").

## Constraints

1. MUST produce master plan + phase files for non-trivial tasks (3+ phases OR 5+ files OR 100+ LOC)
2. MUST keep master plan under 80 lines — overview only, no implementation details
3. MUST keep each phase file under 200 lines — self-contained, Amateur-proof
4. MUST include exact file paths for every task — no vague "set up the database"
5. MUST include test tasks for every phase that produces code
6. MUST include ALL Amateur-Proof sections: data flow, code contracts, tasks, failure scenarios, rejection criteria, cross-phase context, acceptance criteria
7. MUST order phases by dependency — don't plan phase 3 before phase 1's output exists
8. MUST get user approval before writing phase files
9. Phase files MUST be self-contained — coder should NOT need master plan to execute
10. Max 8 phases per master plan — if more, split into sub-projects
11. MUST include failure scenarios table — what happens when things go wrong
12. MUST include rejection criteria — explicit "DO NOT" anti-patterns to prevent common mistakes
13. MUST include cross-phase context — what's assumed from prior phases, what's exported for future

## Sharp Edges

| Failure Mode | Severity | Mitigation |
|---|---|---|
| Monolithic plan file that overflows context | CRITICAL | HARD-GATE: non-trivial tasks MUST use master + phase files |
| Phase file too vague for Amateur to execute | CRITICAL | Amateur-Proof template: ALL 7 mandatory sections required |
| Coder uses wrong approach (toFixed for money, mutation) | CRITICAL | Rejection Criteria section: explicit "DO NOT" list prevents common traps |
| Coder doesn't handle errors properly | HIGH | Failure Scenarios table: when/then/error for EVERY error case |
| Coder doesn't know what other phases expect | HIGH | Cross-Phase Context: explicit imports/exports between phases |
| Coder over-engineers or under-engineers perf | HIGH | Performance Constraints: specific metrics with thresholds |
| Master plan contains implementation detail | HIGH | Max 80 lines, overview only — detail goes in phase files |
| Phase file references other phase files | HIGH | Phase files are self-contained — cross-phase section handles this |
| Plan without scout context — invented file paths | CRITICAL | Step 1: scout first, always |
| Phase with zero test tasks | CRITICAL | HARD-GATE rejects it |
| 10+ phases overwhelming the master plan | MEDIUM | Max 8 phases — split into sub-projects if more |
| Task without File path or Verify command | HIGH | Every task MUST have File + Test + Verify + Commit fields — no vague "implement the feature" tasks |
| Horizontal layer planning (all models → all APIs → all UI) | HIGH | Vertical slices parallelize better. Use wave-based grouping: independent tasks in same wave, dependent tasks in later waves |
| Tasks without `depends_on` in Wave 2+ | MEDIUM | Implicit dependencies break parallel dispatch. Every Wave 2+ task MUST declare `depends_on` |
| Plan ignores locked Decisions from BA | CRITICAL | Decision Compliance section cross-checks requirements.md — locked decisions are non-negotiable |
| Complex feature missing Workflow Registry — components planned but never wired | HIGH | Step 4.5: 4-view registry catches orphaned components, unphased workflows, and missing state transitions before phase files are written |
| Recommending shortcut approach without Completeness Score | MEDIUM | Step 5.5: every alternative needs X/10 Completeness score + dual effort estimate (human vs AI). "Saves 70 LOC" is not a reason when AI makes the delta cost minutes |
| Plan output missing Outcome Block | MEDIUM | Every plan output MUST end with Outcome Block (What Was Planned + Immediate Next Action + How to Measure) — executor drift when omitted |
| Outcome Block "Next Action" is a list, not one action | LOW | One action only — ambiguity about where to start causes re-analysis and lost context |

## Self-Validation

```
SELF-VALIDATION (run before presenting plan to user):
- [ ] Every task has a clear file path — no "update relevant files" vagueness
- [ ] Wave dependencies are acyclic — no task depends on a task in the same or later wave
- [ ] Every code-producing phase has at least one test task
- [ ] Phase files have ALL Amateur-Proof sections (data flow, code contracts, failure scenarios, rejection criteria)
- [ ] Locked decisions from BA are reflected in plan — none contradicted or ignored
- [ ] Every BA requirement has a corresponding Req ID in at least one phase's Traceability Matrix
```

## Done When

- Complexity classified (inline vs master + phase files)
- Scout output read and conventions/patterns identified
- BA requirements consumed (if available)
- Master plan written (< 80 lines) with phase table and key decisions
- Phase files written (< 200 lines each) with ALL Amateur-Proof sections:
  - Data flow diagram, code contracts, tasks with edge cases
  - Failure scenarios table, rejection criteria (DO NOTs)
  - Cross-phase context (assumes/exports), acceptance criteria
- Every code-producing phase has test tasks
- Master plan presented to user with "Awaiting Approval"
- User has explicitly approved
- Self-Validation: all checks passed
- Outcome Block present in every plan output (master plan, phase files, inline plan)
- Outcome Block contains: What Was Planned + Immediate Next Action (single action) + How to Measure table

## Cost Profile

~3000-8000 tokens input, ~2000-5000 tokens output (master + all phase files). Opus for architectural reasoning. Most expensive L2 skill but runs infrequently. Phase files are written once, executed by cheaper models (Sonnet/Haiku).
