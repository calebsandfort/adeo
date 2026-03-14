# Integration Engineer SME Analysis

## Analysis of `[SME:IntegrationEngineer]` Requirements

### Question: Tenant ID Session Context and RLS Testing

**Question:** What's the best approach for setting the tenant ID in the Postgres session context per request? Options include `SET app.current_tenant` via a connection-level variable, or passing it through Drizzle's query context. How does this interact with connection pooling? How do we ensure RLS policies are tested as part of the CI pipeline to prevent regressions?

---

## Recommended Approach: Hybrid SET + Drizzle Middleware

### Primary Recommendation: `SET app.current_tenant` with Drizzle Middleware Wrapper

For the Adeo project with the established tech stack (TimescaleDB/PostgreSQL 16, Drizzle ORM, Next.js frontend container, Docker Compose), I recommend a **hybrid approach** that combines:

1. **PostgreSQL session-level variable** (`SET app.current_tenant`) set on each request
2. **Drizzle middleware/plugin** to automatically inject the SET command when acquiring a connection
3. **RLS policies** that reference `current_setting('app.current_tenant')`

This approach is superior to passing tenant ID through Drizzle's query context for several reasons:

### Why SET Over Query Context

**1. Defense in Depth at Database Layer**

The HLRD explicitly requires "defense in depth at two layers" — application and database. Using `SET app.current_tenant` enables RLS policies to enforce isolation at the database level:

```sql
-- Example RLS policy on emails table
CREATE POLICY tenant_isolation_policy ON emails
FOR ALL
USING (tenant_id = current_setting('app.current_tenant', true)::uuid);
```

If we only pass tenant ID through Drizzle's query context (as a filter parameter), a bug in the application layer (forgotten filter) would result in cross-tenant data leakage. With SET + RLS, even buggy application code cannot return cross-tenant rows — the database itself blocks it.

**2. Simpler Application Code**

With SET, developers write standard Drizzle queries without remembering to include tenant filters:

```typescript
// Without SET approach — error-prone, must remember tenant filter
const emails = await db.query.emails.findMany({
  where: eq(emails.tenantId, request.tenantId) // Easy to forget!
});

// With SET approach — tenant context is automatic
const emails = await db.query.emails.findMany(); // RLS handles isolation
```

**3. Explicit Audit Trail**

The session variable approach leaves a clear audit trail — you can query `current_setting('app.current_tenant')` in logs or debug sessions to verify which tenant context was active during any query.

---

### Implementation Architecture

#### 1. Drizzle Client Configuration with Middleware

Using Drizzle's client middleware pattern to inject the tenant SET command:

```typescript
// frontend/src/lib/db/tenant-middleware.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Middleware } from 'pg-mem';

export function createTenantClient(
  pool: Pool,
  tenantId: string
) {
  // This middleware runs on every query
  const tenantMiddleware: Middleware = async (query, next) => {
    // Set tenant context before executing query
    const setResult = await pool.query(
      `SET app.current_tenant = $1`,
      [tenantId]
    );
    return next(query);
  };

  return drizzle(pool, {
    middleware: [tenantMiddleware],
    schema: { ... }
  });
}
```

However, there's a subtlety: setting the tenant on every single query is inefficient. A better approach uses a **connection-level hook**:

#### 2. Optimized: Set Tenant on Connection Acquisition

```typescript
// frontend/src/lib/db/tenant-client.ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';

let pool: Pool | null = null;
let currentTenantId: string | null = null;

/**
 * Get or create a tenant-scoped pool.
 * In production with PgBouncer, each tenant could have its own pool
 * to ensure complete connection isolation.
 */
export function getTenantDb(tenantId: string) {
  // For simplicity, we use a single pool but set tenant per-request
  // In high-security production, consider per-tenant pools

  if (!pool) {
    pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      // Reserve connections per tenant if using PgBouncer in transaction mode
      max: 20,
    });
  }

  // Create a drizzle client with custom query execution
  const db = drizzle(pool, { schema: { ... } });

  // Wrap to inject tenant context before each query
  return db.$context.withExtensions(async (ctx) => {
    // Ensure tenant is set for this connection
    // Note: This runs once per query in current implementation
    // For better performance, consider PgBouncer with per-tenant connection pools
    await pool!.query(`SET app.current_tenant = $1`, [tenantId]);
  });
}
```

**Actually, the cleanest implementation** uses Drizzle's transaction or raw query hook:

```typescript
// frontend/src/lib/db/tenant-db.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const globalPool = new Pool({ connectionString: process.env.DATABASE_URL! });

// Store tenant context in AsyncLocalStorage for request-scoped isolation
import { AsyncLocalStorage } from 'async_hooks';
const tenantStorage = new AsyncLocalStorage<string>();

export function setTenantContext(tenantId: string) {
  return tenantStorage.run(tenantId, async () => {
    // SET happens once per request, not per query
    await globalPool.query(`SET app.current_tenant = $1`, [tenantId]);
  });
}

export function getCurrentTenantId(): string | undefined {
  return tenantStorage.getStore();
}
```

