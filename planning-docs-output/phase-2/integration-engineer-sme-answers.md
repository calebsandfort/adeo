# Integration Engineer SME Answers

## Phase 2b: Cross-SME Consultation

This document contains answers to all questions routed to the Integration Engineer SME from other SMEs during Phase 2.

---

## Questions from AI/NLP Architecture SME

### Question 1: RLS Policy Testing in CI

**Question:** How should we test Postgres RLS policies in CI to ensure tenant isolation? Should we use transaction-level isolation tests or dedicated RLS test fixtures?

### Answer

I recommend **dedicated RLS test fixtures** over transaction-level isolation tests for the following reasons:

**Why Dedicated RLS Test Fixtures Are Superior:**

1. **Explicit Intent** — RLS tests should clearly verify that cross-tenant queries are blocked. Transaction isolation tests verify serializable isolation between concurrent transactions, which is a different concern than RLS enforcement.

2. **Closer to Production Reality** — Real-world tenant isolation happens via RLS policies on individual connections, not through transaction isolation levels. Testing with separate connections (each with its own SET context) accurately reflects how PgBouncer and application connections behave.

3. **CI Clarity** — Dedicated RLS fixtures make failures immediately obvious: "Tenant A can read Tenant B's data" is a clear security failure, whereas transaction isolation test failures can have ambiguous root causes.

**Recommended Implementation:**

```typescript
// frontend/__tests__/rls-isolation.test.ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { beforeAll, describe, expect, it, afterAll } from 'vitest';

describe('RLS Tenant Isolation', () => {
  let poolA: Pool;
  let poolB: Pool;
  let dbA: any;
  let dbB: any;
  const tenantA = '00000000-0000-0000-0000-000000000001';
  const tenantB = '00000000-0000-0000-0000-000000000002';

  beforeAll(async () => {
    // Create separate connections representing different tenants
    poolA = new Pool({ connectionString: process.env.DATABASE_URL });
    poolB = new Pool({ connectionString: process.env.DATABASE_URL });

    // Set tenant context on each connection
    await poolA.query(`SET app.current_tenant = $1`, [tenantA]);
    await poolB.query(`SET app.current_tenant = $1`, [tenantB]);

    dbA = drizzle(poolA);
    dbB = drizzle(poolB);
  });

  afterAll(async () => {
    await poolA.end();
    await poolB.end();
  });

  it('should NOT allow tenant A to read tenant B emails', async () => {
    // Tenant B inserts an email
    await dbB.insert(emails).values({
      tenantId: tenantB,
      subject: 'Confidential B',
      body: 'Secret content',
    });

    // Tenant A queries — should return empty
    const results = await dbA.query.emails.findMany();
    const tenantBEmails = results.filter((e: any) => e.tenantId === tenantB);

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

  it('should block INSERT of data for wrong tenant', async () => {
    // Attempt to insert with wrong tenant context
    await expect(async () => {
      await dbA.insert(emails).values({
        tenantId: tenantB, // Wrong tenant
        subject: 'Sneaky insert',
        body: 'Should fail',
      });
    }).rejects.toThrow();
  });
});
```

**CI Pipeline Integration:**

Run RLS tests in a dedicated CI job that:
1. Spins up a fresh PostgreSQL/TimescaleDB container
2. Runs Drizzle migrations to create tables
3. Executes RLS policy initialization script
4. Runs RLS isolation test suite
5. Verifies no bypass attempts via SQL injection vectors

**Key Point:** Transaction-level isolation tests (SET TRANSACTION ISOLATION LEVEL) test concurrent write conflicts, not RLS. They test a different failure mode and should be used separately if needed for concurrent access patterns.

---

### Question 2: Tenant Context in Connection Pool

**Question:** When using connection poolers (like PgBouncer), how do we ensure `SET app.current_tenant` persists correctly per request without session state leaking between requests?

### Answer

This is a critical security concern. Here's the comprehensive solution:

**The Problem:**

With PgBouncer in transaction mode:
- Connections are reused across requests
- A connection returned to the pool may retain `app.current_tenant` from the previous tenant
- If the next tenant acquires that connection before it's reset, tenant leakage occurs

**Solution: Configure PgBouncer server_reset_query**

The most reliable approach for Phase 1 is configuring PgBouncer to reset session variables on connection release:

```ini
# pgbouncer.ini
[databases]
adeo = host=db port=5432 dbname=adeo

[pgbouncer]
pool_mode = transaction
server_reset_query = RESET app.current_tenant;
max_client_conn = 1000
default_pool_size = 20
```

**Implementation in Docker Compose:**

```yaml
# docker-compose.yml
services:
  pgbouncer:
    image: pgbouncer/pgbouncer:latest
    environment:
      DATABASE_URL: ${DATABASE_URL}
      POOL_MODE: transaction
      SERVER_RESET_QUERY: RESET app.current_tenant;
    ports:
      - "6432:5432"
```

**Alternative Approaches for Higher Security:**

1. **Per-Tenant Connection Pools** — Each tenant gets its own dedicated pool, eliminating shared connection reuse:

