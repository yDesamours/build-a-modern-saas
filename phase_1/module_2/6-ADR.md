# Chapter 6 — Architecture Decision Records (ADRs)

## What Are Architecture Decision Records?

**Architecture Decision Records (ADRs)** are documents that capture important architectural decisions along with their context and consequences.

Think of ADRs as a journal for your architecture:

- Why did we choose PostgreSQL over MongoDB?
- Why did we start with a monolith?
- Why do we use JWT instead of sessions?
- Why did we choose microservices for service X?

**The Problem ADRs Solve:**

Six months later, a new team member asks: _"Why did we build it this way?"_

Without ADRs:

- ❌ "I think John decided that, but he left the company"
- ❌ "We just followed the tutorial"
- ❌ "No idea, it was like this when I joined"
- ❌ Team makes the same mistakes again

With ADRs:

- ✅ Clear documentation of decisions
- ✅ Context preserved over time
- ✅ New team members understand the "why"
- ✅ Easy to revisit decisions when context changes

---

## ADR vs Design Doc / RFC

Before looking at the format, it's worth being precise about what an ADR is _not_. It is not the place where you explore a problem — weigh a dozen options, prototype two of them, argue back and forth about trade-offs. That exploration belongs in a **design doc** or **RFC** (Request for Comments), a longer, working document meant to be read while the decision is still being made.

An ADR is the concise **summary** of what came out of that process: the context that mattered, the decision made, and its consequences. It's written after (or alongside) the exploration, not instead of it. If a design doc already exists, the ADR can simply link to it rather than reproducing its content — this is exactly what keeps ADRs short and what keeps them useful years later, when nobody has time to reread a ten-page exploration just to find out _what_ was decided.

## ADR Format

### The Light Format

The original template proposed by Michael Nygard — the person who coined "ADR" — is deliberately minimal:

```markdown
# ADR-[NUMBER]: [TITLE]

## Status

[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

## Context

What is the issue we're facing? What factors are at play?

## Decision

What decision did we make?

## Consequences

What are the positive and negative consequences of this decision?
```

Four sections, a few paragraphs each, meant to fit on a single screen. Most ADRs should look like this. Nygard's own guidance is explicit that these should read almost like meeting notes — not a formal spec.

### The Heavy Format

The examples later in this chapter use an extended version of the same skeleton:

```markdown
# ADR-[NUMBER]: [Short Title]

**Date:** YYYY-MM-DD  
**Status:** [Proposed | Accepted | Deprecated | Superseded by ADR-XXX]  
**Deciders:** [List of people involved]  
**Projects:** [Which projects this affects]

## Context

...

## Decision

...

## Consequences

### Positive / Negative / Mitigation Strategies

...

## Alternatives Considered

...

## Review Criteria

...

## References

...
```

This adds `Deciders`, a structured `Alternatives Considered` section, `Mitigation Strategies`, `Review Criteria`, and `References`. It's more thorough — and more expensive to write and to read.

**When does the extra weight earn its keep?** Reach for the heavy format when the decision is high-stakes or costly to reverse, when it crosses team or module boundaries, or when you can already tell it's likely to be questioned again later (e.g. by an auditor, a new hire, or your future self during an incident). For everything else — most day-to-day architectural choices — the light format is not just acceptable, it's the better choice: a short ADR that actually gets written and read beats a long one that gets skipped.

## Which Decisions Deserve an ADR?

Create an ADR for decisions that:

- Are architecturally significant
- Are hard to change later
- Affect multiple modules/teams
- Have significant cost implications
- Have security/compliance implications
- Involve major technology choices

**Examples:**

- ✅ Database selection
- ✅ Authentication mechanism
- ✅ Multi-tenancy approach
- ✅ Monolith vs microservices
- ✅ Cloud provider selection
- ❌ Variable naming convention (too small)
- ❌ Code formatting style (use linter config)
- ❌ Where to put a specific function (code review decision)

### A Concrete Test: One-Way Door vs Two-Way Door

Rather than memorizing a list of examples, it helps to have a test you can apply to a decision you haven't seen before. Amazon popularized a useful framing for this: is the decision a **one-way door** or a **two-way door**?

- A **one-way door** is expensive, slow, or risky to walk back through — once you're on the other side, undoing it means real cost (data migration, a breaking API change, months of rework). These deserve an ADR.
- A **two-way door** is cheap to reverse — you can just walk back through it if it turns out to be wrong. These usually don't need one.

Look back at the examples above through this lens: choosing PostgreSQL over MongoDB is a one-way door (migrating a production database later is a serious project); choosing a variable naming convention is a two-way door (a linter rule change and a find-and-replace, done). The test generalizes to decisions the list doesn't cover — when in doubt, ask which kind of door you're looking at.

## Status Lifecycle

The Status field tracks an ADR through its life:

- **Proposed** — drafted, under discussion, not yet decided
- **Accepted** — the decision is made and in effect
- **Rejected** — the proposal was considered and turned down without ever becoming policy
- **Deprecated** — no longer recommended, but not formally replaced by anything specific
- **Superseded by ADR-XXX** — a newer ADR has explicitly replaced this one

**Rejected is a normal, healthy outcome.** Not every proposal that reaches Review becomes Accepted — and that's not a failure of the process, it's the process working. Recording a Rejected ADR still has value: it tells a future team member "we considered this and decided against it," saving them from re-litigating a settled question without new information.

**Once Accepted, an ADR's content is immutable.** The Context, Decision, and Consequences sections describe what was known and decided _at that time_ — they are not rewritten later to reflect hindsight or new information, even if the decision turns out to have been wrong. If circumstances change, you don't edit the old ADR to say something different; you write a **new** ADR that supersedes it, and the old one's Status line changes to `Superseded by ADR-XXX` while the rest of its text stays exactly as it was originally written. This is what makes a collection of ADRs a trustworthy historical record instead of a wiki page that quietly reflects only the current consensus.

---

## Real ADR Examples for Our SaaS Projects

Let's create actual ADRs for decisions we'll make in our three projects.

### ADR-001: Multi-Tenancy Pattern Selection

````markdown
# ADR-001: Use Shared Database with Shared Schema for Multi-Tenancy

**Date:** 2024-01-15  
**Status:** Accepted  
**Deciders:** Development Team  
**Project:** EnterpriseFlow (Java), CollabSpace (Node.js), ApiCore (Go)

## Context

We need to implement multi-tenancy for our SaaS applications. We're starting with an expected 50-100 customers in the first 6 months, potentially growing to 1,000+ customers within 2 years.

We evaluated three multi-tenancy patterns:

1. Shared database, shared schema (row-level isolation with tenant_id)
2. Shared database, separate schemas per tenant
3. Separate database per tenant

### Factors to Consider:

- Development speed (MVP launch in 3 months)
- Cost optimization (limited seed funding)
- Team size (3 developers)
- Expected customer size (mostly small businesses with similar data volumes)
- Compliance requirements (GDPR, but not healthcare/finance initially)

