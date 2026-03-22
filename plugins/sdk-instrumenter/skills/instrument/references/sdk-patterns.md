# Measure SDK Instrumentation Patterns

Patterns based on `@measuremetrics/measure-sdk` (published on npm).
Install with the project's package manager (e.g., `npm install
@measuremetrics/measure-sdk`). For browser-side tracking, there's
also `@measuremetrics/measure-browser`.

If you encounter type errors, check the actual SDK types — the `Metric`
interface, `CreateMetricAdjustmentRequest`, and
`AdjustmentEntityMemberLink` in the measure-js repo are the source of
truth.

## Client Initialization

Create `lib/measure.ts` in the project root. The client degrades gracefully
when env vars are missing — tracking is silently disabled, not crashy.

```typescript
import { MeasureClient } from '@measuremetrics/measure-sdk';

const isConfigured = Boolean(process.env.MEASURE_API_KEY);

export const measure = isConfigured
  ? new MeasureClient({
      apiKey: process.env.MEASURE_API_KEY!,
      baseURL: process.env.MEASURE_API_URL!,
      organizationId: process.env.MEASURE_ORG_ID!,
      environmentId: process.env.MEASURE_ENV_ID!,
      timeout: 5000,
      retry: { maxRetries: 2 },
    })
  : null;
```

## Metric ID Resolution

Metrics are referenced by UUID in API calls. Resolve by name once, cache
the result. The `Metric` type has a `name` field (not `slug`).

```typescript
const metricCache = new Map<string, string>();
const entityCache = new Map<string, string>();

export async function getMetricId(name: string): Promise<string> {
  if (!measure) throw new Error('[Measure] Not configured');
  if (metricCache.has(name)) return metricCache.get(name)!;
  const { data: metrics } = await measure.metrics.listPaginated();
  const metric = metrics.find(m => m.name === name);
  if (!metric) throw new Error(`[Measure] Metric not found: ${name}`);
  metricCache.set(name, metric.id);
  return metric.id;
}

export async function getEntityId(name: string): Promise<string> {
  if (!measure) throw new Error('[Measure] Not configured');
  if (entityCache.has(name)) return entityCache.get(name)!;
  const { data: entities } = await measure.entities.listPaginated();
  const entity = entities.find(e => e.name === name);
  if (!entity) throw new Error(`[Measure] Entity not found: ${name}`);
  entityCache.set(name, entity.id);
  return entity.id;
}
```

## Entity Member Resolution (with Upsert)

Entity members must exist before adjustments can link to them. Resolution
requires one API call per unique member (cached per process). This helper
upserts on first track — if `getByDurableKey()` returns 404, it creates
the member automatically.

```typescript
import { NotFoundError } from '@measuremetrics/measure-sdk';

const memberCache = new Map<string, string>();

export async function getMemberId(
  entityName: string,
  durableKey: string,
  displayName?: string,
): Promise<string> {
  if (!measure) throw new Error('[Measure] Not configured');
  const cacheKey = `${entityName}:${durableKey}`;
  if (memberCache.has(cacheKey)) return memberCache.get(cacheKey)!;

  const entityId = await getEntityId(entityName);

  try {
    const member = await measure.entityMembers.getByDurableKey(entityId, durableKey);
    memberCache.set(cacheKey, member.id);
    return member.id;
  } catch (error) {
    // Only create if the member genuinely doesn't exist (404).
    // Re-throw everything else (auth, rate limits, network failures).
    if (!(error instanceof NotFoundError)) throw error;

    const member = await measure.entityMembers.create(entityId, {
      durable_key: durableKey,
      property_values: displayName ? { name: displayName } : {},
    });
    memberCache.set(cacheKey, member.id);
    return member.id;
  }
}
```

## Entity Member Links Format

The `entity_member_links` field accepts **both entity names and entity IDs**
as keys. The API resolves names to IDs automatically, so prefer names for
readability. Member values can also be either UUIDs or durable keys — the
API resolves both. If a name or durable key can't be resolved, the API
returns a validation error (not a silent failure).

```typescript
// Recommended — key is entity name, value is member UUID:
entity_member_links: {
  Customer: { entity_member_id: memberId }
}

// Also valid — entity UUID as key:
entity_member_links: { [entityId]: { entity_member_id: memberId } }