```typescript
// frontend/src/lib/db/tenant-pool.ts
const tenantPools = new Map<string, Pool>();

export function getTenantPool(tenantId: string): Pool {
  if (!tenantPools.has(tenantId)) {
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL!,
      max: 5,
    });
    tenantPools.set(tenantId, pool);
  }
  return tenantPools.get(tenantId)!;
}
```

2. **Application-Level Reset on Acquisition** — Explicitly reset tenant context when acquiring a connection from the pool:

```typescript
export async function getTenantConnection(tenantId: string) {
  const client = await pool.connect();
  try {
    // Always reset tenant context on acquisition
    await client.query(`SET app.current_tenant = $1`, [tenantId]);
    return client;
  } catch (e) {
    client.release();
    throw e;
  }
}
```

**Recommendation for Phase 1:**

Use PgBouncer with `server_reset_query = RESET app.current_tenant;`. This provides adequate safety with minimal complexity and is the standard approach for multi-tenant PostgreSQL deployments.

---

## Questions from Legal Domain SME

### Question 3: Entity Linking for Legal

**Question:** The HLRD mentions entity linking (email to case/client). For legal practice, the matching criteria should include case number patterns, client name matching, and possibly opposing counsel identification. What approach would you recommend for the matching logic — deterministic (exact match on case number) or probabilistic (fuzzy matching on names)? Legal data has specific patterns (case numbers like "CV2024-012345") that could enable deterministic matching.

### Answer

I recommend a **hybrid approach: deterministic matching first, probabilistic as fallback**.

**Rationale:**

1. **Legal case numbers have standardized patterns** that enable reliable deterministic matching
2. **Fuzzy matching introduces false positives** that are dangerous in legal contexts (wrong case assignment = malpractice risk)
3. **The cost of missed matching** is lower than the cost of wrong matching

**Proposed Implementation:**

```typescript
// Priority 1: Exact case number match (highest confidence)
function matchByCaseNumber(email: string, cases: Case[]): MatchResult | null {
  const caseNumberPattern = /[A-Z]{2,4}\d{4}[-/]\d{6}/g;
  const foundNumbers = email.match(caseNumberPattern);

  for (const num of foundNumbers || []) {
    const matched = cases.find(c => c.caseNumber === num || c.caseNumber === num.replace('-', ''));
    if (matched) {
      return { entity: matched, confidence: 0.98, method: 'case_number_exact' };
    }
  }
  return null;
}

// Priority 2: Exact client name match (high confidence)
function matchByClientName(email: string, clients: Client[]): MatchResult | null {
  const emailLower = email.toLowerCase();

  for (const client of clients) {
    // Check for client name as substring in email
    const namePattern = new RegExp(`\\b${escapeRegex(client.name)}\\b`, 'i');
    if (namePattern.test(emailLower)) {
      return { entity: client, confidence: 0.85, method: 'client_name_exact' };
    }
  }
  return null;
}

// Priority 3: Fuzzy matching (lower confidence, requires review)
async function fuzzyMatch(email: string, entities: Entity[]): Promise<MatchResult | null> {
  const embeddings = await getEmbeddings(email);
  const results = await searchSimilarEntities(embeddings, entities, threshold: 0.75);

  if (results.length > 0) {
    return { entity: results[0], confidence: results[0].score * 0.8, method: 'fuzzy' };
  }
  return null;
}

// Entity linking pipeline
async function linkEmailToEntity(email: Email, context: TenantContext): Promise<EntityLink[]> {
  const links: EntityLink[] = [];

  // Try deterministic methods first
  const caseMatch = matchByCaseNumber(email.body, context.cases);
  if (caseMatch) links.push(caseMatch);

  const clientMatch = matchByClientName(email.body, context.clients);
  if (clientMatch) links.push(clientMatch);

  // Only use fuzzy if deterministic methods failed
  if (links.length === 0) {
    const fuzzyResult = await fuzzyMatch(email.body, [...context.cases, ...context.clients]);
    if (fuzzyResult) {
      // Mark fuzzy matches for user review
      links.push({ ...fuzzyResult, requiresReview: true });
    }
  }

  return links;
}
```

**Schema Support for Case Numbers:**

```sql
-- Case numbers have specific patterns per jurisdiction
CREATE TABLE cases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  case_number VARCHAR(50) NOT NULL,  -- e.g., "CV2024-012345"
  client_id UUID REFERENCES clients(id),
  case_name VARCHAR(255),
  jurisdiction VARCHAR(50),
  status VARCHAR(20),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast case number lookups
CREATE INDEX idx_cases_case_number ON cases(case_number);
CREATE INDEX idx_cases_tenant ON cases(tenant_id);
```

**Why Not Pure Probabilistic:**

- Legal malpractice risk — matching an email to the wrong case could lead to missed deadlines or incorrect responses
- Case numbers are designed for exact identification — they should be the primary mechanism
- Fuzzy matching should only be a fallback, and those matches should require user confirmation before being used

---

### Question 4: Data Retention for Legal

**Question:** Client communications, case files, and court documents may have specific retention requirements. Should the system track evaluation history and action approvals differently for legal domain (longer retention, audit trail for malpractice defense) compared to other domains?

