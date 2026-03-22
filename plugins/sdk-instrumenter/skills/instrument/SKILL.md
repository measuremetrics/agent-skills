---
name: instrument
description: "Identify and instrument business processes in a Next.js app with the Measure SDK (@measuremetrics/measure-sdk). Use when asked to 'instrument this app', 'add Measure', 'install measure-sdk', 'add metrics tracking', 'instrument business processes', or when onboarding a Next.js B2B SaaS app to Measure. Reads the codebase first to understand the business model, then leads a Kimball-style metric planning conversation before writing any code. Uses agent teams for parallel codebase analysis and adversarial plan review."
---

# Measure SDK Instrumentation

Instrument a Next.js B2B SaaS app with `@measuremetrics/measure-sdk`. This is a **planning-first** process — understand the business before touching code. Uses agent teams to parallelize discovery and stress-test the instrumentation plan.

**References (read as needed, not upfront):**
- `references/metric-planning.md` — Kimball process, metric catalog, schema patterns, push-back checklist
- `references/sdk-patterns.md` — SDK init, `after()` patterns, code examples (Phase 4 only)

---

## Phase 1: Business Discovery

### Step 1.1: Qualify the App

Read the codebase to determine if this is a B2B SaaS business. Check:

1. **Framework**: Look for `next.config.*` — this skill targets Next.js apps only.
2. **Business model signals**: Read the Prisma schema (or equivalent ORM config) and scan for:
   - Subscription/plan tables or columns (`plan`, `price`, `status`, `trial`)
   - Multi-tenant structure (companies/organizations with users underneath)
   - Billing references (`stripeCustomerId`, `priceId`, checkout flows)
   - Customer lifecycle states (`active`, `churned`, `trialing`, `cancelled`)

If the app is **not B2B SaaS** (no subscription model, no multi-tenant structure, consumer app, e-commerce, marketplace), stop and tell the developer: *"Measure's instrumentation skill currently targets B2B SaaS apps. Your app looks like [what it looks like]. You can still use the Measure API directly — here's the SDK."*

If it **is** B2B SaaS, proceed. Summarize what you found for the developer.

### Step 1.2: Parallel Codebase Scan (Agent Team)

Spawn **four Explore agents in parallel** to scan different domains of the codebase simultaneously. Each agent produces a structured report. This is a Parallel Investigation pattern — the sub-tasks are independent.

**Agent 1 — Schema & Data Model:**
```
Research only — do NOT write code.
Read the database schema (Prisma, Drizzle, or raw SQL migrations).
Report:
- All tables/models related to customers, companies, organizations, tenants
- Subscription/plan/billing models — fields, types, enums
- User model and its relationship to company/org
- Status enums and lifecycle fields (created_at, churned_at, trial_ends_at, etc.)
- Price storage: where, what unit (cents? dollars?), how plans map to prices
- Any integration ID columns (stripeCustomerId, hubspotCompanyId, clerkOrgId)
Output as structured findings, not prose.
```

**Agent 2 — Server Actions & API Routes:**
```
Research only — do NOT write code.
Find all server actions (files in app/**/actions.ts or similar) and API routes (app/api/**).
For each mutation function, report:
- Function name and file path
- What it creates/updates/deletes
- What arguments it takes (especially price, plan, status)
- Whether it has access to "old" values before mutation (critical for deltas)
- Whether it uses transactions or has error handling
- Any existing tracking/analytics calls
Output as a table: function | file | mutation type | entities affected | price accessible?
```

**Agent 3 — Webhooks & Integration Points:**
```
Research only — do NOT write code.
Find all webhook handlers (app/api/webhooks/**) and integration configuration.
Report:
- Clerk webhooks: what events are handled, what they create in the DB
- Stripe webhooks: what events are handled, subscription lifecycle coverage
- Any other integrations (HubSpot, QuickBooks, etc.)
- OAuth flows or integration setup code
- .env.example or config for integration API keys
Determine: which business domains are owned by integrations vs the app?
```

**Agent 4 — Middleware & Engagement:**
```
Research only — do NOT write code.
Read middleware.ts and any auth/session handling code.
Report:
- What middleware.ts does (auth checks, redirects, etc.)
- Runtime: Edge or Node.js? (affects SDK compatibility)
- How user identity is resolved (Clerk? cookies? session?)
- Any existing page view or engagement tracking
- Login event tables or session tables (if they exist)
- Feature usage tracking patterns (feature flags, event logging)
Output: engagement instrumentation feasibility assessment.
```

### Step 1.3: Synthesize the Business Process Map

