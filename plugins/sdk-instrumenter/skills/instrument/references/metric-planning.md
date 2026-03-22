# Metric Planning Reference

## The Kimball Process (Adapted for Measure)

Ralph Kimball's dimensional modeling process, adapted for real-time metric instrumentation:

1. **Select the business process** — What questions does the developer (or their board) want to answer? Start with conversation, not schema diagrams.
2. **Declare the grain** — What is a single "event" in this process? One subscription created? One plan change? One page view? The grain determines what an adjustment row represents.
3. **Identify the dimensions** — How does the developer want to slice the data? By customer, by plan tier, by region? These become entities and entity properties in Measure.
4. **Identify the facts** — What are the numeric measurements? MRR amount, customer count, page view count. These become base metrics.

The order matters. Do NOT jump to facts (step 4) before the grain is clear (step 2).

## Standard B2B SaaS Board Questions

When the developer isn't sure what to track, suggest these — every B2B SaaS board asks them:

| Question | Metric | Formula | Source |
|----------|--------|---------|--------|
| "What's our ARR?" | ARR | `total_mrr × 12` | App (MRR components) |
| "How fast are we growing?" | MRR Growth Rate | `net_new_mrr / starting_mrr` | App |
| "Are we retaining revenue?" | NRR | `(start + expansion - contraction - churn) / start` | App |
| "How leaky is the bucket?" | Gross Churn Rate | `churned_mrr / starting_mrr` | App |
| "Is growth efficient?" | Quick Ratio | `(new + expansion) / (contraction + churn)` | App |
| "Are users engaged?" | DAU/MAU | `daily_active / monthly_active` | App (middleware) |
| "What are our unit economics?" | Gross Margin | `(revenue - cogs) / revenue` | Stripe + QuickBooks |
| "How much to acquire a customer?" | CAC | `s&m_spend / new_customers` | QuickBooks + CRM |
| "Is a customer worth acquiring?" | LTV/CAC | `(ARPA / churn_rate) / CAC` | Derived |
| "How healthy is the pipeline?" | Win Rate | `deals_won / (won + lost)` | HubSpot/Salesforce |

## Metric Dependency Trees

Every calculated metric traces back to base metrics. The developer instruments base metrics; Measure computes calculated metrics automatically.

### ARR Tree

```
ARR = total_mrr × 12
  └── total_mrr = Σ(new_mrr + expansion_mrr - contraction_mrr - churned_mrr)
      ├── new_mrr         ← createCustomer() / signup
      ├── expansion_mrr   ← changePlan() where new > old
      ├── contraction_mrr ← changePlan() where new < old
      └── churned_mrr     ← cancelSubscription() / churn

Also computed from same 4 base metrics:
  ├── NRR = (start + expansion - contraction - churn) / start
  ├── Quick Ratio = (new + expansion) / (contraction + churn)
  ├── Gross Churn = churned / starting
  ├── Net New MRR = new + expansion - contraction - churn
  └── MRR Growth Rate = net_new / starting
```

Once you instrument the 4 MRR components, you get ~8 calculated metrics for free.

### Engagement Tree

```
DAU/MAU = daily_active_users / monthly_active_users
  └── page_views (amount=1, per authenticated request)
      └── middleware.ts — fire-and-forget, best effort
```

The app emits raw page views. Measure deduplicates by user and computes DAU/MAU.

## Schema Patterns to Look For

### The customer/company table (always exists)

```
Table: companies | customers | accounts | organizations | tenants
Required: id, name, created_at
Useful:   plan, status, churned_at, stripe_customer_id
```

- `id` → entity member `durable_key`
- `name` → entity member `display_name`
- `plan` → entity property (categorical, filterable — enables "MRR by plan tier")
- `status` → entity property (categorical — active, trial, churned)
- `stripe_customer_id` → signals Stripe integration is possible
- `churned_at` → enables churn timing (if missing, flag as data gap)

### The subscription/plan table (usually exists)