### Answer

**Yes, the legal domain requires distinct retention and audit policies.** Here's the recommended approach:

**1. Domain-Specific Retention Configuration:**

```typescript
// Domain retention policies
const retentionPolicies = {
  legal: {
    evaluationHistory: { duration: '7_years', reason: 'malpractice_defense' },
    actionApprovals: { duration: '7_years', reason: 'bar_compliance' },
    emailMetadata: { duration: '10_years', reason: 'statute_of_limitations' },
  },
  social_media: {
    evaluationHistory: { duration: '1_year', reason: 'campaign_analysis' },
    actionApprovals: { duration: '2_years', reason: 'brand_audit' },
    emailMetadata: { duration: '1_year', reason: 'data_minimization' },
  },
  ecommerce: {
    evaluationHistory: { duration: '3_years', reason: 'tax_compliance' },
    actionApprovals: { duration: '7_years', reason: 'tax_compliance' },
    emailMetadata: { duration: '3_years', reason: 'customer_service' },
  },
};
```

**2. Extended Audit Trail for Legal:**

```sql
-- Evaluation history with legal-specific fields
CREATE TABLE evaluation_audit (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  email_id UUID NOT NULL,
  rule_id UUID NOT NULL,
  triggered BOOLEAN NOT NULL,
  confidence DECIMAL(3,2),
  extracted_context JSONB,
  evaluated_at TIMESTAMPTZ DEFAULT NOW(),
  -- Legal-specific additions
  urgency_classification VARCHAR(20),  -- critical, high, medium, low
  deadline_extracted DATE,
  court_jurisdiction VARCHAR(50),
  opposing_counsel_identified BOOLEAN,
  requires_bar_compliance BOOLEAN DEFAULT FALSE,
  evaluated_by VARCHAR(50) DEFAULT 'llm',  -- llm, hybrid, human
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Action approvals with legal signature tracking
CREATE TABLE action_audit (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  action_id UUID NOT NULL,
  email_id UUID NOT NULL,
  action_type VARCHAR(50) NOT NULL,
  generated_draft TEXT,
  approved_draft TEXT,
  approved_at TIMESTAMPTZ,
  approved_by UUID REFERENCES users(id),
  revision_count INT DEFAULT 0,
  -- Legal-specific additions
  sent_to_court BOOLEAN DEFAULT FALSE,
  sent_to_client BOOLEAN DEFAULT FALSE,
  malpractice_review_required BOOLEAN DEFAULT FALSE,
  disclaimer_included BOOLEAN DEFAULT FALSE,
  signature_confirmed BOOLEAN DEFAULT FALSE,
  executed_at TIMESTAMPTZ,
  execution_method VARCHAR(20),  -- email, fax, mail, portal
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**3. Legal Disclaimer Tracking:**

```typescript
interface LegalActionMetadata {
  requiresDisclaimers: boolean;
  disclaimersIncluded: string[];
  barComplianceChecked: boolean;
  jurisdictionRequirements: string[];
  conflictCheckPerformed: boolean;
  conflictCheckResult: 'cleared' | 'flagged' | 'not_applicable';
}

async function generateLegalAction(
  email: Email,
  rule: Rule,
  context: LegalContext
): Promise<Action> {
  const draft = await generateDraft(email, rule, context);

  // Add required legal disclaimers
  const disclaimers = [
    'This communication is protected by attorney-client privilege.',
    'This is a draft for attorney review only.',
    'The contents do not constitute legal advice until reviewed by an attorney.',
  ];

  return {
    ...draft,
    metadata: {
      ...draft.metadata,
      legal: {
        disclaimersIncluded: disclaimers,
        requiresBarCompliance: context.jurisdictionRequiresBarNumber,
        jurisdiction: context.jurisdiction,
      },
    },
  };
}
```

**4. Retention Implementation:**

```typescript
// Retention enforcement
async function enforceRetentionPolicy(tenantId: string, domain: string) {
  const policy = retentionPolicies[domain];
  if (!policy) return;

  const cutoff = calculateCutoff(policy.evaluationHistory.duration);

  await db.delete(evaluationAudit)
    .where(
      and(
        eq(evaluationAudit.tenantId, tenantId),
        lt(evaluationAudit.createdAt, cutoff)
      )
    );
}
```

**Why This Matters for Legal:**

- **Malpractice defense** — If a client claims an email was missed, audit trail shows evaluation results and decisions
- **Bar compliance** — Most state bars require documentation of communications
- **Statute of limitations** — Legal matters can have extended limitations periods
- **Professional responsibility** — Attorneys must demonstrate adequate communication with clients

---

## Questions from Social Media Domain SME

### Question 5: Twitter/X API Simulation

**Question:** For the demo, how should simulated Twitter data be structured to realistically represent API responses? Should we include engagement metrics (likes, retweets) as part of the trigger evaluation?

### Answer

**Yes, include engagement metrics as part of the trigger evaluation** — they are critical for determining urgency and response priority.

**Simulated Twitter Data Schema:**

```typescript
// Simulated Twitter/X API response structure
interface SimulatedTweet {
  id: string;
  text: string;
  author: {
    id: string;
    username: string;
    name: string;
    followers_count: number;
    verified: boolean;
  };
  created_at: string;
  // Engagement metrics (critical for trigger evaluation)
  public_metrics: {
    retweet_count: number;
    reply_count: number;
    like_count: number;
    quote_count: number;
    impression_count?: number;
  };
  // Context for rule evaluation
  referenced_tweets?: Array<{
    type: 'replied_to' | 'quoted' | 'retweeted';
    id: string;
  }>;
  context_annotations?: Array<{
    domain: { id: string; name: string };
    entity: { id: string; name: string };
  }>;
}