#### 3. Integration with Next.js API Routes

```typescript
// frontend/src/app/api/emails/route.ts
import { getCurrentTenantId } from '@/lib/db/tenant-db';
import { emails } from '@/db/schema/emails';
import { eq } from 'drizzle-orm';

export async function GET() {
  const tenantId = getCurrentTenantId();
  if (!tenantId) {
    throw new Error('Tenant context not established');
  }

  // RLS will automatically filter based on app.current_tenant
  const result = await db.query.emails.findMany();
  return Response.json(result);
}
```

---

### Connection Pooling Interaction

This is a critical consideration for production deployments.

#### The Problem

With standard PostgreSQL connection pooling (PgBouncer in transaction mode):

1. Connections are reused across requests
2. A connection returned to the pool may still have `app.current_tenant` set from the previous tenant's request
3. If the next tenant acquires that connection before it's reset, tenant leakage occurs

#### Solutions

**Option A: PgBouncer in Session Mode**

Simplest but has performance implications — each transaction gets a fresh connection. Not ideal for high-throughput.

**Option B: Reset on Connection Release**

Configure PgBouncer to reset variables on connection release:

```ini
# pgbouncer.ini
server_reset_query = RESET app.current_tenant;
```

**Option C: Per-Tenant Connection Pools (Recommended for High Security)**

For Adeo where tenant isolation is critical, consider per-tenant pool isolation:

```typescript
// frontend/src/lib/db/tenant-pool.ts
const tenantPools = new Map<string, Pool>();

export function getTenantPool(tenantId: string): Pool {
  if (!tenantPools.has(tenantId)) {
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL!,
      max: 5, // Smaller pool per tenant
    });
    tenantPools.set(tenantId, pool);
  }
  return tenantPools.get(tenantId)!;
}
```

**Option D: Application-Level Connection Attribution with PgBouncer**

Use PgBouncer's `stats` mode with the `application_name` attribute:

```sql
-- In connection string
application_name=tenant-{tenantId}

-- In RLS policy (alternative approach)
CREATE POLICY tenant_isolation ON emails
FOR ALL
USING (
  -- Extract tenant from application_name set at connection time
  (SELECT current_setting('app.tenant_id', true)) = tenant_id
);
```

#### Recommendation for Phase 1

For Phase 1 with simulated data sources and limited scale, use **Option B (server_reset_query)** with the standard pool. This provides adequate safety with minimal complexity:

```ini
# docker-compose.yml for pgbouncer
pgbouncer:
  image: pgbouncer/pgbouncer
  environment:
    DATABASE_URL: ${DATABASE_URL}
    POOL_MODE: transaction
    SERVER_RESET_QUERY: RESET app.current_tenant;
```

---

### RLS Policy Testing in CI Pipeline

RLS policies are critical security infrastructure and must be tested automatically.

#### Recommended Test Strategy

**1. Create a Dedicated RLS Test Suite**

```typescript
// frontend/__tests__/rls-isolation.test.ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { beforeAll, describe, expect, it } from 'vitest';

describe('Row-Level Security Isolation', () => {
  let poolA: Pool;
  let poolB: Pool;
  let dbA: any;
  let dbB: any;
  const tenantA = '00000000-0000-0000-0000-000000000001';
  const tenantB = '00000000-0000-0000-0000-000000000002';

  beforeAll(async () => {
    // Create two separate connections representing different tenants
    poolA = new Pool({ connectionString: process.env.DATABASE_URL });
    poolB = new Pool({ connectionString: process.env.DATABASE_URL });

    // Set tenant context
    await poolA.query(`SET app.current_tenant = $1`, [tenantA]);
    await poolB.query(`SET app.current_tenant = $1`, [tenantB]);

    dbA = drizzle(poolA);
    dbB = drizzle(poolB);
  });

  it('should NOT allow tenant A to read tenant B emails', async () => {
    // Tenant B inserts an email
    await dbB.insert(emails).values({
      tenantId: tenantB,
      subject: 'Confidential B',
      body: 'Secret content',
    });

    // Tenant A queries — should return empty or throw
    const results = await dbA.query.emails.findMany();
    const tenantBEmails = results.filter(e => e.tenantId === tenantB);

    expect(tenantBEmails).toHaveLength(0);
  });

  it('should allow tenant A to read their own emails', async () => {
    await dbA.insert(emails).values({
      tenantId: tenantA,
      subject: 'My Email',
      body: 'My content',
    });

    const results = await dbA.query.emails.findMany();
    expect(results.length).toBeGreaterThan(0);
  });
});
```

**2. CI Pipeline Integration**

```yaml
# .github/workflows/rls-tests.yml
name: RLS Isolation Tests

on: [push, pull_request]

jobs:
  rls-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: Start PostgreSQL with TimescaleDB
        uses: pg-advisor/timescaledb-action@v1
        with:
          postgres-version: '16'

      - name: Initialize database with RLS
        run: |
          pnpm drizzle-kit push
          pnpm tsx scripts/init-rls-policies.ts

      - name: Run RLS tests
        run: pnpm vitest run --config vitest.config.ts __tests__/rls-isolation.test.ts

      - name: Verify no bypass attempts
        run: |
          # Additional security test: attempt SQL injection to bypass RLS
          pnpm tsx scripts/test-rls-bypass.ts
```