// WRONG — flat string (will not compile):
entity_member_links: { Customer: memberId }
```

## Track Helper

Wraps metric resolution + adjustment creation. Returns silently when
Measure is not configured.

```typescript
let warnedNotConfigured = false;

function checkConfigured(): boolean {
  if (!measure) {
    if (!warnedNotConfigured) {
      console.warn('[Measure] MEASURE_API_KEY not set — tracking is disabled.');
      warnedNotConfigured = true;
    }
    return false;
  }
  return true;
}

export async function trackAdjustment(
  metricName: string,
  amount: number,
  entityLinks?: Record<string, { entity_member_id: string }>,
): Promise<void> {
  if (!checkConfigured()) return;
  const metricId = await getMetricId(metricName);
  await measure.metricAdjustments.create(metricId, {
    timestamp: new Date().toISOString(),
    amount,
    ...(entityLinks ? { entity_member_links: entityLinks } : {}),
  });
}

/**
 * Convert cents (integer) to dollars without floating-point error.
 * Only needed when the source stores prices as integer cents (e.g., Stripe).
 * If the app stores dollars as DECIMAL, pass the value directly — the
 * Measure API uses arbitrary-precision NUMERIC, so no precision is lost.
 */
export function centsToDollars(cents: number): number {
  return Math.round(cents) / 100;
}

export async function updateMemberProperties(
  entityName: string,
  durableKey: string,
  displayName: string,
  properties: Record<string, string>,
  changeReason?: string,
): Promise<void> {
  if (!checkConfigured()) return;
  const entityId = await getEntityId(entityName);
  const memberId = await getMemberId(entityName, durableKey, displayName);
  await measure!.entityMembers.update(entityId, memberId, {
    property_values: properties,
    ...(changeReason ? { change_reason: changeReason } : {}),
  });
}
```

## Core Pattern: Non-Blocking Adjustment with `after()`

Business metrics must never block the user's operation. Use `after()` from
`next/server`:

```typescript
import { after } from 'next/server';
import { centsToDollars, getMemberId, trackAdjustment } from '@/lib/measure';