// Database schema for simulated tweets
const tweets = pgTable('tweets', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  brandId: uuid('brand_id').references(() => brands.id),
  // Twitter data
  tweetId: varchar('tweet_id', { length: 50 }).notNull(),
  text: text('text').notNull(),
  authorId: varchar('author_id', { length: 50 }).notNull(),
  authorUsername: varchar('author_username', { length: 100 }).notNull(),
  authorName: varchar('author_name', { length: 100 }),
  authorFollowers: integer('author_followers').default(0),
  authorVerified: boolean('author_verified').default(false),
  createdAt: timestamp('created_at').notNull(),
  // Engagement metrics
  retweetCount: integer('retweet_count').default(0),
  replyCount: integer('reply_count').default(0),
  likeCount: integer('like_count').default(0),
  quoteCount: integer('quote_count').default(0),
  impressionCount: integer('impression_count'),
  // Computed for triggers
  engagementScore: integer('engagement_score'), // Computed: likes + retweets * 2 + replies * 3
  isSpike: boolean('is_spike').default(false),
  // For demo purposes
  isSimulated: boolean('is_simulated').default(true),
  simulatedAt: timestamp('simulated_at'),
});
```

**Simulated Data Generation:**

```typescript
// Seed data generator for demo
function generateSimulatedTweets(brand: Brand, count: number): SimulatedTweet[] {
  const tweets: SimulatedTweet[] = [];

  // Generate variety of engagement levels
  const engagementProfiles = [
    { likes: 0, retweets: 0, replies: 1, probability: 0.6 },  // Low engagement (most common)
    { likes: 10, retweets: 2, replies: 3, probability: 0.25 }, // Medium
    { likes: 100, retweets: 25, replies: 15, probability: 0.1 }, // High
    { likes: 500, retweets: 100, replies: 50, probability: 0.05 }, // Viral
  ];

  for (let i = 0; i < count; i++) {
    const profile = weightedRandom(engagementProfiles);
    tweets.push({
      id: `sim_tweet_${i}`,
      text: generateBrandRelevantText(brand, profile),
      author: generateAuthor(profile),
      created_at: randomPastTimestamp(7), // Past 7 days
      public_metrics: profile,
    });
  }

  return tweets;
}
```

**Using Engagement Metrics in Trigger Evaluation:**

```typescript
// Rule evaluation that uses engagement metrics
const socialMediaRules = [
  {
    name: 'Negative Mention with High Engagement',
    condition: async (tweet: SimulatedTweet, brand: Brand) => {
      const isNegative = await analyzeSentiment(tweet.text);
      const hasBrandMention = tweet.text.toLowerCase().includes(brand.name.toLowerCase());
      const isHighEngagement = tweet.public_metrics.like_count > 50 ||
                               tweet.public_metrics.retweet_count > 10;

      return isNegative && hasBrandMention && isHighEngagement;
    },
    priority: 'critical',
  },
  {
    name: 'Viral Positive Mention',
    condition: async (tweet: SimulatedTweet, brand: Brand) => {
      const isPositive = await analyzeSentiment(tweet.text);
      const isViral = tweet.public_metrics.like_count > 200 ||
                      tweet.public_metrics.retweet_count > 50;

      return isPositive && isViral;
    },
    priority: 'high',
  },
];
```

---

### Question 6: Historical Trend Data Detection

**Question:** The system needs to detect "spikes" in mentions. What's the appropriate time window for spike detection (1 hour? 6 hours? 24 hours)? Should this be configurable per brand?

### Answer

**Recommended: Configurable time windows with 1-hour default for critical, 6-hour for standard.**

**Rationale:**

- **1-hour window** — Appropriate for crisis detection and influencer engagement (2-4 hour response window)
- **6-hour window** — Standard for trend detection in social media management
- **24-hour window** — Useful for analytics/reporting but not real-time triggers

**Configuration Schema:**

```typescript
// Brand-level trend detection configuration
interface TrendConfig {
  brandId: string;
  // Time windows (in minutes)
  criticalWindowMinutes: number;   // Default: 60 (1 hour)
  standardWindowMinutes: number;    // Default: 360 (6 hours)
  analyticsWindowMinutes: number;   // Default: 1440 (24 hours)
  // Thresholds
  spikeThresholdCount: number;      // Default: 5 mentions in window
  spikeThresholdPercentIncrease: number;  // Default: 200% increase from baseline
  // What counts as "mention"
  includeReplies: boolean;          // Default: true
  includeRetweets: boolean;         // Default: false
  includeQuotes: boolean;           // Default: true
}