**3. Automated RLS Policy Verification Script**

```typescript
// scripts/init-rls-policies.ts
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL! });

async function initRLSPolicies() {
  const client = await pool.connect();

  try {
    // Enable RLS on all tenant-scoped tables
    const tables = ['emails', 'rules', 'actions', 'clients', 'cases', 'brands', 'products', 'orders'];

    for (const table of tables) {
      // Check if RLS is already enabled
      const result = await client.query(`
        SELECT relrowsecurity
        FROM pg_class
        WHERE relname = $1
      `, [table]);

      if (result.rows[0]?.relrowsecurity) {
        console.log(`RLS already enabled on ${table}`);
        continue;
      }

      // Enable RLS
      await client.query(`ALTER TABLE ${table} ENABLE ROW LEVEL SECURITY;`);

      // Create tenant isolation policy
      await client.query(`
        CREATE POLICY tenant_isolation_${table} ON ${table}
        FOR ALL
        USING (tenant_id = current_setting('app.current_tenant', true)::uuid)
        WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::uuid);
      `);

      console.log(`RLS policy created on ${table}`);
    }
  } finally {
    client.release();
  }
}

initRLSPolicies().then(() => process.exit(0));
```

---

### Security Analysis

**Data Isolation Guarantees:**

1. **Application Layer**: Drizzle queries don't require manual tenant filtering (reduces developer error)
2. **Database Layer**: RLS policies block cross-tenant access even if application has bugs
3. **Connection Layer**: Proper pool configuration prevents connection-based leakage

**Attack Vectors Addressed:**

- SQL injection in user inputs (RLS still applies)
- Developer forgetting tenant filter (RLS blocks)
- Connection pool contamination (server_reset_query)
- Direct database access (RLS enforced at DB level)

---

### Performance Implications

| Aspect | Impact | Mitigation |
|--------|--------|------------|
| SET per request | ~0.5-1ms per query | Use connection-level SET, not per-query |
| RLS policy evaluation | ~1-2ms overhead | Ensure index on tenant_id column |
| Connection pool size | Standard PgBouncer | Monitor with pg_stat_activity |
| Query planning | RLS adds minimal overhead | EXPLAIN ANALYZE in CI to verify |

**Recommended Indexes:**

```sql
-- Must have index on tenant_id for RLS performance
CREATE INDEX idx_emails_tenant ON emails(tenant_id);
CREATE INDEX idx_rules_tenant ON rules(tenant_id);
CREATE INDEX idx_actions_tenant ON actions(tenant_id);
-- Add for all tenant-scoped tables
```

---

### Migration and Evolution Path

**Phase 1 (Current):**
- Single-user-per-tenant model (each user = tenant)
- Shared database, RLS on all tenant-scoped tables
- Standard connection pooling

**Future Phases:**
- **Organization tenancy**: Add `organization_id` column, update RLS to handle both user and org levels
- **Multi-tenant pools**: Per-tenant PgBouncer pools for stronger isolation
- **Cross-tenant analytics**: Add superuser role that bypasses RLS for system analytics

---

## Questions for Other SMEs

### For AI/NLP Architecture SME:

1. **Rule evaluation pipeline** — The HLRD mentions pre-filter LLM calls and full evaluation prompts. How should the tenant context be passed to the rule evaluation? Should each tenant have isolated rule configurations stored in their schema, or should rules be evaluated within the tenant's database context?

2. **Action generation context** — When generating actions (e.g., drafting emails), the LLM needs access to entity context (client names, case numbers). Should this context be retrieved via the tenant-scoped database connection we've designed above, ensuring complete isolation during entity lookups?

### For UX Designer SME:

1. **Multi-tenant dashboard** — The dashboard shows action items from triggered rules. How should we indicate tenant context in the UI for users who might have access to multiple tenants in future phases? Should there be a tenant switcher component in the sidebar?

### For Legal/Social Media/Ecommerce Domain SMEs:

1. **Entity data isolation** — Your domain presets include entities (clients, cases, brands, products). The tenant isolation we're implementing ensures Tenant A cannot see Tenant B's entities. Are there any scenarios where cross-tenant entity lookups might be needed (e.g., a law firm with multiple offices sharing a case)?

---

## Summary

The recommended approach for tenant ID session management in Adeo:

1. **Use `SET app.current_tenant`** at the database session level, not Drizzle query context filtering
2. **Implement via Drizzle middleware** that executes the SET on connection acquisition
3. **Configure PgBouncer** with `server_reset_query = RESET app.current_tenant` to prevent connection contamination
4. **Create RLS policies** on all tenant-scoped tables that reference `current_setting('app.current_tenant')`
5. **Automate RLS testing** in CI with dedicated isolation test suites that verify cross-tenant blocking

This approach provides the defense-in-depth isolation required by the HLRD while keeping developer ergonomics high and performance acceptable for Phase 1 scale.