```
Table: subscriptions | plans | billing
Required: company_id, price (or plan → price mapping), status
Useful:   started_at, ended_at, plan name
```

- `price` → adjustment amount for MRR metrics
- `status` with `cancelled` value → churn detection
- If price is in **cents** (integer), use `centsToDollars()` — never raw `/ 100` (floating-point risk). If stored as DECIMAL dollars, pass directly — the Measure API uses arbitrary-precision NUMERIC.

### The plan change table (rarely exists)

```
Table: plan_changes | subscription_history
Fields: old_plan, new_plan, old_price, new_price, changed_at
```

If this exists, expansion/contraction is clean. If not, the mutation function must query the old price before updating — check whether it does.

### Integration columns (check for these)

| Column on customer table | What it signals |
|-------------------------|-----------------|
| `stripe_customer_id` | Stripe is the billing SoR — consider integration over instrumentation for MRR |
| `hubspot_company_id` | HubSpot CRM data available |
| `clerk_org_id` | Clerk is handling identity |

## Push-Back Checklist

Before writing instrumentation code, verify each item. If any fail, stop and discuss with the developer.

### Grain Issues
- [ ] **Is the grain clear?** Can you state in one sentence what a single adjustment row represents? ("One adjustment per new subscription at the plan's monthly price")
- [ ] **Is "customer" unambiguous?** Does it mean a company, a user, an org? The entity choice affects everything downstream.
- [ ] **Are there multiple event types in one function?** A generic `updateCompany()` that handles signups, upgrades, AND churn needs to be decomposed. Each business event should produce specific metrics.

### Multi-Path Counting
- [ ] **Can the same entity trigger this metric from more than one code path?** For count metrics (`new_customers`, `churned_customers`), trace every path that produces the adjustment. If the same entity can traverse multiple paths — e.g., created as FREE in a server action, then counted again as a new subscriber when Stripe confirms checkout — you'll double-count. This is distinct from grain (each individual event is correct) and from dual-source (both paths are app-owned). The fix is usually to split into separate metrics with precise semantics: `new_customers` (entity first created) vs `new_subscribers` (first paid subscription confirmed).
- [ ] **Does the entity have lifecycle stages that cross instrumentation boundaries?** Freemium models, trial-to-paid conversions, and self-serve-to-sales-assisted paths commonly create entities in one code path and upgrade them in another. If both paths count the entity, every entity that traverses both gets counted twice. Map the full lifecycle before assigning count metrics to code paths.

### Amount Issues
- [ ] **Is the price accessible at the mutation point?** If `changePlan()` doesn't have the old price, you can't compute the expansion/contraction delta.
- [ ] **Is the price in a consistent unit?** Cents vs dollars? Monthly vs annual? Check and normalize. If cents, use `centsToDollars()` — never raw `/ 100` in JavaScript (floating-point risk). If dollars stored as DECIMAL, pass directly to the API.
- [ ] **Is there a plan→price mapping?** If the subscription stores a plan name but not a price, you need the lookup table.
- [ ] **Does churn preserve the price?** If cancellation nulls the plan/price columns, the churned MRR amount is lost.

### Source-of-Truth Issues
- [ ] **Should this come from an integration instead?** If `stripe_customer_id` exists on the customer table, Stripe is likely the billing SoR. Instrumenting MRR from app code when Stripe has the authoritative data creates a dual-source problem.
- [ ] **Is there historical data to worry about?** Forward instrumentation only captures new events. Existing customers need a baseline or integration backfill. Don't pretend forward-only gives you historical trends.

### Baseline Requirements
- [ ] **Classify each metric as cumulative or event.** Cumulative metrics (MRR, customer count) represent running totals — they need baseline adjustments for existing records or the total starts at zero. Event metrics (page views, trial starts) count discrete forward events and don't need baselines.
- [ ] **Plan entity member seeding.** Entity members need to exist with current property values (plan, status, region) for dimension-filtered queries to work. A seed script creates members for existing records; forward events rely on upsert-on-first-track.
- [ ] **Plan entity property updates.** When a dimension changes (customer upgrades plan, status changes), the entity member's properties must be updated in Measure. Without this, "MRR by plan tier" shows stale data. Add property update calls alongside metric adjustments in webhook/action handlers where dimensions change.