// Default configuration
const defaultTrendConfig: TrendConfig = {
  criticalWindowMinutes: 60,
  standardWindowMinutes: 360,
  analyticsWindowMinutes: 1440,
  spikeThresholdCount: 5,
  spikeThresholdPercentIncrease: 200,
  includeReplies: true,
  includeRetweets: false,
  includeQuotes: true,
};
```

**Spike Detection Algorithm:**

```typescript
async function detectSpike(
  brandId: string,
  config: TrendConfig
): Promise<SpikeDetection> {
  const now = new Date();

  // Calculate baseline (average mentions per hour over past 7 days)
  const baseline = await calculateBaseline(brandId, days: 7);

  // Count mentions in critical window
  const criticalStart = new Date(now.getTime() - config.criticalWindowMinutes * 60000);
  const criticalMentions = await countMentions(brandId, {
    startTime: criticalStart,
    endTime: now,
    includeReplies: config.includeReplies,
    includeRetweets: config.includeRetweets,
    includeQuotes: config.includeQuotes,
  });

  // Count mentions in standard window
  const standardStart = new Date(now.getTime() - config.standardWindowMinutes * 60000);
  const standardMentions = await countMentions(brandId, {
    startTime: standardStart,
    endTime: now,
    ...config,
  });

  // Determine spike status
  const criticalSpike = criticalMentions.count >= config.spikeThresholdCount ||
                        criticalMentions.count > (baseline.hourly * (config.spikeThresholdPercentIncrease / 100));

  const standardSpike = standardMentions.count >= config.spikeThresholdCount ||
                        standardMentions.count > (baseline.hourly * 6 * (config.spikeThresholdPercentIncrease / 100));

  return {
    isSpiking: criticalSpike || standardSpike,
    severity: criticalSpike ? 'critical' : standardSpike ? 'elevated' : 'normal',
    mentionsInWindow: criticalMentions.count,
    standardMentionsInWindow: standardMentions.count,
    baselineHourly: baseline.hourly,
    percentAboveBaseline: ((criticalMentions.count / baseline.hourly) - 1) * 100,
    windowUsed: criticalSpike ? 'critical' : 'standard',
  };
}
```

**Database Schema for Trend Tracking:**

```sql
-- Track mention counts over time for spike detection
CREATE TABLE mention_trends (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  brand_id UUID NOT NULL,
  window_start TIMESTAMPTZ NOT NULL,
  window_minutes INT NOT NULL,
  mention_count INT NOT NULL,
  sentiment_breakdown JSONB,  -- { positive: 10, negative: 2, neutral: 5 }
  computed_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(tenant_id, brand_id, window_start, window_minutes)
);

-- Index for efficient trend queries
CREATE INDEX idx_mention_trends_brand_time ON mention_trends(brand_id, window_start DESC);
```

**Why Configurable:**

Different brands have different baselines:
- A major brand with 100K mentions/day needs different thresholds than a local business with 10 mentions/day
- A brand with frequent viral moments (entertainment) has different "normal" than a B2B brand
- Configurability allows the system to adapt to each brand's social media velocity

---

## Questions from E-commerce Domain SME

### Question 7: Product Entity Schema for Multi-Marketplace

**Question:** How should the product entity schema handle multi-marketplace listings? A single product might have different SKUs on Shopify vs. Amazon, different inventory levels, and different review profiles. Should the entity model support marketplace-specific sub-records?

### Answer

**Yes, use a hybrid approach: master product with marketplace-specific sub-records.**

**Recommended Schema:**

```sql
-- Master product (brand/tenant level)
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100),  -- apparel, electronics, home, etc.
  brand VARCHAR(100),
  base_sku VARCHAR(100),
  -- Common fields across marketplaces
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Marketplace-specific product data
CREATE TABLE product_marketplaces (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id),
  tenant_id UUID NOT NULL,
  marketplace VARCHAR(50) NOT NULL,  -- 'shopify', 'amazon', 'ebay', 'walmart'
  -- Marketplace-specific identifiers
  external_product_id VARCHAR(100) NOT NULL,  -- Amazon ASIN, Shopify product ID
  sku VARCHAR(100),                           -- Marketplace-specific SKU
  -- Marketplace-specific inventory
  inventory_count INT DEFAULT 0,
  reserved_count INT DEFAULT 0,
  -- Thresholds (can differ by marketplace)
  low_stock_threshold INT DEFAULT 10,
  reorder_point INT DEFAULT 20,
  -- Marketplace-specific pricing
  price DECIMAL(10, 2),
  currency VARCHAR(3) DEFAULT 'USD',
  -- Review data (marketplace-specific)
  review_count INT DEFAULT 0,
  average_rating DECIMAL(2, 1),
  last_review_sync TIMESTAMPTZ,
  -- Status
  status VARCHAR(20) DEFAULT 'active',  -- active, inactive, pending
  listed_at TIMESTAMPTZ,
  delisted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(marketplace, external_product_id)
);