Collect all four agent reports. Synthesize into a **business process map** — this is your job as the lead, not a raw concatenation.

Look for:
- **Contradictions** — does Agent 2 show MRR mutations in server actions while Agent 3 shows Stripe handling subscriptions? That's a dual-source risk. Flag it.
- **Gaps** — did no agent find a churn code path? That's a missing business process. Ask the developer.
- **Integration boundaries** — if Agent 3 found Stripe webhooks handling `subscription.created`, that domain belongs to Stripe, not to app instrumentation.

Present the synthesized map:

```
Business processes this app owns:
├── Customer signup
│   ├── Code: src/app/actions/customers.ts:createCustomer()
│   ├── Creates: Company + Subscription (in transaction)
│   └── Price: subscription.price (cents)
├── Plan changes
│   ├── Code: src/app/actions/customers.ts:changePlan()
│   ├── Updates: Subscription.plan + Subscription.price
│   └── ⚠️  Old price: queried before update (confirmed by Agent 2)
├── Cancellation
│   ├── Code: src/app/actions/customers.ts:cancelSubscription()
│   └── Sets: Subscription.status = 'cancelled', Company.status = 'churned'
├── Feature usage
│   ├── Code: src/app/actions/product.ts:trackFeatureUsage()
│   └── Records: FeatureEvent with feature slug + user + company
└── Page views
    ├── Code: middleware.ts (Edge runtime)
    └── Auth: Clerk, userId from session

Owned by external systems (don't instrument):
├── Billing → Stripe (webhook: subscription.created/updated/deleted)
├── User identity → Clerk (webhook: user.created)
└── Expenses → not connected
```

### Step 1.4: Recommend Metrics to the Developer

Read `references/metric-planning.md` for the B2B SaaS metric catalog and dependency trees.

Based on the business process map, recommend specific metrics — each traced to the code path you found. Be specific about what the developer gets and what's missing:

```
Based on what I found in your codebase, here's what I'd recommend:

From your subscription lifecycle:
  → new_mrr, expansion_mrr, contraction_mrr, churned_mrr
  → new_customers, churned_customers
  → Measure computes from these: ARR, NRR, Quick Ratio, Gross Churn, ARPA

From middleware:
  → page_views → Measure computes DAU/MAU

From your core product interaction:
  → [whatever the core value-creation event is — e.g., "report generated",
     "deal logged", "metrics computed"]
  → This is your activation signal — did the user get value?

I noticed Stripe webhooks handle subscription.created and subscription.updated.
If Stripe is your billing system of record, it should own MRR components through
a Measure integration connector — not app instrumentation. Which is authoritative
for billing: your app's server actions or Stripe?
```

**Always identify the core value-creation event.** Page views tell you someone showed up. The core product interaction tells you they got value. If the developer says "just page views," push back: *"What action tells you a user actually got value from your product? That's the metric that enables activation rate."*

**Recommend full funnels.** When you identify an onboarding flow (signup → setup → activation → conversion), recommend metrics at each stage. Partial funnels can't compute conversion rates. Always recommend paired metrics: `trial_started` + `trial_converted`, `onboarding_started` + `onboarding_completed`.

Ask the developer:
- **"Does this match your understanding of what this app owns?"**
- **"What business questions do you or your board want to answer?"** — their priorities may add metrics or deprioritize ones you found.
- **"Is [system] the source of truth for [domain]?"** — for every integration boundary you found.

### Step 1.5: Push Back When Context Is Missing

Read the push-back checklist in `references/metric-planning.md`. Do NOT proceed if:

- **No price data at the mutation point** — if `changePlan()` overwrites the plan without preserving the old price, you can't compute expansion/contraction deltas.
- **No clear grain** — if "customer" could mean a company, a user, or a subscription, clarify. Kimball: declare the grain before identifying dimensions.
- **Ambiguous business events** — if one function handles signups AND trial conversions AND plan changes, get the developer to explain which state transitions matter.
- **Integration-better metrics** — if Stripe webhooks handle the subscription lifecycle, push back on instrumenting MRR from app code.
- **Missing timestamps** — if `churned_at` doesn't exist, you know they churned but not when.
- **Dual-source risk** — if both app code AND Stripe handle the same business event, clarify which is authoritative. Instrumenting both creates double-counting.
- **No entity member provisioning path** — adjustments link to entity members. The API resolves durable keys automatically, but the member must exist — if it doesn't, the API returns a validation error. For each entity, ask: how do members get created? Options: webhook provisioning, baseline seed, or upsert-on-first-track (the `getMemberId` helper handles this by catching `NotFoundError` and creating the member). If none exist, flag it.