### Entity Member Provisioning
- [ ] **How do entity members get created in Measure?** Adjustments can reference entity members by durable key (the API resolves them), but the member must exist or the API returns a validation error. For each entity that adjustments link to, verify one of: (a) a webhook creates the member when the DB record is created, (b) a baseline seed creates members for existing records, (c) the instrumentation code upserts on first use via the `getMemberId` helper (catches `NotFoundError`, creates the member). If none of these exist, flag it.
- [ ] **Are existing records covered?** New instrumentation only captures forward events. Existing customers/users need entity members created via a seed script or integration backfill.

### Timing Issues
- [ ] **Can you determine WHEN the event happened?** `churned_at` is ideal; `status = 'churned'` without a timestamp means you know the state but not the timing.
- [ ] **Is the mutation atomic?** If customer creation and subscription creation happen in separate transactions, which one triggers the metric? What if one fails?

### Runtime Compatibility
- [ ] **Middleware runtime check.** Next.js middleware defaults to Edge runtime. The Measure SDK may use Node.js-only APIs. Before instrumenting middleware, check: does the middleware file set `export const runtime = 'nodejs'`? If not, you have two options — and this is a **tradeoff decision**, not just a compatibility check:
  - **Set `runtime = 'nodejs'`**: simpler, but moves ALL middleware execution from globally-distributed Edge to single-region Node.js. This adds latency to every request the middleware handles, not just tracked ones.
  - **Use a route handler instead**: create a lightweight `/api/track` endpoint that the middleware or client calls. Middleware stays on Edge; tracking runs in Node.js only where needed.
  Present both options and the latency tradeoff to the developer. Don't default to changing the runtime without discussing impact.

### Product Analytics Coverage
- [ ] **What is the core value-creation event?** Every product has one thing users do that represents the core value. For a calculator, it's computing results. For a CRM, it's logging a deal. If the developer says "just page views," push back: "Page views tell you someone showed up. What tells you they got value?" This metric enables activation rate (users who did X within N days / total signups).
- [ ] **Is the funnel complete?** When you find an onboarding flow (signup → setup → activation → conversion), recommend metrics at each stage, not just the endpoints. Partial funnels can't compute conversion rates.

### Conversion Pairs
- [ ] **Pair start and end metrics.** When instrumenting `trial_started`, always recommend `trial_converted`. When instrumenting `onboarding_started`, recommend `onboarding_completed`. Measure can compute conversion rates from paired metrics. One without the other is less useful.

### Missing Data (document, don't guess)
- [ ] **Don't fabricate expansion/contraction from a single plan column.** If the app overwrites `plan` without logging the old value, you genuinely can't compute deltas from the app. Say so. Recommend Stripe integration.
- [ ] **Don't assume historical DAU.** Without a login_events table, engagement is forward-only from middleware instrumentation.
- [ ] **Don't instrument what you can't verify.** If you can't confirm the adjustment reaches Measure (no test environment, no API key), document it as "code written, not verified."

## The Instrumentation Spectrum

What the app needs to instrument depends on connected integrations:

```
Full instrumentation (no integrations):
├── MRR components from subscription server actions
├── Customer/user counts from signup actions
├── Page views from middleware
├── Feature usage from event handlers
└── Trial events from signup flow

With Stripe + Clerk (typical production):
├── Stripe handles: MRR components, revenue, subscription lifecycle
├── Clerk handles: user lifecycle, session events
└── App instruments ONLY: feature usage, activation, domain-specific events

With Stripe + Clerk + QuickBooks + HubSpot:
└── App instruments ONLY: product analytics (feature adoption, NPS, activation)
```

As integrations are connected, the app's instrumentation surface **shrinks**. Always ask: "Is an integration connected or connectable for this data domain?"