-- Inventory history (marketplace-specific)
CREATE TABLE inventory_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_marketplace_id UUID NOT NULL REFERENCES product_marketplaces(id),
  tenant_id UUID NOT NULL,
  quantity_change INT NOT NULL,  -- Positive for receive, negative for sale
  quantity_after INT NOT NULL,
  reason VARCHAR(50),  -- sale, return, restock, adjustment, transfer
  occurred_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Entity Model in Application Code:**

```typescript
// Unified product entity with marketplace details
interface Product {
  id: string;
  tenantId: string;
  name: string;
  description?: string;
  category: ProductCategory;
  brand?: string;
  baseSku?: string;
  // All marketplace listings
  marketplaces: ProductMarketplace[];
  // Aggregated data (computed)
  totalInventory: number;
  averageRating: number;
  totalReviewCount: number;
}

interface ProductMarketplace {
  id: string;
  marketplace: Marketplace;
  externalProductId: string;
  sku?: string;
  inventory: {
    available: number;
    reserved: number;
    lowStockThreshold: number;
    reorderPoint: number;
  };
  pricing: {
    amount: number;
    currency: string;
  };
  reviews: {
    count: number;
    averageRating: number;
  };
  status: 'active' | 'inactive' | 'pending';
}

// Access patterns
function getProductForMarketplace(productId: string, marketplace: Marketplace): ProductMarketplace | null {
  const product = await db.query.products.findFirst({
    where: eq(products.id, productId),
    with: {
      marketplaces: {
        where: eq(productMarketplaces.marketplace, marketplace),
      },
    },
  });
  return product?.marketplaces[0] || null;
}

function getTotalInventory(productId: string): number {
  // Sum across all marketplaces
  return db.select({ sum: sql`SUM(inventory_count)` })
    .from(productMarketplaces)
    .where(eq(productMarketplaces.productId, productId));
}
```

**Why This Approach:**

1. **Single source of truth** — Master product record for common attributes
2. **Marketplace flexibility** — Each marketplace can have different SKUs, inventory, pricing
3. **Easy aggregation** — Total inventory, average rating across marketplaces
4. **Future-proof** — Add new marketplaces without schema changes
5. **Rule evaluation** — Rules can evaluate per-marketplace or aggregated

---

### Question 8: Simulated Data Providers

**Question:** For the simulated data sources, how should we represent the difference between Shopify and Amazon API payloads? Should we have separate simulated providers for each, or a unified provider with marketplace configuration?

### Answer

**Use a unified provider with marketplace-specific configuration and transformation layers.**

**Architecture:**

```typescript
// Unified simulated data provider
interface SimulatedDataProvider {
  marketplace: Marketplace;
  config: MarketplaceConfig;
  // Generate simulated data
  generateOrders(count: number): Promise<SimulatedOrder[]>;
  generateProducts(count: number): Promise<SimulatedProduct[]>;
  generateCustomers(count: number): Promise<SimulatedCustomer[]>;
  generateReviews(count: number): Promise<SimulatedReview[]>;
  generateInventoryAlerts(count: number): Promise<SimulatedAlert[]>;
}

// Marketplace-specific configurations
const marketplaceConfigs: Record<Marketplace, MarketplaceConfig> = {
  shopify: {
    orderFields: ['id', 'email', 'total_price', 'financial_status', 'fulfillment_status'],
    productFields: ['id', 'title', 'variants', 'inventory_quantity'],
    customerFields: ['id', 'email', 'first_name', 'last_name'],
    reviewFields: ['id', 'product_id', 'rating', 'body', 'author'],
    terminology: {
      order: 'Order',
      product: 'Product',
      customer: 'Customer',
      price: 'total_price',
      status: 'fulfillment_status',
    },
  },
  amazon: {
    orderFields: ['AmazonOrderId', 'BuyerEmail', 'OrderTotal', 'OrderStatus', 'FulfillmentChannel'],
    productFields: ['ASIN', 'ItemName', 'SKU', 'Quantity'],
    customerFields: ['BuyerName', 'BuyerEmail', 'ShippingAddress'],
    reviewFields: ['ReviewId', 'ASIN', 'OverallRating', 'ReviewText', 'ReviewerId'],
    terminology: {
      order: 'Order',
      product: 'ASIN',
      customer: 'Buyer',
      price: 'OrderTotal.Amount',
      status: 'OrderStatus',
    },
  },
};
```

**Data Generation with Marketplace-Specific Payloads:**