export async function createCustomer(data: CustomerInput) {
  // 1. Business operation completes first
  const customer = await db.customer.create({ data });

  // 2. Tracking happens in background, never blocks the user
  after(async () => {
    try {
      const memberId = await getMemberId('Customer', customer.id, customer.name);
      await trackAdjustment('new_mrr', centsToDollars(customer.planPrice), {
        Customer: { entity_member_id: memberId },
      });
    } catch (error) {
      console.error('[Measure] Failed to track new_mrr:', error);
    }
  });

  return customer;
}
```

## Pattern: Plan Change (Expansion or Contraction)

```typescript
after(async () => {
  try {
    const deltaCents = newPrice - oldPrice; // arithmetic in cents (integers)
    if (deltaCents === 0) return;

    const metricName = deltaCents > 0 ? 'expansion_mrr' : 'contraction_mrr';
    const memberId = await getMemberId('Customer', customerId, customerName);
    await trackAdjustment(metricName, centsToDollars(Math.abs(deltaCents)), {
      Customer: { entity_member_id: memberId },
    });
  } catch (error) {
    console.error('[Measure] Failed to track plan change:', error);
  }
});
```

## Pattern: Page Views (Middleware)

Fire-and-forget in middleware. Important considerations:

- **Exclude admin routes** — only track product routes
- **Verify runtime** — the SDK may require Node.js, but switching middleware
  from Edge to Node.js adds latency to every request. See the Push-Back
  Checklist in `metric-planning.md` for the full tradeoff analysis (Edge vs
  Node.js vs route handler) before defaulting to `runtime = 'nodejs'`.
- **Never block** — fire-and-forget with `.catch()` that logs errors

```typescript
// Inside clerkMiddleware or middleware function, after auth:
if (isProductRoute(request) && !isAdminRoute(request)) {
  getMemberId('User', userId)
    .then((memberId) =>
      trackAdjustment('page_views', 1, {
        User: { entity_member_id: memberId },
      }),
    )
    .catch((error) => {
      console.error(`[Measure] Failed to track page_views for user ${userId}:`, error);
    });
}
```

## Pattern: Webhook-Sourced Metrics

When a webhook is the authoritative source (e.g., Stripe confirms a trial
started), instrument inside the webhook handler — not in the server action
that initiated the flow. This avoids counting abandoned flows.

```typescript
// In Stripe webhook handler, after DB update:
if (subscriptionStatus === 'TRIALING') {
  getMemberId('Customer', customer.id, customer.name)
    .then((memberId) =>
      trackAdjustment('trial_started', 1, {
        Customer: { entity_member_id: memberId },
      }),
    )
    .catch((error) =>
      console.error(`[Measure] Failed to track trial_started for customer ${customer.id}:`, error),
    );
}
```

Always recommend paired metrics for conversion funnels:
- `trial_started` + `trial_converted` (subscription.status → active)
- `onboarding_started` + `onboarding_completed`

## Entity Member Provisioning

Entity members must exist in Measure before adjustments can link to them.
If `getByDurableKey()` returns 404, the adjustment's entity link silently
fails. Plan for provisioning:

**Option A — Upsert on first track:** Create the entity member if the
`getByDurableKey()` call 404s, then retry the link. Adds latency on first
event per member.

**Option B — Webhook provisioning:** Create entity members in the same
webhook that creates the DB record (e.g., Clerk `user.created` webhook
also creates a User entity member in Measure).

**Option C — Baseline seed script:** Run once to create entity members
for all existing records. Forward events rely on Option A or B.

The skill should ask: **"How do entity members get provisioned?"** during
planning. If there's no answer, flag it — adjustments will silently fail.

## Baseline Seeding

Forward-only instrumentation misses existing state. Cumulative metrics
(MRR, customer count) need baseline adjustments so totals are correct
from day one. Entity members need seeding so dimension-filtered queries
work immediately.

### Which metrics need baselines?

- **Cumulative amounts** (new_mrr, churned_mrr) — YES. Without baselines,
  ARR starts at $0 instead of the real total.
- **Cumulative counts** (new_customers) — YES. Total customer count starts
  at 0 without seeding existing customers.
- **Event counts** (page_views, trial_started) — NO. These count discrete
  events going forward. History starts at instrumentation time.
- **Transitions** (trial_converted) — NO. Past transitions are history.

### Seed script pattern

Run once after instrumentation is deployed. Creates entity members for
all existing records and publishes baseline adjustments for cumulative
metrics. Use the actual creation timestamp so time-series charts show
the real history.

```typescript
import { measure, getMetricId, getEntityId } from '@/lib/measure';
import { prisma } from '@/lib/prisma';
import { PLANS } from '@/lib/stripe';