## Decision

We will implement **Shared Database with Shared Schema** (Pattern 1) using a tenant_id discriminator column.

### Implementation Details:

- Every table will include a `tenant_id` column
- All queries will automatically filter by `tenant_id` using middleware
- Database indexes will include `tenant_id` as the first column
- ORM/query builders will enforce tenant isolation at the application layer

### Code Example:

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT projects_tenant_id_idx PRIMARY KEY (tenant_id, id)
);

CREATE INDEX idx_projects_tenant ON projects(tenant_id);
```
````

```javascript
// Middleware enforces tenant context
app.use((req, res, next) => {
  if (req.user) {
    req.tenant = req.user.tenant;
    db.setTenantContext(req.tenant.id);
  }
  next();
});
```

## Consequences

### Positive:

- **Fast development:** Simplest pattern to implement, single database schema
- **Cost-effective:** One database instance serves all customers (~$50/month vs $50/month per customer)
- **Easy maintenance:** Single schema to migrate, optimize, and backup
- **Simple operations:** One database to monitor and maintain
- **Query optimization:** Can optimize queries across all tenants
- **Reporting:** Easy to run analytics across all tenants

### Negative:

- **Data leakage risk:** If we forget `tenant_id` in a query, data could leak between tenants
- **Performance isolation:** One tenant's large dataset could slow down others ("noisy neighbor" problem)
- **Limited customization:** Difficult to offer schema customization per tenant
- **Scaling complexity:** As we grow, migration to other patterns becomes harder
- **Compliance constraints:** May not meet requirements for highly regulated customers (healthcare, finance)

### Mitigation Strategies:

1. **Prevent data leakage:**
   - Use ORM/query builder that enforces tenant filtering
   - Automated tests that verify tenant isolation
   - Code review checklist includes tenant_id verification
   - Database triggers to prevent cross-tenant queries

2. **Monitor performance per tenant:**
   - Track query performance by tenant_id
   - Set up alerts for tenants exceeding thresholds
   - Implement rate limiting per tenant

3. **Plan for migration:**
   - Structure code in repository pattern (easy to change data access)
   - Document migration path to separate schemas/databases
   - Design aggregates that can be extracted if needed

## Alternatives Considered

### Alternative 1: Shared Database, Separate Schemas

**Pros:**

- Better isolation than shared schema
- Can customize schema per tenant if needed
- Easier to move individual tenants later

**Cons:**

- More complex migrations (must run for each schema)
- PostgreSQL recommends <100 schemas per database
- Connection pooling more complex
- Slower development

**Rejected because:** Adds complexity we don't need yet. When we reach 500+ customers, we can revisit.

### Alternative 2: Separate Databases per Tenant

**Pros:**

- Maximum isolation and security
- Easy to customize per tenant
- Easy to migrate tenants between servers
- Best for compliance

**Cons:**

- Most expensive (database cost per tenant)
- Complex operations (monitor hundreds of databases)
- Difficult to run cross-tenant analytics
- Slow development

**Rejected because:** Too expensive and complex for our stage. May use for enterprise tier in future.

## Review Criteria

This decision should be reviewed when:

- [ ] We reach 500 customers (resource utilization concern)
- [ ] We onboard enterprise customers requiring dedicated infrastructure
- [ ] We enter regulated industries (healthcare, finance)
- [ ] Performance monitoring shows consistent "noisy neighbor" issues
- [ ] We need to offer schema customization

## References

- [AWS SaaS Architecture Fundamentals](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/)
- [Salesforce Multi-Tenant Architecture](https://developer.salesforce.com/page/Multi_Tenant_Architecture)
- Module 1.2: Multi-Tenancy Concepts

````

---

### ADR-002: Monolith vs Microservices

```markdown
# ADR-002: Start with Modular Monolith Architecture

**Date:** 2024-01-16
**Status:** Accepted
**Deciders:** Development Team, CTO
**Projects:** EnterpriseFlow (Java), CollabSpace (Node.js), ApiCore (Go)

## Context

We need to decide on the overall system architecture for our SaaS applications. The choice is between:
1. Modular Monolith (single deployable unit with well-organized modules)
2. Microservices (multiple independent services)

### Current Situation:
- Team: 3 full-stack developers
- Timeline: MVP in 3 months
- Expected users: Start with 1,000 users, grow to 100,000 in Year 1
- Infrastructure budget: $500/month initially
- DevOps expertise: Limited (no dedicated DevOps engineer)

### Business Goals:
- Launch quickly to validate product-market fit
- Iterate rapidly based on customer feedback
- Keep operational costs low
- Scale when needed (not prematurely)

## Decision

We will start with a **Modular Monolith** architecture for all three projects.

### What This Means:

**Single Deployable Unit:**
```

Application (Single Deployment)
├── Auth Module
├── Projects Module
├── Tasks Module
├── Billing Module
├── Notifications Module
└── Analytics Module

```

**Clear Module Boundaries:**
- Each module has its own folder structure
- Modules communicate through well-defined interfaces (services)
- No direct database access across modules
- Each module could theoretically become a microservice later

**Technology Stack:**
- **Java Project:** Spring Boot monolith with Maven modules
- **Node.js Project:** NestJS with module structure
- **Go Project:** Standard Go project layout with packages

### Example Structure (Node.js):

```

src/
├── modules/
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.repository.ts
│   │   ├── auth.module.ts
│   │   └── index.ts              # Public API
│   │
│   ├── projects/
│   │   ├── projects.controller.ts
│   │   ├── projects.service.ts
│   │   ├── projects.repository.ts
│   │   ├── projects.module.ts
│   │   └── index.ts
│   │
│   └── billing/
│       ├── billing.controller.ts
│       ├── billing.service.ts
│       ├── billing.repository.ts
│       ├── billing.module.ts
│       └── index.ts
│
├── shared/
│   ├── database/
│   ├── middleware/
│   └── utils/
│
└── main.ts

```

**Module Communication Rules:**
```typescript
// ❌ BAD: Direct database access across modules
import { db } from '../shared/database';
const user = await db.users.findOne({ id: userId });

// ✅ GOOD: Use the module's public service
import { userService } from '../auth';
const user = await userService.getUser(userId);
```

## Consequences

### Positive:

**Development Speed:**
- Single codebase = easier to navigate and understand
- No network calls between modules = faster development
- Shared code easily accessible
- Simple debugging (everything in one process)

**Operational Simplicity:**
- One deployment pipeline instead of 10+
- One application to monitor
- One log stream to search
- Simple infrastructure (2-3 servers vs 20+)

**Cost Efficiency:**
- Lower infrastructure costs (~$100/month vs $500+/month)
- No need for service mesh, API gateway, etc.
- Fewer servers to manage