```typescript
class EcommerceSimulator {
  constructor(private marketplace: Marketplace) {}

  async generateOrderNotification(): Promise<ShopifyOrder | AmazonOrder> {
    const baseData = this.generateBaseOrderData();

    if (this.marketplace === 'shopify') {
      return {
        id: `shopify_${baseData.orderNumber}`,
        email: baseData.customerEmail,
        total_price: baseData.total.toFixed(2),
        currency: 'USD',
        financial_status: baseData.paymentStatus,
        fulfillment_status: baseData.fulfillmentStatus,
        line_items: baseData.items.map(item => ({
          id: `line_${item.id}`,
          title: item.name,
          quantity: item.quantity,
          price: item.price.toFixed(2),
          sku: item.sku,
        })),
        created_at: baseData.createdAt.toISOString(),
      } as ShopifyOrder;
    } else {
      return {
        AmazonOrderId: `AMZ_${baseData.orderNumber}`,
        BuyerEmail: baseData.customerEmail,
        OrderTotal: {
          Amount: baseData.total.toFixed(2),
          CurrencyCode: 'USD',
        },
        OrderStatus: baseData.amazonOrderStatus,
        FulfillmentChannel: baseData.fulfillmentChannel,
        OrderItems: baseData.items.map(item => ({
          ASIN: item.asin,
          QuantityOrdered: item.quantity,
          ItemPrice: {
            Amount: (item.price * item.quantity).toFixed(2),
            CurrencyCode: 'USD',
          },
        })),
        PurchaseDate: baseData.createdAt.toISOString(),
      } as AmazonOrder;
    }
  }

  async generateReviewAlert(): Promise<ShopifyReview | AmazonReview> {
    const baseData = this.generateBaseReviewData();

    if (this.marketplace === 'shopify') {
      return {
        id: `review_${baseData.id}`,
        product_id: `shopify_prod_${baseData.productId}`,
        rating: baseData.rating,
        body: baseData.body,
        author: baseData.author,
        created_at: baseData.createdAt.toISOString(),
        verified_buyer: baseData.verifiedBuyer,
      } as ShopifyReview;
    } else {
      return {
        ReviewId: `R${baseData.id}`,
        ASIN: `B${baseData.productId}`,
        OverallRating: baseData.rating,
        ReviewText: baseData.body,
        ReviewerId: `AR${baseData.authorId}`,
        ReviewDate: baseData.createdAt.toISOString(),
        VerifiedPurchase: baseData.verifiedBuyer,
      } as AmazonReview;
    }
  }
}
```

**Unified Normalization Layer:**

```typescript
// Normalize marketplace-specific data to unified schema
function normalizeOrder(order: ShopifyOrder | AmazonOrder): UnifiedOrder {
  if ('id' in order && order.id.startsWith('shopify_')) {
    return {
      orderId: order.id,
      marketplace: 'shopify',
      customerEmail: order.email,
      total: parseFloat(order.total_price),
      status: mapShopifyStatus(order.fulfillment_status),
      items: order.line_items.map(li => ({
        productId: li.sku,
        quantity: li.quantity,
        price: parseFloat(li.price),
      })),
    };
  } else {
    return {
      orderId: order.AmazonOrderId,
      marketplace: 'amazon',
      customerEmail: order.BuyerEmail,
      total: parseFloat(order.OrderTotal.Amount),
      status: mapAmazonStatus(order.OrderStatus),
      items: order.OrderItems.map(oi => ({
        productId: oi.ASIN,
        quantity: oi.QuantityOrdered,
        price: parseFloat(oi.ItemPrice.Amount),
      })),
    };
  }
}
```

**Demo Setup:**

```typescript
// Initialize both marketplace simulators for demo
const shopifySimulator = new EcommerceSimulator('shopify');
const amazonSimulator = new EcommerceSimulator('amazon');

// Generate demo data
const demoData = {
  shopify: {
    orders: await shopifySimulator.generateOrders(20),
    products: await shopifySimulator.generateProducts(10),
    reviews: await shopifySimulator.generateReviews(15),
  },
  amazon: {
    orders: await amazonSimulator.generateOrders(20),
    products: await amazonSimulator.generateProducts(10),
    reviews: await amazonSimulator.generateReviews(15),
  },
};
```

**Why This Approach:**

1. **Single codebase** — One simulator implementation with configuration
2. **Realistic payloads** — Each marketplace returns data in its actual format
3. **Easy comparison** — Demo can show how same scenario appears in different marketplaces
4. **Extensible** — Adding Walmart/eBay only requires new config
5. **Testable** — Normalization layer can be tested independently

---

## Questions from UX Designer SME

### Question 9: Synthetic Data Realism

**Question:** The HLRD mentions "simulated data sources" seeded with synthetic data. How realistic should the email subjects and body content be for testing the UX? Should we invest in LLM-generated synthetic emails for more varied edge cases, or is hand-crafted seed data sufficient for demonstrating the dashboard patterns?

### Answer

**I recommend a hybrid approach: hand-crafted seed data for core patterns, LLM-generated for edge cases and variety.**

**Rationale:**

- **Hand-crafted data** ensures core use cases are clearly demonstrated
- **LLM-generated data** provides the variety needed to test edge cases and prompt tuning
- For demo purposes, hand-crafted data is sufficient initially, with LLM generation added as needed

**Recommended Implementation:**

```typescript
// 1. Core seed data: Hand-crafted for clear demonstration
const coreEmailSeeds: SeedEmail[] = [
  {
    category: 'legal_deadline',
    subject: 'Court Order - Response Due March 15',
    body: `Dear Attorney,