async function seed() {
  if (!measure) throw new Error('MEASURE_API_KEY required for seeding');

  const entityId = await getEntityId('Customer');
  const mrrMetricId = await getMetricId('new_mrr');
  const custMetricId = await getMetricId('new_customers');

  // Guard: abort if members already exist (re-running creates duplicates)
  const { data: existing } = await measure.entityMembers.listPaginated(entityId);
  if (existing.length > 0) {
    console.error(
      `[Measure] Seed aborted: entity members already exist. ` +
      'Re-running would create duplicate adjustments.\n' +
      'If a previous run partially failed, write a targeted script that seeds only ' +
      'the missing customers (query your DB for customers not yet in Measure ' +
      'via getByDurableKey()).',
    );
    process.exit(1);
  }

  const customers = await prisma.customer.findMany({
    where: { status: { not: 'CANCELLED' } },
    include: { subscription: true },
  });

  let succeeded = 0;
  let failed = 0;

  for (const customer of customers) {
    try {
      // 1. Create entity member with current property values
      const member = await measure.entityMembers.create(entityId, {
        durable_key: customer.id,
        property_values: {
          name: customer.name,
          plan: customer.plan,
          status: customer.status,
        },
      });

      // 2. Baseline MRR adjustment (backdated to creation)
      const price = centsToDollars(PLANS[customer.plan].price);
      if (price > 0) {
        await measure.metricAdjustments.create(mrrMetricId, {
          timestamp: customer.createdAt.toISOString(),
          amount: price,
          entity_member_links: {
            Customer: { entity_member_id: member.id },
          },
          change_reason: 'Baseline: existing customer at instrumentation',
        });
      }

      // 3. Baseline customer count
      await measure.metricAdjustments.create(custMetricId, {
        timestamp: customer.createdAt.toISOString(),
        amount: 1,
        entity_member_links: {
          Customer: { entity_member_id: member.id },
        },
        change_reason: 'Baseline: existing customer at instrumentation',
      });

      succeeded++;
    } catch (error) {
      failed++;
      console.error(`[Measure] Failed to seed customer ${customer.id}:`, error);
    }
  }

  console.log(`Seeded ${succeeded} customers (${failed} failed)`);
  if (failed > 0) {
    console.error(
      `[Measure] ${failed} customers failed. ` +
      'Do NOT re-run the full script — it will duplicate already-seeded customers.',
    );
    process.exit(1);
  }
}
```

### When to run

Run the seed script **once** after the first deploy with Measure
instrumentation. This script is **not idempotent** — re-running it will
create duplicate adjustments that inflate MRR and customer counts. The
guard above aborts if entity members already exist.

### Incremental seeding (after partial failure or new customers)

If the initial seed partially failed, or new customers were added between
seed and go-live, use a targeted script that checks each customer
individually:

```typescript
async function seedIncremental() {
  if (!measure) throw new Error('MEASURE_API_KEY required for seeding');

  const entityId = await getEntityId('Customer');
  const mrrMetricId = await getMetricId('new_mrr');
  const custMetricId = await getMetricId('new_customers');

  const customers = await prisma.customer.findMany({
    where: { status: { not: 'CANCELLED' } },
  });

  let created = 0;
  let skipped = 0;

  for (const customer of customers) {
    try {
      // Skip if already seeded
      await measure.entityMembers.getByDurableKey(entityId, customer.id);
      skipped++;
      continue;
    } catch (error) {
      if (!(error instanceof NotFoundError)) throw error;
    }

    // Not found — seed this customer
    try {
      const member = await measure.entityMembers.create(entityId, {
        durable_key: customer.id,
        property_values: {
          name: customer.name,
          plan: customer.plan,
          status: customer.status,
        },
      });

      const price = centsToDollars(PLANS[customer.plan].price);
      if (price > 0) {
        await measure.metricAdjustments.create(mrrMetricId, {
          timestamp: customer.createdAt.toISOString(),
          amount: price,
          entity_member_links: {
            Customer: { entity_member_id: member.id },
          },
          change_reason: 'Baseline: incremental seed',
        });
      }

      await measure.metricAdjustments.create(custMetricId, {
        timestamp: customer.createdAt.toISOString(),
        amount: 1,
        entity_member_links: {
          Customer: { entity_member_id: member.id },
        },
        change_reason: 'Baseline: incremental seed',
      });

      created++;
    } catch (error) {
      console.error(`[Measure] Failed to seed customer ${customer.id}:`, error);
    }
  }

  console.log(`Incremental seed: ${created} created, ${skipped} already existed`);
}
```

## Entity Member Property Updates

Entity members have versioned properties. When a dimension changes
(customer upgrades plan, status changes), update the entity member so
dimension-filtered queries reflect current state.

```typescript
// After updating customer plan in the DB:
after(async () => {
  try {
    await updateMemberProperties(
      'Customer',
      customer.id,
      customer.name,
      { plan: newPlan, status: newStatus },
      `Plan changed to ${newPlan}`,
    );
  } catch (error) {
    console.error('[Measure] Failed to update entity member:', error);
  }
});
```

Without property updates, "MRR by plan tier" shows stale data — a
customer who upgraded from PRO to ENTERPRISE still appears under PRO
in Measure.

## SDK Resource Reference

Key resources on `MeasureClient`:

| Resource | Method | Use for |
|----------|--------|---------|
| `metrics` | `.listPaginated()`, `.create()`, `.get()` | Finding/creating metric definitions |
| `metricAdjustments` | `.create(metricId, data)` | Publishing measurements |
| `entities` | `.listPaginated()`, `.create()` | Finding/creating entity types |
| `entityMembers` | `.create(entityId, data)`, `.getByDurableKey(entityId, dk)` | Managing entity instances |
| `calculatedMetrics` | `.create()` | Defining derived metrics (ARR, NRR, etc.) |
| `goals` | `.create()` | Setting targets |
| `dashboards` | `.create()` | Creating dashboards |

## Instrumentation Spectrum

See `metric-planning.md` for guidance on what to instrument based on
connected integrations. The key principle: as integrations are connected,
the app's instrumentation surface shrinks.