**Team Productivity:**
- No coordination needed between service teams (we're small!)
- Easy to refactor across modules
- Shared understanding of entire system

**Transaction Management:**
- Database transactions work naturally
- ACID guarantees for business operations
- No distributed transaction complexity

**Testing:**
- Simple integration tests (no service mocking)
- Easy to test entire user flows
- No contract testing needed

### Negative:

**Scaling Constraints:**
- Must scale entire application (can't scale modules independently)
- Resource-intensive modules affect everything
- May need to over-provision resources

**Deployment Coupling:**
- Changes to one module require full application deployment
- Higher risk deployments (entire app restarts)
- Can't deploy modules independently

**Technology Lock-in:**
- All modules must use same language/framework
- Can't use specialized tools for specific modules
- Stuck with single tech stack

**Team Scaling:**
- As team grows (>10 engineers), coordination becomes harder
- Merge conflicts in single codebase
- Harder to have independent team ownership

### Mitigation Strategies:

**For Scaling:**
- Design for horizontal scaling from day 1
- Use load balancers
- Keep stateless where possible
- Identify bottlenecks early through monitoring

**For Deployment:**
- Use feature flags for gradual rollouts
- Comprehensive automated testing
- Blue-green deployment strategy
- Quick rollback capability

**For Module Isolation:**
- Enforce module boundaries through linting rules
- Code review checklist for inter-module dependencies
- Consider ESLint/TSLint rules to prevent boundary violations

**For Future Migration:**
- Keep modules loosely coupled
- Use dependency injection
- Use events for cross-module communication where appropriate
- Document which modules are candidates for extraction

## Alternatives Considered

### Alternative 1: Microservices from Day 1

**Pros:**
- Independent scaling
- Technology flexibility per service
- Independent deployment
- Clear service boundaries

**Cons:**
- Much slower development
- Complex operations (multiple deployments, monitoring, logs)
- Network latency between services
- Distributed transaction complexity
- Higher infrastructure costs
- Requires experienced DevOps

**Rejected because:**
- Team too small (3 developers)
- No dedicated DevOps engineer
- Premature optimization
- Don't have scaling problems yet (< 1,000 users initially)

### Alternative 2: Serverless Functions (FaaS)

**Pros:**
- No server management
- Pay per use
- Auto-scaling

**Cons:**
- Complex local development
- Vendor lock-in
- Cold start latency
- Difficult debugging
- Complex state management

**Rejected because:**
- Need consistent performance (cold starts problematic)
- Complex business logic better suited to traditional architecture
- Team unfamiliar with serverless patterns

## Migration Path to Microservices

If we need to migrate later (expected at 50K+ users or 10+ engineers):

**Phase 1: Identify Service Candidates** (6-12 months)
- Background jobs (first candidate - independent, async)
- Notifications (clear boundary, high volume)
- Billing (requires high availability, infrequent changes)

**Phase 2: Extract First Service** (12-18 months)
- Start with least coupled module
- Set up service infrastructure (deployment, monitoring)
- Establish inter-service communication (REST or gRPC)
- Keep rest as monolith

**Phase 3: Extract More Services** (18-24 months)
- Extract based on scaling needs
- Extract based on team boundaries
- Don't extract everything (keep core as monolith)

**Decision Triggers for Migration:**
- Team grows beyond 10 engineers
- Different modules have vastly different scaling needs
- Deployment coordination becomes a bottleneck
- Need to use different technology stacks

## Review Criteria

This decision should be reviewed when:
- [ ] Team grows to 10+ engineers
- [ ] Application reaches 50,000 users
- [ ] Deployment frequency drops below weekly
- [ ] Specific modules need independent scaling (e.g., 10x resource difference)
- [ ] Performance bottlenecks can't be solved in monolith
- [ ] Operational costs exceed $2,000/month due to over-provisioning

## References
- [Martin Fowler - MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Sam Newman - Monolith to Microservices](https://samnewman.io/books/monolith-to-microservices/)
- Module 2.1: Monolith vs Microservices
````

---

### ADR-003: Database Selection

````markdown
# ADR-003: Use PostgreSQL as Primary Database

**Date:** 2024-01-17  
**Status:** Accepted  
**Deciders:** Development Team  
**Projects:** EnterpriseFlow (Java), CollabSpace (Node.js), ApiCore (Go)

## Context

We need to select a primary database for our SaaS applications. Key requirements:

- Multi-tenant data isolation
- ACID transactions (critical for billing, user management)
- Complex queries (reports, analytics)
- JSON support (flexible data structures)
- Strong consistency
- Battle-tested reliability
- Good performance up to millions of records

Database options considered:

1. PostgreSQL (relational)
2. MySQL (relational)
3. MongoDB (document NoSQL)
4. DynamoDB (key-value NoSQL)

## Decision

We will use **PostgreSQL** as the primary database for all three projects.

**Version:** PostgreSQL 14+ (latest stable)

**Deployment:**

- Development: Local PostgreSQL via Docker
- Production: Managed PostgreSQL (AWS RDS, Google Cloud SQL, or DigitalOcean Managed Databases)

### Why PostgreSQL:

**1. Relational Model Fits Our Domain:**

- Clear relationships: Projects → Tasks, Tenants → Users
- Need JOINs for complex queries
- Referential integrity (foreign keys)

**2. ACID Transactions:**

```sql
-- Example: Transfer subscription from one plan to another
BEGIN;
  UPDATE subscriptions SET plan_id = $1 WHERE id = $2;
  INSERT INTO billing_events (subscription_id, event_type) VALUES ($2, 'plan_changed');
  UPDATE tenant_stats SET plan_changes = plan_changes + 1 WHERE tenant_id = $3;
COMMIT;
-- All or nothing - critical for billing
```
````

**3. JSON Support:**

```sql
-- Flexible metadata without schema changes
CREATE TABLE projects (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  name VARCHAR(255),
  metadata JSONB  -- Flexible field for custom attributes
);

-- Query JSON fields efficiently
SELECT * FROM projects
WHERE metadata->>'status' = 'active'
  AND (metadata->'tags')::jsonb ? 'urgent';

CREATE INDEX idx_projects_metadata ON projects USING GIN (metadata);
```

**4. Multi-Tenancy Support:**

- Row-level security (RLS) for additional tenant isolation
- Efficient indexes including tenant_id
- Schemas can be used if we migrate to separate-schema pattern

**5. Advanced Features:**

- Full-text search (good for MVP, can add Elasticsearch later)
- CTEs (Common Table Expressions) for complex queries
- Window functions for analytics
- Array types for tags, categories
- Generated columns
- Triggers for audit logs

**6. Tooling and Ecosystem:**

- Excellent ORM support (Hibernate, Sequelize, GORM)
- pgAdmin for database management
- Rich extension ecosystem (PostGIS, pg_trgm, etc.)
- Great monitoring tools

## Consequences

### Positive:

**Reliability:**

- Battle-tested in production for 30+ years
- Strong ACID guarantees
- Well-understood failure modes

**Developer Productivity:**

- SQL is widely known
- Excellent documentation
- Great error messages
- Strong typing helps catch errors

**Performance:**

- Good performance for OLTP workloads (up to millions of records)
- Efficient indexing strategies
- Query planner optimizations
- Connection pooling available (PgBouncer)

**Cost:**

- Managed PostgreSQL: ~$50-100/month for startup
- Scales cost-effectively to 100K users
- No per-operation pricing (unlike DynamoDB)

**Flexibility:**

- Can add NoSQL-like features via JSONB
- Can add full-text search
- Can add time-series data (TimescaleDB extension)

### Negative:

**Scaling Limits:**

- Vertical scaling has limits (single server)
- Horizontal scaling requires sharding (complex)
- Write scaling more difficult than reads
- Large datasets (>1TB) become challenging

**Complexity:**

- Schema migrations can be risky
- Query optimization requires expertise
- Connection pooling needed for high concurrency

**Cloud Native:**

- Not as "cloud native" as DynamoDB
- Requires capacity planning (can't auto-scale infinitely)

### Mitigation Strategies:

**For Scaling:**

- Start with read replicas (easy in PostgreSQL)
- Implement caching layer (Redis) for hot data
- Archive old data to reduce database size
- Plan for sharding if we exceed 1M records per tenant

**For Operations:**

- Use managed PostgreSQL service (automated backups, updates)
- Set up monitoring (pg_stat_statements, connection count, slow queries)
- Automated backup testing
- Connection pooling from day 1 (PgBouncer)

**For Migrations:**

- Use migration tools (Flyway, Liquibase, or ORM migrations)
- Test migrations on production-like data
- Zero-downtime migration strategies
- Rollback plans for every migration

## Alternatives Considered

### Alternative 1: MongoDB

**Pros:**

- Schema flexibility
- Horizontal scaling easier (sharding built-in)
- JSON-native (natural fit for JavaScript)
- Good for rapid prototyping

**Cons:**

- No JOINs (must denormalize or multiple queries)
- No ACID transactions across documents (until recently)
- Eventual consistency complexities
- More complex to model relational data
- Multi-tenant isolation less robust

**Rejected because:**

- Our domain is naturally relational (projects, tasks, users, tenants)
- Need ACID transactions for billing
- Team more familiar with SQL
- Denormalization increases complexity

### Alternative 2: MySQL

**Pros:**

- Similar to PostgreSQL
- Slightly simpler to operate
- Large community
- Good for read-heavy workloads

**Cons:**

- Less feature-rich than PostgreSQL
- Weaker JSON support
- Less sophisticated query optimizer
- Less extensible

**Rejected because:**

- PostgreSQL's advanced features (JSONB, full-text search, CTEs) valuable
- PostgreSQL performance comparable or better for our use case
- Team preference for PostgreSQL

### Alternative 3: DynamoDB (or other NoSQL)

**Pros:**

- Serverless (no capacity planning)
- Infinite scaling
- Pay per request

**Cons:**

- No ad-hoc queries (must plan access patterns upfront)
- No JOINs (complex data modeling)
- More expensive at moderate scale
- Vendor lock-in (AWS specific)
- No local development (different behavior in DynamoDB Local)

**Rejected because:**

- Early-stage SaaS needs flexibility (unknown access patterns)
- Cost-inefficient for predictable workloads
- Relational model better fits our domain

## Schema Design Principles

To maximize PostgreSQL benefits:

**1. Normalize, but Don't Over-Normalize:**

```sql
-- Good: Normalized relationships
CREATE TABLE tenants (
  id UUID PRIMARY KEY,
  name VARCHAR(255),
  plan VARCHAR(50)
);

CREATE TABLE users (
  id UUID PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  email VARCHAR(255),
  name VARCHAR(255)
);

-- Use JSONB for truly flexible data
CREATE TABLE projects (
  id UUID PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  name VARCHAR(255),
  custom_fields JSONB  -- Customer-specific fields
);
```

**2. Index Strategy:**

```sql
-- Always index tenant_id (for multi-tenancy)
CREATE INDEX idx_users_tenant ON users(tenant_id);

-- Composite indexes for common queries
CREATE INDEX idx_projects_tenant_status ON projects(tenant_id, status);

-- Partial indexes for common filters
CREATE INDEX idx_active_projects ON projects(tenant_id)
WHERE status = 'active';

-- GIN indexes for JSONB and full-text search
CREATE INDEX idx_projects_custom_fields ON projects USING GIN (custom_fields);
```

**3. Use Appropriate Types:**

```sql
-- UUID for distributed IDs
id UUID PRIMARY KEY DEFAULT gen_random_uuid()

-- JSONB (not JSON) for queryable documents
metadata JSONB

-- TIMESTAMP WITH TIME ZONE for dates
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()

-- ENUM types for controlled values
CREATE TYPE subscription_status AS ENUM ('active', 'past_due', 'canceled');
```

## Performance Targets

With PostgreSQL, we expect:

- < 50ms for simple queries (SELECT by ID)
- < 200ms for complex queries (JOINs, aggregations)
- 1,000+ queries per second on modest hardware
- Support for 100+ concurrent connections (with pooling)

If we exceed these, consider:

- Query optimization
- Additional indexes
- Caching layer
- Read replicas

## Review Criteria

This decision should be reviewed when:

- [ ] Single database exceeds 500GB
- [ ] Query performance consistently above targets despite optimization
- [ ] Need for real-time analytics (may add ClickHouse)
- [ ] Need for time-series data at scale (may add TimescaleDB)
- [ ] Horizontal scaling becomes mandatory

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Use The Index, Luke!](https://use-the-index-luke.com/)
- [PostgreSQL at Scale (Blog Series)](https://www.cybertec-postgresql.com/en/blog/)

````

---

### ADR-004: Authentication Method

```markdown
# ADR-004: Use JWT for API Authentication

**Date:** 2024-01-20
**Status:** Accepted
**Deciders:** Development Team, Security Consultant
**Projects:** All three projects

## Context

We need to implement authentication for our APIs. Users will access the application from:
- Web browsers
- Mobile apps (future)
- Third-party integrations (API access)

Requirements:
- Stateless authentication (for horizontal scaling)
- Support for multiple client types
- Secure token management
- Ability to revoke access
- Support for refresh tokens

## Decision

We will use **JWT (JSON Web Tokens)** for API authentication with the following approach:

### Implementation:

**Access Token:**
- Short-lived (15 minutes)
- Contains: user_id, tenant_id, roles
- Signed with HS256 (HMAC SHA-256)
- Stored in memory on client

**Refresh Token:**
- Long-lived (7 days)
- Stored in database with revocation capability
- Used to obtain new access tokens
- Stored in httpOnly cookie (web) or secure storage (mobile)

**Token Structure:**
```json
{
  "sub": "user_id",
  "tenant_id": "tenant_uuid",
  "roles": ["user", "admin"],
  "iat": 1642684800,
  "exp": 1642685700
}
````

**Authentication Flow:**

```
1. User logs in with email/password
2. Server validates credentials
3. Server generates:
   - Access token (15 min expiry)
   - Refresh token (7 day expiry, stored in DB)
4. Client stores access token in memory
5. Client stores refresh token in httpOnly cookie
6. Client includes access token in Authorization header
7. When access token expires:
   - Client uses refresh token to get new access token
8. When refresh token expires:
   - User must login again
```

## Consequences

### Positive:

- **Stateless:** No session storage needed, scales horizontally
- **Flexible:** Works for web, mobile, and third-party APIs
- **Secure:** Tokens are signed, tampering detected
- **Performance:** No database lookup on every request
- **Standard:** Industry-standard approach with good library support

### Negative:

- **Token size:** JWTs larger than session IDs (~200 bytes vs 32 bytes)
- **Revocation complexity:** Can't immediately revoke access tokens
- **Secret management:** Must securely manage signing keys
- **Token theft:** If stolen, valid until expiry

### Mitigation:

- Short access token expiry (15 min limits damage from theft)
- Refresh token revocation (stored in DB, can be revoked immediately)
- Secure secret storage (environment variables, secrets management)
- HTTPS only (prevent token theft via network sniffing)
- XSS protection (httpOnly cookies for refresh tokens)

## Alternatives Considered

### Alternative 1: Session-Based Authentication

**Pros:**

- Simple to implement
- Easy to revoke (delete session)
- Smaller identifiers (session ID)

**Cons:**

- Requires session storage (Redis/database)
- Doesn't scale horizontally as easily
- Not suitable for mobile apps or third-party APIs
- Sticky sessions or shared session store needed

**Rejected because:** Need stateless authentication for horizontal scaling and API access.

### Alternative 2: OAuth 2.0 with Third-Party Provider

**Pros:**

- Delegate authentication to experts
- Social login (Google, GitHub)
- No password management

**Cons:**

- Dependency on external service
- Additional complexity
- Users must have accounts with provider
- Privacy concerns for some users

**Decision:** Will add as optional authentication method later (ADR-010), but need native authentication first.

### Alternative 3: API Keys (for API access only)

**Pros:**

- Simple for programmatic access
- Long-lived, easy to manage

**Cons:**

- Not suitable for user authentication
- No expiration (unless implemented)
- Difficult to scope permissions

**Decision:** Will use for third-party API access in addition to JWT (ADR-012).

## Implementation Notes

```javascript
// Token generation (Node.js)
const jwt = require('jsonwebtoken');

function generateAccessToken(user) {
  return jwt.sign(
    {
      sub: user.id,
      tenant_id: user.tenantId,
      roles: user.roles
    },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );
}

function generateRefreshToken(user) {
  const token = jwt.sign(
    { sub: user.id },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  );

  // Store in database for revocation capability
  await db.refreshTokens.create({
    userId: user.id,
    token: token,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  return token;
}

// Middleware to verify token
function verifyToken(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

## Review Criteria

Review this decision when:

- [ ] Token theft becomes a significant security issue
- [ ] Access token size causes performance problems
- [ ] Need for more fine-grained permission control
- [ ] Regulatory requirements change

## References

- [JWT.io](https://jwt.io)
- [OWASP JWT Cheat Sheet](https://owasp.org/www-project-cheat-sheets/)
- [RFC 7519 - JSON Web Token](https://tools.ietf.org/html/rfc7519)

````

---

### ADR-005: Caching Strategy

```markdown
# ADR-005: Use Redis for Application-Level Caching

**Date:** 2024-01-22
**Status:** Accepted
**Deciders:** Development Team
**Projects:** All three projects

## Context

As our application grows, we need caching to:
- Reduce database load
- Improve response times
- Handle traffic spikes
- Reduce API costs (external services)

Expected caching needs:
- User sessions (JWT validation data)
- Frequently accessed data (tenant info, user profiles)
- Rate limiting counters
- Temporary data (password reset tokens)
- API response caching
- Computed results (analytics, reports)

Evaluation criteria:
- Performance (low latency)
- Reliability
- Ease of use
- Cost
- Scalability
- TTL (time-to-live) support
- Data structure support

## Decision

We will use **Redis** as our primary caching solution.

### Why Redis:

**1. Performance:**
- In-memory storage (sub-millisecond latency)
- Single-threaded (predictable performance)
- Optimized for read-heavy workloads

**2. Rich Data Structures:**
```javascript
// Strings (simple key-value)
await redis.set('user:123:profile', JSON.stringify(userProfile), 'EX', 3600);

// Hashes (structured data)
await redis.hset('user:123', {
  name: 'John Doe',
  email: 'john@example.com',
  lastLogin: Date.now()
});

// Lists (ordered data)
await redis.lpush('notifications:user:123', notification);

// Sets (unique values)
await redis.sadd('tenant:123:active_users', userId);

// Sorted Sets (leaderboards, time-series)
await redis.zadd('rate_limit:api_key:abc', Date.now(), requestId);

// Bitmaps (analytics, presence)
await redis.setbit('daily_active_users:2024-01-22', userId, 1);
````

**3. Built-in Features:**

- TTL (automatic expiration)
- Pub/Sub (real-time messaging)
- Lua scripting (atomic operations)
- Transactions
- Persistence options (optional)

**4. Ecosystem:**

- Excellent library support (all languages)
- Redis Cluster for scaling
- Redis Sentinel for high availability
- Compatible with managed services (AWS ElastiCache, Azure Cache)

### Caching Strategy:

**Cache-Aside Pattern (Lazy Loading):**

```javascript
async function getUser(userId) {
  // 1. Try cache first
  const cached = await redis.get(`user:${userId}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Cache miss - fetch from database
  const user = await db.users.findById(userId);

  // 3. Store in cache for next time
  await redis.setex(
    `user:${userId}`,
    3600, // 1 hour TTL
    JSON.stringify(user),
  );

  return user;
}
```

**Write-Through Pattern (for critical data):**

```javascript
async function updateUser(userId, updates) {
  // 1. Update database
  const user = await db.users.update(userId, updates);

  // 2. Update cache immediately
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}
```

**Cache Invalidation:**

```javascript
async function deleteUser(userId) {
  // 1. Delete from database
  await db.users.delete(userId);

  // 2. Invalidate cache
  await redis.del(`user:${userId}`);
}
```

### TTL Strategy:

| Data Type           | TTL        | Reason                               |
| ------------------- | ---------- | ------------------------------------ |
| User profiles       | 1 hour     | Changes infrequently, reduce DB load |
| Tenant settings     | 5 minutes  | May change during configuration      |
| Rate limit counters | 1 minute   | Short window for rate limiting       |
| Session data        | 15 minutes | Matches JWT expiry                   |
| API responses       | 5 minutes  | Balance freshness and performance    |
| Computed analytics  | 1 hour     | Expensive to compute                 |

## Consequences

### Positive:

**Performance:**

- 10-100x faster than database queries
- Sub-millisecond latency
- Reduces database load significantly

**Scalability:**

- Can handle millions of requests per second
- Easy to add more cache capacity
- Reduces database as bottleneck

**Flexibility:**

- Multiple data structures for different use cases
- TTL handles cleanup automatically
- Pub/Sub for real-time features

**Cost:**

- Managed Redis: ~$15-50/month for small instance
- Saves database costs by reducing load
- Reduces need for larger database instances

**Developer Experience:**

- Simple API
- Well-documented
- Great tooling (Redis CLI, RedisInsight)

### Negative:

**Complexity:**

- Cache invalidation is hard
- Must handle cache misses gracefully
- Potential for stale data
- Memory limitations (must fit in RAM)

**Reliability:**

- Single point of failure (without replication)
- Data loss on restart (unless persistence enabled)
- Cache stampede risk (many requests for expired key)

**Cost:**

- Additional infrastructure component
- Memory can be expensive at scale

**Debugging:**

- Harder to debug cached data issues
- Must monitor cache hit rates

### Mitigation:

**For Cache Invalidation:**

```javascript
// Use consistent key patterns
const keys = {
  user: (userId) => `user:${userId}`,
  userProjects: (userId) => `user:${userId}:projects`,
  project: (projectId) => `project:${projectId}`,
  tenantUsers: (tenantId) => `tenant:${tenantId}:users`,
};

// Invalidate related caches together
async function updateProject(projectId, updates) {
  await db.projects.update(projectId, updates);

  // Invalidate all related caches
  await redis.del(keys.project(projectId), keys.userProjects(updates.userId));
}
```

**For Cache Stampede:**

```javascript
// Lock to prevent multiple concurrent cache fills
async function getExpensiveData(key) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  // Try to acquire lock
  const lockKey = `lock:${key}`;
  const locked = await redis.set(lockKey, "1", "EX", 10, "NX");

  if (locked) {
    try {
      // Only this request computes the data
      const data = await computeExpensiveData();
      await redis.setex(key, 3600, JSON.stringify(data));
      return data;
    } finally {
      await redis.del(lockKey);
    }
  } else {
    // Wait for other request to populate cache
    await sleep(100);
    return getExpensiveData(key); // Retry
  }
}
```

**For Reliability:**

- Use Redis Sentinel for automatic failover
- Enable AOF persistence for durability
- Monitor cache hit rates (target >80%)
- Have graceful fallback to database

**For Memory Management:**

```javascript
// Set maxmemory policy
// redis.conf: maxmemory-policy allkeys-lru

// Monitor memory usage
const info = await redis.info("memory");
console.log("Memory used:", info.used_memory_human);

// Use appropriate TTLs to prevent memory bloat
```

## Alternatives Considered

### Alternative 1: Memcached

**Pros:**

- Simple, focused on caching
- Multi-threaded (better CPU utilization)
- Slightly faster for simple key-value

**Cons:**

- Only key-value storage (no data structures)
- No persistence
- No Pub/Sub
- Less feature-rich

**Rejected because:** Need data structures (sets, sorted sets) for rate limiting, leaderboards, etc.

### Alternative 2: Application Memory (In-Process Cache)

**Pros:**

- Fastest possible (no network latency)
- Free (no additional infrastructure)
- Simple to implement

**Cons:**

- Not shared across application instances
- Memory constrained by application
- Lost on application restart
- Cache invalidation harder in clustered environment

**Rejected because:** Need shared cache across multiple application instances for horizontal scaling.

### Alternative 3: Database Query Cache

**Pros:**

- No additional infrastructure
- Automatic invalidation on data change
- Built into database

**Cons:**

- Slower than dedicated cache
- Limited control over caching strategy
- Doesn't reduce database connection usage

**Rejected because:** Not sufficient for our performance needs. Will use as supplementary optimization.

## Implementation Guidelines

**1. Never Cache Sensitive Data:**

```javascript
// ❌ DON'T cache passwords, tokens, credit cards
await redis.set("user:123:password", hashedPassword); // NO!

// ✅ DO cache non-sensitive, frequently accessed data
await redis.set("user:123:profile", JSON.stringify({ name, email }));
```

**2. Always Set TTL:**

```javascript
// ❌ DON'T set data without expiration
await redis.set("key", "value"); // Will stay forever!

// ✅ DO set appropriate TTL
await redis.setex("key", 3600, "value"); // Expires in 1 hour
```

**3. Handle Cache Misses Gracefully:**

```javascript
// ❌ DON'T assume cache always has data
const data = JSON.parse(await redis.get('key')); // Could be null!

// ✅ DO fallback to database
const cached = await redis.get('key');
const data = cached ? JSON.parse(cached) : await db.query(...);
```

**4. Use Namespaced Keys:**

```javascript
// ❌ DON'T use simple keys (collision risk)
await redis.set("123", data);

// ✅ DO namespace keys
await redis.set("user:123", data);
await redis.set("project:123", data);
```

## Monitoring

Key metrics to track:

- Cache hit rate (target: >80%)
- Cache miss rate
- Memory usage (stay below 80% of max)
- Eviction rate (should be low)
- Response time (P50, P95, P99)
- Connection count

```javascript
// Log cache metrics
async function getCacheMetrics() {
  const info = await redis.info();

  return {
    hitRate: info.keyspace_hits / (info.keyspace_hits + info.keyspace_misses),
    memoryUsed: info.used_memory,
    memoryMax: info.maxmemory,
    connectedClients: info.connected_clients,
    evictedKeys: info.evicted_keys,
  };
}
```

## Review Criteria

Review this decision when:

- [ ] Cache hit rate consistently below 70%
- [ ] Memory costs exceed $200/month
- [ ] Need for geographical distribution (multi-region)
- [ ] Need for different caching strategies per module

## References

- [Redis Documentation](https://redis.io/documentation)
- [Caching Strategies](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/Strategies.html)
- [Redis Best Practices](https://redis.io/topics/best-practices)

````

---

## Superseding an ADR in Practice

The `Superseded by ADR-XXX` status is easy to state and easy to get wrong in practice, so it's worth walking through a concrete case. Say that, a year after ADR-001, the team has grown past 500 tenants and is hitting the "noisy neighbor" problem the Review Criteria flagged. The team decides to migrate large tenants to separate schemas.

**A new ADR is written, not a rewrite of ADR-001:**

```markdown
# ADR-014: Migrate Large Tenants to Separate Schemas

**Date:** 2025-03-10
**Status:** Accepted
**Supersedes:** ADR-001
**Deciders:** Development Team, Infrastructure Lead
**Projects:** EnterpriseFlow, CollabSpace, ApiCore

## Context

ADR-001 established shared-schema multi-tenancy, with an explicit review
trigger at 500 customers. We are now at 620 customers, and monitoring has
shown three tenants whose query load consistently degrades performance
for others sharing the same tables ("noisy neighbor," anticipated in
ADR-001's risk section).

## Decision

Large tenants (top 5% by data volume/query load) will be migrated to
their own PostgreSQL schema within the same database instance...

## Consequences
...
````

**And ADR-001 itself is not touched, except for its Status line:**

```markdown
# ADR-001: Use Shared Database with Shared Schema for Multi-Tenancy

**Date:** 2024-01-15
**Status:** Superseded by ADR-014
**Deciders:** Development Team
**Project:** EnterpriseFlow (Java), CollabSpace (Node.js), ApiCore (Go)

## Context

[... unchanged from the original 2024-01-15 text ...]
```

Notice what didn't change: the Context, Decision, and Consequences of ADR-001 still read exactly as they did on 2024-01-15 — including the parts that, with hindsight, undersold the noisy-neighbor risk. That's intentional. The record shows what the team actually knew and decided at the time; ADR-014 is where the update to that thinking lives.

## A Weak ADR, for Contrast

Every example so far has been a good one. Here's a short, deliberately weak ADR to make the gaps easier to spot in your own drafts:

```markdown
# ADR-009: Use MongoDB

**Status:** Accepted

## Context

We needed a database for the new notifications service.

## Decision

We're using MongoDB.

## Consequences

It should work well and is easy to scale.
```

What's missing, compared to the real examples above:

- **No specific context.** "We needed a database" doesn't say what the notifications service actually requires (query patterns, consistency needs, expected volume) — there's nothing here a future reader could use to judge whether the decision still makes sense.
- **No alternatives considered.** There's no sign any other option was evaluated, which makes it impossible to tell whether this was a reasoned choice or just "the first thing that came to mind."
- **Consequences with no substance.** "It should work well" isn't a consequence, positive or negative — compare this to ADR-003's detailed, honest list of scaling limits and mitigation strategies for PostgreSQL.

None of this requires the heavy format to fix — even the light format, filled in honestly, would have caught all three gaps.

---

## Creating Your Own ADRs

### ADR Template (Heavy Format)

Use this template for architectural decisions significant enough to warrant it (see "ADR Format" above for the lighter alternative, which is the better default for most decisions):

```markdown
# ADR-[NUMBER]: [Short Title]

**Date:** YYYY-MM-DD  
**Status:** [Proposed | Accepted | Rejected | Deprecated | Superseded by ADR-XXX]  
**Deciders:** [List of people involved]  
**Projects:** [Which projects this affects]

## Context

What is the issue we're trying to solve?

### Background

- Current situation
- Constraints
- Requirements
- Business goals

### Factors to Consider

- List key factors influencing the decision

## Decision

What did we decide to do?

### Implementation Details

- Specific choices
- Technical approach
- Code examples (if relevant)

## Consequences

### Positive

- Benefits of this decision
- What problems does it solve?

### Negative

- Downsides and tradeoffs
- What problems might it create?

### Mitigation Strategies

- How we'll address the negative consequences

## Alternatives Considered

### Alternative 1: [Name]

## **Pros:**

## **Cons:**

## **Rejected because:**

### Alternative 2: [Name]

[Same format]

## Review Criteria

This decision should be reviewed when:

- [ ] Condition 1
- [ ] Condition 2

## References

- Links to relevant documentation
- Related ADRs
- External resources
```

---

## ADR Workflow

### 1. When to Create an ADR

Create an ADR for decisions that:

- Are architecturally significant
- Are hard to change later
- Affect multiple modules/teams
- Have significant cost implications
- Have security/compliance implications
- Involve major technology choices

**Examples:**

- ✅ Database selection
- ✅ Authentication mechanism
- ✅ Multi-tenancy approach
- ✅ Monolith vs microservices
- ✅ Cloud provider selection
- ❌ Variable naming convention (too small)
- ❌ Code formatting style (use linter config)
- ❌ Where to put a specific function (code review decision)

### 2. ADR Creation Process

**Step 1: Draft (Status: Proposed)**

- Developer identifies need for architectural decision
- Creates ADR in draft format
- Researches alternatives
- Documents context and options

**Step 2: Review**

- Share with team for feedback
- Update ADR based on discussions
- May require multiple iterations
- Some proposals end here: if the team decides against the proposal, the ADR moves to **Rejected** rather than Accepted — that's a valid, useful outcome of the process, not a dead end to hide

**Step 3: Decision (Status: Accepted)**

- Team/tech lead makes final decision
- ADR updated to "Accepted"
- Implementation begins

**Step 4: Implementation**

- Follow the decision
- Update ADR if implementation reveals new information

**Step 5: Review (Periodic)**

- Revisit based on review criteria
- May lead to new ADR superseding old one

### 3. ADR File Organization

```
docs/
└── architecture/
    └── decisions/
        ├── README.md                    # Index of all ADRs
        ├── 0001-multi-tenancy.md
        ├── 0002-monolith-architecture.md
        ├── 0003-database-selection.md
        ├── 0004-authentication-method.md
        ├── 0005-caching-strategy.md
        └── template.md                  # Template for new ADRs
```

**README.md Index:**

```markdown
# Architecture Decision Records

## Active ADRs

| Number | Title                 | Status   | Date       |
| ------ | --------------------- | -------- | ---------- |
| 001    | Multi-Tenancy Pattern | Accepted | 2024-01-15 |
| 002    | Monolith Architecture | Accepted | 2024-01-16 |
| 003    | Database Selection    | Accepted | 2024-01-17 |
| 004    | Authentication Method | Accepted | 2024-01-20 |
| 005    | Caching Strategy      | Proposed | 2024-01-22 |

## Superseded ADRs

| Number | Title | Superseded By | Date |
| ------ | ----- | ------------- | ---- |
| -      | -     | -             | -    |

## How to Create an ADR

1. Copy `template.md`
2. Name it `XXXX-short-title.md` (increment number)
3. Fill in all sections
4. Submit for review
5. Update index when accepted
```

---

## Tools for Managing ADRs

### CLI Tool: adr-tools

```bash
# Install
npm install -g adr-log

# Initialize
adr init docs/architecture/decisions

# Create new ADR
adr new "Use PostgreSQL as primary database"

# List all ADRs
adr list

# Generate documentation
adr generate toc
```

### Version Control

ADRs should be:

- ✅ Committed to git
- ✅ Reviewed via pull requests
- ✅ Linked to relevant issues/tickets
- ✅ Updated when superseded

```bash
# Example workflow
git checkout -b adr/004-authentication-method
# Edit ADR file
git add docs/architecture/decisions/0004-authentication-method.md
git commit -m "ADR-004: JWT-based authentication"
git push origin adr/004-authentication-method
# Create pull request for team review
```

---

## Team Adoption

Knowing the format isn't the hard part — getting a team to actually keep writing ADRs is. Two objections come up almost every time:

- **"This takes too long."** True if every ADR is written in the heavy format. Start with the light format for everything, and only reach for the heavy one when the one-way-door test clearly says the decision deserves it.
- **"Nobody reads them later anyway."** Often true if ADRs live in a folder nobody visits. Two things help: link the relevant ADR directly from the code or config it justifies (a comment pointing at `docs/architecture/decisions/0003-database-selection.md` gets read at exactly the moment someone is confused), and keep the README index current so a newcomer can skim titles in under a minute.

Practical ways to build the habit rather than mandate it from day one:

- Start with a single volunteer or a single team, not an org-wide policy — let a handful of real, useful ADRs make the case before asking everyone to adopt the practice.
- Add "Does this PR make a one-way-door decision? If so, link an ADR." to the pull request template for changes that plausibly qualify — this puts the prompt exactly where the decision is being made, instead of relying on people to remember a process on their own.
- Keep the first several ADRs deliberately light. Proving the habit is cheap comes before asking anyone to invest in the heavy format.

---

## Key Takeaways

### Why ADRs Matter

1. **Preserves Context:** Future developers understand WHY decisions were made
2. **Prevents Rehashing:** Don't revisit settled decisions without new information
3. **Onboarding Tool:** New team members can read ADRs to understand architecture
4. **Accountability:** Clear ownership of decisions
5. **Learning Tool:** Team learns from past decisions

### ADR Best Practices

**DO:**

- ✅ Write ADRs for significant architectural decisions
- ✅ Keep them concise but complete
- ✅ Include alternatives considered
- ✅ Update status when superseded
- ✅ Review periodically
- ✅ Use consistent format
- ✅ Store in version control
- ✅ Link to relevant code/documentation

**DON'T:**

- ❌ Write ADRs for trivial decisions
- ❌ Make decisions without documenting
- ❌ Let ADRs become outdated
- ❌ Skip the "alternatives considered" section
- ❌ Write novels (keep it focused)
- ❌ Use ADRs for project management (use tickets)

### Common Mistakes

**Mistake 1: Not Writing ADRs**

- Problem: Future team doesn't know why decisions were made
- Solution: Make ADRs part of your workflow

**Mistake 2: Too Much Detail**

- Problem: ADRs become unreadable
- Solution: Focus on decision and rationale, link to detailed docs

**Mistake 3: No Alternatives**

- Problem: Looks like no evaluation happened
- Solution: Always document at least 2 alternatives

**Mistake 4: Status Never Updated**

- Problem: Outdated ADRs confuse people
- Solution: Review ADRs regularly, update status

---

## Exercises

### Exercise 1: Create Your First ADR

**Task:** Write an ADR for one of these decisions:

**Option A:** Choosing a logging framework

- Winston vs Bunyan vs Pino (Node.js)
- Logback vs Log4j2 (Java)
- Zap vs Logrus (Go)

**Option B:** Choosing a testing framework

- Jest vs Mocha (Node.js)
- JUnit vs TestNG (Java)
- testify vs Ginkgo (Go)

**Option C:** Choosing a form validation library

- Joi vs Yup vs Zod (Node.js)
- Hibernate Validator vs Apache Commons (Java)
- go-playground/validator vs ozzo-validation (Go)

**Deliverable:** Complete ADR document following the template

---

### Exercise 2: Review and Critique

**Task:** Review the ADR-003 (Database Selection) and answer:

1. What is the most important factor in choosing PostgreSQL?
2. What is the biggest risk of this decision?
3. When should this decision be reviewed?
4. What alternative would you have chosen and why?

**Deliverable:** 1-page written analysis

---

### Exercise 3: ADR for Your Project

**Task:** Think about a recent technical decision in a project you've worked on. Write an ADR for it.

**Requirements:**

- Use the template provided
- Include at least 2 alternatives
- Document both positive and negative consequences
- Include review criteria

**Deliverable:** Complete ADR document

---

### Exercise 4: ADR Index

**Task:** Create an ADR index (README.md) for a hypothetical project with these decisions:

1. Multi-tenancy pattern
2. Authentication method
3. Database selection
4. Caching strategy
5. Background job processing
6. File storage solution
7. Email service provider
8. Monitoring/logging solution

**Deliverable:** README.md with organized index and status of each ADR

---

## Resources

### Tools

**ADR Tools:**

- [adr-tools](https://github.com/npryce/adr-tools) - Command-line tool for managing ADRs
- [adr-log](https://github.com/adr/adr-log) - Generate architectural decision log
- [Markdown ADR](https://adr.github.io/madr/) - Markdown Architectural Decision Records

**Templates:**

- [Michael Nygard's Template](https://github.com/joelparkerhenderson/architecture-decision-record/tree/main/templates/decision-record-template-by-michael-nygard)
- [MADR Template](https://adr.github.io/madr/)
- [Y-Statements](https://medium.com/olzzio/y-statements-10eb07b5a177)

### Reading

**Essential:**

1. **"Documenting Architecture Decisions" by Michael Nygard**
   - https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
   - The original blog post that started ADRs

2. **"Architecture Decision Records" by Joel Parker Henderson**
   - https://github.com/joelparkerhenderson/architecture-decision-record
   - Comprehensive guide with examples

3. **"ADR GitHub Organization"**
   - https://adr.github.io/
   - Collection of ADR resources and tools

**Books:**

1. **"Software Architecture in Practice" by Bass, Clements, Kazman**
   - Chapter on documenting architecture decisions

2. **"Design It!" by Michael Keeling**
   - Practical guide to software architecture, includes ADR usage

### Examples from Real Projects

**Open Source ADR Examples:**

1. **Spotify's Backstage**
   - https://github.com/backstage/backstage/tree/master/docs/architecture-decisions

2. **GOV.UK**
   - https://github.com/alphagov/govuk-aws/tree/master/doc/architecture/decisions

3. **Markdown Architectural Decision Records (MADR)**
   - https://github.com/adr/madr/tree/master/docs/decisions

---

## Summary

**Architecture Decision Records are essential for:**

- 📝 Documenting important architectural choices
- 🤔 Capturing the "why" behind decisions
- 📚 Creating institutional knowledge
- 🔄 Enabling informed re-evaluation
- 👥 Onboarding new team members

**Key Format Elements:**

- Status (Proposed/Accepted/Rejected/Deprecated/Superseded)
- Context (the problem)
- Decision (what we chose)
- Consequences (positive and negative)
- Alternatives (what we didn't choose and why)

**Remember:**

- ADRs are historical records, not living documents — once Accepted, their content doesn't change; supersede instead of rewriting
- Focus on significant decisions (not every choice) — the one-way-door test helps decide
- Be honest about tradeoffs (no perfect solutions)
- Keep them concise but complete — light format by default, heavy format when the stakes justify it
- Store in version control with code