Please be advised that the Court has issued an order in the above-referenced matter requiring your client's response within 30 days.

The motion for summary judgment filed by opposing counsel is attached. Failure to respond may result in the Court granting the motion.

Please contact our office if you have any questions.

Sincerely,
Court Clerk`,
    shouldTrigger: true,
    entityMatch: { caseNumber: 'CV2024-012345' },
  },
  {
    category: 'legal_hearing_change',
    subject: 'HEARING CANCELLED - Martinez v. Smith',
    body: `The hearing scheduled for March 15, 2024 at 9:00 AM has been cancelled due to the Court's calendar conflict.

A new hearing date will be scheduled and notice will be provided.

Please update your records accordingly.`,
    shouldTrigger: true,
    entityMatch: { caseNumber: 'CV2024-012345', clientName: 'Martinez' },
  },
  {
    category: 'legal_client_inquiry',
    subject: 'Question about my case',
    body: `Hi,

I wanted to check in on the status of my case. I've been worried about what's happening and haven't heard from you in a while.

When can I expect an update?

Thanks,
Maria`,
    shouldTrigger: true,
    entityMatch: { clientName: 'Maria' },
  },
  // ... 20-30 core seeds covering each domain
];

// 2. Edge case generation: LLM for variety
async function generateEdgeCaseEmails(
  domain: 'legal' | 'social_media' | 'ecommerce',
  count: number,
  existingPatterns: SeedEmail[]
): Promise<SeedEmail[]> {
  const prompt = `Generate ${count} emails for ${domain} domain that test edge cases:

Existing patterns to avoid duplicating:
${existingPatterns.map(e => `- ${e.category}: ${e.subject}`).join('\n')}

For each email provide:
- subject: Realistic subject line
- body: Email body
- category: What type of situation it represents
- shouldTrigger: Whether it should trigger a rule (true/false)
- edgeCaseType: The specific edge case (e.g., "sarcasm", "mixed_sentiment", "ambiguous_deadline")

Make some emails intentionally difficult:
- Sarcasm or irony
- Multiple entities mentioned
- Ambiguous deadlines ("sometime next week")
- Very short or incomplete emails
- Emails that look like they should trigger but shouldn't`;

  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
    response_format: { type: 'json_object' },
  });

  return JSON.parse(response.choices[0].message.content).emails;
}

// 3. Seed data generator
async function seedDatabase(tenantId: string, domain: string) {
  // Insert core hand-crafted data
  for (const seed of coreEmailSeeds.filter(e => e.domain === domain)) {
    await db.insert(emails).values({
      tenantId,
      subject: seed.subject,
      body: seed.body,
      from: seed.from,
      category: seed.category,
      isSimulated: true,
    });
  }

  // Optionally generate edge cases (for advanced testing)
  if (process.env.GENERATE_EDGE_CASES === 'true') {
    const edgeCases = await generateEdgeCaseEmails(domain, 50, coreEmailSeeds);
    for (const edge of edgeCases) {
      await db.insert(emails).values({
        tenantId,
        subject: edge.subject,
        body: edge.body,
        category: edge.category,
        isSimulated: true,
        isEdgeCase: true,
      });
    }
  }
}
```

**Recommended Seed Data Quantities:**

| Purpose | Count | Method |
|---------|-------|--------|
| Core demonstration | 20-30 emails | Hand-crafted |
| Pattern variety | 50-100 emails | LLM-generated |
| Edge case testing | 50-100 emails | LLM-generated with specific prompts |
| Rule validation | 100-200 emails | LLM-generated |

**For Phase 1 Demo Specifically:**

For demonstrating dashboard patterns (the primary goal of Phase 1), hand-crafted seed data is **sufficient and preferable**:

1. **Clarity** — Each email clearly demonstrates a specific trigger scenario
2. **Predictability** — Testers know what should happen
3. **Coverage** — Can ensure all major rule types are represented
4. **No setup overhead** — No LLM calls needed for demo data

Reserve LLM generation for:
- Rule testing/validation (need many examples)
- Prompt refinement (need edge cases to tune)
- Stress testing (need volume)

---

## Summary of Answers

| Question | Answer Summary |
|----------|----------------|
| RLS Testing in CI | Use dedicated RLS test fixtures with separate connections, not transaction-level isolation tests |
| Tenant Context in Pool | Configure PgBouncer with `server_reset_query = RESET app.current_tenant;` |
| Entity Linking (Legal) | Hybrid: deterministic (case number, exact name) first, probabilistic as fallback |
| Data Retention (Legal) | Yes, distinct 7-year retention for legal audit trail and malpractice defense |
| Twitter/X Simulation | Include engagement metrics — critical for urgency determination |
| Trend Detection | Configurable windows: 1-hour critical, 6-hour standard, 24-hour analytics |
| Product Schema | Master product with marketplace-specific sub-records |
| Simulated Providers | Unified provider with marketplace-specific transformation layers |
| Synthetic Data Realism | Hybrid: hand-crafted for core demo, LLM for edge cases |

---

*Answers prepared by Integration Engineer SME for Phase 2b of the Adeo requirements elaboration workflow.*