Ask the developer to resolve ambiguities before proceeding.

---

## Phase 2: Metric Model & Instrumentation Plan

### Step 2.1: Build the Metric Model

For each agreed-upon business question, trace the dependency tree:

```
"What's our ARR?"
  → ARR = total_mrr × 12        (calculated — Measure computes)
    → total_mrr = Σ(new + expansion - contraction - churn)
      ├── new_mrr         ← createCustomer()    (instrument)
      ├── expansion_mrr   ← changePlan()         (instrument)
      ├── contraction_mrr ← changePlan()         (instrument)
      └── churned_mrr     ← cancelSubscription() (instrument)
```

Present the full model separating three categories:

| Metric | Type | Source |
|--------|------|--------|
| `new_mrr` | Base — instrument from app | `createCustomer()` |
| `expansion_mrr` | Base — instrument from app | `changePlan()` |
| ARR | Calculated — Measure computes | `total_mrr × 12` |
| NRR | Calculated — Measure computes | retention formula |
| Gross Margin | Calculated — needs integration | Stripe + QuickBooks |

### Step 2.2: Map Each Base Metric to Code

For each base metric, document the full instrumentation specification:

```
Base metric: new_mrr
  → Trigger: new subscription created
  → Code path: src/app/actions/customers.ts:createCustomer()
  → Amount: subscription.price (cents → divide by 100)
  → Entity link: company.id → Customer durable_key
  → Old price accessible? N/A (new subscription)
  → Timestamp: now()
  → Status: Ready
```

If a mutation point **cannot be found**, report it and ask:
- "I can't find where plan changes happen — is there an `updatePlan()`?"
- "Price is stored in cents but there's no plan→price mapping for preset names"

### Step 2.3: Adversarial Review (Agent Team)

Before presenting the final plan, spawn a **skeptic agent** to challenge it. This is an Adversarial Debate pattern — catches blind spots before the developer approves.

**Skeptic agent prompt:**
```
Research only — do NOT write code.
Review the following instrumentation plan for a B2B SaaS app using Measure SDK.

[paste the metric model table and base metric → code path mappings]

Challenge this plan using these techniques:
- For each base metric: is the amount correct? Is the old value accessible for
  deltas? Could this double-count with an integration?
- For entity links: is the durable_key stable? Could it change?
- For calculated metrics: do the formulas have all required inputs? Are there
  edge cases (division by zero for NRR in period 1, Quick Ratio with no churn)?
- Grain check: is each adjustment exactly one business event, or could one
  function call produce multiple adjustments that should be atomic?
- Multi-path counting: for count metrics, can the same entity trigger the metric
  from more than one instrumented code path? Trace the full entity lifecycle —
  creation, upgrade, downgrade, churn — and verify each stage is counted exactly
  once across all paths. Freemium-to-paid and trial-to-active are common traps.
- Data quality: what happens if the app crashes between the DB write and the
  Measure API call? What data is lost?
- Integration conflicts: any risk of double-counting if an integration is
  connected later?

Output: structured list of concerns, ranked by severity.
```

Review the skeptic's findings. Address valid concerns by adjusting the plan or flagging them for the developer. Dismiss concerns that aren't applicable.

### Step 2.4: Present the Instrumentation Plan

Present the final plan with a summary table and all flagged gaps:

| Base Metric | Code Path | Amount | Entity Link | Status |
|-------------|-----------|--------|-------------|--------|
| `new_mrr` | `createCustomer()` | `centsToDollars(price)` | `company.id` | Ready |
| `expansion_mrr` | `changePlan()` | `centsToDollars(new - old)` | `company.id` | Ready — old price queried before update |
| `churned_mrr` | `cancelSubscription()` | `centsToDollars(price)` | `company.id` | Ready |
| `page_views` | `middleware.ts` | `1` | `user.id` | ⚠️ Edge runtime |

Include any concerns from the adversarial review that the developer should know about.

**Re-surface any unresolved blocking questions** from Phase 1 before asking for approval. These are blocking — do not proceed without answers:
- Core value-creation event (what tells you a user got value?)
- Source-of-truth for each data domain (app vs integration?)
- Grain definition for each count metric (what does "one new customer" mean?)

**Get explicit approval** before writing any code.

---

## Phase 3: Provision Measure Resources

Before instrumenting code, the developer needs a Measure account with entities and metrics configured. This phase handles account setup and resource provisioning.

### Step 3.1: Account Setup

Ask the developer:

**"Do you have a Measure account, or do we need to set one up?"**

**Existing account:**
- Ask for their API key, organization ID, and environment ID
- Set these as env vars locally (`.env.local`)
- Verify the connection works: call `measure.metrics.listPaginated()` — if it succeeds, you're connected

**New account:**
- Direct them to [measure.dev](https://measure.dev) to sign up
- Walk them through creating an organization (typically the company name)
- Walk them through creating an environment (e.g., `production`, `development`)
- Have them generate an API key with admin permissions
- Set the credentials as env vars locally

Either way, you need these four values before proceeding:
```
MEASURE_API_KEY=...
MEASURE_API_URL=...
MEASURE_ORG_ID=...
MEASURE_ENV_ID=...
```

### Step 3.2: Create Entities

From the metric model (Phase 2), create entity definitions with the properties you identified as useful for slicing metrics. Run these as a **one-time setup script** (e.g., `scripts/measure-setup.ts`) or in a REPL — these are admin operations, not application runtime code.

For a typical B2B SaaS app, this means:

```typescript
// Customer entity — the billing/subscription unit
await measure.entities.create({
  name: 'Customer',
  cardinality: 'unbounded',
  properties: [
    { name: 'plan', property_type: 'categorical', is_filterable: true, is_facetable: true },
    { name: 'status', property_type: 'categorical', is_filterable: true, is_facetable: true },
  ],
});

// User entity — the identity unit (if tracking engagement)
await measure.entities.create({
  name: 'User',
  cardinality: 'unbounded',
  properties: [],
});
```

Adapt the entity names and properties to match what you found in the codebase. Entity names here must match the keys used in `entity_member_links` in the instrumentation code.

### Step 3.3: Create Metrics

Create each base metric from the approved metric model. Set `can_decrement` based on whether the metric can go negative (MRR components can, counts typically can't). Link allowed entities.

```typescript
// Example: create base metrics
const customerEntity = await measure.entities.listPaginated();
const custEntityId = customerEntity.data.find(e => e.name === 'Customer')!.id;

await measure.metrics.create({
  name: 'new_mrr',
  description: 'MRR from new subscriptions',
  unit: 'currency',
  precision_recorded: 2,
  can_decrement: false,
  allowed_entities: [{ entity_id: custEntityId, is_required: false }],
});

// contraction_mrr: can_decrement is FALSE because the amount is always
// the positive magnitude of the contraction (e.g., $50 lost), not a
// negative number. The metric name conveys the direction.
await measure.metrics.create({
  name: 'contraction_mrr',
  description: 'MRR lost from plan downgrades',
  unit: 'currency',
  precision_recorded: 2,
  can_decrement: false,
  allowed_entities: [{ entity_id: custEntityId, is_required: false }],
});

// Repeat for expansion_mrr, churned_mrr, new_customers,
// churned_customers, trial_started, trial_converted, page_views, etc.
```

### Step 3.4: Create Calculated Metrics (if supported)

If the SDK supports `calculatedMetrics.create()`, define the derived metrics from the dependency trees in Phase 2:
- ARR = total_mrr × 12
- NRR, Quick Ratio, Gross Churn, etc.

If calculated metrics aren't available yet, document what the developer should create manually in the Measure dashboard.

### Step 3.5: Verify Setup

Confirm all resources exist:
- List entities and verify names match what the instrumentation code expects
- List metrics and verify names match
- Print a summary for the developer

---

## Phase 4: Instrument

Only after Phase 3 resources are provisioned. Read `references/sdk-patterns.md` for code patterns.

### Step 4.1: SDK Setup

1. Install the SDK using the project's package manager (`npm install @measuremetrics/measure-sdk`, `bun add`, `pnpm add`, etc. — detect from the lockfile). For browser-side tracking, there's also `@measuremetrics/measure-browser`.
2. **Verify the SDK types against the reference.** Read the installed SDK's examples (`node_modules/@measuremetrics/measure-sdk/examples/`) and type definitions to confirm the reference patterns match. The SDK exports typed error classes (`NotFoundError`, `RateLimitError`, etc.) — use `instanceof` checks, not status code inspection.
3. If the target app uses **Turbopack** (Next.js 15+), add `serverExternalPackages: ['@measuremetrics/measure-sdk']` to `next.config.ts`.
4. Create `lib/measure.ts` with client init + ID resolution helpers (see `references/sdk-patterns.md`)
5. Add env vars to `.env.example`: `MEASURE_API_KEY`, `MEASURE_API_URL`, `MEASURE_ORG_ID`, `MEASURE_ENV_ID`
6. **The client must degrade gracefully** — if `MEASURE_API_KEY` is not set, disable tracking silently instead of crashing the app. See the `isConfigured` pattern in `references/sdk-patterns.md`.
7. If using middleware, **verify the runtime** — check whether the file sets `export const runtime = 'nodejs'`. If it defaults to Edge, either set Node.js explicitly or confirm the SDK is Edge-compatible. Don't leave this as "needs verification." See the runtime tradeoff notes in `references/metric-planning.md`.

### Step 4.2: Instrument Business Processes

For each approved business process, use a **subagent** to write the instrumentation. The lead (you) orchestrates and reviews.

**When to use subagents vs do it yourself:**
- **Subagent**: When the process is self-contained (one server action, clear inputs/outputs) and you can fully specify what needs to happen. Run in a worktree for isolation.
- **Do it yourself**: When the instrumentation touches multiple files, requires judgment calls about edge cases, or the developer needs to see your reasoning step by step.

**Subagent prompt template:**
```
Read references/sdk-patterns.md in the skill directory first.

Instrument the following business process with Measure SDK:
- Business process: [name]
- Code path: [file:function]
- Metrics to publish: [list with amounts and entity links]
- SDK patterns: use after() for non-blocking, catch and log errors inside after()
  blocks (never let tracking errors propagate to the user's request),
  entity member creation if needed

Write the instrumentation code. Do not change the function's business logic —
only add Measure tracking calls inside after() blocks.

After writing, document results in docs/measure-instrumentation.md with:
process name, code path, metrics, grain, amount, entity link, status, notes.
```

### Step 4.3: Review Each Instrumentation

After each subagent completes (or after you instrument directly), verify:
- [ ] Tracking is inside `after()` — never blocks the business operation
- [ ] Error handling inside `after()` catches and logs with context (metric name, entity ID) — never throws to the user's request
- [ ] Seed scripts and setup code throw on failure (the opposite of runtime tracking)
- [ ] Entity member is created before adjustment links to it
- [ ] `entity_member_links` keys use entity **names** (e.g., `"Customer"`) for readability — the API also accepts entity IDs but names are preferred
- [ ] Amount is correct (use `centsToDollars()` for cents, pass decimals directly — never raw `/ 100`)
- [ ] No multi-path double-counting — can the same entity trigger this metric from another instrumented path?
- [ ] `docs/measure-instrumentation.md` is updated
- [ ] If the file has existing tests, add test coverage for tracking calls (mock the Measure client)

### Step 4.4: Baseline & Entity Seeding

After business processes are instrumented, address existing data:

1. **Classify each metric**: cumulative (needs baseline) vs event (forward-only). See `references/sdk-patterns.md` for the taxonomy.
2. **Create a seed script** for cumulative metrics and entity members. The script should:
   - Create entity members for all existing records with current property values (plan, status, etc.)
   - Publish baseline adjustments for cumulative metrics (MRR, customer count) backdated to creation timestamps
   - Be idempotent or guarded against re-runs
3. **Add entity member property updates** to webhook/action handlers where dimensions change. When a customer upgrades plan or changes status, the entity member's properties must be updated in Measure — otherwise dimension-filtered queries show stale data.
4. **Document what's forward-only** — page views, trial events, and other event metrics don't need baselines. Make this explicit in `docs/measure-instrumentation.md`.

Present the seed script for review. It's a one-time operation that affects production data.

### Step 4.5: Iterate

Pick the next process from the approved plan. Continue until all are instrumented or blocked.

### Step 4.6: Handoff

After all processes are done:

1. **Verify the build** — run `tsc --noEmit` (or the project's type-check command) to confirm no type errors were introduced.
2. **Review `docs/measure-instrumentation.md`** — is coverage complete? Any processes marked "Blocked" that need the developer's input?
3. **Suggest next steps** — create a branch, suggest a commit message.
4. **Flag remaining work** — entity member provisioning, runtime verification, metrics that were deferred or need integrations.

---

## Known API Limitations

- **Durable key resolution**: `entity_member_links` accepts both UUIDs and durable keys as member values — the API resolves durable keys automatically. Entity keys accept both names and UUIDs.
- **Idempotency keys**: Not yet accepted by the adjustments API. Document as known gap — duplicate adjustments from retries are possible.

## What NOT to Instrument

- Metrics that integrations handle better (Stripe for MRR if connected, QuickBooks for expenses)
- Read-only operations (unless tracking engagement via page views)
- Internal admin actions (unless they affect business metrics)
- Anything where the grain is unclear — ask first
